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

QEMU simulates a computer that includes RAM (physical memory)
starting at physical address 0x80000000 and continuing through at least 0x88000000,
which xv6 calls PHYSTOP.

QEMU は物理アドレス 0x8000_0000 から始まり少なくとも 0x8800_0000 まで続く RAM (物理メモリ) を
持つコンピュータをシミュレートする。
xv6 は 0x8800_0000 を PHYSTOP と呼ぶ。

The QEMU simulation also includes I/O devices such as a disk interface.

QEMU のシミュレーションにはディスクインタフェースのような I/O デバイスも含まれる。

QEMU exposes the device interfaces to software as memory-mapped control registers
that sit below 0x80000000 in the physical address space.

QEMU は 物理アドレス空間 0x8000_0000 以下にあるメモリマップされたコントロールレジスタとして、
デバイスインタフェースをソフトウェアに公開している。

The kernel can interact with the devices by reading/writing these special physical addresses;
such reads and writes communicate with the device hardware rather than with RAM.

カーネルはこれらの特殊な物理アドレスを読み書きすることでデバイスとやり取りすることができる。
そのような読み書きは RAM とではなくデバイスハードウェアとの通信となる。

Chapter 4 explains how xv6 interacts with devices.

4章で xv6 がどのようにしてデバイスとやり取りするのかを説明する。

The kernel gets at RAM and memory-mapped device registers using “direct mapping;”
that is, mapping the resources at virtual addresses that are equal to the physical address.

カーネルは「ダイレクトマッピング」を使って RAM やメモリマップトデバイスにアクセスする。
つまり、物理アドレスと等しい仮想アドレスにリソースをマップするということである。

For example, the kernel itself is located at KERNBASE=0x80000000
in both the virtual address space and in physical memory.

例えば、カーネル自身は物理メモリと仮想アドレスの両方において KERNBASE=0x8000_0000 に位置する。

Direct mapping simplifies kernel code that reads or writes physical memory.

ダイレクトマッピングは物理メモリを読み書きするカーネルコードを簡潔にできる。

For example, when fork allocates user memory for the child process,
the allocator returns the physical address of that memory;
fork uses that address directly as a virtual address when it is copying the parent’s user memory to the child.

例えば、fork が子プロセスのためのユーザメモリを確保する時、
アロケータはそのメモリの物理アドレスを返す。
fork は親のユーザメモリを子にコピーする際、アロケータの返した物理アドレスをそのまま
仮想アドレスとして使う。

There are a couple of kernel virtual addresses that aren’t direct-mapped:

ダイレクトマップされないカーネル仮想アドレスが2つある。

• The trampoline page. It is mapped at the top of the virtual address space;
user page tables have this same mapping.
Chapter 4 discusses the role of the trampoline page,
but we see here an interesting use case of page tables;
a physical page (holding the trampoline code) is mapped twice
in the virtual address space of the kernel:
once at top of the virtual address space and once with a direct mapping.
• The kernel stack pages.
Each process has its own kernel stack, which is mapped high
so that below it xv6 can leave an unmapped guard page.
The guard page’s PTE is invalid (i.e., PTE_V is not set),
so that if the kernel overflows a kernel stack,
it will likely cause an exception and the kernel will panic.
Without a guard page an overflowing stack would overwrite other kernel memory,
resulting in incorrect operation.
A panic crash is preferable.

* トランポリンページ。これは仮想アドレス空間の先頭にマップされる。
ユーザページテーブルもこれと同じマッピングを持つ。
4章でトランポリンページの役割を議論するが、ここではページテーブルの興味深い使い方を見ていく。
1つの物理ページ (トランポリンコードを含む) はカーネルの仮想アドレス空間に二度マップされる。
1回は仮想アドレス空間の先頭に、もう1回はダイレクトマップされる。
* カーネルスタックページ。
それぞれのプロセスは自身専用のカーネルスタックを持ち、上位アドレスにマップされるため、
xv6 は 下位アドレスをアンマップ状態のガードページのままにできる。
ガードページの PTE は無効 (つまり PTE_V がセットされない) であり、
これによりカーネルがカーネルスタックをオーバーフローさせた時、
例外が発生し、カーネルはパニックする可能性が高い。
ガードページなしでオーバーフローすると他のカーネルメモリを上書きすることになり、
不正な動作を引き起こす。
パニックのクラッシュが望ましい。

While the kernel uses its stacks via the high-memory mappings,
they are also accessible to the kernel through a direct-mapped address.

カーネルは自分のスタックを上位メモリへのマッピング経由で使用する。
これはダイレクトマップされたアドレス経由でもアクセス可能である。

An alternate design might have just the direct mapping,
and use the stacks at the direct-mapped address.

他の設計としてはダイレクトマップのみとし、ダイレクトマップされたアドレスで
スタックを使うというのもあり得るかもしれない。

In that arrangement, however, providing guard pages would involve unmapping
virtual addresses that would otherwise refer to physical memory,
which would then be hard to use.

しかしそのやり方だと、ガードページを提供するために物理メモリを参照していた仮想アドレスを
アンマップすることになり、その部分の物理メモリを使いづらくなってしまう。

The kernel maps the pages for the trampoline and the kernel text
with the permissions PTE_R and PTE_X.

カーネルはトランポリンとカーネルテキストのためのページを
パーミッション PTE_R と PTE_X でマップする。

The kernel reads and executes instructions from these pages.

カーネルはそれらのページを読んだり実行したりできる。

The kernel maps the other pages with the permissions PTE_R and PTE_W,
so that it can read and write the memory in those pages.

カーネルは他のページをパーミッション PTE_R と PTE_W でマップする。
これによりそれらのページのメモリは読み書きできる。

The mappings for the guard pages are invalid.

ガードページのマッピングは無効である。

## Code: creating an address space

コード: アドレス空間の生成

Most of the xv6 code for manipulating address spaces and page tables resides in
vm.c (kernel/vm.c:1).

アドレス空間とページテーブルの操作をする xv6 のコードの大部分は vm.c (`kernel/vm.c:1`) にある。

The central data structure is pagetable_t, which is really a pointer to
a RISC-V root page-table page;
a pagetable_t may be either the kernel page table, or one of the per-process page tables.

中心的なデータ構造は pagetable_t で、これは RISC-V ルートページテーブルページ
そのものへのポインタである。
pagetable_t はカーネルのページテーブルの場合もあれば、
プロセスごとのページテーブルのうちの1つの可能性もある。

The central functions are walk, which finds the PTE for a virtual address,
and mappages, which installs PTEs for new mappings.

中心的な関数は walk、これは仮想アドレスに対して PTE を求めるもので、
それと mappages、これは PTE を新しいマッピングにインストールするものである。

Functions starting with kvm manipulate the kernel page table;
functions starting with uvm manipulate a user page table;
other functions are used for both.

kvm で始まる関数はカーネルページテーブルを操作し、
uvm で始まる関数はユーザページテーブルを操作する。
他の関数は両方の目的で使用される。

copyout and copyin copy data to and from user virtual addresses provided as
system call arguments;
they are in vm.c because they need to explicitly translate those addresses
in order to find the corresponding physical memory.

copyout と copyin は、システムコールの引数として提供されたユーザ仮想アドレスに、
またはユーザ仮想アドレスから、データをコピーする。
それらは仮想アドレスを対応する物理アドレスに明示的に変換する必要があるため、
vm.c の中にある。

Early in the boot sequence, main calls kvminit (kernel/vm.c:54) to create
the kernel’s page table using kvmmake (kernel/vm.c:20).

ブートシーケンスの早い段階で、main は kvminit (`kernel/vm.c:54`) を、
カーネルのページテーブルを kvmmake (`kernel/vm.c:20`) を使って作成するために呼ぶ。

This call occurs before xv6 has enabled paging on the RISC-V,
so addresses refer directly to physical memory.

この呼び出しは cv6 が RISC-V のページング機能を有効にする前に行われる。
このためアドレスは直接物理メモリを指している。

kvmmake first allocates a page of physical memory to hold the root page-table page.

kvmmake は初めに物理メモリの1ページをルートページテーブルページを格納するために確保する。

Then it calls kvmmap to install the translations that the kernel needs.

その後 kvmmap を呼んでカーネルが必要とする変換をインストールする。

The translations include the kernel’s instructions and data, physical memory up to PHYSTOP,
and memory ranges which are actually devices.

この変換にはカーネルの命令とデータ、PHYSTOP までの物理メモリ、実デバイスのメモリ範囲が
含まれる。

proc_mapstacks (kernel/proc.c:33) allocates a kernel stack for each process.

proc_mapstacks (`kernel/proc.c:33`) はプロセスごとのカーネルスタックを確保する。

It calls kvmmap to map each stack at the virtual address generated by KSTACK,
which leaves room for the invalid stack-guard pages.

proc_mapstacks は kvmmap を呼んで KSTACK によって生成された仮想アドレスに
それぞれのスタックをマップする。
そこには無効なスタックガードページのための空きも残してある。

kvmmap (kernel/vm.c:132) calls mappages (kernel/vm.c:143),
which installs mappings into a page table for a range of virtual addresses
to a corresponding range of physical addresses.

kvmmap (`kernel/vm.c:132`) は mappages (`kernel/vm.c:143`) を呼ぶ。
mappages はページテーブルに、ある範囲の仮想アドレスから対応する範囲の物理アドレスへの
マップをインストールする。

It does this separately for each virtual address in the range, at page intervals.

これはページ単位で、その範囲内のそれぞれの仮想アドレスに対して行われる。

For each virtual address to be mapped,
mappages calls walk to find the address of the PTE for that address.

それぞれのこれからマップされる仮想アドレスについて、
mappages はそのアドレスの PTE のアドレスを求めるために walk を呼ぶ。

It then initializes the PTE to hold the relevant physical page number,
the desired permissions (PTE_W, PTE_X, and/or PTE_R),
and PTE_V to mark the PTE as valid (kernel/vm.c:158).

それから PTE を、関連付ける物理ページ番号、要求されたパーミッション
(PTE_W, PTE_X, and/or PTE_R)、この PTE が有効であるとマークする PTE_V、
を保持するように初期化する(`kernel/vm.c:158`)。

walk (kernel/vm.c:86) mimics the RISC-V paging hardware
as it looks up the PTE for a virtual address (see Figure 3.2).

walk (`kernel/vm.c:86`) は RISC-V ページングハードウェアが
仮想アドレスから PTE をルックアップするのを模倣する(図 3.2 参照)。

walk descends the 3-level page table 9 bits at the time.

walk は3段ページテーブルを一度に 9 bit ずつ降りていく。

It uses each level’s 9 bits of virtual address to find the PTE of
either the next-level page table or the final page (kernel/vm.c:92).

そこではそれぞれのレベルの仮想アドレスの中の 9 bit を次のレベルのページテーブルまたは
最後のページの PTE を求めるために使う。

If the PTE isn’t valid, then the required page hasn’t yet been allocated;
if the alloc argument is set, walk allocates a new page-table page and
puts its physical address in the PTE.

もし PTE が有効でなかったら、要求されたページはまだ確保されていない。
もし alloc 引数がセットされていたら、walk は新しいページテーブルページを確保し、
その物理アドレスを PTE の中に格納する。

It returns the address of the PTE in the lowest layer in the tree (kernel/vm.c:102).

walk はツリーの一番下のレイヤの PTE のアドレスを返す (`kernel/vm.c`)。

The above code depends on physical memory being direct-mapped
into the kernel virtual address space.

ここまでのコードは物理メモリがカーネル仮想アドレス空間にダイレクトマップされていることに
依存している。

For example, as walk descends levels of the page table,
it pulls the (physical) address of the next-level-down page table from a PTE (kernel/vm.c:94),
and then uses that address as a virtual address to fetch the PTE
at the next level down (kernel/vm.c:92).

例えば、walk がページテーブルのレベルを下りていく際、
次の下のレベルの ページテーブルの (物理) アドレスを PTE から引いてくる (`kernel/vm.c:94`)。
そしてそのアドレスを仮想アドレスとして使用し次の下のレベルの PTE を取得する (`kernel/vm.c:92`)。

main calls kvminithart (kernel/vm.c:62) to install the kernel page table.

main は kvminithart (`kernel/vm.c:62`) をカーネルページテーブルを準備するために呼ぶ。

It writes the physical address of the root page-table page into the register satp.

kvminithart はルートページテーブルページの物理アドレスをレジスタ satp に書き込む。

After this the CPU will translate addresses using the kernel page table.

この後、CPU はカーネルページテーブルを使ってアドレスを変換するようになる。

Since the kernel uses an identity mapping,
the now virtual address of the next instruction will map to the right physical memory address.

カーネルは同一マッピングを使っているので、次の命令の仮想アドレスは
正しい物理メモリアドレスにマップされる。

Each RISC-V CPU caches page table entries in a Translation Look-aside Buffer (TLB),
and when xv6 changes a page table, it must tell the CPU to invalidate
corresponding cached TLB entries.

それぞれの RISC-V CPU はページテーブルエントリを Translation Look-aside Buffer (TLB) に
キャッシュする。xv6 がページテーブルを変更した時、対応する TLB エントリを
無効化 (invalidate) するよう CPU に伝えなければならない。

If it didn’t, then at some point later the TLB might use an old cached mapping,
pointing to a physical page that in the meantime has been allocated to another process,
and as a result, a process might be able to scribble on some other process’s memory.

もしそうしなかったなら、その後あるところで TLB が古いキャッシュされたマッピングを使ってしまい、
その古いマッピングの指す物理ページがその間に他のプロセスに確保され、
その結果、あるプロセスが他のプロセスのメモリを書き壊してしまう可能性がある。

The RISC-V has an instruction sfence.vma that flushes the current CPU’s TLB.

RISC-V は現在の CPU の TLB をフラッシュする sfence.vma 命令を持つ。

Xv6 executes sfence.vma in kvminithart after reloading the satp register,
and in the trampoline code that switches to a user page table
before returning to user space (kernel/trampoline.S:89).

xv6 は kvminithart 内で satp を書き換えた後と、
ユーザページテーブルへ切り替えるトランポリンコード内でユーザ空間へ戻る前に
sfence.vma を実行する。

It is also necessary to issue sfence.vma before changing satp,
in order to wait for completion of all outstanding loads and stores.

satp を書き換える前にも、未完了のロードやストアが完了するまで待つために
sfence.vma を発行する必要がある。
(注: satp の設定命令より上にあるメモリアクセス命令がアウトオブオーダ実行によって
satp の変更より前に完了してしまうと、新しい変換テーブルでメモリアクセスが
行われてしまう可能性がある。おそらくどこかで似たような話を後述。)

This wait ensures that preceding updates to the page table have completed,
and ensures that preceding loads and stores use the old page table, not the new one.

この待機命令はこれより前のページテーブルの更新が完了したことを保証し、
これより前のロードやストア命令が新しいものではなく古いページテーブルを使用することを保証する。

To avoid flushing the complete TLB,
RISC-V CPUs may support address space identifiers (ASIDs) [3].

全 TLB のフラッシュを避けるため、RISC-V CPU はアドレス空間 ID (ASID) をサポートする場合がある。

The kernel can then flush just the TLB entries for a particular address space.

カーネルは特定のアドレス空間の TLB エントリのみをフラッシュすることができる。

Xv6 does not use this feature.

xv6 はこの機能を使用していない。

## Physical memory allocation

物理メモリのアロケーション

The kernel must allocate and free physical memory at run-time for page tables,
user memory, kernel stacks, and pipe buffers.

カーネルは実行時にページテーブル、ユーザメモリ、カーネルスタック、パイプバッファのために
物理メモリを確保しなければならない。

Xv6 uses the physical memory between the end of the kernel and PHYSTOP for run-time allocation.

xv6 はカーネルの終わりから PHYSTOP までの間の物理メモリを実行時アロケーションのために用いる。

It allocates and frees whole 4096-byte pages at a time.

xv6 は 4096 byte のページ全体を一度に確保または解放する。

It keeps track of which pages are free by threading a linked list through the pages themselves.

xv6 はどのページがフリーなのかをページ自身でリンクリストをつなぐことによって管理する。

Allocation consists of removing a page from the linked list;
freeing consists of adding the freed page to the list.

アロケーションはリンクリストからページを削除することによって行う。
解放はページをリストに追加することによって行う。

## Code: Physical memory allocator

コード: 物理メモリアロケータ

The allocator resides in kalloc.c (kernel/kalloc.c:1).

アロケータは kalloc.c にある (`kernel/kalloc.c:1`)。

The allocator’s data structure is a free list of physical memory pages that are available for allocation.

アロケータのデータ構造はアロケートに使用可能な物理メモリページのフリーリストである。

Each free page’s list element is a struct run (kernel/kalloc.c:17).

フリーページリストのそれぞれの要素は struct run である (`kernel/kalloc.c:17`)。

Where does the allocator get the memory to hold that data structure?

アロケータはそのデータ構造をメモリのどこに持たせるのだろうか？

It store each free page’s run structure in the free page itself,
since there’s nothing else stored there.

フリーページの struct run 構造体をフリーページ自身に格納する。
なぜならそこには他に何も格納されていないからである。

The free list is protected by a spin lock (kernel/kalloc.c:21-24).

フリーリストはスピンロックで守られている (`kernel/kalloc.c:21-24`)。

The list and the lock are wrapped in a struct to make clear that
the lock protects the fields in the struct.

リストとロックはロックが構造体のフィールドを保護していることを明確にするため、
1つの構造体にまとめられている。

For now, ignore the lock and the calls to acquire and release;
Chapter 6 will examine locking in detail.

今のところは、ロックと acquire と release の呼び出しについては無視する。
6章でロックについて詳しく説明する。

The function main calls kinit to initialize the allocator (kernel/kalloc.c:27).

main 関数は kinit を呼んでアロケータを初期化する (`kernel/kalloc.c:27`)。

kinit initializes the free list to hold every page between the end of the kernel and PHYSTOP.

kinit はフリーリストを初期化しカーネルの終わりから PHYSTOP までの間のすべてのページを
保持するようにする。

Xv6 ought to determine how much physical memory is available
by parsing configuration information provided by the hardware.

xv6 はどのくらいの量の物理メモリが利用可能なのかをハードウェアから提供される
構成情報をパースすることによって判定すべきである。

Instead xv6 assumes that the machine has 128 megabytes of RAM.

代わりに xv6 はマシンが 128 MiB の RAM を持っていると仮定する。

kinit calls freerange to add memory to the free list via per-page calls to kfree.

kinit freerange を呼んで、kfree をページごとに呼ぶことによりメモリをフリーリストに追加する。

A PTE can only refer to a physical address that is aligned on a 4096-byte boundary
(is a multiple of 4096), so freerange uses PGROUNDUP to ensure that
it frees only aligned physical addresses.

PTE は 4096 byte 境界にアラインされた (4096 の倍数である) 物理アドレスとか参照することができない。
よって freerange は PGROUNDUP (注: PaGe Round Up) を使って
アラインされた物理アドレスのみを解放することを保証する。

The allocator starts with no memory; these calls to kfree give it some to manage.

アロケータはメモリなしから開始し、これらの kfree の呼び出しで何らかのメモリを管理し始める。

The allocator sometimes treats addresses as integers in order to perform arithmetic on them
(e.g., traversing all pages in freerange),
and sometimes uses addresses as pointers to read and write memory
(e.g., manipulating the run structure stored in each page);
this dual use of addresses is the main reason that the allocator code is full of C type casts.

アロケータは時々、算術演算を行うためにアドレスを整数として扱うことがある
(例: freetange ですべてのページを辿る)。
そしてまた時々、メモリを読み書きするためにアドレスをポインタとして扱うことがある
(例: 各ページに格納されている struct run 構造体を操作する)。
このアドレスに対する2種類の使用法が、アロケータのコードが C の型キャストだらけになっている
主な理由である。

The other reason is that freeing and allocation inherently change the type of the memory.

もう1つの理由は解放と確保がメモリの型を本質的に変えてしまうからだ。

The function kfree (kernel/kalloc.c:47) begins by setting every byte
in the memory being freed to the value 1.

関数 kfree (`kernel/kalloc.c:47`) は解放しようとするメモリのすべてのバイトを
まず1で埋める。

This will cause code that uses memory after freeing it (uses “dangling references”)
to read garbage instead of the old valid contents;
hopefully that will cause such code to break faster.

これにより解放後のメモリを使った (「ダングリング参照」を使った) コードは
古い有効な値ではなくゴミが読めるようになる。
これによりそのようなコードは早めに壊れてくれると嬉しい。

Then kfree prepends the page to the free list: it casts pa to a pointer to struct run,
records the old start of the free list in r->next, and sets the free list equal to r.

次に kfree はページをフリーリストにつなぐ準備をする。
`pa` を `struct run` へのポインタにキャストし、`r->next` に元のフリーリストの始点を保存し、
フリーリストを r に等しくする。
(注: リンクリストの先頭に要素を追加しているだけである。)

kalloc removes and returns the first element in the free list.

kalloc はフリーリストの先頭の要素を削除して返す。

## Process address space

Each process has a separate page table, and when xv6 switches between processes,
it also changes page tables.

各プロセスは別々のページテーブルを持ち、xv6 がプロセス間を切り替えた時、
ページテーブルも同時に切り替える。

Figure 3.4 shows a process’s address space in more detail than Figure 2.3.

図 3.4 に図 2.3 よりも詳細なプロセスアドレス空間の様子を示す。

A process’s user memory starts at virtual address zero and can grow up to MAXVA
(kernel/riscv.h:360), allowing a process to address in principle 256 Gigabytes of memory.

プロセスのユーザメモリは仮想アドレスのゼロから始まり、MAXVA (`kernel/riscv.h:80`) まで
伸び、プロセスは基本的に 256 GiB までのメモリをアドレス指定できる。

A process’s address space consists of pages that contain the text of the program
(which xv6 maps with the permissions PTE_R, PTE_X, and PTE_U),
pages that contain the pre-initialized data of the program,
a page for the stack, and pages for the heap.

プロセスのアドレス空間はそのプログラムのテキスト領域を含むページ
(xv6 はこれをパーミッション PTE_R, PTE_X, PTE_U でマップする)、
そのプログラムの前もって初期化されたデータ領域を持つページ、
スタックのためのページ、ヒープのためのページからなる。

Xv6 maps the data, stack, and heap with the permissions PTE_R, PTE_W, and PTE_U.

xv6 はデータ、スタック、ヒープをパーミッション PTE_R, PTE_W, PTE_U でマップする。

Using permissions within a user address space is a common technique to harden a user process.

ユーザアドレス空間にパーミッションを使うのは、ユーザプロセスを堅牢化する一般的な技法である。

If the text were mapped with PTE_W,
then a process could accidentally modify its own program;
for example, a programming error may cause the program to write to a null pointer,
modifying instructions at address 0, and then continue running,
perhaps creating more havoc.

もしテキスト領域が PTE_W でマップされていたら、
プロセスは誤って自分自身のプログラムを書き換えてしまう可能性がある。
例えば、プログラミング上の誤りによりプログラムがヌルポインタに書き込んでしまい、
アドレス 0 の命令が変更され、実行が続き、おそらくさらなる大混乱が起こるだろう。

To detect such errors immediately, xv6 maps the text without PTE_W;
if a program accidentally attempts to store to address 0,
the hardware will refuse to execute the store and raises a page fault (see Section 4.6).

そのようなエラーを即座に検出するために、xv6 はテキスト領域を PTE_W なしでマップする。
もしプログラムが誤ってアドレス 0 に書き込もうとしたら、ハードウェアはストアの実行を拒否して
ページフォールトを発生させるだろう (4.6 節参照)。

The kernel then kills the process and prints out an informative message
so that the developer can track down the problem.

カーネルはプロセスを kill し、開発者が問題を見つけられるように情報メッセージを出力する。

Similarly, by mapping data without PTE_X, a user program cannot accidentally jump to
an address in the program’s data and start executing at that address.

同様に、PTE_X なしでマップすることにより、ユーザプログラムは誤ってデータ領域内のアドレスに
ジャンプしそのアドレスから実行を始めるということがない。

In the real world, hardening a process by setting permissions carefully also aids in
defending against security attacks.

現実世界では、プロセスをパーミッションを注意深く設定することによって堅牢にすることには
セキュリティ攻撃に対して防御するという目的もある。

An adversary may feed carefully-constructed input to a program (e.g., a Web server)
that triggers a bug in the program in the hope of turning that bug into an exploit [14].

攻撃者はプログラム中のバグを引き起こすような入念に構築された入力を
プログラム (例: ウェブサーバ) に送り込み、
そのバグがエクスプロイトになるのを狙ってくるかもしれない。

Setting permissions carefully and other techniques,
such as randomizing of the layout of the user address space,
make such attacks harder.

パーミッションを注意深く設定するのと、ユーザアドレス空間のレイアウトのランダマイズのような
その他の技術はそのような攻撃を難しくする。

The stack is a single page, and is shown with the initial contents as created by exec.

スタックは1つのページで、exec によって生成された初期データが入って見える。

Strings containing the command-line arguments, as well as an array of pointers to them,
are at the very top of the stack.

コマンドライン引数を格納している文字列と、それらへのポインタの配列が、
スタックの一番上に置かれる。

Just under that are values that allow a program to start at main
as if the function main(argc, argv) had just been called.

そのすぐ下は、関数 main(argc. argv) がちょうど呼ばれたかのように
プログラムが main から始まるようにするための値が置かれる。

To detect a user stack overflowing the allocated stack memory,
xv6 places an inaccessible guard page right below the stack by clearing the PTE_U flag.

ユーザスタックが、確保されたスタックメモリをオーバーフローしたのを検出するため、
xv6 はア PTE_U フラグをクリアしてアクセス不能にしたガードページをスタックのすぐ下に配置する。

If the user stack overflows and the process tries to use an address below the stack,
the hardware will generate a page-fault exception
because the guard page is inaccessible to a program running in user mode.

もしユーザスタックがオーバーフローしてプロセスがスタックの下のアドレスを使おうとした場合、
ハードウェアはページフォールト例外を生成する。
なぜならガードページはユーザモードで実行されているプログラムからはアクセス不能だからだ。

A real-world operating system might instead automatically allocate more memory
for the user stack when it overflows.

現実世界のオペレーティングシステムはスタックがオーバーフローした時、
代わりに自動的に追加のメモリを確保するかもしれない。

When a process asks xv6 for more user memory, xv6 grows the process’s heap.

プロセスが xv6 にさらなるユーザメモリを要求した時、xv6 はプロセスのヒープを拡張する。

Xv6 first uses kalloc to allocate physical pages.

xv6 は最初に kalloc を使って物理ページを確保する。

It then adds PTEs to the process’s page table that point to the new physical pages.

次に新しい物理ページを指す PTE をそのプロセスのページテーブルに追加する。

Xv6 sets the PTE_W, PTE_R, PTE_U, and PTE_V flags in these PTEs.

xv6 はそれらの PTE 内に PTE_W, PTE_R, PTE_U, PTE_V を設定する。

Most processes do not use the entire user address space;
xv6 leaves PTE_V clear in unused PTEs.

ほとんどのプロセスはユーザアドレス空間全体を使うことはない。
xv6 は使われていない PTE 内の PTE_V をクリア状態のままにする。

We see here a few nice examples of use of page tables.

ここで、いくつかのページテーブルの使い方の良い例を見られる。

First, different processes’ page tables translate user addresses to
different pages of physical memory, so that each process has private user memory.

初めに、異なるプロセスのページテーブルはユーザアドレスを異なる物理メモリのページに変換する。
これにより各プロセスはプライベートなユーザメモリを持つことができる。

Second, each process sees its memory as having contiguous virtual addresses starting at zero,
while the process’s physical memory can be non-contiguous.

第二に、各プロセスからは自分のメモリがゼロから始まる連続した仮想アドレスを持っているように見える。
プロセスの物理メモリは非連続かもしれないながら。

Third, the kernel maps a page with trampoline code at the top of
the user address space (without PTE_U), thus a single page of physical memory
shows up in all address spaces, but can be used only by the kernel.

第三に、カーネルはトランポリンコードが入った1つのページをユーザアドレス空間の先頭に
(PTE_U フラグなしで) マップする。
したがって物理メモリ中の1ページが全てのアドレス空間に現れるが、
カーネルからしか使用できない。

## Code: sbrk

コード: sbrk

sbrk is the system call for a process to shrink or grow its memory.

sbrk はプロセスが自身のメモリを小さくしたり大きくしたりするためのシステムコールである。

The system call is implemented by the function growproc (kernel/proc.c:260).

このシステムコールは growproc 関数 に実装されている (`kernel/proc.c:260`)。

growproc calls uvmalloc or uvmdealloc, depending on whether n is positive or negative.

growproc は uvmalloc または uvmdealloc を、n が正か負かに応じて呼ぶ。

uvmalloc (kernel/vm.c:226) allocates physical memory with kalloc,
and adds PTEs to the user page table with mappages.

uvmalloc (`kernel/vm.c:226`) は kalloc で物理メモリを確保し、
PTE をユーザページテーブルにマッピングと共に追加する。

uvmdealloc calls uvmunmap (kernel/vm.c:171), which uses walk to find PTEs and
kfree to free the physical memory they refer to.

uvmdealloc は uvmunmap (`kernel/vm.c:171`) を呼び、uvmunmap は walk を使って
PTE を求め、kfree を使って参照されていた物理メモリを解放する。

Xv6 uses a process’s page table not just to tell the hardware how to map user virtual addresses,
but also as the only record of which physical memory pages are allocated to that process.

xv6 はプロセスのページテーブルをハードウェアにユーザ仮想アドレスをどのようにマップするかを
教えるだけでなく、どの物理メモリページがそのプロセスのために確保されているかの
唯一の記録としても使う。

That is the reason why freeing user memory (in uvmunmap) requires
examination of the user page table.

これがユーザメモリの解放 (uvmunmap) 内でユーザページテーブルを調べる必要がある理由である。

## Code: exec

コード: exec

exec is a system call that replaces a process’s user address space with data read from a file,
called a binary or executable file.

exec はプロセスのユーザアドレス空間をファイルから読んだデータで置き換えるシステムコールである。
そのファイルはバイナリまたは実行可能ファイルと呼ばれる。

A binary is typically the output of the compiler and linker,
and holds machine instructions and program data.

バイナリは通常コンパイラとリンカの出力であり、機械語命令とプログラムデータが含まれる。

exec (kernel/exec.c:23) opens the named binary path using namei (kernel/exec.c:36),
which is explained in Chapter 8.

exec (`kernel/exec.c:23`) は名前の付いたバイナリのパスを namei (`kernel/exec.c:36`) を
使ってオープンする。
namei については8章で説明する。

Then, it reads the ELF header.

次に、ELF ヘッダを読む。

Xv6 binaries are formatted in the widely-used ELF format, defined in (kernel/elf.h).

xv6 バイナリは広く使われている ELF フォーマットでフォーマットされており、
`kernel/elh.f` で定義されている。

An ELF binary consists of an ELF header, struct elfhdr (kernel/elf.h:6),
followed by a sequence of program section headers, struct proghdr (kernel/elf.h:25).

ELF バイナリは ELF ヘッダ struct elfhdr (`kernel/elf.h:6`)、
続いてプログラムセクションヘッダ struct proghdr (`kernel/elf.h`) の列からなる。

Each progvhdr describes a section of the application that must be loaded into memory;
xv6 programs have two program section headers:
one for instructions and one for data.

それぞれの proghdr はメモリにロードしなければならないアプリケーションのセクションを記述する。
xv6 プログラムは2つのプログラムセクションヘッダを持つ。
1つは命令でもう1つはデータである。

The first step is a quick check that the file probably contains an ELF binary.

An ELF binary starts with the four-byte “magic number” 0x7F, ‘E’, ‘L’, ‘F’,
or ELF_MAGIC (kernel/elf.h:3).

ELF バイナリは4バイトの「マジックナンバー」0x7F, 'E', 'L', 'F'、
つまり ELF_MAGIC (`kernel/elf.h:3`) で始まる。

If the ELF header has the right magic number, exec assumes that the binary is well-formed.

もし ELF ヘッダが正しいマジックナンバーを持っていたら、exe はそのバイナリが
正しい形式であると仮定する。

exec allocates a new page table with no user mappings with proc_pagetable (kernel/exec.c:49),
allocates memory for each ELF segment with uvmalloc (kernel/exec.c:65), and loads each segment
into memory with loadseg (kernel/exec.c:10).

exec はユーザマッピングのない新しいページテーブルを proc_pagetable (`kernel/exec.c:49`) で
確保し、それぞれの ELF セグメントを uvmalloc (`kernel/exec.c:65`) で確保し、
それぞれのセグメントをメモリへ loadseg (`kernel/exec.c:10`) でロードする。

loadseg uses walkaddr to find the physical address of the allocated memory
at which to write each page of the ELF segment, and readi to read from the file.

loadseg は walkaddr を使って ELF セグメントのそれぞれのページを書き込む先である
確保されたメモリの物理アドレスを求める。

The program section header for /init, the first user program created with exec,
looks like this:

/init のプログラムセクションヘッダは次のようになっている。
/init は exec で生成される最初のユーザプログラムである。

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

We see that the text segment should be loaded at virtual address 0x0 in memory
(without write permissions) from content at offset 0x1000 in the file.

テキスト領域はメモリ上の仮想アドレス 0x0 へ (write パーミッションなしで)
ファイル中のオフセット 0x1000 の内容からロードすべきということが分かる。

We also see that the data should be loaded at address 0x1000,
which is at a page boundary, and without executable permissions.

また、データ領域はアドレス 0x1000 へ、実行パーミッションなしでロードすべきと分かる。
0x1000 はページ境界である。

A program section header’s filesz may be less than the memsz,
indicating that the gap between them should be filled with zeroes
(for C global variables) rather than read from the file.

プログラムセクションヘッダの filesz は memsz より小さい可能性がある。
これは両者の差分にあたる部分はファイルから読むのではなく
ゼロで埋めるべき (C のグローバル変数のため) ということを示す。

For /init, the data filesz is 0x10 bytes and memsz is 0x30 bytes,
and thus uvmalloc allocates enough physical memory to hold 0x30 bytes,
but reads only 0x10 bytes from the file /init.

/init に関しては、データの filesz は 0x19 バイトで memsz は 0x30 バイトなので、
uvmalloc は 0x30 バイトを保持するのに十分な物理メモリを確保するが、
ファイル /init からは 0x10 バイトしか読まない。

Now exec allocates and initializes the user stack.

次に exec はユーザスタックを確保し初期化する。

It allocates just one stack page.

スタックを1ページだけ確保する。

exec copies the argument strings to the top of the stack one at a time,
recording the pointers to them in ustack.

exec は引数文字列をスタックトップに1度に1つコピーし、
ustack にそのポインタを記録していく。

It places a null pointer at the end of what will be the argv list passed to main.

main に渡される argv リストとなるものの最後にヌルポインタを置く。

The first three entries in ustack are the fake return program counter,
argc, and argv pointer.

ustack の最初の3つのエントリはフェイクのリターンプログラムカウンタ
argc、argv ポインタとなる。

exec places an inaccessible page just below the stack page,
so that programs that try to use more than one page will fault.

exec はスタックページのすぐ下にアクセス不能なページを配置する。
これにより1ページより多く使おうとするプログラムはフォールトを起こすことになる。

This inaccessible page also allows exec to deal with arguments that are too large;
in that situation, the copyout (kernel/vm.c:352) function that
exec uses to copy arguments to the stack will notice that
the destination page is not accessible, and will return -1.

このアクセス不能なページにより exec は引数が大きすぎる事態にも対処することができる。
そのような状況では、exec が引数をスタックにコピーするのに使うcopyout (`kernel/vm.c:352`) 関数は
コピー先ページがアクセス不能であることに気づき、-1 を返す。

During the preparation of the new memory image,
if exec detects an error like an invalid program segment,
it jumps to the label bad, frees the new image, and returns -1.

exec must wait to free the old image until it is sure that the system call will succeed:
if the old image is gone, the system call cannot return -1 to it.

exec はシステムコールが成功するということが確かになるまで古いイメージを解放するのを
待たなければならない。
もし古いイメージが消えてしまうと、システムコールは -1 を返すことができない。

The only error cases in exec happen during the creation of the image.

exec 中のでエラーが起きるのはイメージの生成中のみだ。

Once the image is complete, exec can commit to the new page table (kernel/exec.c:125)
and free the old one (kernel/exec.c:129).

イメージが完全ならば、exec は新しいページテーブルにコミットでき (`kernel/exec.c:125`)、
古いページテーブルを解放できる (`kernel/exec.c:129`)。

exec loads bytes from the ELF file into memory at addresses specified by the ELF file.

exec は ELF ファイルから ELF ファイルに指定されたアドレスのメモリへバイト列をロードする。

Users or processes can place whatever addresses they want into an ELF file.

ユーザやプロセスは ELF ファイル中に好きなアドレスを書いておくことができる。

Thus exec is risky, because the addresses in the ELF file may refer to the kernel,
accidentally or on purpose.

したがって exec はリスキーである。
ELF ファイル中のアドレスは事故による偶然であれ、意図的であれ、
カーネルを参照している可能性があるからだ。

The consequences for an unwary kernel could range from a crash
to a malicious subversion of the kernel’s isolation mechanisms
(i.e., a security exploit).

不用心なカーネルの結果はクラッシュからカーネルの隔離メカニズムの破壊
(つまり、セキュリティエクスプロイト) まで多岐にわたる。

Xv6 performs a number of checks to avoid these risks.

xv6 はこれらの危険を回避するために数多くのチェックを行っている。

For example if(ph.vaddr + ph.memsz < ph.vaddr) checks for
whether the sum overflows a 64-bit integer.

例えば `if(ph.vaddr + ph.memsz < ph.vaddr)` は和が 64 bit 整数を
オーバーフローしないかをチェックしている。

The danger is that a user could construct an ELF binary with a ph.vaddr
that points to a user-chosen address, and ph.memsz large enough
that the sum overflows to 0x1000, which will look like a valid value.

ここでの危険というのは、ユーザが自分で選んだアドレスを指す ph.vaddr と、
和がオーバーフローして 0x1000 (そしてこれは正しい値に見える。) になるように
大きな ph.memsz を持ったELF バイナリを作れることだ。

In an older version of xv6 in which the user address space also contained the kernel
(but not readable/writable in user mode), the user could choose an address
that corresponded to kernel memory and would thus copy data from the ELF binary into the kernel.

ユーザアドレス空間がカーネルも含んでいた (しかしユーザモードからは読み書きできない)
xv6 の古いバージョンでは、ユーザはカーネルメモリに対応するようなアドレスを選ぶことができ、
したがって ELF バイナリからカーネルへデータをコピーできた。
(注: おそらく x86 用バージョン)

In the RISC-V version of xv6 this cannot happen,
because the kernel has its own separate page table;
loadseg loads into the process’s page table, not in the kernel’s page table.

RISC-V バージョンの xv6 ではこれは起こらない。
なぜならカーネルは自分専用の独立したページテーブルを持っているからだ。
loadseg はプロセスのページテーブルへロードするのであって、
カーネルのページテーブルへではない。

It is easy for a kernel developer to omit a crucial check,
and real-world kernels have a long history of missing checks
whose absence can be exploited by user programs to obtain kernel privileges.

カーネル開発者が重要なチェックを怠ってしまうというのはよくあることで、
実世界のカーネルにはユーザプログラムがカーネル権限を得られるような
長いチェック漏れの歴史がある。

It is likely that xv6 doesn’t do a complete job of validating user-level data
supplied to the kernel, which a malicious user program might be able to
exploit to circumvent xv6’s isolation.

xv6 はカーネルに与えられるユーザレベルデータの検証を完全に行えていない可能性が高く、
そうだとしたら悪意のあるユーザプログラムが xv6 の分離を回避できてしまうだろう。

## Real world

Like most operating systems, xv6 uses the paging hardware for memory protection and mapping.

ほとんどのオペレーティングシステムと同様に、xv6 はページングハードウェアを
メモリ保護やマッピングのために用いている。

Most operating systems make far more sophisticated use of paging than xv6
by combining paging and page-fault exceptions, which we will discuss in Chapter 4.

ほとんどのオペレーティングシステムはページングとページフォルト例外を組み合わせることによって、
xv6 よりもはるかに高度なページングの使い方をしている。
これについては4章で議論する。

Xv6 is simplified by the kernel’s use of a direct map between virtual and physical addresses,
and by its assumption that there is physical RAM at address 0x8000000,
where the kernel expects to be loaded.

xv6 はカーネルが仮想と物理アドレスの間でダイレクトマップを使用することと、
物理 RAM がアドレス 0x8000_0000 (ここにカーネルがロードされると期待する) にあると
仮定することによって単純化されている。

This works with QEMU, but on real hardware it turns out to be a bad idea;
real hardware places RAM and devices at unpredictable physical addresses,
so that (for example) there might be no RAM at 0x8000000,
where xv6 expect to be able to store the kernel.

これは QEMU ではうまくいくが、現実のハードウェア上では悪いアイデアだとわかる。
現実のハードウェアは RAM やデバイスをあらかじめ予測できない物理アドレスに置くので、
その結果 (例えば) 0x8000_0000、ここは xv6 がカーネルが格納されると期待する場所である、
には RAM がないかもしれない。

More serious kernel designs exploit the page table
to turn arbitrary hardware physical memory layouts into
predictable kernel virtual address layouts.

より本格的なカーネル設計では、任意のハードウェア物理メモリレイアウトを
予測可能なカーネル仮想アドレスレイアウトに変換するためにページテーブルを活用する。

RISC-V supports protection at the level of physical addresses,
but xv6 doesn’t use that feature.

RISC-V は物理アドレスレベルでの保護をサポートするが、xv6 はその機能を使用していない。

On machines with lots of memory
it might make sense to use RISC-V’s support for “super pages.”

多くのメモリを積んだマシンでは、RISC-V がサポートする「スーパーページ」を使用するのが
よいかもしれない。

Small pages make sense when physical memory is small,
to allow allocation and page-out to disk with fine granularity.

スモールページは物理メモリが小さい時に有効である。
アロケーションやディスクへのページアウトを細かい粒度で行えるからだ。

For example, if a program uses only 8 kilobytes of memory,
giving it a whole 4-megabyte super-page of physical memory is wasteful.

例えば、プログラムが 8KB のメモリしか使わないのならば、
物理メモリ中の 4 MB スーパーページ全体を与えるのは無駄が多い。

Larger pages make sense on machines with lots of RAM,
and may reduce overhead for page-table manipulation.

より大きなページは RAM の多いマシン上で有効であり、
ページテーブル操作のオーバーヘッドを低減できる。

The xv6 kernel’s lack of a malloc-like allocator that can provide memory for small objects
prevents the kernel from using sophisticated data structures
that would require dynamic allocation.

xv6 カーネルは小さなオブジェクトのためのメモリを提供できる malloc ライクなアロケータを欠いているので、
カーネルは動的確保を必要とするような高度なデータ構造を使用できない。

A more elaborate kernel would likely allocate many different sizes of small blocks,
rather than (as in xv6) just 4096-byte blocks;
a real kernel allocator would need to handle small allocations as well as large ones.

より複雑なカーネルでは、(xv6 のように) ちょうど 4096 バイトのブロックというよりは、
たくさんの異なるサイズの小さなブロックが確保されることが多い。
現実のカーネルアロケータは大きなものだけでなく、小さなアロケーションもハンドルする必要がある。

Memory allocation is a perennial hot topic,
the basic problems being efficient use of limited memory and
preparing for unknown future requests [9].

メモリアロケーションは永遠にホットなトピックであり、
基本的な問題は、限られたメモリを効率的に使用することと、
未知の未来のリクエストに対して備えることである。

Today people care more about speed than space efficiency.

今日では、空間効率よりも速度が優先されることが多い。

## Exercises

1. Parse RISC-V’s device tree to find the amount of physical memory the computer has.
1. Write a user program that grows its address space by one byte by calling sbrk(1).
  Run the program and investigate the page table for the program
  before the call to sbrk and after the call to sbrk.
  How much space has the kernel allocated?
  What does the PTE for the new memory contain?
1. Modify xv6 to use super pages for the kernel.
1. Unix implementations of exec traditionally include special handling for shell scripts.
  If the file to execute begins with the text #!,
  then the first line is taken to be a program to run to interpret the file.
  For example, if exec is called to run myprog arg1 and myprog ’s first
  line is #!/interp, then exec runs /interp with command line /interp myprog arg1.
  Implement support for this convention in xv6.
1. Implement address space layout randomization for the kernel.

訳

1. RISC-V のデバイスツリーをパースし、コンピュータが持つ物理メモリの量を発見せよ。
1. sbrk(1) を呼ぶことでアドレス空間を1バイトずつ拡張するユーザプログラムを書け。
  そのプログラムを実行し、sbrk を呼ぶ前と後でそのプログラムのページテーブルが
  どうなるか調査せよ。
  カーネルはどれだけの領域を確保したか？
  新しいメモリのための PTE には何が入っているか？
1. カーネルについてはスーパーページを使用するよう、xv6 を変更せよ。
1. exec の UNIX 実装には伝統的にシェルスクリプトのための特別な対応が含まれている。
  実行するファイルがテキスト `#!` で始まっていれば、最初の行はこのファイルを解釈するために
  実行されるプログラムとみなされる。
  例えば、exec が `myprog arg1` を実行するために呼び出され myprog の最初の行が
  `#!/interp` だった場合、exec は `/interp` をコマンドライン `/interp myprog arg1` で
  実行する。
  この変換のサポートを xv6 に実装せよ。
1. アドレス空間レイアウトランダマイゼーション (ASLR) をカーネルに実装せよ。
