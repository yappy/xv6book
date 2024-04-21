# Page tables

Page tables are the most popular mechanism through which the operating system provides
each process with its own private address space and memory.

ページテーブルはオペレーティングシステムが各プロセスに自分だけのプライベートアドレス空間と
メモリを提供するために使う最もポピュラーな仕組みである。

Page tables determine what memory addresses mean, and what parts of physical memory can be accessed.

ページテーブルはメモリアドレスが何を意味するか、
そして物理メモリのどの部分にアクセスできるかを決定する。

They allow xv6 to isolate different process’s address spaces and
to multiplex them onto a single physical memory.

ページテーブルにより xv6 は異なるプロセスのアドレス空間を分離し、
それらを1つの物理メモリに多重化することができる。

Page tables are a popular design because they provide a level of indirection
that allow operating systems to perform many tricks.

ページテーブルはオペレーティングシステムが様々なトリックを行えるような
間接参照のレベルを提供するため、ポピュラーな設計である。

Xv6 performs a few tricks:
mapping the same memory (a trampoline page) in several address spaces,
and guarding kernel and user stacks with an unmapped page.

xv6 は少数のトリックを行う。
同じメモリ (トランポリンページ) を複数のアドレス空間にマップする、
カーネルおよびユーザスタックをマップされていないページでガードする。

The rest of this chapter explains the page tables that the RISC-V hardware provides
and how xv6 uses them.

本章の残りでは、 RISC-V ハードウェアが提供するページテーブルについてと、
xv6 がそれをどう使うのかについて説明する。

## Paging hardware

ページングハードウェア

As a reminder, RISC-V instructions (both user and kernel) manipulate virtual addresses.

思い出しておくこととして、RISC-V 命令 (ユーザとカーネルの両方) は仮想アドレスを操作する。

The machine’s RAM, or physical memory, is indexed with physical addresses.

マシンの RAM、つまり物理メモリは物理アドレスでインデックス付けされている。

The RISC-V page table hardware connects these two kinds of addresses,
by mapping each virtual address to a physical address.

RISC-V ページテーブルハードウェアはこれら2種類のアドレスを、
それぞれの仮想アドレスを物理アドレスにマップすることで接続している。

Xv6 runs on Sv39 RISC-V,
which means that only the bottom 39 bits of a 64-bit virtual address are used;
the top 25 bits are not used.

xv6 は Sv39 RISC-V で動作する。
これは 64 bit 仮想アドレスのうち下位 39 bit のみを使用することを意味する。
上位 25 bit は使用されない。

In this Sv39 configuration, a RISC-V page table is logically
an array of 227 (134,217,728) page table entries (PTEs).

この Sv39 構成では、RISC-V ページテーブルは論理的には 2^27 (134,217,728) 個の
ページテーブルエントリ (PTE) の配列である。
(注: 2^39 / 2^27 を割り算すると 2^12 = 2^2 \* 2^10 = 4 \* 1024 = 4 KiB/page である)
(注: 論理的には、というのはこれを全部メモリに置くとするとメモリを食いすぎるので省略の工夫がある、ということ)

Each PTE contains a 44-bit physical page number (PPN) and some flags.

それぞれの PTE には 44 bit の物理ページ番号 physical page number (PPN) と
いくつかのフラグが入る。

The paging hardware translates a virtual address by using
the top 27 bits of the 39 bits to index into the page table to find a PTE,
and making a 56-bit physical address whose top 44 bits come from the PPN in the PTE and
whose bottom 12 bits are copied from the original virtual address.

ページングハードウェアは仮想アドレスの 39 bit のうち上位 27 bit を
ページテーブルの中から PTE を探すインデックスとして使用し、
上位 44 bit を PTE の PPN から、下位 12 bit を元の仮想アドレスからコピーして、
56 bit の物理アドレスを生成する。

Figure 3.1 shows this process with a logical view of the page table
as a simple array of PTEs (see Figure 3.2 for a fuller story).

図 3.1 にこのプロセスを、シンプルな PTE 配列としての論理的なビューで示している
(より完全なストーリーについては 図 3.2 を見よ)。

A page table gives the operating system control over virtual-to-physical address translations
at the granularity of aligned chunks of 4096 (212) bytes.

ページテーブルはオペレーティングシステムに、アラインされた 4096 (2^12) バイトチャンクの
粒度での仮想-物理アドレス変換の制御を提供する。

Such a chunk is called a page.

このようなチャンクをページと呼ぶ。

In Sv39 RISC-V, the top 25 bits of a virtual address are not used for translation.

Sv39 RISC-V では、仮想アドレスのうち上位 25 bit は変換に使用されない。

The physical address also has room for growth:
there is room in the PTE format for the physical page number to grow by another 10 bits.

物理アドレスは bit 数を増やす余地がある。
PTE フォーマットには 10 bit まで PPN (physical page number) を増やせる場所が残っている。
(注: 図 3.2 参照)

The designers of RISC-V chose these numbers based on technology predictions.

RISC-V の設計者は技術予測に基づいてこれらの数字を選んだ。

239 bytes is 512 GB, which should be enough address space for applications running
on RISC-V computers.

2^39 バイトは 512 GB で、RISC-V コンピュータ上で動くアプリケーションにとって
十分なアドレス空間であるはずだ。

256 is enough physical memory space for the near future to fit many I/O devices and DRAM chips.

2^56 は近い将来において多くの I/O デバイスと DRAM チップを扱うのに十分な
物理メモリ空間と言える。

If more is needed, the RISC-V designers have defined Sv48 with 48-bit virtual addresses [3].

もしもっと必要ならば、RISC-V 設計者は 48 bit 仮想アドレス を持つ Sv48 を定義している。

As Figure 3.2 shows, a RISC-V CPU translates a virtual address into a physical in three steps.

図 3.2 に示すように、RISC-V CPU は仮想アドレスを物理アドレスに3段階のステップで変換する。

A page table is stored in physical memory as a three-level tree.

ページテーブルは3段階のツリーとして物理メモリに格納される。

The root of the tree is a 4096-byte page-table page that contains 512 PTEs,
which contain the physical addresses for page-table pages in the next level of the tree.

ツリーのルートは 512 個の PTE を持つ 4096 バイトのページテーブルページで、
次のレベルのツリーのページテーブルページの物理アドレスが入っている。

Each of those pages contains 512 PTEs for the final level in the tree.

これらのページそれぞれが 512 個のツリーでの最後のレベルの PTE を含んでいる。

The paging hardware uses the top 9 bits of the 27 bits to select a PTE in the root page-table page,
the middle 9 bits to select a PTE in a page-table page in the next level of the tree,
and the bottom 9 bits to select the final PTE.

ページングハードウェアは 27 bit のうち上位 9 bit をルートページテーブルページから PTE を選択するのに使い、
真ん中の 9 bit を次のレベルのページテーブルページから PTE を選択するのに使い、
下位 9 bit を最後の PTE を選択するのに使う。

(In Sv48 RISC-V a page table has four levels,
and bits 39 through 47 of a virtual address index into the top-level.)
(Sv48 RISC-V ではページテーブルは4段で、bit 39-47 がトップレベルでの仮想アドレスインデックスになる。)

If any of the three PTEs required to translate an address is not present,
the paging hardware raises a page-fault exception,
leaving it up to the kernel to handle the exception (see Chapter 4).

1つのアドレスを変換するのに必要とされる3つの PTE のうち1つでも存在しなかった場合
ページングハードウェアはページフォールト例外を発生させ、
カーネルが例外を処理するのに任せる。

The three-level structure of Figure 3.2 allows a memory-efficient way of recording PTEs,
compared to the single-level design of Figure 3.1.

この図 3.2 の3段構造は PTE を保存するのを、図 3.1 の1段設計に比べて省メモリにできる。

In the common case in which large ranges of virtual addresses have no mappings,
the three-level structure can omit entire page directories.

仮想アドレスのうち大部分にマッピングが存在しない通常のケースにおいて、
この3レベル構造はページディレクトリ全体を省略できる。

For example, if an application uses only a few pages starting at address zero,
then the entries 1 through 511 of the top-level page directory are invalid,
and the kernel doesn’t have to allocate pages those for 511 intermediate page directories.

例えば、もしアプリケーションがアドレスゼロから始まる少数のページしか使用しないならば、
(最初の0以外の) 1 から 511 のトップレベルのページディレクトリは無効 (invalid) となり、
カーネルは 511 個の中間ページディレクトリのためのページを確保しなくて済む。

Furthermore, the kernel also doesn’t have to allocate pages for
the bottom-level page directories for those 511 intermediate page directories.

さらに、カーネルは 511 個の中間ページディレクトリのための一番下のレベルのページも
確保する必要がない。

So, in this example, the three-level design saves 511 pages for
intermediate page directories and 511 × 512 pages for bottom-level page directories.

したがって、この例では、この3段設計で 511 ページの中間ページディレクトリと
511 * 512 ページの一番下のレベルのページディレクトリを節約できている。

Although a CPU walks the three-level structure in hardware
as part of executing a load or store instruction,
a potential downside of three levels is that
the CPU must load three PTEs from memory to perform the translation of the virtual address
in the load/store instruction to a physical address.

CPU はロードやストア命令の実行の一部としてハードウェアで3段構造をウォークするのだが、
3段の方法自体の欠点としては、ロード/ストア命令の仮想アドレスを物理アドレスに変換するために
CPU が3つの PTE をメモリからロードしなければならないことだ。

To avoid the cost of loading PTEs from physical memory,
a RISC-V CPU caches page table entries in a Translation Look-aside Buffer (TLB).

物理メモリから PTE をロードするコストを回避するために、
RISC-V CPU はページテーブルエントリを Translation Look-aside Buffer (TLB) にキャッシュする。

Each PTE contains flag bits that tell the paging hardware
how the associated virtual address is allowed to be used.

それぞれの PTE には、関連付けられた仮想アドレスが使えるかどうかを
ページングハードウェアに教えるフラグビットが含まれている。

PTE_V indicates whether the PTE is present:
if it is not set, a reference to the page causes an exception (i.e., is not allowed).

PTE_V は PTE が存在するかどうかを示す。
このビットがセットされていなかったら、このページへの参照は例外を発生させる
(つまり、この仮想アドレスへの参照は許可されない)。

PTE_R controls whether instructions are allowed to read to the page.

PTE_R は命令がこのページを読めるかどうかを制御する。

PTE_W controls whether instructions are allowed to write to the page.

PTE_W は命令がこのページに書けるかどうかを制御する。

PTE_X controls whether the CPU may interpret the content of the page as instructions and execute them.

PTE_X は CPU がこのページの内容を命令として解釈して実行できるかどうかを制御する。

PTE_U controls whether instructions in user mode are allowed to access the page;
if PTE_U is not set, the PTE can be used only in supervisor mode.

PTE_U はユーザモードの命令がこのページにアクセスできるかを制御する。
もし PTE_U がセットされていなかったら、この PTE はスーパーバイザモードでしか使用できない。

Figure 3.2 shows how it all works.

図 3.2 に全体の動きを示す。

The flags and all other page hardware-related structures are defined in (kernel/riscv.h)

フラグと他のページングハードウェア関連の構造体は `kernel/riscv.h` で定義されている。

To tell a CPU to use a page table, the kernel must write the physical address of
the root page table page into the satp register.

CPU にページテーブルを使うよう指示するには、カーネルはルートページテーブルのページの
物理アドレスを satp レジスタに設定しなければならない。

A CPU will translate all addresses generated by subsequent instructions
using the page table pointed to by its own satp.

CPU は後続の命令によって生成されたすべてのアドレスを、自身の satp に指された
ページテーブルを使って変換する。

Each CPU has its own satp so that different CPUs can run different processes,
each with a private address space described by its own page table.

異なる CPU が異なるプロセスを実行できるよう、それぞれの CPU は独自の satp を持っており、
それぞれが独自のページテーブルによって記述されるプライベートなアドレス空間で実行される。

Typically a kernel maps all of physical memory into its page table so that
it can read and write any location in physical memory using load/store instructions.

通常、あらゆる箇所の物理メモリをロード/ストア命令で読み書きできるようにするため、
カーネルは物理メモリのすべてを自身のページテーブルにマップする。

Since the page directories are in physical memory, the kernel can program
the content of a PTE in a page directory by writing to the virtual address of the PTE
using a standard store instruction.

ページディレクトリは物理メモリ内にあるため、カーネルはページディレクトリ内の PTE の内容を
PTE の仮想アドレスへ標準的なストア命令を使って書き込むことで変更できる。

A few notes about terms.

用語について少し注意しておく。

Physical memory refers to storage cells in DRAM.

物理メモリとは DRAM 中のストレージセルのことを指す。

A byte of physical memory has an address, called a physical address.

物理メモリの各バイトにはアドレスが振られており、物理アドレスと呼ばれる。

Instructions use only virtual addresses,
which the paging hardware translates to physical addresses,
and then sends to the DRAM hardware to read or write storage.

命令からは仮想アドレスのみが使える。
仮想アドレスはページングハードウェアが物理アドレスに変換する。
その後、ストレージを読み書きするために DRAM ハードウェアに送られる。

Unlike physical memory and virtual addresses, virtual memory isn’t a physical object,
but refers to the collection of abstractions and mechanisms the kernel provides
to manage physical memory and virtual addresses.

物理メモリや仮想アドレスと違って、仮想メモリは物理的なオブジェクトではなく、
物理メモリと仮想アドレスを管理するためにカーネルが提供する
抽象化とメカニズムを集めたもののことを指す。

## Kernel address space

Xv6 maintains one page table per process, describing each process’s user address space,
plus a single page table that describes the kernel’s address space.

xv6 はプロセスごとに1つのページテーブルを管理し、
ページテーブルはそれぞれのプロセスのユーザアドレス空間を表し、
さらにカーネルのアドレス空間を表す1つのページテーブルを管理する。

The kernel configures the layout of its address space to give itself access to
physical memory and various hardware resources at predictable virtual addresses.

カーネルは自分のアドレス空間のレイアウトを、自分が物理メモリと様々なハードウェアリソースに
予測可能な仮想アドレスでアクセスできるように構成する。

Figure 3.3 shows how this layout maps kernel virtual addresses to physical addresses.

図 3.3 にこのレイアウトがどのようにカーネルの仮想アドレスを物理アドレスにマップするかを示す。

The file (kernel/memlayout.h) declares the constants for xv6’s kernel memory layout.

ファイル `kernel/memlayout.h` は xv6 のカーネルメモリレイアウトのための定数を宣言している。

QEMU simulates a computer that includes RAM (physical memory) starting at physical ad-
dress 0x80000000 and continuing through at least 0x88000000, which xv6 calls PHYSTOP.
The QEMU simulation also includes I/O devices such as a disk interface. QEMU exposes the de-
vice interfaces to software as memory-mapped control registers that sit below 0x80000000 in the
physical address space. The kernel can interact with the devices by reading/writing these special
physical addresses; such reads and writes communicate with the device hardware rather than with
RAM. Chapter 4 explains how xv6 interacts with devices.
The kernel gets at RAM and memory-mapped device registers using “direct mapping;” that
is, mapping the resources at virtual addresses that are equal to the physical address. For example,
the kernel itself is located at KERNBASE=0x80000000 in both the virtual address space and in
physical memory. Direct mapping simplifies kernel code that reads or writes physical memory. For
example, when fork allocates user memory for the child process, the allocator returns the physical
address of that memory; fork uses that address directly as a virtual address when it is copying the
parent’s user memory to the child.
There are a couple of kernel virtual addresses that aren’t direct-mapped:
• The trampoline page. It is mapped at the top of the virtual address space; user page tables
have this same mapping. Chapter 4 discusses the role of the trampoline page, but we see
here an interesting use case of page tables; a physical page (holding the trampoline code) is
mapped twice in the virtual address space of the kernel: once at top of the virtual address
space and once with a direct mapping.
• The kernel stack pages. Each process has its own kernel stack, which is mapped high so
that below it xv6 can leave an unmapped guard page. The guard page’s PTE is invalid (i.e.,
PTE_V is not set), so that if the kernel overflows a kernel stack, it will likely cause an excep-
tion and the kernel will panic. Without a guard page an overflowing stack would overwrite
other kernel memory, resulting in incorrect operation. A panic crash is preferable.
While the kernel uses its stacks via the high-memory mappings, they are also accessible to the
kernel through a direct-mapped address. An alternate design might have just the direct mapping,
and use the stacks at the direct-mapped address. In that arrangement, however, providing guard
pages would involve unmapping virtual addresses that would otherwise refer to physical memory,
which would then be hard to use.
The kernel maps the pages for the trampoline and the kernel text with the permissions PTE_R
and PTE_X. The kernel reads and executes instructions from these pages. The kernel maps the other
pages with the permissions PTE_R and PTE_W, so that it can read and write the memory in those
pages. The mappings for the guard pages are invalid.

3.3 Code: creating an address space
Most of the xv6 code for manipulating address spaces and page tables resides in vm.c (ker-
nel/vm.c:1). The central data structure is pagetable_t, which is really a pointer to a RISC-V
35
root page-table page; a pagetable_t may be either the kernel page table, or one of the per-
process page tables. The central functions are walk, which finds the PTE for a virtual address,
and mappages, which installs PTEs for new mappings. Functions starting with kvm manipulate
the kernel page table; functions starting with uvm manipulate a user page table; other functions are
used for both. copyout and copyin copy data to and from user virtual addresses provided as
system call arguments; they are in vm.c because they need to explicitly translate those addresses
in order to find the corresponding physical memory.
Early in the boot sequence, main calls kvminit (kernel/vm.c:54) to create the kernel’s page ta-
ble using kvmmake (kernel/vm.c:20). This call occurs before xv6 has enabled paging on the RISC-V,
so addresses refer directly to physical memory. kvmmake first allocates a page of physical mem-
ory to hold the root page-table page. Then it calls kvmmap to install the translations that the kernel
needs. The translations include the kernel’s instructions and data, physical memory up to PHYSTOP,
and memory ranges which are actually devices. proc_mapstacks (kernel/proc.c:33) allocates a
kernel stack for each process. It calls kvmmap to map each stack at the virtual address generated
by KSTACK, which leaves room for the invalid stack-guard pages.
kvmmap (kernel/vm.c:132) calls mappages (kernel/vm.c:143), which installs mappings into a
page table for a range of virtual addresses to a corresponding range of physical addresses. It does
this separately for each virtual address in the range, at page intervals. For each virtual address to
be mapped, mappages calls walk to find the address of the PTE for that address. It then initializes
the PTE to hold the relevant physical page number, the desired permissions (PTE_W, PTE_X, and/or
PTE_R), and PTE_V to mark the PTE as valid (kernel/vm.c:158).
walk (kernel/vm.c:86) mimics the RISC-V paging hardware as it looks up the PTE for a virtual
address (see Figure 3.2). walk descends the 3-level page table 9 bits at the time. It uses each
level’s 9 bits of virtual address to find the PTE of either the next-level page table or the final page
(kernel/vm.c:92). If the PTE isn’t valid, then the required page hasn’t yet been allocated; if the
alloc argument is set, walk allocates a new page-table page and puts its physical address in the
PTE. It returns the address of the PTE in the lowest layer in the tree (kernel/vm.c:102).
The above code depends on physical memory being direct-mapped into the kernel virtual ad-
dress space. For example, as walk descends levels of the page table, it pulls the (physical) address
of the next-level-down page table from a PTE (kernel/vm.c:94), and then uses that address as a
virtual address to fetch the PTE at the next level down (kernel/vm.c:92).
main calls kvminithart (kernel/vm.c:62) to install the kernel page table. It writes the phys-
ical address of the root page-table page into the register satp. After this the CPU will translate
addresses using the kernel page table. Since the kernel uses an identity mapping, the now virtual
address of the next instruction will map to the right physical memory address.
Each RISC-V CPU caches page table entries in a Translation Look-aside Buffer (TLB), and
when xv6 changes a page table, it must tell the CPU to invalidate corresponding cached TLB
entries. If it didn’t, then at some point later the TLB might use an old cached mapping, point-
ing to a physical page that in the meantime has been allocated to another process, and as a re-
sult, a process might be able to scribble on some other process’s memory. The RISC-V has an
instruction sfence.vma that flushes the current CPU’s TLB. Xv6 executes sfence.vma in
kvminithart after reloading the satp register, and in the trampoline code that switches to a
36
user page table before returning to user space (kernel/trampoline.S:89).
It is also necessary to issue sfence.vma before changing satp, in order to wait for comple-
tion of all outstanding loads and stores. This wait ensures that preceding updates to the page table
have completed, and ensures that preceding loads and stores use the old page table, not the new
one.
To avoid flushing the complete TLB, RISC-V CPUs may support address space identifiers
(ASIDs) [3]. The kernel can then flush just the TLB entries for a particular address space. Xv6
does not use this feature.
3.4 Physical memory allocation
The kernel must allocate and free physical memory at run-time for page tables, user memory,
kernel stacks, and pipe buffers.
Xv6 uses the physical memory between the end of the kernel and PHYSTOP for run-time alloca-
tion. It allocates and frees whole 4096-byte pages at a time. It keeps track of which pages are free
by threading a linked list through the pages themselves. Allocation consists of removing a page
from the linked list; freeing consists of adding the freed page to the list.
3.5 Code: Physical memory allocator
The allocator resides in kalloc.c (kernel/kalloc.c:1). The allocator’s data structure is a free list
of physical memory pages that are available for allocation. Each free page’s list element is a
struct run (kernel/kalloc.c:17). Where does the allocator get the memory to hold that data struc-
ture? It store each free page’s run structure in the free page itself, since there’s nothing else stored
there. The free list is protected by a spin lock (kernel/kalloc.c:21-24). The list and the lock are
wrapped in a struct to make clear that the lock protects the fields in the struct. For now, ignore the
lock and the calls to acquire and release; Chapter 6 will examine locking in detail.
The function main calls kinit to initialize the allocator (kernel/kalloc.c:27). kinit initializes
the free list to hold every page between the end of the kernel and PHYSTOP. Xv6 ought to de-
termine how much physical memory is available by parsing configuration information provided
by the hardware. Instead xv6 assumes that the machine has 128 megabytes of RAM. kinit calls
freerange to add memory to the free list via per-page calls to kfree. A PTE can only refer to
a physical address that is aligned on a 4096-byte boundary (is a multiple of 4096), so freerange
uses PGROUNDUP to ensure that it frees only aligned physical addresses. The allocator starts with
no memory; these calls to kfree give it some to manage.
The allocator sometimes treats addresses as integers in order to perform arithmetic on them
(e.g., traversing all pages in freerange), and sometimes uses addresses as pointers to read and
write memory (e.g., manipulating the run structure stored in each page); this dual use of addresses
is the main reason that the allocator code is full of C type casts. The other reason is that freeing
and allocation inherently change the type of the memory.
37
The function kfree (kernel/kalloc.c:47) begins by setting every byte in the memory being freed
to the value 1. This will cause code that uses memory after freeing it (uses “dangling references”)
to read garbage instead of the old valid contents; hopefully that will cause such code to break faster.
Then kfree prepends the page to the free list: it casts pa to a pointer to struct run, records the
old start of the free list in r->next, and sets the free list equal to r. kalloc removes and returns
the first element in the free list.
3.6 Process address space
Each process has a separate page table, and when xv6 switches between processes, it also changes
page tables. Figure 3.4 shows a process’s address space in more detail than Figure 2.3. A process’s
user memory starts at virtual address zero and can grow up to MAXVA (kernel/riscv.h:360), allowing
a process to address in principle 256 Gigabytes of memory.
A process’s address space consists of pages that contain the text of the program (which xv6
maps with the permissions PTE_R, PTE_X, and PTE_U), pages that contain the pre-initialized data
of the program, a page for the stack, and pages for the heap. Xv6 maps the data, stack, and heap
with the permissions PTE_R, PTE_W, and PTE_U.
Using permissions within a user address space is a common technique to harden a user process.
If the text were mapped with PTE_W, then a process could accidentally modify its own program;
for example, a programming error may cause the program to write to a null pointer, modifying
instructions at address 0, and then continue running, perhaps creating more havoc. To detect such
errors immediately, xv6 maps the text without PTE_W; if a program accidentally attempts to store
to address 0, the hardware will refuse to execute the store and raises a page fault (see Section 4.6).
The kernel then kills the process and prints out an informative message so that the developer can
track down the problem.
Similarly, by mapping data without PTE_X, a user program cannot accidentally jump to an
address in the program’s data and start executing at that address.
In the real world, hardening a process by setting permissions carefully also aids in defending
against security attacks. An adversary may feed carefully-constructed input to a program (e.g., a
Web server) that triggers a bug in the program in the hope of turning that bug into an exploit [14].
Setting permissions carefully and other techniques, such as randomizing of the layout of the user
address space, make such attacks harder.
The stack is a single page, and is shown with the initial contents as created by exec. Strings
containing the command-line arguments, as well as an array of pointers to them, are at the very
top of the stack. Just under that are values that allow a program to start at main as if the function
main(argc, argv) had just been called.
To detect a user stack overflowing the allocated stack memory, xv6 places an inaccessible guard
page right below the stack by clearing the PTE_U flag. If the user stack overflows and the process
tries to use an address below the stack, the hardware will generate a page-fault exception because
the guard page is inaccessible to a program running in user mode. A real-world operating system
might instead automatically allocate more memory for the user stack when it overflows.
When a process asks xv6 for more user memory, xv6 grows the process’s heap. Xv6 first uses
kalloc to allocate physical pages. It then adds PTEs to the process’s page table that point to the
new physical pages. Xv6 sets the PTE_W, PTE_R, PTE_U, and PTE_V flags in these PTEs. Most
processes do not use the entire user address space; xv6 leaves PTE_V clear in unused PTEs.
We see here a few nice examples of use of page tables. First, different processes’ page tables
translate user addresses to different pages of physical memory, so that each process has private user
memory. Second, each process sees its memory as having contiguous virtual addresses starting at
zero, while the process’s physical memory can be non-contiguous. Third, the kernel maps a page
with trampoline code at the top of the user address space (without PTE_U), thus a single page of
physical memory shows up in all address spaces, but can be used only by the kernel.
3.7 Code: sbrk
sbrk is the system call for a process to shrink or grow its memory. The system call is implemented
by the function growproc (kernel/proc.c:260). growproc calls uvmalloc or uvmdealloc, de-
pending on whether n is positive or negative. uvmalloc (kernel/vm.c:226) allocates physical mem-
ory with kalloc, and adds PTEs to the user page table with mappages. uvmdealloc calls
uvmunmap (kernel/vm.c:171), which uses walk to find PTEs and kfree to free the physical
memory they refer to.
Xv6 uses a process’s page table not just to tell the hardware how to map user virtual addresses,
but also as the only record of which physical memory pages are allocated to that process. That is
the reason why freeing user memory (in uvmunmap) requires examination of the user page table.
39
3.8 Code: exec
exec is a system call that replaces a process’s user address space with data read from a file, called
a binary or executable file. A binary is typically the output of the compiler and linker, and holds
machine instructions and program data. exec (kernel/exec.c:23) opens the named binary path using
namei (kernel/exec.c:36), which is explained in Chapter 8. Then, it reads the ELF header. Xv6
binaries are formatted in the widely-used ELF format, defined in (kernel/elf.h). An ELF binary
consists of an ELF header, struct elfhdr (kernel/elf.h:6), followed by a sequence of program
section headers, struct proghdr (kernel/elf.h:25). Each progvhdr describes a section of the
application that must be loaded into memory; xv6 programs have two program section headers:
one for instructions and one for data.
The first step is a quick check that the file probably contains an ELF binary. An ELF binary
starts with the four-byte “magic number” 0x7F, ‘E’, ‘L’, ‘F’, or ELF_MAGIC (kernel/elf.h:3). If
the ELF header has the right magic number, exec assumes that the binary is well-formed.
exec allocates a new page table with no user mappings with proc_pagetable (kernel/exec.c:49),
allocates memory for each ELF segment with uvmalloc (kernel/exec.c:65), and loads each segment
into memory with loadseg (kernel/exec.c:10). loadseg uses walkaddr to find the physical ad-
dress of the allocated memory at which to write each page of the ELF segment, and readi to read
from the file.
The program section header for /init, the first user program created with exec, looks like
this:

```text
# objdump -p user/_init
user/_init: file format elf64-little
Program Header:
0x70000003 off 0x0000000000006bb0 vaddr 0x0000000000000000
paddr 0x0000000000000000 align 2**0
filesz 0x000000000000004a memsz 0x0000000000000000 flags r--
LOAD off 0x0000000000001000 vaddr 0x0000000000000000
paddr 0x0000000000000000 align 2**12
filesz 0x0000000000001000 memsz 0x0000000000001000 flags r-x
LOAD off 0x0000000000002000 vaddr 0x0000000000001000
paddr 0x0000000000001000 align 2**12
filesz 0x0000000000000010 memsz 0x0000000000000030 flags rw-
STACK off 0x0000000000000000 vaddr 0x0000000000000000
paddr 0x0000000000000000 align 2**4
filesz 0x0000000000000000 memsz 0x0000000000000000 flags rw-
```

We see that the text segment should be loaded at virtual address 0x0 in memory (without write
permissions) from content at offset 0x1000 in the file. We also see that the data should be loaded
at address 0x1000, which is at a page boundary, and without executable permissions.
A program section header’s filesz may be less than the memsz, indicating that the gap be-
tween them should be filled with zeroes (for C global variables) rather than read from the file. For
/init, the data filesz is 0x10 bytes and memsz is 0x30 bytes, and thus uvmalloc allocates
enough physical memory to hold 0x30 bytes, but reads only 0x10 bytes from the file /init.
40
Now exec allocates and initializes the user stack. It allocates just one stack page. exec copies
the argument strings to the top of the stack one at a time, recording the pointers to them in ustack.
It places a null pointer at the end of what will be the argv list passed to main. The first three entries
in ustack are the fake return program counter, argc, and argv pointer.
exec places an inaccessible page just below the stack page, so that programs that try to use
more than one page will fault. This inaccessible page also allows exec to deal with arguments
that are too large; in that situation, the copyout (kernel/vm.c:352) function that exec uses to copy
arguments to the stack will notice that the destination page is not accessible, and will return -1.
During the preparation of the new memory image, if exec detects an error like an invalid
program segment, it jumps to the label bad, frees the new image, and returns -1. exec must wait
to free the old image until it is sure that the system call will succeed: if the old image is gone, the
system call cannot return -1 to it. The only error cases in exec happen during the creation of the
image. Once the image is complete, exec can commit to the new page table (kernel/exec.c:125) and
free the old one (kernel/exec.c:129).
exec loads bytes from the ELF file into memory at addresses specified by the ELF file. Users
or processes can place whatever addresses they want into an ELF file. Thus exec is risky, because
the addresses in the ELF file may refer to the kernel, accidentally or on purpose. The consequences
for an unwary kernel could range from a crash to a malicious subversion of the kernel’s isolation
mechanisms (i.e., a security exploit). Xv6 performs a number of checks to avoid these risks. For
example if(ph.vaddr + ph.memsz < ph.vaddr) checks for whether the sum overflows a
64-bit integer. The danger is that a user could construct an ELF binary with a ph.vaddr that
points to a user-chosen address, and ph.memsz large enough that the sum overflows to 0x1000,
which will look like a valid value. In an older version of xv6 in which the user address space also
contained the kernel (but not readable/writable in user mode), the user could choose an address that
corresponded to kernel memory and would thus copy data from the ELF binary into the kernel. In
the RISC-V version of xv6 this cannot happen, because the kernel has its own separate page table;
loadseg loads into the process’s page table, not in the kernel’s page table.
It is easy for a kernel developer to omit a crucial check, and real-world kernels have a long
history of missing checks whose absence can be exploited by user programs to obtain kernel priv-
ileges. It is likely that xv6 doesn’t do a complete job of validating user-level data supplied to the
kernel, which a malicious user program might be able to exploit to circumvent xv6’s isolation.
3.9 Real world
Like most operating systems, xv6 uses the paging hardware for memory protection and mapping.
Most operating systems make far more sophisticated use of paging than xv6 by combining paging
and page-fault exceptions, which we will discuss in Chapter 4.
Xv6 is simplified by the kernel’s use of a direct map between virtual and physical addresses, and
by its assumption that there is physical RAM at address 0x8000000, where the kernel expects to be
loaded. This works with QEMU, but on real hardware it turns out to be a bad idea; real hardware
places RAM and devices at unpredictable physical addresses, so that (for example) there might
be no RAM at 0x8000000, where xv6 expect to be able to store the kernel. More serious kernel
41
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

## Exercises

* Parse RISC-V’s device tree to find the amount of physical memory the computer has.
* RISC-V のデバイスツリーをパースし、コンピュータが持つ物理メモリの量を発見せよ。
* Write a user program that grows its address space by one byte by calling sbrk(1). Run
the program and investigate the page table for the program before the call to sbrk and after
the call to sbrk. How much space has the kernel allocated? What does the PTE for the new
memory contain?
1. Modify xv6 to use super pages for the kernel.
1. Unix implementations of exec traditionally include special handling for shell scripts. If the
file to execute begins with the text #!, then the first line is taken to be a program to run to
interpret the file. For example, if exec is called to run myprog arg1 and myprog ’s first
line is #!/interp, then exec runs /interp with command line /interp myprog arg1.
Implement support for this convention in xv6.
1. Implement address space layout randomization for the kernel.
