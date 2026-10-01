---
date: '2026-09-25T12:00:00-04:00'
draft: false
title: "Building a node on libbitcoinkernel: a tour of kernel-node's structure"
---

kernel-node is an experimental full node built on libbitcoinkernel, which packages Bitcoin Core's consensus and storage code as a library with a C API. It validates blocks but doesn't serve them to other nodes. It syncs from several peers at once, and most of its design is about the download path.

Two things in kernel-node's block-processing thread show the central constraint of programming against libbitcoinkernel. First, when the node submits a block, it ignores the return value:

```rust
let _ = chainman.process_block(&block);   // src/bin/node.rs, block processing thread
```

Second, the node only submits a block whose parent is already on the active chain (with one exception, covered later), and it checks that by asking the kernel rather than tracking a tip of its own:

```rust
pub fn is_on_active_chain(&self, block_hash: &bitcoin::BlockHash) -> bool {   // src/peer.rs
    let hash = bitcoinkernel::BlockHash::from(block_hash.to_byte_array());
    match self.chainman.get_block_tree_entry(&hash) {
        Some(entry) => self.chainman.active_chain().contains(&entry),
        None => false,
    }
}
```

Both come from one fact about the kernel: submitting a block and connecting it to the chain are separate steps, and `process_block`'s return value only covers the first. I'll walk through the rest of kernel-node's structure to show how that split shapes the way the node syncs and buffers blocks.

## The kernel/node split

Each job belongs to either the kernel or the node, never both. The kernel, libbitcoinkernel, handles anything that involves the consensus rules (the rules every Bitcoin node must agree on to accept a block) or the chain data kept on disk:

- Signature and script verification, and the checks on weight, amounts, and limits.
- The UTXO set (`chainstate/`, a LevelDB).
- Block and undo storage (`blocks/blk*.dat`, `blocks/rev*.dat`) and the block index.
- The block tree and the most-work decision, including reorgs.

The node handles everything else: peer discovery, the wire protocol, its own threads, and the wallet. Its job is to find blocks, decide which one the kernel can use next, and hand it over. The node uses `bitcoin` crate types and the kernel uses its own, so a small adapter converts between them.

You build the kernel from two objects. A `Context` holds the configuration and the callbacks the kernel uses to report events. A `ChainstateManager` owns the databases, and most operations go through it.

```rust
let context = create_context(network.chain_type(), fatal.clone(), Arc::clone(&wallet), scan_tx);

let chainman = ChainstateManagerBuilder::new(&context, &data_dir, &blocks_dir)
    .unwrap()
    .worker_threads(((available_parallelism().unwrap().get() / 2) + 1).try_into().unwrap())
    .build()
    .unwrap();

if let Err(err) = node_state.chainman.import_blocks() { /* log and exit */ }
```

Passing `data_dir` and `blocks_dir` hands the kernel ownership of the disk: it opens the block index, the chainstate, and the block files, and starts its script verification threads (`worker_threads` sets the count to one more than half the machine's logical cores). An existing Bitcoin Core datadir works as is. `import_blocks()` activates the best chain, and only replays `blk*.dat` on a reindex.

## Results come through callbacks

The callbacks registered on the `Context`, not `process_block`, tell you whether a block connected. `create_context` registers ten of them, and these two show the pattern:

```rust
.with_block_connected_validation(move |block, _entry| {
    if wallet.lock().unwrap().keys.is_none() { return; }
    let block_hash = block.hash();
    if scan_tx.send(ScanEvent::Connected { block, block_hash }).is_err() {
        fatal_connected.trigger(Category::NODE, "Scan channel closed unexpectedly ...");
    }
})
.with_block_checked_validation(move |block, state| {
    match state.mode() {
        ValidationMode::Valid => { /* log */ }
        _ => error!(target: Category::KERNEL, "Received an invalid block!"),
    }
})
```

The rest of the node is built around how these handlers behave.

They run synchronously on the block-processing thread, inside the `process_block` call, so anything slow in a handler holds up that call and every block after it. That's why `block_connected` does the minimum: it checks whether the wallet has keys, wraps the block in a `ScanEvent`, pushes it down the channel, and leaves the work to the scan thread.

The kernel passes the `BlockTreeEntry` as a borrow that ends when the call returns, so the borrow checker won't let a handler store it in a struct or send it to another thread. A handler has to either use the entry right away, as `block_disconnected` does when it reads `entry.height()`, or forward the block hash and let a later thread look the entry up again. `block_connected` does the second: it sends `block_hash`, and the scan thread calls `get_block_tree_entry`.

`block_checked` reports whether a block is valid, and `block_connected` reports that it joined the active chain. A block stored on a side branch triggers neither. And one `process_block` call can trigger both several times, as blocks the kernel stored earlier connect on top of the one just submitted.

## Stored versus connected

`process_block` tells you whether it stored the block on disk, found a duplicate, or rejected it. None of those tells the node whether the active chain advanced or whether this block is now valid and connected. The kernel can store a block (it knows the header, it has the body) and still leave it unconnected because its parent hasn't connected yet. `process_block` can even return `NewBlock` for a block that fails `ConnectBlock` in the same call, since `Rejected` only covers failures before the block is stored. So the node ignores the return value and gets the answers it needs elsewhere:

| The node wants to know | `process_block` return | Where the answer comes from |
|---|---|---|
| Is the block on disk? | yes (`NewBlock` / `Duplicate`) | the return value |
| Did the active chain advance? | no | `block_connected` callback |
| Is this block valid? | no (may be deferred) | `block_checked` callback |
| Has this block's parent connected? | no | `is_on_active_chain` → `active_chain().contains` |

kernel-node syncs headers first. The kernel doesn't require that, but the download queue walks block tree entries, and a hash has no entry until its header has been processed. So when an `Inv` announces blocks, the node starts by asking for their headers:

```rust
NetworkMessage::Inv(inventory) => {
    // ...
    if !block_hashes.is_empty() {
        // The queue walks block tree entries, which need these headers first.
        let locators = build_block_locators(node_state.chainman.best_entry().unwrap());
        (PeerStateMachine::AwaitingHeaders, vec![create_getheaders_message(locators)])
    } // ...
}
```

Only once those headers exist does it fill the download queue. `populate_download_queue` starts at `chainman.best_entry()` and follows `prev()` back to the active chain, which gives it the fork point and every hash in between.

Blocks have to connect in order: `ConnectBlock` validates block N against the UTXO set as block N-1 left it, and writes undo data as it goes so a reorg can roll the changes back. A block whose parent isn't connected has no state to validate against, and outside the fork fallback below, the node avoids putting the kernel in that position.

## Parallel downloading in batches

The peer manager starts eight peer threads (`DEFAULT_MAX_PEERS = 8`, or one with `--connect`), and each holds a single connection at a time. They share one download plan behind a mutex, so a slow peer doesn't hold up the others:

```rust
pub struct DownloadState {   // src/peer.rs
    queue: VecDeque<BlockHash>,
    in_flight: HashSet<BlockHash>,
    buffer: HashMap<BlockHash /* prev */, bitcoinkernel::Block>,
}
```

A peer reserves work with `pop_batch`, which moves up to `DOWNLOAD_BATCH_SIZE` (16) hashes from the queue into `in_flight`. A hash that's already in flight fails `HashSet::insert` and gets skipped, so two peers never ask for the same block. When a block arrives, only the peer whose `release` call removes its hash from `in_flight` buffers it:

```rust
if node_state.download.lock().unwrap().release(&block_hash) {
    node_state.buffer_block(prev_blockhash, block.convert());
}
```

When a peer disconnects, its thread calls `release_in_flight`, which pushes everything the peer still owes back onto the front of the queue for another peer to pick up. With eight peers and a batch of sixteen, `pop_batch` keeps at most 128 requests outstanding, and a disconnect costs the sync one batch and nothing else.

Blocks arrive in whatever order the peers send them, so the node files each buffered block under its parent's hash and takes them out in the order they can connect. `take_connectable` looks for a buffered block whose parent is on the active chain, and falls back to a second check when it finds none:

```rust
fn take_connectable(&mut self, is_connected: impl Fn(&BlockHash) -> bool) -> Option<bitcoinkernel::Block> {
    if let Some(prev) = self.buffer.keys().copied().find(&is_connected) {
        return self.buffer.remove(&prev);
    }
    if !self.queue.is_empty() || !self.in_flight.is_empty() {
        return None;
    }
    let prev = self.buffer_head()?;
    self.buffer.remove(&prev)
}
```

The first check hands `process_block` one block at a time, and only a block whose parent the kernel has already connected. The second handles a fork below the current tip. The node buffers the fork's blocks, but their parents are on a branch the kernel already has rather than on the active chain, so the first check never fires. Once the queue and the in-flight set are both empty, meaning no more blocks are coming, the node hands over the oldest buffered block (`buffer_head` finds the entry that isn't the child of another buffered block) and lets the kernel decide, by accumulated work, whether that branch becomes the active chain. Without the second check those blocks would sit in the buffer forever.

The buffer between the peer threads and the block-processing thread has no size limit. Peers go on reserving batches whether or not validation keeps up, so the buffer grows by however much downloads outpace validation. A limit would put every peer back to waiting on the slowest one, because once the buffer filled, it couldn't drain until that peer's block arrived. Bitcoin Core instead limits downloads to a 1024-block window and disconnects the peer that's stalling it. kernel-node's backstop is a 20-minute `STALE_BLOCK_DURATION`: if no block reaches the kernel in that time, the block-processing thread disconnects every peer and the manager picks new ones.

## Reading the chain

Reading the tip is cheap. Bitcoin Core keeps the block index and the active chain in RAM, so the IPC handler gets the tip with one lookup and a couple of field reads. It opens no files, the cost doesn't grow with chain length, and it's safe to do while validation runs on another thread:

```rust
let tip = self.chainman.active_chain().tip();
r.set_height(tip.height() as u32);
r.set_hash(BlockHash::from_byte_array(tip.block_hash().to_bytes()).to_string());
```

The second read is expensive. For each transaction, the silent-payments wallet needs the outputs it spent (its prevouts). Those leave the UTXO set once the block connects, so the wallet gets them from the undo data the kernel wrote to `rev*.dat`. `read_spent_outputs` reads that file to find the block's record and decode the coin each input spent. Doing that inside `block_connected` would put a disk seek per block on the block-processing thread, which is why the scan thread does it instead:

```rust
let entry = chainman_for_scan.get_block_tree_entry(&block_hash) /* or fatal */;
let spent_outputs = chainman_for_scan.read_spent_outputs(&entry) /* or fatal */;
let count = wallet.lock().unwrap().scan_block(block, spent_outputs, block_height);
```

For each input, `scan_block_inner` then collects the three fields BIP-352 needs to recover the sender's public key: the input's `scriptSig`, its witness, and the script of the output it spent.

```rust
for (kernel_tx, tx_spent) in kernel_block.transactions().skip(1).zip(spent_outputs.iter()) {
    // ...
    let coin = tx_spent.coin(idx).expect("input/spent-output count mismatch");
    let witness: Vec<Vec<u8>> = input.witness_stack().items().collect();
    let script_sig = input.script_sig().unwrap();
    // coin.output().script_pubkey() is the prevout script the scan requires
}
```

The `scriptSig` and witness are in the block, and the prevout script comes from the kernel's undo data, which lets the wallet rebuild input keys without a UTXO set of its own. Where the key lives depends on the input type: the `scriptSig` for P2PKH, the witness for P2WPKH and P2SH-P2WPKH, and the prevout script itself for P2TR. A scanner that looks in only one place will miss payments. The `.skip(1)` drops the coinbase transaction (it spends no earlier outputs) and lines up `spent_outputs[i]` with transaction `i+1`.

A third read uses the block index itself. `build_block_locators` walks back from an entry with `BlockTreeEntry::ancestor(height)` instead of following `prev` one block at a time, so each hop goes through the block-index skip list at O(log N). It seeds `getheaders` locators from `best_entry()` (what follows the headers you already have) and `getblocks` locators from `active_chain().tip()` (what follows the blocks at your tip).

## Building on the kernel

The kernel gives you a small set of operations, among them `process_block_header`, `process_block`, `active_chain`, `best_entry`, `get_block_tree_entry`, `read_spent_outputs`, `import_blocks`, and `interrupt`, plus synchronous callbacks that report events. The consensus-critical parts you'd worry about most (script validation, the UTXO set, reorgs) are the parts you never touch. kernel-node submits headers before asking for bodies, uses the shared `DownloadState` to hand blocks over in an order the kernel can connect, and moves expensive reads to a scan thread so they stay off the block-processing thread. 
