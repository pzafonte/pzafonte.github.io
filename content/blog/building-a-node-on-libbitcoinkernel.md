---
date: '2026-09-25T12:00:00-04:00'
draft: false
title: "Building a node on libbitcoinkernel: a tour of kernel-node's structure"
---

kernel-node is an experimental full node built on libbitcoinkernel, Bitcoin Core's consensus and storage code compiled to a C API library. It validates blocks but does not serve them. It syncs from several peers at once, and the download path is where most of its design decisions live.

Two aspects of kernel-node's block-processing thread represent a good example to show the central constraint in programming against libbitcoinkernel. First, when the node submits a block it throws the return value away:

```rust
let _ = chainman.process_block(&block);   // src/bin/node.rs, block processing thread
```

Second, the node only ever submits a block whose parent is already on the active chain, and it decides that by asking the kernel rather than tracking a tip of its own:

```rust
pub fn is_on_active_chain(&self, block_hash: &bitcoin::BlockHash) -> bool {   // src/peer.rs
    let hash = bitcoinkernel::BlockHash::from(block_hash.to_byte_array());
    match self.chainman.get_block_tree_entry(&hash) {
        Some(entry) => self.chainman.active_chain().contains(&entry),
        None => false,
    }
}
```

Both only make sense if submitting a block and connecting it to the chain are different operations, and if the return value of `process_block` does not tell you which one happened. That split between submitting a block and learning whether it connected shapes how you sync, where you buffer, and what you ask the kernel afterward. I will follow the rest of kernel-node's structure to explain why.


## The kernel/node split

Each operation belongs to one side of the boundary. libbitcoinkernel owns everything that touches consensus or persistent state:

- Script and signature verification, amounts, weight limits.
- The UTXO set (`chainstate/`, a LevelDB).
- Block and undo storage (`blocks/blk*.dat`, `blocks/rev*.dat`) and the block index.
- The block tree and the most-work decision, including reorgs.
- Its own pool of script-verification threads.

The node owns everything else: peer discovery, the wire protocol, the choice of which block to hand the kernel next, all threading, and the wallet. The node runs one loop that finds blocks, decides which one the kernel can use, and hands it over. Every "is this valid?" belongs to the kernel. The node deals in `bitcoin` crate types while the kernel deals in its own, and a small adapter module isolates conversion between them.

You build the kernel from two objects. A `Context` carries configuration and the callbacks through which the kernel reports what happened. A `ChainstateManager` owns the databases and does the work.

```rust
let context = create_context(network.chain_type(), fatal.clone(), Arc::clone(&wallet), scan_tx);

let chainman = ChainstateManagerBuilder::new(&context, &data_dir, &blocks_dir)
    .unwrap()
    .worker_threads(((available_parallelism().unwrap().get() / 2) + 1).try_into().unwrap())
    .build()
    .unwrap();

node_state.chainman.import_blocks().unwrap();
```

Passing `data_dir` and `blocks_dir` hands the kernel ownership of disk. The kernel opens the block index, the chainstate, and the flat files, and starts its verification threads. `worker_threads` sizes that script-verification pool at half the machine's parallelism plus one. `import_blocks()` replays whatever already sits in `blk*.dat` to rebuild the block tree, so restarting does not re-download the chain, and an existing Bitcoin Core datadir works directly.

## Results come through callbacks

Because `process_block` will not tell you whether a block connected, the callbacks you register on the `Context` are not optional. `create_context` wires ten of them. The two that carry a block payload set the pattern.

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

These handlers have three properties and structures the rest of the node.

**They run synchronously, on the thread that called `process_block`, before it returns.** Anything slow inside a handler adds to that call's latency and delays validation of every later block. `block_connected` does the minimum. It checks whether the wallet even holds keys, and if so packages a `ScanEvent` and sends it down a channel. The scan thread does the real work later. Keeping the callback this small isn't optional, since the kernel runs it on the validation thread.

**The kernel passes the `BlockTreeEntry` as a borrow whose lifetime ends when the call returns.** A handler cannot store it in a struct or send it down a channel. So a handler either uses the entry immediately, as `block_disconnected` does when it reads `entry.height()`, or forwards the block hash and lets a later thread re-derive the entry. `block_connected` takes the second route. It sends `block_hash`, and the scan thread re-fetches the entry with `get_block_tree_entry`. The borrow checker enforces the structure the synchronous-callback model needs. You cannot accidentally move a kernel handle onto another thread.

**`block_checked` reports validity, `block_connected` reports chain membership, and they are distinct events.** The kernel can check a block valid without connecting it (it extends a non-best branch), and both callbacks can fire several times inside one `process_block` call as a run of buffered blocks connects at once. So the node cannot read chain status from the single return value.

## The return value isn't enough

`process_block` reports whether it newly wrote the block to disk, found a duplicate, or rejected it. None of those answers the two questions the node has, whether the active chain advanced and whether this specific block is now valid and connected. The kernel can store a block on disk (it knows the header, it has the body) and still leave it unconnected because the parent has not connected yet. So the node discards the return and reads the answer from `block_connected`. `is_on_active_chain` above makes the same decision from the other side. To discover whether a block's parent has connected, the node asks the kernel's `active_chain` instead of trusting any state it kept itself.

For the kernel's block submission, headers-first synchronization is a precondition. The kernel accepts a block's header before it touches the body, and it cannot place a header whose parent is not already in the block tree, so the node can only submit a body once the tree holds its header. kernel-node's peer state machine enforces the ordering directly. When a synced peer announces new blocks with an `inv`, the node does not request the bodies. Rather, it resyncs headers first.

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

The node builds the download queue by walking the header chain. `populate_download_queue` starts from `chainman.best_entry()`, the most-work header the kernel knows, and walks `prev()` back until it reaches a block already on the active chain, which yields both the fork point and every hash between there and the best header. The node only requests bodies for hashes that already exist as headers in the tree.

The kernel connects in order because of ordinary Bitcoin Core consensus behavior. `ConnectBlock` validates block N against the state block N-1 left behind. The UTXO set is a single mutable database rather than a per-block snapshot. Median-time-past reads the previous eleven blocks. Difficulty retarget reads the previous 2016. The kernel appends undo data on each connect so a reorg can walk back. A block whose parent has not connected has no state to validate against, and the node's job is to never put the kernel in that position during normal operation.

A single process_block call tells the node little on its own.

| The node wants to know | `process_block` return | Where it actually reads it |
|---|---|---|
| Is the block on disk? | yes (`NewBlock` / `Duplicate`) | the return value |
| Did the active chain advance? | no | `block_connected` callback |
| Is this block valid? | no (may be deferred) | `block_checked` callback |
| Has this block's parent connected? | no | `is_on_active_chain` → `active_chain().contains` |

## Ordering parallel downloads

By default the peer manager runs eight peer threads (`DEFAULT_MAX_PEERS = 8`, and `--connect` pins it to one). They share a single download plan, so they fetch each block once and a slow peer does not stall the sync. That plan is one struct behind one mutex.

```rust
pub struct DownloadState {   // src/peer.rs
    queue: VecDeque<BlockHash>,
    in_flight: HashSet<BlockHash>,
    buffer: HashMap<BlockHash /* prev */, bitcoinkernel::Block>,
}
```

A peer reserves work with `pop_batch`, which moves up to `DOWNLOAD_BATCH_SIZE` (16) hashes from the queue into `in_flight`. The reservation rides on the return value of `HashSet::insert`. A hash already in flight fails the insert, so `pop_batch` skips it and two peers never request the same block. When a block arrives, `release` removes it from `in_flight`, and only the peer whose removal succeeds buffers it, since another peer may have delivered it first.

```rust
if node_state.download.lock().unwrap().release(&block_hash) {
    node_state.buffer_block(prev_blockhash, block.convert());
}
```

A peer that dies calls `release_in_flight`, which pushes everything it still owed back to the front of the queue for another peer to pick up. With eight peers and a batch of sixteen, at most 128 requests stay outstanding at once, and a disconnect costs the sync one batch and nothing else.

Blocks arrive in whatever order eight peers happen to send them, so the node keys the buffer by parent hash and drains it in connectable order. `take_connectable` looks for a buffered block whose parent already sits on the active chain. It carries a second clause below the first.

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

The first branch keeps each `process_block` call to a single block whose parent the kernel already connected. The second handles a branch that forks below the current tip. The node buffers its blocks, none of their parents sit on the active chain, and the first branch never fires, because the parents are not missing. They sit on a chain the node already has. Once the queue and the in-flight set both empty out, no more blocks will come, so the node hands over the oldest buffered block (`buffer_head` finds the buffer entry that is not the child of another buffered block) and lets the kernel decide, by accumulated work, whether that branch should become the active chain. The node delegated that decision to the kernel. Otherwise those blocks would sit in the buffer forever.

The peer threads and the validation thread meet at this shared buffer with no bounded queue between them, so a peer keeps reserving batches whether or not validation has kept up, and the buffer grows to the distance between download speed and validation speed. Bounding it would let a full buffer stall every peer on the slowest one's write, reintroducing the coupling parallel download removes, so it stays unbounded. The one guard is a 20-minute `STALE_BLOCK_DURATION` backstop. If no block connects in that window, the block-processing thread drops every peer so the manager picks new ones.

## Reading the chain

Feeding blocks is one use of the kernel. Querying it is another.

One type of read is cheap, because the tip lives in the in-memory block index. Bitcoin Core keeps the block index and the active chain resident in RAM, so the handler behind the Cap'n Proto control socket reads the tip directly, a pointer walk and a couple of field reads with no file opened and no cost that grows with the chain.

```rust
let tip = self.chainman.active_chain().tip();
r.set_height(tip.height() as u32);
r.set_hash(BlockHash::from_byte_array(tip.block_hash().to_bytes()).to_string());
```

The IPC thread can call `active_chain().tip()` safely while validation runs elsewhere.

The other read is expensive, because the data it needs is gone from memory. For each transaction, the silent-payments wallet needs the outputs it spent, and those prevouts leave the UTXO set once the block connects. They survive only in the undo data the kernel wrote to `rev*.dat` at connect time, so `read_spent_outputs` opens that file, seeks the block's record, and decodes every spent coin for every input in the block, a cost that scales with the block's input count. Running that inside the `block_connected` callback would put a disk seek and a per-input decode on the validation thread for every block, so the callback only forwards the hash and the scan thread does the read.

```rust
let entry = chainman_for_scan.get_block_tree_entry(&block_hash) /* or fatal */;
let spent_outputs = chainman_for_scan.read_spent_outputs(&entry) /* or fatal */;
let count = wallet.lock().unwrap().scan_block(block, spent_outputs, block_height);
```

For each input, `scan_block_inner` then reconstructs the three fields BIP-352 needs to recover the sender's public key, the input's `scriptSig`, its witness, and the script of the output it spent.

```rust
for (kernel_tx, tx_spent) in kernel_block.transactions().skip(1).zip(spent_outputs.iter()) {
    // ...
    let coin = tx_spent.coin(idx).expect("input/spent-output count mismatch");
    let witness: Vec<Vec<u8>> = input.witness_stack().items().collect();
    let script_sig = input.script_sig().unwrap();
    // coin.output().script_pubkey() is the prevout script the scan requires
}
```

The `scriptSig` and witness sit in the block already. The prevout script does not. It belongs to an output created in an earlier block. The kernel supplies it through the same undo data it wrote when it connected the block, so the wallet recovers every input's public key without keeping its own UTXO set or prevout index. Exposing both the `scriptSig` and the witness matters because different input types carry the key in different places (pay-to-pubkey-hash in the `scriptSig`, SegWit and Taproot in the witness), and a scan that only sees one would miss real payments. The `.skip(1)` drops the coinbase, which spends nothing, so `spent_outputs[i]` aligns with transaction `i+1`.

A third read uses the kernel's index as a data structure. `build_block_locators` walks back from an entry to build a `getheaders`/`getblocks` locator, reaching each ancestor with `BlockTreeEntry::ancestor(height)` rather than following `prev` one block at a time, using the block index skip list at O(log N) per hop. The same routine seeds `getheaders` from `best_entry()` (what comes after the headers you have) and `getblocks` from `active_chain().tip()` (what comes after the blocks you have).

## Building on the kernel

The kernel library hands you a small set of operations, among them `process_block_header`, `process_block`, `active_chain`, `best_entry`, `get_block_tree_entry`, `read_spent_outputs`, `import_blocks`, and `interrupt`. Alongside them sit callbacks that report events synchronously on the validation thread. The consensus-critical parts you might worry about most (script validation, the UTXO set, reorgs) are the parts you never touch. Almost all of the design effort goes into the three things the kernel library does not do for you. The node gets headers into the tree before requesting bodies, orders the blocks it submits so each connects against a parent that already connected, and keeps the callbacks small enough that nothing expensive runs on the validation thread. kernel-node handles them with the shared `DownloadState` queue, a blocking handoff to the validation thread, and the scan-thread indirection.

kernel-node is experimental and under heavy development, so the specifics here may change. Even so, this should give you a good sense of the project's contours and of what building a node on the kernel involves.