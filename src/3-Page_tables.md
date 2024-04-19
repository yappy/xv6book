# Page tables

Page tables are the most popular mechanism through which the operating system provides each
process with its own private address space and memory. Page tables determine what memory ad-
dresses mean, and what parts of physical memory can be accessed. They allow xv6 to isolate
different process’s address spaces and to multiplex them onto a single physical memory. Page ta-
bles are a popular design because they provide a level of indirection that allow operating systems
to perform many tricks. Xv6 performs a few tricks: mapping the same memory (a trampoline page)
in several address spaces, and guarding kernel and user stacks with an unmapped page. The rest of
this chapter explains the page tables that the RISC-V hardware provides and how xv6 uses them.
3.1 Paging hardware
As a reminder, RISC-V instructions (both user and kernel) manipulate virtual addresses. The ma-
chine’s RAM, or physical memory, is indexed with physical addresses. The RISC-V page table
hardware connects these two kinds of addresses, by mapping each virtual address to a physical
address.
Xv6 runs on Sv39 RISC-V, which means that only the bottom 39 bits of a 64-bit virtual address
are used; the top 25 bits are not used. In this Sv39 configuration, a RISC-V page table is logically
an array of 227 (134,217,728) page table entries (PTEs). Each PTE contains a 44-bit physical page
number (PPN) and some flags. The paging hardware translates a virtual address by using the top 27
bits of the 39 bits to index into the page table to find a PTE, and making a 56-bit physical address
whose top 44 bits come from the PPN in the PTE and whose bottom 12 bits are copied from the
original virtual address. Figure 3.1 shows this process with a logical view of the page table as a
simple array of PTEs (see Figure 3.2 for a fuller story). A page table gives the operating system
control over virtual-to-physical address translations at the granularity of aligned chunks of 4096
(212) bytes. Such a chunk is called a page.
In Sv39 RISC-V, the top 25 bits of a virtual address are not used for translation. The physical
address also has room for growth: there is room in the PTE format for the physical page number
to grow by another 10 bits. The designers of RISC-V chose these numbers based on technology
predictions. 239 bytes is 512 GB, which should be enough address space for applications running
31
Virtual address
Physical Address
12
Offset
12
PPN Flags
0
1
10
Page table
27
EXT
2^2744
44
Index
25
64
56Figure 3.1: RISC-V virtual and physical addresses, with a simplified logical page table.
on RISC-V computers. 256 is enough physical memory space for the near future to fit many I/O
devices and DRAM chips. If more is needed, the RISC-V designers have defined Sv48 with 48-bit
virtual addresses [3].
As Figure 3.2 shows, a RISC-V CPU translates a virtual address into a physical in three steps.
A page table is stored in physical memory as a three-level tree. The root of the tree is a 4096-byte
page-table page that contains 512 PTEs, which contain the physical addresses for page-table pages
in the next level of the tree. Each of those pages contains 512 PTEs for the final level in the tree.
The paging hardware uses the top 9 bits of the 27 bits to select a PTE in the root page-table page,
the middle 9 bits to select a PTE in a page-table page in the next level of the tree, and the bottom
9 bits to select the final PTE. (In Sv48 RISC-V a page table has four levels, and bits 39 through 47
of a virtual address index into the top-level.)
If any of the three PTEs required to translate an address is not present, the paging hardware
raises a page-fault exception, leaving it up to the kernel to handle the exception (see Chapter 4).
The three-level structure of Figure 3.2 allows a memory-efficient way of recording PTEs, com-
pared to the single-level design of Figure 3.1. In the common case in which large ranges of virtual
addresses have no mappings, the three-level structure can omit entire page directories. For exam-
ple, if an application uses only a few pages starting at address zero, then the entries 1 through 511
of the top-level page directory are invalid, and the kernel doesn’t have to allocate pages those for
511 intermediate page directories. Furthermore, the kernel also doesn’t have to allocate pages for
the bottom-level page directories for those 511 intermediate page directories. So, in this example,
the three-level design saves 511 pages for intermediate page directories and 511 × 512 pages for
bottom-level page directories.
Although a CPU walks the three-level structure in hardware as part of executing a load or store
instruction, a potential downside of three levels is that the CPU must load three PTEs from memory
to perform the translation of the virtual address in the load/store instruction to a physical address.
To avoid the cost of loading PTEs from physical memory, a RISC-V CPU caches page table entries
in a Translation Look-aside Buffer (TLB).
32
Physical Page Number6
A
5 4 3
U
2
W
1
V
07891063
V
R
W
X
U
A
D

- Valid
- Readable
- Writable
- Executable
- User
- Accessed
- Dirty (0 in page directory)
Virtual address Physical Address
129
L1 L0 Offset
12
PPN Offset
PPN Flags
0
1
10
Page Directory
satp
L2
PPN Flags
0
1
44 10
Page Directory
PPN Flags
0
1
511
10
Page Directory
99
EXT
9
511
511
44
44
44
D U X RG
A - Accessed
-G - Global
RSW
Reserved for supervisor software
53ReservedFigure 3.2: RISC-V address translation details.
Each PTE contains flag bits that tell the paging hardware how the associated virtual address is
allowed to be used. PTE_V indicates whether the PTE is present: if it is not set, a reference to the
page causes an exception (i.e., is not allowed). PTE_R controls whether instructions are allowed
to read to the page. PTE_W controls whether instructions are allowed to write to the page. PTE_X
controls whether the CPU may interpret the content of the page as instructions and execute them.
PTE_U controls whether instructions in user mode are allowed to access the page; if PTE_U is not
set, the PTE can be used only in supervisor mode. Figure 3.2 shows how it all works. The flags and
all other page hardware-related structures are defined in (kernel/riscv.h)
To tell a CPU to use a page table, the kernel must write the physical address of the root page-
table page into the satp register. A CPU will translate all addresses generated by subsequent
instructions using the page table pointed to by its own satp. Each CPU has its own satp so that
different CPUs can run different processes, each with a private address space described by its own
page table.
Typically a kernel maps all of physical memory into its page table so that it can read and write
any location in physical memory using load/store instructions. Since the page directories are in
physical memory, the kernel can program the content of a PTE in a page directory by writing to
the virtual address of the PTE using a standard store instruction.
A few notes about terms. Physical memory refers to storage cells in DRAM. A byte of physical
memory has an address, called a physical address. Instructions use only virtual addresses, which
the paging hardware translates to physical addresses, and then sends to the DRAM hardware to read
33
or write storage. Unlike physical memory and virtual addresses, virtual memory isn’t a physical
object, but refers to the collection of abstractions and mechanisms the kernel provides to manage
physical memory and virtual addresses.0
Trampoline
Unused
Unused
UnusedKstack 0
Guard page
Kstack 1
Guard page
0x1000
0
R-X
Virtual Addresses
CLINT
Kernel text
boot ROM
Physical Addresses
2^56-1
Unused
and other I/O devices
0x02000000
0x0C000000 PLIC
UART0
VIRTIO disk
0x10000000
0x10001000
KERNBASE
(0x80000000)
PHYSTOP
(0x88000000)
MAXVA
Kernel data
R-X
RW-
Physical memory (RAM)
VIRTIO disk
UART0
PLIC
RW-
RW-
RW-
Free memory RW-
...

---
---
RW-
RW-
Figure 3.3: On the left, xv6’s kernel address space. RWX refer to PTE read, write, and execute
permissions. On the right, the RISC-V physical address space that xv6 expects to see.
3.2 Kernel address space
Xv6 maintains one page table per process, describing each process’s user address space, plus a sin-
gle page table that describes the kernel’s address space. The kernel configures the layout of its ad-
dress space to give itself access to physical memory and various hardware resources at predictable
virtual addresses. Figure 3.3 shows how this layout maps kernel virtual addresses to physical ad-
dresses. The file (kernel/memlayout.h) declares the constants for xv6’s kernel memory layout.
34
designs exploit the page table to turn arbitrary hardware physical memory layouts into predictable
kernel virtual address layouts.
RISC-V supports protection at the level of physical addresses, but xv6 doesn’t use that feature.
On machines with lots of memory it might make sense to use RISC-V’s support for “super
pages.” Small pages make sense when physical memory is small, to allow allocation and page-out
to disk with fine granularity. For example, if a program uses only 8 kilobytes of memory, giving
it a whole 4-megabyte super-page of physical memory is wasteful. Larger pages make sense on
machines with lots of RAM, and may reduce overhead for page-table manipulation.
The xv6 kernel’s lack of a malloc-like allocator that can provide memory for small objects
prevents the kernel from using sophisticated data structures that would require dynamic allocation.
A more elaborate kernel would likely allocate many different sizes of small blocks, rather than (as
in xv6) just 4096-byte blocks; a real kernel allocator would need to handle small allocations as
well as large ones.
Memory allocation is a perennial hot topic, the basic problems being efficient use of limited
memory and preparing for unknown future requests [9]. Today people care more about speed than
space efficiency.
3.10 Exercises

1. Parse RISC-V’s device tree to find the amount of physical memory the computer has.
2. Write a user program that grows its address space by one byte by calling sbrk(1). Run
the program and investigate the page table for the program before the call to sbrk and after
the call to sbrk. How much space has the kernel allocated? What does the PTE for the new
memory contain?
3. Modify xv6 to use super pages for the kernel.
4. Unix implementations of exec traditionally include special handling for shell scripts. If the
file to execute begins with the text #!, then the first line is taken to be a program to run to
interpret the file. For example, if exec is called to run myprog arg1 and myprog ’s first
line is #!/interp, then exec runs /interp with command line /interp myprog arg1.
Implement support for this convention in xv6.
5. Implement address space layout randomization for the kernel.
