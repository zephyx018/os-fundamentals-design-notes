# OS Fundamentals Design Note: Virtual Memory, Page Replacement and Thrashing

*Capstone project: OS fundamentals design note*

## 1. Problem

Programs need more memory than is physically available, and many programs must share the same RAM safely. An operating system has to give each process the illusion of a large, private address space while using a limited pool of physical memory efficiently. The design question is: how do we map, cache and replace memory so that processes run fast, stay isolated, and the system does not collapse under load?

## 2. Design Overview

The solution is **demand paging** built on three cooperating pieces:

1. **Paging.** Virtual memory is split into fixed-size pages (for example 8 KB, which needs 13 offset bits because 8 KB = 2^13 bytes). Physical memory is split into frames of the same size. A per-process page table maps page numbers to frame numbers.
2. **The TLB.** A Translation Lookaside Buffer is a small, fast hardware cache of recent page-to-frame mappings. Without it, every memory access would need an extra page table access.
3. **Page replacement.** When a page fault occurs and no frame is free, the OS must evict a page to make room.

## 3. Key Design Decisions and Trade-offs

### 3.1 Fixed-size pages

Fixed-size frames mean any free frame can satisfy any request, so **external fragmentation is eliminated**. The cost is **internal fragmentation** in the last page of each allocation, and page table memory overhead.

### 3.2 TLB for speed

On a TLB hit the frame number is returned immediately. On a TLB miss for a page that is present in memory, the page table is walked to find the frame and the entry is loaded into the TLB. This works because of locality of reference, so hit rates are typically above 95%. The trade-off is that a context switch may require flushing the TLB (or tagging entries with an address-space ID), which is a major part of context switch cost.

Effective access time (single-level page table):

`EAT = h × (t_tlb + t_mem) + (1 − h) × (t_tlb + 2 × t_mem)`

### 3.3 Choosing a replacement policy

| Policy | Idea | Strength | Weakness |
| --- | --- | --- | --- |
| FIFO | Evict the oldest page | Very simple | Can evict hot pages; suffers Belady's anomaly |
| LRU | Evict the least recently used page | Good approximation of future use; stack algorithm | Costly to implement exactly |
| Optimal | Evict the page used farthest in the future | Lowest possible faults | Needs future knowledge; used only as a benchmark |

**Belady's anomaly** is the surprising case where giving a process more frames produces more page faults. It occurs with FIFO. Stack algorithms such as LRU and Optimal are immune, which is a strong argument for LRU-like policies (often approximated with a reference bit or clock algorithm in practice).

### 3.4 Working set and thrashing

The **working set** of a process is the set of pages it referenced in a recent window of Δ references. If a process is not given enough frames to hold its working set, it faults constantly, and the CPU spends more time paging than computing. When many processes do this at once, the OS may see low CPU utilization and wrongly admit even more processes, making things worse. This collapse is **thrashing**.

The correct response is to **reduce the degree of multiprogramming** by suspending or swapping out some processes, so the remaining ones can keep their working sets resident. A related approach is to track page-fault frequency and adjust each process's frame allocation accordingly.

## 4. Worked Example

Suppose a process has 3 frames and references pages in the order `1 2 3 4 1 2 5 1 2 3 4 5`. Under FIFO this causes 9 page faults. With 4 frames, FIFO causes 10 faults. Adding a frame made things worse, which demonstrates Belady's anomaly. Running the same string under LRU never shows this behavior.

## 5. Summary of Recommendations

- Use paging with fixed-size frames to avoid external fragmentation.
- Rely on a TLB (with ASIDs where possible) to keep translation cheap.
- Prefer an LRU-approximating replacement policy over plain FIFO.
- Monitor working sets or page-fault rate, and reduce multiprogramming when thrashing is detected.

## 6. Conclusion

Virtual memory works because several simple mechanisms reinforce each other: paging removes fragmentation, the TLB hides translation cost, replacement policies decide what stays in RAM, and working-set control keeps the whole system out of thrashing. Each mechanism has a trade-off, and the OS design is the balance among them.
