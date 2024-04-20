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
