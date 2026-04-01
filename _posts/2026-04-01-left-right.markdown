---
layout: post
title:  "Left-right"
date:   2026-04-01
categories: swift concurrency
---
One day, my YouTube algorithm recommended a video by [Jon Gjengset](https://thesquareplanet.com) called [The Cost of Concurrency Coordination](https://www.youtube.com/watch?v=tND-wBBZ8RY&t=2567s). I strongly recommend watching it if you are interested in concurrency locking mechanisms, how they actually work at the CPU core level, and the left-right concurrency control technique - which was new to me at the time. It essentially enables wait-free read operations for any data structure. I wanted to learn it and implement it in Swift, and stumbled onto some cool things along the way.

## Why This Problem Exists At All

Modern Apple Silicon chips have multiple cores - an M3 Pro has 12, an M2 Ultra has 24. Each core has its own L1 and L2 cache. When two cores want to read the same data, that's fine - they can both have a copy in their local cache simultaneously. When one core wants to write, it has to invalidate every other core's cached copy, wait for acknowledgements, then do the write. This cross-core communication is expensive and doesn't scale - the more cores you have, the more acknowledgements you need.

This coordination is managed by the [MESI protocol](https://en.wikipedia.org/wiki/MESI_protocol) - the cache coherence 
protocol used by Apple Silicon (or some MESI flavour). Every cache line is in one of four states:

- **Modified** - I have the only copy, and it's dirty (different from RAM)
- **Exclusive** - I have the only copy, and it matches RAM
- **Shared** - multiple cores have this line, all matching RAM
- **Invalid** - my copy is stale, someone else modified it

When Core 0 wants to write to a line in `Shared` state, it broadcasts an invalidate message to all other cores, waits for acknowledgements, transitions to `Modified`, then writes. The more cores you have, the more acknowledgements you need to collect before you can write. This is the fundamental scalability problem that every concurrent data structure is working around.

This is why concurrent access to shared mutable state is hard. It's not just a software problem. It's a hardware problem. Every lock, every atomic operation, every memory barrier exists because of this physical reality.

The cache line is the unit of transfer. On Apple Silicon it's 128 bytes. The CPU never moves less than 128 bytes between caches. This has a critical implication: if two completely unrelated variables happen to sit within the same 128-byte region of memory, and two different cores write to them, those cores will thrash each other's caches even though they're touching different data. This is false sharing - and it's one of the most common reasons parallel code doesn't scale.

I knew what a lock was. I knew what a data race was. But the idea that two variables could fight over cache just by sitting next to each other in memory - that was new to me, and honestly a bit mind-blowing.

This diagram is what everything else in this article is about. Each core has its own private caches - L1, L2 - and they all share L3 and RAM. When two cores read the same data, no problem. When one wants to write, it has to tell every other core to throw away their cached copy. That round-trip is what makes concurrent writes expensive, and it's the reason locks, atomics, and memory barriers exist in the first place.

```
                    ┌─────────────────────────────────────────┐
                    │              Main RAM                   │
                    │            ~100+ cycles                 │
                    └─────────────────┬───────────────────────┘
                                      │
                    ┌─────────────────┴───────────────────────┐
                    │             L3 / SLC Cache              │
                    │              ~40 cycles                 │
                    └──────────────┬──────────────────────────┘
                                   │
              ┌────────────────────┴─────────────────────┐
              │                                          │
    ┌─────────┴────────┐                       ┌─────────┴────────┐
    │      Core 0      │                       │      Core 1      │
    │                  │                       │                  │
    │  ┌────────────┐  │                       │  ┌────────────┐  │
    │  │  L2 Cache  │  │                       │  │  L2 Cache  │  │
    │  │ ~12 cycles │  │                       │  │ ~12 cycles │  │
    │  └─────┬──────┘  │                       │  └─────┬──────┘  │
    │        │         │                       │        │         │
    │  ┌─────┴──────┐  │                       │  ┌─────┴──────┐  │
    │  │  L1 Cache  │  │                       │  │  L1 Cache  │  │
    │  │  ~4 cycles │  │                       │  │  ~4 cycles │  │
    │  └─────┬──────┘  │                       │  └─────┬──────┘  │
    │        │         │                       │        │         │
    │  ┌─────┴──────┐  │                       │  ┌─────┴──────┐  │
    │  │  Registers │  │                       │  │  Registers │  │
    │  │  ~0 cycles │  │                       │  │  ~0 cycles │  │
    │  └────────────┘  │                       │  └────────────┘  │
    └──────────────────┘                       └──────────────────┘
```

## What Locks Actually Are

Every lock in existence is built on one hardware primitive: [Compare-And-Swap (CAS)](https://en.wikipedia.org/wiki/Compare-and-swap). On ARM it's implemented as a pair of instructions - `ldaxr` (load-acquire exclusive) and `stlxr` (store-release exclusive). The hardware guarantees these are atomic - indivisible, no intermediate state visible to other cores.
CAS does this atomically:

```
if *address == expected {
    *address = new
    return success
} else {
    return failure  
}
```

If two cores race to CAS the same address, exactly one wins. The loser retries. Everything else - every lock, every mutex, every semaphore you've ever used - is built on top of this one instruction.

[os_unfair_lock](https://developer.apple.com/documentation/os/os_unfair_lock) (what [OSAllocatedUnfairLock](https://developer.apple.com/documentation/os/osallocatedunfairlock) wraps) uses CAS to try to acquire. If it succeeds - nobody else held it - the whole thing takes ~5ns and never touches the kernel. If it fails, it spins briefly then calls into the kernel to sleep the thread. "Unfair" means no FIFO queue - when released, any waiter can grab it. This is the lock you want in Swift when you need raw performance.

[NSLock](https://developer.apple.com/documentation/foundation/nslock) wraps [pthread_mutex_t](https://pubs.opengroup.org/onlinepubs/7908799/xsh/pthread_mutex_lock.html). Heavier - ~25ns uncontended, fair (FIFO ordering), more features. For the write path of left-right it doesn't matter, because writes are rare. I used it because it's simple and the overhead is irrelevant when you're only writing once every few seconds.

Here's the fundamental problem with locks for reads:

```
acquire lock      ← all readers queue here, one at a time
read data
release lock
```

Even if nobody is writing, readers block each other acquiring and releasing the lock. On 8 cores all trying to read the same data, 7 are waiting at any moment. Read throughput doesn't scale with core count - it's essentially serial. You bought an 8-core machine and you're using one.

[pthread_rwlock](https://pubs.opengroup.org/onlinepubs/009696899/functions/pthread_rwlock_rdlock.html) improves this by allowing concurrent readers, but readers still have to atomically increment a shared reader count on every read. That shared counter is a single cache line being hammered by every reader on every core - bouncing between L1 caches constantly. Under high concurrency, that counter becomes your bottleneck. It's why `pthread_rwlock` can actually be slower than a plain mutex under high read contention. You solved one problem and created another.

## The ARM Memory Model

Here's something that surprises most people: the CPU does not execute your instructions in the order you wrote them. Both the compiler and the CPU reorder operations to keep execution units busy. In single-threaded code this is invisible because the CPU tracks dependencies. In multi-threaded code it's a real problem.

Consider Thread A:

```
data = 42
ready = true
```

And Thread B:

```
while !ready { }
print(data)  // Is this always 42?
```

On x86 - yes. x86's strong memory model ([TSO - Total Store Order](https://en.wikipedia.org/wiki/Memory_ordering)) prevents store-store reordering. On ARM - no. The CPU can reorder those two stores. Thread B might see `ready = true` but still read `data = 0`. This is not a bug. It's a documented feature of the ARM memory model, designed to allow aggressive out-of-order execution for performance. Apple Silicon is ARM. Your M-series Mac can do this to you.

A memory barrier tells the CPU: do not reorder loads/stores across this point. On ARM the full barrier is `dmb ish` - expensive because it drains the store buffer and waits for acknowledgements from all other cores. You don't want this in a hot path.
The ARM architecture gives you lighter-weight alternatives:

- [ldar](https://developer.arm.com/documentation/ddi0596/2020-12/Base-Instructions/LDAR--Load-Acquire-Register-) - load-acquire: this load cannot be reordered with any memory operation after it
- [stlr](https://developer.arm.com/documentation/ddi0596/2021-06/Base-Instructions/STLR--Store-Release-Register-) - store-release: this store cannot be reordered with any memory operation before it

These are cheaper than a full barrier but sufficient for most synchronization patterns - specifically the acquire-release pairing.

## The Left-Right Algorithm

The insight behind left-right is simple: what if readers never had to touch any shared mutable state at all? No lock to acquire, no counter to increment, no cache line to contend on. Just data, sitting there, waiting to be read.

The way you achieve this is by keeping two complete copies of your data structure. At any moment, one copy is active - readers use it. The other is inactive - the writer owns it. Readers never touch the inactive copy. The writer never touches the active copy while readers are on it.

```
         ┌──────────────┐         ┌──────────────┐
Readers  │    ACTIVE    │         │   INACTIVE   │  Writer
  ──────▶│    Copy A    │         │    Copy B    │◀──────
  ──────▶│              │         │              │
  ──────▶└──────────────┘         └──────────────┘
                    ▲
                    │
               readIndex
          (atomic bool - tells
          readers which copy
              to use)
```

### The Write Sequence

Step 1. The writer applies the operation to the inactive copy (B). Readers are all on A - this is completely safe, no coordination needed.

```
         ┌──────────────┐         ┌──────────────┐
Readers  │    ACTIVE    │         │   INACTIVE   │  Writer
  ──────▶│    Copy A    │         │    Copy B'   │◀── writes here
  ──────▶│  (old value) │         │  (new value) │
         └──────────────┘         └──────────────┘
```

Step 2. The writer atomically flips `readIndex`. New readers now go to B' - the updated copy. Old readers that were already reading A continue until they finish. You can't stop them mid-read.

```
         ┌──────────────┐         ┌──────────────┐
Old      │  now         │         │  now ACTIVE  │  New readers
readers  │  INACTIVE    │         │    Copy B'   │◀──────
finishing│    Copy A    │         │  (new value) │◀──────
  ──────▶│  (old value) │         │              │
         └──────────────┘         └──────────────┘
```

Step 3. The writer waits for all readers that were on A to finish. This is called the drain. Once drained, A is truly inactive - no thread is touching it.

Step 4. The writer applies the same operation to A. Both copies are now identical and up to date.

```
         ┌──────────────┐         ┌──────────────┐
         │   INACTIVE   │         │    ACTIVE    │  Readers
Writer ──▶│   Copy A'   │         │    Copy B'   │◀──────
         │  (new value) │         │  (new value) │◀──────
         └──────────────┘         └──────────────┘
```

The next write starts from this state, with A' as the inactive copy. The two sides alternate on every write.

### The Drain Problem

After the flip, the writer needs to wait for readers that were mid-read on the old copy to finish. But how does it know when they're done - without shared mutable state?

This is the clever part. Each reader gets its own epoch counter - a single integer that lives on its own private cache line. The protocol is simple:

```
Even value → not currently reading
Odd value  → currently reading
```

The read protocol:

```
increment my counter  (even → odd)   "I am starting a read"
load readIndex                        "which copy do I use?"
read the data
increment my counter  (odd → even)   "I am done"
```

After the flip, the writer snapshots all epoch counters. Any counter that was odd at the moment of the flip belongs to a reader that was mid-read on the old copy. The writer spins on those specific counters until they go even. Readers that start after the flip go to the new copy - the writer doesn't care about them.

```
Writer after flip:

epoch[0] = 4  (even) → not reading, skip
epoch[1] = 7  (odd)  → was reading on old copy, wait...
epoch[2] = 2  (even) → not reading, skip
epoch[3] = 11 (odd)  → was reading on old copy, wait...

... epoch[1] becomes 8, epoch[3] becomes 12 ...

All clear. Apply operation to now-inactive copy.
```

Each reader writes only to its own epoch counter - its own private cache line. No other thread writes to it. The writer only reads the counters during the drain, it never writes them. So there is no cache line bouncing between cores during the read path.

This is the whole algorithm. Two copies, an atomic flip, epoch counters, a drain. Everything else in the implementation is making this correct and efficient in Swift.

## Swift Atomics

Swift has no built-in atomic operations. For a long time, if you needed atomics in Swift you had to drop into C or use deprecated OSAtomic APIs. [swift-atomics](https://github.com/apple/swift-atomics) is Apple's official answer to that - a package that exposes atomic operations on primitive types with the full C++ memory model, available from pure Swift.

### ManagedAtomic vs UnsafeAtomic

There are two atomic types you'll use. The difference is ownership of storage.
`ManagedAtomic<T>` is a class. Heap allocated, ARC managed. When you write:

```
let counter = ManagedAtomic<Int>(0)
```

What you get in memory:
```
Stack:
┌──────────────────┐
│  counter (ptr)   │──────┐  8 bytes, a pointer
└──────────────────┘      │
                          ▼
Heap:
┌──────────────────────────────────┐
│  ARC refcount    │  8 bytes      │
├──────────────────────────────────┤
│  type metadata   │  8 bytes      │
├──────────────────────────────────┤
│  atomic value    │  8 bytes      │  ← the actual Int
└──────────────────────────────────┘
```

Every operation requires following that pointer. One extra memory access on every read or write. For a single atomic value this is fine - the overhead is negligible. For an array of atomics where layout matters, it's a problem.

`UnsafeAtomic<T>` does not own its storage. You provide the memory, you manage the lifetime. The key type is `UnsafeAtomic<T>.Storage` - a fixed-size chunk of properly aligned bytes that you embed directly in your own struct:

```
struct EpochCounter {
    var storage: UnsafeAtomic<UInt>.Storage = .init(0)
}
```

When you put these in an array, the atomic values are contiguous in memory exactly where you put them. This is what lets you control layout - essential for cache line padding.

To operate on the storage you create a temporary handle:

```
withUnsafeMutablePointer(to: &storage) { ptr in
    UnsafeAtomic<UInt>(at: ptr).wrappingIncrement(ordering: .releasing)
}
```

The handle is just a pointer wrapper. It doesn't own anything, doesn't allocate anything. It exists for the duration of the closure and disappears. The value lives in `storage`.
```
Array of EpochCounters with UnsafeAtomic:

┌──────────────────────────────────┐
│  UInt value  │  120 bytes pad   │  ← EpochCounter[0], 128 bytes, own cache line
├──────────────────────────────────┤
│  UInt value  │  120 bytes pad   │  ← EpochCounter[1], 128 bytes, own cache line
└──────────────────────────────────┘
No pointers. No heap. Values exactly where you need them.

Array of EpochCounters with ManagedAtomic:

┌─────┬─────┬─────┬─────┐
│ ptr │ ptr │ ptr │ ptr │  ← array of pointers, nicely laid out
└──┬──┴──┬──┴──┬──┴──┬──┘
   │     │     │     │
   ▼     ▼     ▼     ▼
  heap  heap  heap  heap   ← actual values scattered, same cache line, false sharing
```

For left-right, UnsafeAtomic is the only option that makes the cache line padding meaningful.

### Memory Orderings

This is the part that confused me the most at first. The ordering parameter on an atomic operation doesn't affect whether the operation is atomic - it's always atomic. It controls how the operation interacts with surrounding memory operations - what the CPU and compiler are allowed to reorder around it.

There are four you need to know:

`.relaxed`

No ordering constraints. The operation is atomic but the CPU can reorder it freely with respect to everything around it. Use when you only care about atomicity, not about what other memory is visible.

```
nextSlot.loadThenWrappingIncrement(ordering: .relaxed)
```

On ARM this compiles to a plain atomic instruction with no barrier - fastest possible.

`.acquiring` (loads only)

This load cannot be reordered with any memory operation after it. Think of it as a one-way barrier - nothing below this line can float above it.

```
// ARM: ldar
epochs[i].load(ordering: .acquiring)
```

`.releasing` (stores only)

This store cannot be reordered with any memory operation before it. Nothing above this line can sink below it.

```
// ARM: stlr
epoch.wrappingIncrement(ordering: .releasing)
```

**The acquire-release pairing** is the fundamental synchronization mechanism in left-right. A `.releasing` store on Thread A synchronizes with an `.acquiring` load of the same variable on Thread B. Once that synchronization is established, Thread B is guaranteed to see all memory writes Thread A did before the release.

Without the pairing:
```
Thread A:
data = 42
ready.store(true, .relaxed)   ← no ordering guarantee

Thread B:
ready.load(.relaxed)          ← sees true
print(data)                   ← might still see 0 on ARM
```

With the pairing:
```
Thread A:
data = 42
ready.store(true, .releasing)  ← stlr: data write cannot sink below this

Thread B:
ready.load(.acquiring)         ← ldar: nothing below can float above this
print(data)                    ← guaranteed to see 42
```

The `stlr`/`ldar` pair establishes a happens-before relationship across threads. This is exactly the same hardware mechanism described in the ARM memory model section - now you're using it directly from Swift.

### How This Maps to Left-Right

Now every ordering choice in the implementation has a concrete reason:

```
// Reader entering - stlr
// prevents the data read below from floating above this signal
cell.epochs[slot].increment()  // .releasing

// Which copy? - plain ldr
// the .releasing increment above already acts as a barrier
let useRight = cell.readIndex.load(ordering: .relaxed)

// Read the data
let result = body(useRight ? cell.right : cell.left)

// Reader done - stlr
// prevents the data read above from sinking below this signal
cell.epochs[slot].increment()  // .releasing
```

```
// Writer flipping readIndex - stlr
// the inactive copy write cannot become visible after this flip
readIndex.store(inactiveIndex, ordering: .releasing)

// Writer polling during drain - ldar
// synchronizes with reader's .releasing increment
// guarantees writer sees reader's data access as complete
while epochs[i].load(ordering: .acquiring) == seenEpoch { }
```

Without `.releasing` on the reader's exit increment, the CPU could reorder the data read to after the "I'm done" signal - the writer would think the reader finished when it hadn't. Without `.acquiring` on the writer's drain poll, the CPU could speculate past the while loop before the reader's stores were actually visible.

On ARM these compile to `stlr` and `ldar` - lightweight, no full `dmb ish` barrier anywhere in the hot path. On x86 they compile to plain `mov` instructions because TSO gives you acquire-release semantics for free. `swift-atomics` handles the difference transparently.

## The Swift Implementation

Now that we understand the algorithm, the hardware, and the primitives, the implementation becomes straightforward. Every line has a reason.

### EpochCounter

The first thing we need is the epoch counter. One per reader, padded to a full cache line so there's no false sharing between slots.

```
struct EpochCounter {
    var storage: UnsafeAtomic<UInt>.Storage = .init(0)

    #if arch(arm64)
    private let _pad: (UInt64, UInt64, UInt64, UInt64,
                       UInt64, UInt64, UInt64, UInt64,
                       UInt64, UInt64, UInt64, UInt64,
                       UInt64, UInt64, UInt64) =
        (0,0,0,0,0,0,0,0,0,0,0,0,0,0,0)
    #else
    private let _pad: (UInt64, UInt64, UInt64,
                       UInt64, UInt64, UInt64, UInt64) =
        (0,0,0,0,0,0,0)
    #endif

    init() {}

    func increment() {
        withUnsafeMutablePointer(to: &storage) { ptr in
            UnsafeAtomic<UInt>(at: ptr).wrappingIncrement(ordering: .releasing)
        }
    }

    func load() -> UInt {
        withUnsafeMutablePointer(to: &storage) { ptr in
            UnsafeAtomic<UInt>(at: ptr).load(ordering: .acquiring)
        }
    }
}
```

`UnsafeAtomic<UInt>.Storage` is 8 bytes embedded directly in the struct - no heap, no pointer. The padding fills the rest of the cache line: 120 bytes on ARM64 (128 - 8), 56 bytes on x86 (64 - 8). The `#if` arch block handles both platforms.

`increment()` uses `.releasing` - the data read must complete before this store becomes visible to the writer. `load()` uses .acquiring - the writer synchronizes with the reader's release and sees the data access as truly complete. These two orderings are a pair. One without the other is wrong.

You can verify the layout is correct at compile time:

```
#if arch(arm64)
assert(MemoryLayout<EpochCounter>.stride == 128)
#else
assert(MemoryLayout<EpochCounter>.stride == 64)
#endif
```

`stride` not `size` - stride is what determines spacing between elements in an array, which is what cache line isolation depends on.

### LeftRight

The container holds the two copies, the atomic flip index, the epoch counter array, and the write lock.

```
public final class LeftRight<T: Sendable>: @unchecked Sendable {

    internal var left: T
    internal var right: T

    internal let readIndex: ManagedAtomic<Bool>
    internal let epochs: UnsafeMutableBufferPointer<EpochCounter>

    private let writeLock = NSLock()
    private let nextSlot = ManagedAtomic<Int>(0)

    public init(_ initial: T, maxReaders: Int = 64) {
        self.left = initial
        self.right = initial
        self.readIndex = ManagedAtomic(false)
        self.epochs = UnsafeMutableBufferPointer<EpochCounter>.allocate(capacity: maxReaders)
        self.epochs.initialize(repeating: EpochCounter())
    }

    deinit {
        epochs.deinitialize()
        epochs.deallocate()
    }

    public func makeReader() -> Reader<T> {
        let slot = nextSlot.loadThenWrappingIncrement(ordering: .relaxed) % epochs.count
        return Reader(cell: self, slot: slot)
    }
}
```

`@unchecked Sendable` - Swift can't prove this is safe because `left` and `right` are unprotected `var` properties. You're telling the compiler to trust you. The epoch counters and atomic orderings are what make that claim true.

`ManagedAtomic<Bool>` for `readIndex` - this is fine as a class reference. There's only one of it, read once per read operation. The pointer dereference is not your bottleneck.

`UnsafeMutableBufferPointer<EpochCounter>` for the epoch array - not `[EpochCounter]`. A Swift array would give you CoW semantics and no control over memory layout. The buffer pointer gives you direct access to contiguous memory that you manage yourself. `allocate` followed by `initialize` - you need both. `allocate` reserves the memory, `initialize` puts valid `EpochCounter` values in it. Without `initialize` you have garbage values in your epoch counters and the even/odd protocol breaks immediately.

`nextSlot` uses `.relaxed` - handing out slot indices is just a counter. No thread is synchronizing on this value, no ordering guarantees needed.

`deinit` calls `deinitialize` then `deallocate` - in that order, always. deinitialize runs the deinitializer on each element. deallocate frees the memory.

### Reader

Each reader thread gets its own `Reader` - created once, kept for the lifetime of the thread. It knows its slot and holds a reference to the cell.

```
public struct Reader<T: Sendable> {
    internal let cell: LeftRight<T>
    internal let slot: Int
}

extension Reader {
    public func read<R>(_ body: (T) -> R) -> R {
        cell.epochs[slot].increment()

        let useRight = cell.readIndex.load(ordering: .relaxed)

        let result = body(useRight ? cell.right : cell.left)

        cell.epochs[slot].increment()

        return result
    }
}
```

This is the hot path. Four operations:

1. Increment epoch counter - signals entering read, `.releasing `ensures data access can't float above this
2. Load `readIndex` - `.relaxed` is correct, the increment above already established ordering
3. Read the data - the actual work, no lock, no contention
4. Increment epoch counter again - signals done, `.releasing` ensures data access can't sink below this

No lock. No shared counter. No cache line bouncing between cores. Each reader touches only its own private epoch slot and then the data. This is why reads scale.

### The Write Path

```
extension LeftRight {
    public func write(_ operation: (inout T) -> Void) {
        writeLock.lock()
        defer { writeLock.unlock() }

        let currentReadIndex = readIndex.load(ordering: .acquiring)
        let inactiveIndex = !currentReadIndex

        if inactiveIndex == false {
            operation(&left)
        } else {
            operation(&right)
        }

        readIndex.store(inactiveIndex, ordering: .releasing)

        let snapshot = (0..<epochs.count).map { epochs[$0].load() }

        waitForReaders(snapshot: snapshot)

        if currentReadIndex == false {
            operation(&left)
        } else {
            operation(&right)
        }
    }

    private func waitForReaders(snapshot: [UInt]) {
        for (i, seenEpoch) in snapshot.enumerated() {
            guard seenEpoch & 1 == 1 else { continue }
            while epochs[i].load() == seenEpoch {
                sched_yield()
            }
        }
    }
}
```

Step by step:

1. **Lock.** `NSLock` enforces single-writer. In correct usage this never contends - you're calling `write` from one thread. The lock is there to catch misuse, not to handle real contention.

2. **Load `readIndex` with `.acquiring`.** The writer needs to see the current state of the world - specifically any stores that happened before the last flip.

3. **Apply to inactive copy.** The copy readers are not on. No reader will touch it. No synchronization needed for the write itself.

4. **Flip `readIndex` with `.releasing`.** Redirects new readers to the updated copy. `.releasing` ensures the write to the inactive copy is visible to all cores before any reader gets redirected to it. Without this, a reader could arrive at the new copy before the writer's mutations are visible - silent corruption on ARM.

5. **Snapshot.** Immediately after the flip, read all epoch counters. Any counter that is odd at this moment was mid-read on the old copy. These are the only readers the writer needs to wait for.

6. **Drain.** For each slot that was odd at snapshot time, spin until the counter changes. Any change means the reader finished. `sched_yield()` yields the thread to the OS scheduler - if the reader isn't currently scheduled, this lets it run rather than the writer burning CPU waiting for something that can't happen yet.

7. **Apply to now-inactive copy.** The old active copy is fully drained. Apply the same operation. Both copies are now identical.

## What I Learned

I started this as a learning exercise. I wanted to understand how left-right works, implement it in Swift, and see what I'd learn along the way. I didn't expect to end up this deep into ARM instruction sets, cache coherence protocols, and memory ordering models.

A few things that stuck with me:

**False sharing was the biggest surprise.** The idea that two completely unrelated variables can fight over cache just by sitting next to each other in memory - that's not something you learn from writing everyday Swift code. You only discover it when you have to care about where things live in memory.

**Memory orderings stopped being magic words.** Before this, `.acquiring` and `.releasing` felt like incantations you copy from Stack Overflow and hope for the best. After implementing left-right, they're just ARM instructions - `ldar` and `stlr` - with a clear, specific job. I understand what breaks if I get them wrong, because I deliberately got them wrong and watched TSan catch it.

**The algorithm itself is surprisingly small.** Two copies, an atomic flip, epoch counters, a drain. That's it. The implementation is about 135 lines of Swift.

**Swift gets you surprisingly close to the metal.** `swift-atomics`, `UnsafeMutableBufferPointer`, manual memory layout - these aren't C. They're Swift. You can write lock-free concurrent data structures in Swift without dropping into C, and the result is correct, readable, and fast.

The implementation is a learning exercise and is scoped accordingly - full replacement writes only, fixed reader slots, Apple Silicon focused. There's more to build. But as a foundation for understanding how this class of primitive works, I'm happy with it.


## Sources

- [swift-left-right](https://github.com/juraskrlec/swift-left-right) - my Swift implementation
- [Left-Right: A Classical Algorithm](https://concurrencyfreaks.blogspot.com/2013/12/left-right-classical-algorithm.html) - Pedro Ramalhete and Andreia Correia, the original description of the primitive
- [jonhoo/left-right](https://github.com/jonhoo/left-right) - Jon Gjengset's Rust implementation, the direct inspiration for this project
- [The Cost of Concurrency Coordination](https://www.youtube.com/watch?v=tND-wBBZ8RY&t=2567s) - Jon Gjengset's video that started all of this
- [apple/swift-atomics](https://github.com/apple/swift-atomics) - Apple's official Swift atomics package
- [ARM Architecture Reference Manual](https://developer.arm.com/documentation/ddi0487/latest) - the definitive source on `ldar`, `stlr`, and the ARM memory model