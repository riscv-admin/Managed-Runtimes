# Assessment of GC barrier checks from the perspective of potential hardware support (work in progress)

A GC barrier is a piece of special purpose code inserted into compiled application code.
Depending on where they appear, there are read and write barriers, that is, they are inserted somewhere
near a memory read operation or a memory write operation. (More specifically, read-reference or write-reference
operations. A reference is an object pointer. There are some exceptions we want to leave out for simplicity.
So we will refer to them as just read/write.)

Typically 3 GC features require barriers:
* Concurrent Mark (CM): typically requires write or read barrier
* Concurrent Copy (CC): typically requires read barrier
* Generational GC (GenGC): typically requires write barrier



For folks not familiar with these terms:
You could do a 5-min search about them. But honestly they are not really stopping you from understanding the examples.
We try to summarize the common barrier layouts as follows. These examples are abstracted away from the barriers'
purposes---you won't know why they are there by looking at the example. But hopefully you don't need to.
A load is a load. A test is a test. There is no difficulty in understanding these.

What these features have in common is that they bring "concurrency" to the table, which is to say, compared to
the non-concurrent version of mark or copy (generational GC is practically always concurrent so there is no
non-concurrent generational GC), mutators (i.e., application threads) and GC threads run at the same time. There needs
to be some sort of synchronization between them. By inserting barrier code into application code, mutators can
synchronize with GC threads. In order for GC algorithm to not break under concurrency, the barrier gurantees certain
things, e.g., for CM it's strong/weak tri-color invariant, for CC it's commonly to-space invariant.



Barriers need to be short and fast. Usually they are laid down with at least a fast and a slow path.
When certain conditions are met, mutators can skip barrier altogether, that is, going fast-path
(a good example is the GC state check below),
Since in the fastest path, mutators still need to at least check for a fast/slow path condition,
a very critical part in a barrier then is this check. This check's frequency is a ratio
of the frequency of read or write operations. A barrier can have multiple nested checks. The outermost one
will be the cheapest; the innermost one will be the most precise, but probably laid down further from the
read/write due to higher cost. These checks are interesting because they are a major source of barrier cost and
the relevant part in a barrier when we talk about potential hardware optimization.
Inside the barrier slow path, there could be some heavy-lifting work like memcpy, but this shouldn't be counted towards
barrier cost because you have to pay it if you want automatic garbage collection. Whether the memcpy is in
application code executed by application threads (mutators) or in runtime code executed by GC threads doesn't
change the cost.
So we mainly discuss just these checks and omit the rest.



## Type 1: GC state check
In many cases, the first check in the barrier is a GC state check.
This can be seen in CM or CC, read or write barriers. A write barrier for concurrent
marking purpose (SATB-style) can look like:
```
gc_state_check:
  load r1, (#gc_state_addr) ;; load gc state: is collection in progress or not?
  test r1, 0x1              ;; test for slow path (not shown in example)
  jne slow_path
write:
  store (rd), rs
```
This barrier is merely saying "before we write anything, if GC is in progress, do some (heavy) work".
In practice, GC state check costs very little. We revisit this later.

## Type 2: Region check
Sometimes a pointer's src or dst needs to be checked against a table for information about the
src or dst's GC state. This can be seen in CC read barriers or GenGC write barriers. For example,
a read barrier for concurrent copy can look like:
```
read:
  load rd, (rs)
region_check:
  test #table_base(rd>>20), 0x1 ;; look up in the table (containing GC metadata) for a pointer load from the memory;
                                ;; if it points to the from-space, go slow path (not shown in example)
  jne slow_path
```
``>>20`` here implies that the heap is managed as many 1m-sized regions (this is configurable).
You can see this in OpenJDK Shenandoah and Android RT CC (TableLookup).
Another example is a write barrier for generational GC:
```
write:
  store (rd), rs                ;; update a field with a pointer
region_check:
  test #table_base(rd>>20), 0x1 ;; look up in the table (containing GC metadata) for the *location* of this update;
                                ;; if it's in the old generation, go slow path (not shown in example)
  jne slow_path
```
They are all very similar---compute the index from a raw pointer in some way (more generally, do pointer arithmetics)
then look the index up in a table (containing GC metadata). If the table says this pointer belongs to a region
that needs special handling, the execution jumps to a slow path (not shown in example).

Why do we put these two barriers together? Because they can be optimized by this interesting
memory protection feature:
[s390's Guarded-Storage](https://itdks.su.bcebos.com/51917c6806dc413b949c8c42d71c778b.pdf),
[SOAR](https://dl.acm.org/doi/abs/10.1145/800015.808182) (I didn't know this until
[Jecel mentioned it](https://lists.riscv.org/g/tech-j-ext/message/378). Thanks!).
By protecting the memory regions that need slow-path work, we can implement the barrier very concisely.

This optimization requires that the region size is a multiple of protection granularity. This requirement
kills the oppotunity to optimize some barriers. Say protection granularity is 4k. OpenJDK G1's
barrier for generational GC would check "cards" (512 bytes) in a card table. It looks technically like the example
above but it's too fine-grained for protection to discriminate different cards, so it cannot use this trick directly.

## Type 3: Bitmap check
Marking commonly uses bitmap because its space efficiency to keep track of the marking state of each object.
This can be see in CM write barriers. It Looks like this (SATB-style):
```
bitmap_check:
  load r1, (rd)                ;; load the old pointer before the following update overwrites it
  testbit #bitmap_addr(r1>>64) ;; 'testbit' is short for the sequence of instructions to lookup a pointer in a mark bitmap;
                               ;; if not marked, go slow path (not shown in example)
  je slow_path
write:
  store (rd), rs               ;; update a field with a pointer
```
This appears in V8's CM.
The ``testbit`` sequence is quite long, so GC might not inline such bitmap check into application code and instead
do it in (a call to) runtime code. This is the case for G1 and Shenandoah CM.
By the way, ``bitmap_check`` is only *logically* above ``write`` but it can be shortcutted by other checks like a
GC-state check. That's why it can appear in a distance away from the ``write``, or implemented in the runtime code.
It's similar for our other examples.

## Type 4: Tagged pointers
Some GCs maintain tagged pointers. In their (for example) read barriers they can tell if a pointer points
to a from-space object by looking at the tag. If it does, the execution goes into a slow path (not shown in example).
This is currently seen in CC or CM's read barriers.
The barrier looks like:
```
read:
  load rd, (rs)
tag_check:
  load r1, (#mask_addr) ;; load a bit mask (GC metadata); it encodes the "bad bits" (meaning need action)
  test rd, r1           ;; test the tag bits against the mask in the loaded pointer
  jne slow_path
```
Currently Azul C4, OpenJDK ZGC afaik use this type of barrier. C4 is not open-sourced, but ZGC
would place the tag bits in the non-canonical bits. ZGC would need to be able to dereference tagged pointers.
This mean ZGC will need N times the virtual space to manage a certain-sized heap.
N depends on the number of dereferenceable tag bit states, which is 3 for ZGC now and possibly more
for the future generational ZGC.

By moving the tags to some "ignored bits", ZGC would need less virtual space, and thus fewer page table entries,
which is in theory good for performance. For some unclear reason ZGC did not use the already-available TBI
(Top-Byte Ignore) in their ARM port. [C4](https://dl.acm.org/doi/10.1145/2076022.1993491) on the other hand,
has specialized hardware support. If run on their custom hardware, It can do tag checking using a special instruction.

## Type 5: Header check
Metadata can be encoded into object or page headers, or headers of any self-defined memory management unit.
In which case the barrier finds
the header of the object or page pointed to by a pointer, and checks for slow path.
It's seen in ART CC (BakerReadBarrier), V8 (barrier for generational GC).

Honestly, the list of
shapes of barriers cannot be exhausted so we would not expand further. But the above should cover the most
common ones. Null check appears in barriers too but it's so generic so we don't discuss it here.



## Real code
Here is a piece of real read barrier code from OpenJDK Shenandoah to give you a taste of what it's like.

[LRB of Shenandoah GC](shenandoah.md)


# Cost of read/write barrier
I have an "impression number" for the cost of read/write barriers---10% and 5% respectively.
If you prefer, here are some studies of barrier cost:
* [Shenandoah](https://shipilev.net/talks/jugbb-Sep2019-shenandoah.pdf) (write barrier cost up to 2.4%;
read barrier cost up to 11.7%)
* [ZGC](https://cr.openjdk.java.net/~pliden/slides/ZGC-Jfokus-2018.pdf) (read barriers (tagged-pointer style) cost 4%)
* [Lisp](https://www.researchgate.net/publication/2773436_Barrier_Methods_for_Garbage_Collection) (write barriers cost 2%-7%;
read barriers cost up to ~15%)
* [G1](https://users.cecs.anu.edu.au/~steveb/pubs/papers/g1-vee-2020.pdf) (two write barriers in G1 cost ~12%)
* [Go](https://go.dev/blog/ismmkeynote) (implies write barriers cost 4%-7%; also write barriers
made them give up "ROC" and generational GC)

Read barriers generally cost more because reads are much more common than writes.
Also this data is pure barrier cost. The cost of concurrency is much higher than barrier cost.
To make precise measurement is hard and it depends on specific runtime, hardware, and workload.

How do we understand the cost?
It mainly comes from:
* Instruction execution cost: The sum of time it takes to execute the barrier instructions, weighing in
the probability of each execution path. Colder path probably also suffer from miss prediction.
If the barrier has a memory load, it also affects the data cache.
* Code size of barrier: For each read or write, there is a barrier, each has at
least a couple of instructions inlined in the hot path, and more instructions in the cold path including
a call to runtime slow path handler---it's a lot of code. This affects function inlining, instruction
cache, register allocation.



> Further notes on tag checking vs. region checking
>
> The tagged-pointer-based checks are faster than region checks in general.
> First of all, tag checking is shorter; doing a shift then a table entry load is slightly longer in number of instrs.
> Secondly, the ``#mask_addr`` is usually a
> constant, and loading from a constant address is faster than loading a table entry, which has variable address.
> Some GC (e.g., Shenandoah) would put the region check in its
> "middle path", and insert a explicit GC-state check in its fast path to shortcut the middle path. Even though
> it's perfectly fine to encode GC states in the table, e.g., when GC is not in action, zero all table entries.
> It shows that Shenandoah considers region check expensive enough to worth another if-check to shortcut it.
> OTOH, the tag checking relies on the fact that GC maintains tagged pointers for the entire heap, whereas region checks
> relies on maintaining a table (cheaper). Those are design choices that different GC algorithms took.
>
> Anyway, this is probably too implementation specific.
> The point is, if there is a hardware magic that helps us get rid of the cost of one type of barrier.
> I personally would be more happy if it can get rid of the region-checking barriers. But that's to assume the effort
> to create such a magic is the same. OTOH, the GC algorithms are not non-negotiable. If one algorithm is superior
> because of a hardware magic helps elide its barrier, GC will be more likely to adopt it.



# State of the art in barrier reduction
Here we list some interesting directions that are undertaken by GCs. The first two are rather recent (2022).

## Read-barrier-less concurrent compaction using kernel feature userfaultfd
[Android 13](https://android-developers.googleblog.com/2022/08/android-13-is-in-aosp.html)
seems to try to upgrade its GC from Concurrent Copy to Concurrent Mark Compact (CMC), who has utilized a kernel
feature called userfaultfd to reach read-barrier-lessness. In short, userfaultfd lets user-space program
handle the page fault events itself. We all know that normally upon page fault, if it's a write to a missing page,
kernel would try to supply an empty one; if it's a read to a missing page, kernel would supply a zero page.
Well with userfaultfd, user-space program can now listen to the faults, and redefine the behaviour upon
page faults. That means, e.g., we can supply a page with arbitrary content upon a read.

Here comes Concurrent Compaction. Compaction is a special kind of Copy. In GC terminology it usually implies
that the dest addresses of live object copying is precomputed and more organized. But this doesn't change
the fact that if it's to be concurrent, it needs a read barrier. Android's CMC eliminates read barriers by
remapping the heap to a different address, so that mutators all see a big vacuum where there is no physcial pages
and trigger page faults; then GC threads would handle the page faults themselfs by copying the remapped heap
back to vacuum page-by-page; in the mean time, GC updates all object references in a page with the correct
post-compaction addresses before waking up a mutator thread that blocks on a page fault.

CMC claims to reach 10% reduction in code size due to elimination of read barriers.

## The new Generational ZGC (GenZ) under development
The [GenZ](https://bugs.openjdk.org/browse/JDK-8272979) description is an interesting read. It
demonstrates what the Z devs think when designing the barriers, e.g., how the work are distributed into fast and
slow path; it solves some of the mysteries why ZGC haven't used TBI-like feature---they plan to remove
multi-mapping in GenZ and there are a lot more metadata bits now; it mentions using code patching to squeeze
more performance from the barrier.

The code patching is worth some awareness because its architecture relevance, i.e., not every arch supports
patching and not every instruction supports patching.
([More reading](http://cr.openjdk.java.net/~jrose/jvm/hotspot-cmc.html).)
Restrictions aside, let's see how it works. Remember we talked about [GC state checks](## Type 1: GC state check).
GC state as a variable could stay unchanged for a long time---it usually changes when GC switches phases. So
to avoid having to load a state from the memory only to discover it's the same boring value again and again,
ZGC can patch the mem-load with imm-load, saving the memory visit cost. (The document didn't mention whether
it's a load or not. I infer that. In tag-checking GC, it's also not a "state" but specifically a "color", but
the idea can be generalized.)

## Trap-based barrier reduction with memory protection
We mentioned [s390's Guarded-Storage](https://itdks.su.bcebos.com/51917c6806dc413b949c8c42d71c778b.pdf) which can
be used to implement an efficient barrier. This can be generalize as a trap-based barrier reduction technique.
The gist of it is letting the trap (synchronized exception) conditional replace the
software condition (so the barrier is near 0 code size), letting the trap handler replace the barrier slow path,
and letting GC set the proper trap condition.

Memory protection can be used to accomplish this. For example, x86's Memory Protection Keys For Userspace will
do, albeit a little awkward because it's not designed for this. More importantly,
the critical issue with this kind of technique is the performance of traps. Specifically, can traps be core-local?
Can they be user-spaced? Because trapping into kernel is very expensive.
For these properties we need more architectural support, like what s390 provides.

## Trap-based barrier reduction with tag checking
This is similar to the above approach (can even be considered a special case). The representative is
[C4](https://dl.acm.org/doi/10.1145/2076022.1993491). C4 installs "NMT" bits in a pointer, which if not matching the
pre-conditioned bits, will trap a mutator that dereferences it. C4 makes the trap very fast with a special TLB
mode in custom hardware/kernel.

Similarly, ARM has Memory Tagging Extension, but not providing the fast traps that we hope for.



# Application, prevalence, dependency (WIP)
Instead of listing all types of GCs and all types of their barriers,
the goal is to find the most reasonable thing to work on so as to help a largest subset of them.

There are two general considerations for (server) GC: throughput and response time.
Throughput benefits hugely from generational GC; RT benefits hugely from CC.
Throughput workloads reject CC because CC depends on the higher-costing read barriers (write-barrier CC exists
[theoretically](https://dl.acm.org/doi/pdf/10.1145/376656.376810)).

In weaker hardware (smart phones, embedded systems), memory and energy conservation become another concern.
