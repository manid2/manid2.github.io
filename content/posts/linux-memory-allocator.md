+++
title = "Linux memory allocator"
description = """Brief discussion on Linux memory allocator."""
draft = false
date = "2023-02-04"
author = "Mani Kumar"
categories = ["linux", "memory", "allocator"]
tags = ["linux", "memory", "allocator"]
+++

This post is a brief discussion on linux memory allocator and should help with
understanding the memory allocations in linux OS from physical memory to user
space memory. It is not intended to be a complete documentation to understand
the specifics in memory allocation and management instead it acts as an
introductory material with pointers to the actual documentation that contains
more in depth content about the concepts mentioned here.

All the information in this post is based on the latest version v6.2 of Linux
kernel.

Physical memory
---------------

Physical memory is detected by linux kernel at boot time. It maybe addressed
from 0x0 (low) to a maximal (high) address spanning contiguous range. But in
practice it may not be contiguous or start exactly at 0x0 address, there can
be several contiguous ranges at completely distinct addresses and can contain
"holes" that are not accessible for the CPU.

In large scale machines such as servers with multiple processors the memory
maybe arranged into banks that incur a different cost to access depending on
the “distance” from the processor. This arrangement is called Non-Uniform
Memory Access (NUMA). In the NUMA system, each processor has part of the
physical memory "attached". The memory has a single address space. Therefore,
any processor could access any memory location directly using its physical
address but the time to access it depends on the distance to the processor
which results in a non uniform memory access time.

For general purpose computers such as desktops the memory can be accessed by
all CPUs uniformly in the same way a single processor accesses its memory.
This type of memory architecture for multiple processors is called Uniform
memory access (UMA). All processors have equal access time to any memory
location and access to the memory is balanced, these systems are also called
SMP (symmetric multiprocessor) systems.

To handle this diversity and complexity of physical memory in different memory
architectures the linux kernel uses these memory models to provide abstraction
for physical memory.

### FLATMEM

This the simplest memory model and is suitable for non-NUMA systems with
contiguous, or mostly contiguous, physical memory. In this memory model a
global `mem_map` array that maps the entire physical memory where each element
of this array is of type `struct page` which describes a page in the physical
memory. Each element in this array is accessed using the page frame number
(PFN) minus `ARCH_PFN_OFFSET` the first page frame number in the system. While
this model is efficient it could not handle large holes in the physical
address space.

### SPARSEMEM

To handle discontiguous physical memory and support NUMA architectures
DISCONTIGMEM model was developed. It introduced the notion of a memory "node"
which is the basis of NUMA memory management. Each node carries an independent
memory management subsystem with its own memory information. The node data is
represented by `struct pglist_data` with typedef `pg_data_t` and contains the
node local memory map. Assuming that each node has contiguous physical memory,
having an `mem_map` array of `struct page` elements per node solves the
problem of large holes in the FLATMEM. But with DISCONTIGMEM it is necessary
to determine which node has the given page in memory to convert its PFN into
page.  In this model splitting the node creates unnecessary fragmentation and
also doesn't supports several advanced features such as memory hot-plug and
hot-remove.

This limitation was overcome with the new model SPARSEMEM. It abstracts the
use of discontiguous memory maps as a collection of sections of arbitrary size
defined by the architectures. Each section is a `struct mem_section` logically
a pointer to an array of `struct page` and it is stored with logic to aid in
the sections management. With SPARSEMEM the advantage is that it doesn't
require each NUMA node's physical address space to be contiguous and it can
handle overlapping address space between nodes which DISCONTIGMEM discards
that memory. With SPARSEMEM there are two ways to convert a PFN to the
corresponding `struct page` first is the “classic sparse” and the second is
“sparse vmemmap”.  This is selected at build time and it is determined by the
value of `CONFIG_SPARSEMEM_VMEMMAP`. The "classic sparse" encodes the section
number of a page in `page->flags` and uses high bits of a PFN to access the
section that maps that page frame. Inside a section, the PFN is the index to
the array of pages. The sparse vmemmap uses a virtually mapped memory map to
optimize `pfn_to_page ()` and `page_to_pfn ()` operations.

Virtual memory
--------------

Virtual memory was developed to avoid managing the physical memory directly by
the application software as it is very limited, complex and varies across
architectures. Using the virtual memory kernel allows protected and controlled
access to physical memory. Application software processes can access only
virtual address space. On a 32 bit system the there are 2^32 unique memory
addresses which gives 4GB of virtual memory and on 64 bit systems it is 2^64
very huge number. The actual memory can be much lesser than that and the
kernel allocates physical memory fairly for all processes in the system.

This is achieved by translating the virtual addresses into physical addresses
by CPU using the `page tables` maintained by kernel. To make this address
translation simple and efficient the virtual and physical memories are divided
into fixed size chunks called `pages` or `page frames`. Each physical memory
page is mapped to one or more virtual pages using the page tables. The page
tables are organized hierarchically as they can be larger than the physical
memory.

The tables at the lowest level of the hierarchy contain physical addresses of
actual pages used by processes. The tables at higher levels contain physical
addresses of the pages belonging to the lower levels. The pointer to the top
level page table resides in a register. When the CPU performs the address
translation, it uses this register to access the top level page table. The
high bits of the virtual address are used to index an entry in the top level
page table. That entry is then used to access the next level in the hierarchy
with the next bits of the virtual address as the index to that level page
table. The lowest bits in the virtual address define the offset inside the
actual page.

When there is no page table entry (PTE) for a given virtual memory page then
the processor fails to translate into physical page when needed i.e.
read/write operations. The processor then notifies the kernel of `page fault`
and kernel decides to allocate physical memory page if the PTE is "valid" and
the physical memory page is available. If PTE is not valid then it means that
the process has attempted to access a virtual address that it should not have
e.g. writing to random addresses in memory, in this case the kernel will
terminate the process to protect the other processes in the system. If the PTE
is valid but no free physical memory pages are available then the kernel
invokes  Out Of Memory (OOM) manager which simply verifies that the system is
"truly" out of memory and if so, select a process to kill.

In Linux any number of virtual memory pages are allocated to the processes as
requested but physical memory pages are only allocated when needed i.e.
attempt to read/write into a virtual memory. This way for the processes the
memory is always "available" and kernel manages all the complexities of the
physical memory.

Cache memory
------------

To make accessing the system memory fast and efficient among multiple
processes with requests to access data and instructions cache memory is used
to act as a buffer between the CPU registers and system memory. CPU registers
are very limited in size but are as fast as CPU. System memory is much larger
but not as fast as CPU as it is located outside CPU. Cache memory operates at
same speed as the CPU and is not kept waiting when data is to be accessed from
it. Its purpose is to function as a very fast copy of the contents of selected
portions of the system memory

Cache memory is configured to read/write data/instructions from/to system
memory. When data is to be read from the system memory then system checks if
the data is in cache, if available in cache then it is quickly used by CPU. If
not available in cache then it is fetched from system memory and used while
also copying it into cache so that it can be quickly reused later.

When writing data from the CPU it is first written to the cache memory and
marked as "dirty". The dirty pages in cache memory are periodically written to
the backing storage device. When linux decides to reuse them for other
purposes it synchronizes them with the backing storage.

Linux kernel recognizes the cache memory as the Page Cache which stores the
data in fixed size chunks of memory called "pages". In older linux kernels
there was also buffer cache which contained the data buffers used by the block
device drivers. Buffer cache is now outdated and only Page Cache is available
in linux kernel.

### Cache levels

In latest computers cache memory is available in multiple levels with lower
levels being closer to CPU and hence fastest and smaller in size. Higher
level caches are larger in size but smaller then system memory and slightly
faster but slower than lower level caches.

Most systems have 2 levels of cache and some systems such as the high
performance servers have 3 levels. These are namely L1, L2 and L3 caches. L1
cache is often located directly on the CPU chip and runs at the same speed as
the CPU. L2 cache is often part of the CPU module, runs nearly at CPU speeds,
and is usually a bit larger and slower than L1 cache. L3 cache is often called
Last-Level Cache (LLC) and is the largest and slowest cache memory unit.

System calls to allocate memory
-------------------------------

In Linux all user space memory is allocated using these system calls, they are
called from inside user space memory management functions _libc_ `malloc()`
and `free()`. As `malloc()` does more than just allocate memory to user space
processes such as to keep record of all the memory allocated to all the user
space processes including their stats. Also provides fast and efficient way to
allocate and manage the memory. Due to these reasons it is better to avoid
using these system calls directly instead use `malloc()`. Here we can learn
about what these system calls do in brief.

### brk () and sbrk ()

These system calls sets up the _program break_ which is the location in
process virtual memory pointing to the end of _data segment_ which contains
global variables and local static variables. Before _shared libraries_ and
`mmap()` there was no concept of _heap_ hence the memory obtained was part of
_data segment_ and now it is part of the _heap_. Increasing the _program
break_ increases the memory available to the process effectively allocating
memory to the process and decreasing it reduces the process memory i.e.
deallocates memory.

`brk()` takes an initial address as argument and sets the end of the data
segment to this value.

`sbrk()` takes an increment value and increments the program break by this
amount. Calling sbrk() with an increment of 0 is used to find the current
location of the program break which is also the initial heap.

### mmap ()

`mmap()` system call creates a mapping in the process virtual address space it
takes a _start_ address, _length_ of the mapping, _protection_ and _flags_ to
control the mapping. The start address given to `mmap()` is just a hint to the
kernel about where to start the mapping in linux the kernel will pick a
"nearby" page boundary. Kernel then attempts to create a mapping there and if
another mapping is already exists then the kernel picks a new address that may
or may not depend on the hint. The address of the new mapping is returned.
The _glibc_ `malloc()` uses this system call when allocating "large" blocks of
memory instead of using `brk()` as it is more efficient for large memory
allocation.

References
----------

* [Physical Memory Model][1]
* [Memory: the flat, the discontiguous, and the sparse][2]
* [sparsemem memory model][3]
* [4.2.2. Cache Memory][4]
* [Cache Miss – What It Is and How to Reduce It][5]
* [What does the brk() system call do?][6]
* [Linux system calls source code][7]
* `man brk(2), mmap(2), malloc(3)`

[1]: https://www.kernel.org/doc/html/latest/mm/memory-model.html
[2]: https://lwn.net/Articles/789304/
[3]: https://lwn.net/Articles/134804/
[4]: https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/4/html/introduction_to_system_administration/s2-memory-cache
[5]: https://www.hostinger.com/tutorials/cache-miss
[6]: https://stackoverflow.com/a/6990428
[7]: https://elixir.bootlin.com/linux/latest/source/tools/include/nolibc/sys.h
