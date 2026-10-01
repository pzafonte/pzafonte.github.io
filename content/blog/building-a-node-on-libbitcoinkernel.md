---
date: '2026-09-25T12:00:00-04:00'
draft: false
title: "Building a node on libbitcoinkernel: a tour of kernel-node's structure"
---

kernel-node is an experimental full node based on libbitcoinkernel, which compiles Bitcoin Core's consensus and storage code into a C API library. It validates blocks but does not serve them. It syncs from several peers at once, and most design decisions focus on the download path.

Two aspects of kernel-node's block-processing thread represent a good example to show the central constraint in programming against libbitcoinkernel. First, when the node submits a block, it disregards the return value:

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

Both only make sense if submitting a block and connecting it to the chain are different operations, and if the return value of `process_block` does not tell you which one happened. The division between block submission and chain connection informs your synchronization strategy and buffer management. I will follow the rest of kernel-node's structure to explain why.

## The kernel/node split

Operations function entirely on one side of the boundary. libbitcoinkernel owns everything touching consensus and persistent state:

- Signature and script verification, weight, amounts, and limits.
- The UTXO set (`chainstate/`, a LevelDB).
- Block and undo storage (`blocks/blk*.dat`, `blocks/rev*.dat`) and the block index.
- The block tree and the most-work decision, including reorgs.
- Dedicated threads that verify scripts.

The node handles everything else: peer discovery, wire protocol, block selection for the kernel, thread control, and wallet management. The node runs one loop that finds blocks, decides which one the kernel can use, and gives the chosen block to the kernel. All validity checks are performed by the kernel. The node uses `bitcoin` crate types while the kernel uses its own types, and a small adapter handles conversions between the node's and kernel's types.

You build the kernel from two objects. A `Context` holds the configuration and callbacks that let the kernel report events. A `ChainstateManager` manages the databases and performs the operations.

```rust
let context = create_context(network.chain_type(), fatal.clone(), Arc::clone(&wallet), scan_tx);

let chainman = ChainstateManagerBuilder::new(&context, &data_dir, &blocks_dir)
    .unwrap()
    .worker_threads(((available_parallelism().unwrap().get() / 2) + 1).try_into().unwrap())
    .build()
    .unwrap();

node_state.chainman.import_blocks().unwrap();
```

Passing `data_dir` and `blocks_dir` hands the kernel ownership of disk. The kernel opens the block index, the chainstate, and the flat files, and starts its verification threads. `worker_threads` sets that pool to one more than half the machine's logical cores. The block index and chainstate persist. Restarting does not re-download the chain, and an existing Bitcoin Core datadir works directly. `import_blocks()` activates the best chain, replaying `blk*.dat` only on a reindex.

## Results come through callbacks

`process_block` does not indicate whether a block connected. Callbacks registered on the `Context` do. `create_context` registers ten callbacks; these two define the pattern:

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

These handlers have three properties that structure the rest of the node.

They run synchronously, on the thread that called `process_block`, before it returns. Anything slow inside a handler increases the call's latency and delays validation of subsequent blocks. `block_connected` does the minimum. It verifies whether the wallet contains keys, creates a `ScanEvent` and pushes it through the channel, and lets the scan thread perform the work. Small callbacks are not optional since the kernel runs them on the validation thread.

The kernel passes the `BlockTreeEntry` as a borrow whose lifetime ends when the call returns. A handler cannot store it in a struct or send it down a channel; it must either use the entry immediately, as `block_disconnected` does when it reads `entry.height()`, or forward the block hash and let a later thread re-derive the entry. `block_connected` takes the second route. It sends `block_hash`, and the scan thread calls `get_block_tree_entry`. The borrow checker enforces the structure the synchronous-callback model needs. You cannot accidentally move a kernel handle onto another thread.

`block_checked` reports a block's validity while `block_connected` reports its chain membership. A block stored on a non-best branch triggers neither, and one `process_block` call can trigger both several times as blocks the kernel stored earlier connect behind it. The node cannot read chain status from the return value alone.

## Stored versus connected

`process_block` provides information about whether it successfully stored the block on disk, found a duplicate, or rejected it. None of those answers the two questions the node has: whether the active chain advanced, and whether this specific block is now valid and connected. The kernel can store a block on disk (it knows the header, it has the body) and still leave it unconnected because the parent has not connected yet. `process_block` can even return `NewBlock` for a block that fails `ConnectBlock` during the same call, since `Rejected` only covers failures before the block is stored. The node discards the return and reads the answer from `block_connected`. `is_on_active_chain` above makes the same decision from the other side. To determine whether a block's parent is connected, the node checks the kernel's `active_chain` rather than relying on local state.

kernel-node syncs headers first. The kernel doesn't require it, but the download queue walks block tree entries, and a hash has no entry until its header is processed.

When handling `NetworkMessage::Inv`, the node checks `block_hashes` and, if not empty, requests their headers first:

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

When an `Inv` arrives, the node calls `build_block_locators(node_state.chainman.best_entry().unwrap())`, sends `create_getheaders_message(locators)`, and only after the headers exist populates the download queue. `populate_download_queue` initiates at `chainman.best_entry()` and traces `prev()` back to the active chain, identifying the fork point and the intervening hashes. The node only requests bodies for hashes that already exist as headers in the tree.

Bitcoin Core consensus logic determines the order of kernel connections. `ConnectBlock` validates block N by comparing it to the UTXO state left by block N-1. The kernel appends undo data during `ConnectBlock` so a reorg can revert changes. A block whose parent isn't connected has no validation state. The node must avoid leaving the kernel in that situation.

| The node wants to know | `process_block` return | Where it actually reads it |
|---|---|---|
| Is the block on disk? | yes (`NewBlock` / `Duplicate`) | the return value |
| Did the active chain advance? | no | `block_connected` callback |
| Is this block valid? | no (may be deferred) | `block_checked` callback |
| Has this block's parent connected? | no | `is_on_active_chain` → `active_chain().contains` |

## Parallel downloading in batches

The peer manager creates eight peer threads. `DEFAULT_MAX_PEERS = 8`; `--connect` pins it to one. They share a single download plan, guarded by a single mutex, so they fetch each block once and a slow peer does not stall the sync:

```rust
pub struct DownloadState {   // src/peer.rs
    queue: VecDeque<BlockHash>,
    in_flight: HashSet<BlockHash>,
    buffer: HashMap<BlockHash /* prev */, bitcoinkernel::Block>,
}
```

A peer reserves work with `pop_batch`, which moves up to `DOWNLOAD_BATCH_SIZE` (16) hashes from the queue into `in_flight`. The reservation relies on `HashSet::insert` returning true. A hash already in flight fails the insert. `pop_batch` skips it, and two peers never request the same block. When `release` removes the hash from `in_flight`, the peer that removed that hash buffers the arriving block:

```rust
if node_state.download.lock().unwrap().release(&block_hash) {
    node_state.buffer_block(prev_blockhash, block.convert());
}
```

A peer that dies calls `release_in_flight`, which pushes everything it still owes back to the front of the queue for another peer to pick up. With eight peers and a batch of sixteen, `pop_batch` keeps at most 128 requests outstanding, and a disconnect costs the sync one batch and nothing else.

Because blocks are delivered in the order the peers send them, the node keys the buffer by parent hash and drains it in connectable order. `take_connectable` searches for a buffered block whose parent is on the active chain. It carries a second clause below the first:

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

The first branch guarantees `process_block` only processes one block at a time whose parent the kernel has already attached. The second handles a divergence occurring below the current tip. The node buffers its blocks, none of their parents sit on the active chain, and the first branch never fires, even though the parents are not missing. They sit on a chain the node already has. Once the queue and the in-flight set both empty out, no more blocks will come. The node then hands over the oldest buffered block (`buffer_head` finds the buffer entry that is not the child of another buffered block) and lets the kernel decide, by accumulated work, whether that branch should become the active chain. The node delegates that decision to the kernel. Otherwise those blocks would sit in the buffer forever.

The peer threads and the validation thread meet at this shared buffer with no bounded queue between them. Peers continue reserving batches whether or not validation has kept up, and the buffer grows by the difference between download and validation speed. Creating a bound would allow a full buffer to stall all peers, because it can't drain until the slowest peer's block arrives. That would reintroduce the coupling the parallel download removes, so the system remains unbounded. Bitcoin Core instead limits downloads to a 1024-block window and disconnects the peer stalling it. kernel-node's backstop is a 20-minute `STALE_BLOCK_DURATION`. If no block reaches the kernel in that time, the block-processing thread disconnects all peers and the manager selects new ones.

## Reading the chain

The kernel receives blocks and provides query operations; reads that access the tip are cheap because the tip is stored in the in-memory block index. Bitcoin Core maintains the block index and active chain in RAM. The handler reads the tip directly, which is a pointer walk and a couple of field reads. No files are opened, and the cost does not grow with chain length:

```rust
let tip = self.chainman.active_chain().tip();
r.set_height(tip.height() as u32);
r.set_hash(BlockHash::from_byte_array(tip.block_hash().to_bytes()).to_string());
```

The IPC thread can call `active_chain().tip()` safely while validation runs elsewhere.

The other read is expensive, because the data it needs is gone from memory. For each transaction, the silent-payments wallet needs the outputs it spent, and those prevouts leave the UTXO set once the block connects. They survive only in the undo data the kernel wrote to `rev*.dat` at connect time. `read_spent_outputs` accesses this file to find the block's record and decode every input's spent coin. Because reads would add a disk seek per block if done inside `block_connected`, the callback forwards only the hash and a separate scan thread performs the read:

```rust
let entry = chainman_for_scan.get_block_tree_entry(&block_hash) /* or fatal */;
let spent_outputs = chainman_for_scan.read_spent_outputs(&entry) /* or fatal */;
let count = wallet.lock().unwrap().scan_block(block, spent_outputs, block_height);
```

For each input, `scan_block_inner` then reconstructs the three fields BIP-352 needs to recover the sender's public key: the input's `scriptSig`, its witness, and the script of the output it spent.

```rust
for (kernel_tx, tx_spent) in kernel_block.transactions().skip(1).zip(spent_outputs.iter()) {
    // ...
    let coin = tx_spent.coin(idx).expect("input/spent-output count mismatch");
    let witness: Vec<Vec<u8>> = input.witness_stack().items().collect();
    let script_sig = input.script_sig().unwrap();
    // coin.output().script_pubkey() is the prevout script the scan requires
}
```

Both `scriptSig` and witness are included in the block; the prevout script comes from the spent output in the earlier block. The kernel's undo data lets the wallet rebuild input keys without a UTXO set. Check both `scriptSig` (P2PKH) and witness (SegWit/Taproot), because different input types store keys in different places and a scanner that looks at only one will miss payments. Call `.skip(1)` to skip the coinbase transaction (it spends no earlier outputs) and line up `spent_outputs[i]` with transaction `i+1`.

Another read uses the kernel's index as a data structure. `build_block_locators` walks back from an entry to produce `getheaders`/`getblocks` locators. It reaches each ancestor by calling `BlockTreeEntry::ancestor(height)` instead of traversing the `prev` pointer one block at a time. Each hop goes through the block-index skip list and costs O(log N). The same routine seeds `getheaders` from `best_entry()` (what follows the headers you already have) and `getblocks` from `active_chain().tip()` (what follows the blocks at your tip).

## Building on the kernel

The kernel library provides a limited set of operations: among them `process_block_header`, `process_block`, `active_chain`, `best_entry`, `get_block_tree_entry`, `read_spent_outputs`, `import_blocks`, and `interrupt`, with synchronous callbacks that report events on the validation thread. The consensus-critical parts you might worry about most (script validation, the UTXO set, reorgs) are the parts you never touch. The node submits headers before requesting bodies, hands over each block only once its parent has connected, and keeps callbacks brief to avoid heavy work on the validation thread. kernel-node uses a shared `DownloadState` queue to hand off work to the validation thread and scan-thread indirection to avoid costly work on the validation thread. These interfaces are under active development and may change.
