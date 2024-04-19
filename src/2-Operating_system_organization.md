# Operating system organization

オペレーティングシステムの構成

A key requirement for an operating system is to support several activities at once.

オペレーティングシステムにとっての1つの鍵となる要件は、
いくつもの動作を同時にサポートすることだ。

For example, using the system call interface described in Chapter 1
a process can start new processes with fork.

例えば、1章で説明したシステムコールインタフェースを使って、
プロセスは fork によって新しいプロセスを開始することができる。

The operating system must time-share the resources of the computer among these processes.

オペレーティングシステムはそれらのプロセスの間でコンピュータのリソースを
time-share しなければならない。

For example, even if there are more processes than there are hardware CPUs,
the operating system must ensure that all of the processes get a chance to execute.

例えば、ハードウェア CPU の数より多くのプロセスが存在する場合でも、
オペレーティングシステムはすべてのプロセスが実行の機会を得られるよう保証しなければならない。

The operating system must also arrange for isolation between the processes.

オペレーティングシステムはプロセス間の分離 (isolation) も行わなければならない。

That is, if one process has a bug and malfunctions,
it shouldn’t affect processes that don’t depend on the buggy process.

つまり、もしあるプロセスにバグがあり誤動作したとしても、
そのバギーなプロセスに依存していないプロセスに影響があるべきではない。

Complete isolation, however, is too strong,
since it should be possible for processes to intentionally interact;
pipelines are an example.

しかしながら完全な分離というのもやりすぎで、なぜならプロセスは意図的に
相互作用を行う可能性があるためである。
パイプラインがその例だ。

Thus an operating system must fulfill three requirements:
multiplexing, isolation, and interaction.

したがってオペレーティングシステムは3つの要件を満たさなければならない。
multiplexing, isolation, and interaction (多重化、分離、相互作用)

This chapter provides an overview of how operating systems are
organized to achieve these three requirements.

本章ではオペレーティングシステムがどのようにしてこれらの3つの要件を満たすよう
構成されるのかの概観を与える。

It turns out there are many ways to do so, but this text focuses on
mainstream designs centered around a monolithic kernel,
which is used by many Unix operating systems.

そのようにする方法はたくさんあることが分かるが、本テキストではモノリシックカーネルを
中心とした主流の設計方法に焦点を当てる。
モノリシックなカーネル設計は多くの Unix OS で使われている。

This chapter also provides an overview of an xv6 process,
which is the unit of isolation in xv6, and the creation of the first process when xv6 starts.

本章では xv6 プロセス、これは xv6 における分離の単位である、
それと xv6 の開始時に最初のプロセスを生成するところ、の概観を与える。

Xv6 runs on a multi-core(*) RISC-V microprocessor,
and much of its low-level functionality (for example, its process implementation) is
specific to RISC-V.

xv6 はマルチコア(*) RISC-V マイクロプロセッサ上で動き、
その低レベルな機能 (例えば、プロセスの実装) は RISC-V に特有なものとなる。

RISC-V is a 64-bit CPU, and xv6 is written in “LP64” C,
which means long (L) and pointers (P) in the C programming language are 64 bits,
but int is 32-bit.

RISC-V は 64 bit CPU であり、xv6 は "LP64" C で書かれている。
これは C 言語における long (L) と pointers (P) が 64 bit だが、
int は 32 bit であることを意味する。

This book assumes the reader has done a bit of machine-level programming on some architecture,
and will introduce RISC-V-specific ideas as they come up.

本書では読者は何らかのアーキテクチャ上でマシンレベルのプログラミングを
少しはやったことがあることを前提とする。
そして RISC-V 特有のアイデアが出てきたら紹介していく。

A useful reference for RISC-V is “The RISC-V Reader: An Open Architecture Atlas” [15].

RISC-V の有用な参考書としては "The RISC-V Reader: An Open Architecture Atlas" がある。

The user-level ISA [2] and the privileged architecture [3] are the official specifications.

ユーザレベル ISA と特権アーキテクチャは公式の仕様書である。

The CPU in a complete computer is surrounded by support hardware,
much of it in the form of I/O interfaces.

完全な形のコンピュータにおける CPU は補助的なハードウェアに囲まれており、
その多くは I/O インタフェースの形をとっている。

Xv6 is written for the support hardware simulated by qemu’s “-machine virt” option.

xv6 は qemu の "-machine virt" オプションによりシミュレートされた
ハードウェアを使うように書かれている。

This includes RAM, a ROM containing boot code,
a serial connection to the user’s keyboard/screen, and a disk for storage.

これには RAM、ブートコードの入った ROM、ユーザのキーボードやスクリーンとつながった
シリアル通信、ストレージのためのディスクが含まれる。

(*) By “multi-core” this text means multiple CPUs that share memory but execute in parallel,
each with its own set of registers.

本書における「マルチコア」とは、メモリを共有するが並列に実行され、
それぞれが自分自身のレジスタセットを持つような複数個の CPU のことを指す。

This text sometimes uses the term multiprocessor as a synonym for multi-core,
though multiprocessor can also refer more specifically to a computer
with several distinct processor chips.

本書では時々マルチプロセッサという用語をマルチコアと同義で用いるが、
マルチプロセッサはより狭い意味で、いくつかの分かれたプロセッサチップからなるコンピュータを
指すこともある。
(注: 1枚のチップに複数コアを詰め込んだ密結合のマルチコア CPU と比較して、
複数のプロセッサチップを高速通信網でつないだ疎結合のスパコン的なシステムの方を
マルチプロセッサという用語で指すことがある、ということ。)

## Abstracting physical resources

物理リソースの抽象化

The first question one might ask when encountering an operating system is why have it at all?

オペレーティングシステムに出会ったとき、一番初めの疑問はこういうものかもしれない。
いったいなぜこれを使うのか？

That is, one could implement the system calls in Figure 1.2 as a library,
with which applications link.

これはつまり、図 1.2 のようなシステムコールをライブラリとして実装し、
アプリケーションにリンクするという形にもできるということだ。

In this plan, each application could even have its own library tailored to its needs.

このやり方では、それぞれのアプリケーションが自身の要求に合わせた独自のライブラリを
持つこともできる。

Applications could directly interact with hardware resources and
use those resources in the best way for the application
(e.g., to achieve high or predictable performance).

アプリケーションはハードウェアリソースと直接やりとりすることができ、
それらのリソースをアプリケーションにとってベストな方法で使用することができる
(例えば、高い、または予測可能なパフォーマンスを達成する)。

Some operating systems for embedded devices or real-time systems are organized in this way.

組み込みデバイスやリアルタイムシステムのためのいくつかのオペレーティングシステムは、
この方法で構成されている。

The downside of this library approach is that, if there is more than one application running,
the applications must be well-behaved.

このライブラリによるアプローチの欠点は、2つ以上のアプリケーションが実行されている時、
そのアプリケーションは行儀よく動かなくてはならないことだ。

For example, each application must periodically give up the CPU
so that other applications can run.

例えば、それぞれのアプリケーションは一定周期ごとに CPU を手放し、
他のアプリケーションが実行できるようにする必要がある。

Such a cooperative time-sharing scheme may be OK
if all applications trust each other and have no bugs.

そのような協調的タイムシェアリング戦略は、すべてのアプリケーションがお互いを信頼し、
かつすべてのアプリケーションに一切のバグがないという条件の下で OK と言えるかもしれない。
(注: 対義語はプリエンプティブ)

It’s more typical for applications to not trust each other, and to have bugs,
so one often wants stronger isolation than a cooperative scheme provides.

アプリケーションは他を信用できず、バグがある、という状況の方が典型的であり、
したがって協調的戦略よりも強い分離が望ましいケースが多い。

To achieve strong isolation it’s helpful to forbid applications
from directly accessing sensitive hardware resources,
and instead to abstract the resources into services.

強い分離を達成するためには、アプリケーションがセンシティブなハードウェアリソースに
直接アクセスするのを禁止し、代わりにそのリソースをサービスに抽象化するのが有効である。

For example, Unix applications interact with storage only through the file system’s
open, read, write, and close system calls, instead of reading and writing the disk directly.

例えば、Unix アプリケーションはストレージに対してディスクを直接読み書きするのではなく、
ファイルシステムのopen, read, write, close というシステムコールを介してのみやりとりする。

This provides the application with the convenience of pathnames,
and it allows the operating system (as the implementer of the interface)
to manage the disk.

この方法はアプリケーションにパス名という利便性を提供する。
そしてオペレーティングシステム (インタフェースの実装者) がディスクを管理することができる。

Even if isolation is not a concern, programs that interact intentionally
(or just wish to keep out of each other’s way) are likely to
find a file system a more convenient abstraction than direct use of the disk.

分離が必要ない場合であっても、意図的に他とやり取りする
(または単に他の邪魔をしたくないだけの) プログラムは、
ディスクを直接扱うよりもファイルシステムの方が便利な抽象化であると感じることが多い。

Similarly, Unix transparently switches hardware CPUs among processes,
saving and restoring register state as necessary,
so that applications don’t have to be aware of time sharing.

似たような話で、Unix はプロセス間でハードウェア CPU を透過的に切り替え。
このとき必要に応じてレジスタの退避と復帰を行うため、
アプリケーションはタイムシェアリングに関して意識する必要がない。

This transparency allows the operating system to share CPUs
even if some applications are in infinite loops.

この透過性により、オペレーティングシステムはいくつかのアプリケーションが
無限ループしている場合でも CPU を共有することができる。

As another example, Unix processes use exec to build up their memory image,
instead of directly interacting with physical memory.

他の例でいうと、Unix プロセスは自分のメモリイメージを組み上げるのに
物理メモリを直接操作する代わりに exec を使用する。

This allows the operating system to decide where to place a process in memory;
if memory is tight, the operating system might even store some of a process’s data on disk.

これによりオペレーティングシステムはプロセスをメモリ中のどこに置くかを決めることができる。
もしメモリが逼迫しているなら、オペレーティングシステムはプロセスのデータの一部を
ディスクに置くことができるかもしれない。
(注: スワップのことを指している)

exec also provides users with the convenience of a file system
to store executable program images.

exec は実行可能プログラムイメージを格納するためにファイルシステムの利便性を
ユーザに提供しているともいえる。

Many forms of interaction among Unix processes occur via file descriptors.

Unix プロセス間のやりとりの多くはファイルディスクリプタを介した形で行われる。

Not only do file descriptors abstract away many details
(e.g., where data in a pipe or file is stored),
they are also defined in a way that simplifies interaction.

ファイルディスクリプタは多くの詳細 (例えばパイプやファイルのデータがどこに格納されているのか) を
捨象するだけでなく、やり取りを単純化する方法で定義されている。

For example, if one application in a pipeline fails,
the kernel generates an end-of-file signal for the next process in the pipeline.

例えば、パイプライン中の1つのアプリケーションが失敗した場合、
カーネルはパイプライン中の次のプロセスに向けて EOF のシグナルを生成する。

The system-call interface in Figure 1.2 is carefully designed to provide
both programmer convenience and the possibility of strong isolation.

図 1.2 に示したシステムコールインタフェースはプログラマの利便性と
強力な分離を両立させるよう、注意深く設計されている。

The Unix interface is not the only way to abstract resources,
but it has proven to be a very good one.

Unix インタフェースはリソースを抽象化する唯一の方法ではないが、
とてもよい一つの方法であると証明されている。

2.2 User mode, supervisor mode, and system calls

ユーザモード、スーパーバイザモード、システムコール

Strong isolation requires a hard boundary between applications and the operating system.

強い分離のためにはアプリケーションとオペレーティングシステムの間に強い境界が必要である。

If the application makes a mistake, we don’t want the operating system
to fail or other applications to fail.

アプリケーションが誤りを犯した場合、オペレーティングシステムに障害が起きてほしくないし、
他のアプリケーションにも障害が起きてほしくない。

Instead, the operating system should be able to clean up the failed application and
continue running other applications.

そうではなく、オペレーティングシステムは障害が発生したアプリケーションを綺麗に片付け、
他のアプリケーションを実行し続けられるべきである。

To achieve strong isolation, the operating system must arrange that
applications cannot modify (or even read) the operating system’s data structures and instructions and
that applications cannot access other processes’ memory.

強い分離を達成するためには、オペレーティングシステムは
アプリケーションがオペレーティングシステムのデータ構造や命令を変更(または読み取りも)
できないように、また、他のプロセスのメモリにアクセスできないようにしなければならない。

CPUs provide hardware support for strong isolation.

CPU は強い分離のためのハードウェアサポートを提供する。

For example, RISC-V has three modes in which the CPU can execute instructions:
machine mode, supervisor mode, and user mode.

例えば、RISC-V は CPU が命令を実行するのに使える3つのモードがある。
マシンモード、スーパーバイザモード、ユーザモード。

Instructions executing in machine mode have full privilege; a CPU starts in machine mode.

マシンモードで実行される命令は全権限を持つ。
CPU はマシンモードで開始する。

Machine mode is mostly intended for configuring a computer.

マシンモードは大まかに言ってコンピュータの設定を行うためのものである。

Xv6 executes a few lines in machine mode and then changes to supervisor mode.

xv6 はマシンモードで数行を実行し、その後スーパバイザモードへ変更する。

In supervisor mode the CPU is allowed to execute privileged instructions:
for example, enabling and disabling interrupts,
reading and writing the register that holds the address of a page table, etc.

スーパーバイザモードでは CPU は特権命令を実行することができる。
例えば、割り込みを有効または無効にする、
ページテーブルのアドレスを保持するレジスタを読み書きする、等。

If an application in user mode attempts to execute a privileged instruction,
then the CPU doesn’t execute the instruction,
but switches to supervisor mode so that supervisor-mode code can terminate the application,
because it did something it shouldn’t be doing.

ユーザモードでアプリケーションが特権命令を実行しようとした場合、CPU はその命令を実行しないが、
スーパーバイザモードのコードがアプリケーションを終了できるようスーパーバイザモードへ切り替わる。
なぜならユーザモードのアプリケーションがやってはいけないことをやったからだ。

Figure 1.1 in Chapter 1　illustrates this organization.

1章の図1.1にこの構成を示している。

An application can execute only user-mode instructions (e.g., adding numbers, etc.)
and is said to be running in user space,
while the software in supervisor mode can also execute privileged instructions and
is said to be running in kernel space.

アプリケーションはユーザモード命令のみを実行できる (例: 足し算を行う、等)。
そしてこれはユーザ空間で動いているという。
対してスーパーバイザモードのソフトウェアは特権命令を実行することができ、
これはカーネル空間で動いているという。

The software running in kernel space (or in supervisor mode) is called the kernel.

カーネル空間で (つまりスーパーバイザモードで) 動作しているソフトウェアのことを
カーネルと呼ぶ。

An application that wants to invoke a kernel function
(e.g., the read system call in xv6) must transition to the kernel;
an application cannot invoke a kernel function directly.

カーネルの機能を呼び出したいアプリケーション (例: xv6 の read システムコール) は
カーネルに遷移しなくてはならない。
アプリケーションはカーネルの機能を直接呼び出すことはできない。

CPUs provide a special instruction that switches the CPU
from user mode to supervisor mode and enters the kernel at an entry point
specified by the kernel.

CPU は、CPU をユーザモードからスーパーバイザモードに切り替えてカーネルに指定された
エントリポイントからカーネルに入る特別な命令を提供する。

(RISC-V provides the ecall instruction for this purpose.)

RISC-V は ecall 命令をこの目的で提供する。
(注: environment-call。RISC-V 用語で自分を管理する一段上の世界を environment と呼ぶ。)

Once the CPU has switched to supervisor mode,
the kernel can then validate the arguments of the system call
(e.g., check if the address passed to the system call is part of the application’s memory),
decide whether the application is allowed to perform the requested operation
(e.g., check if the application is allowed to write the specified file),
and then deny it or execute it.

CPU がスーパーバイザモードに切り替わると、カーネルはシステムコールの引数を検証することができる。
(例: システムコールに渡されたアドレスがアプリケーションのメモリの一部であるかをチェックする)
そしてアプリケーションが要求した操作を行ってよいかどうかを決定できる。
(例: アプリケーションが指定されたファイルに書き込んでよいかをチェックする)
そしてリクエストを拒否するか実行するかできる。

It is important that the kernel control the entry point for transitions to supervisor mode;
if the application could decide the kernel entry point, a malicious application could,
for example, enter the kernel at a point where the validation of arguments is skipped.

カーネルがスーパーバイザモードへの遷移のエントリポイントを制御するのは重要なことである。
もしアプリケーションがカーネルエントリポイントを決定できてしまうと、
悪意のあるアプリケーションが、例えば、引数の検証をスキップした後の場所から
カーネルの中に入れてしまう。

## Kernel organization

カーネルの構成

A key design question is what part of the operating system should run in supervisor mode.

設計上のキーとなる問題は、オペレーティングシステムのどの部分をスーパーバイザモードで
実行すべきかということだ。

One possibility is that the entire operating system resides in the kernel,
so that the implementations of all system calls run in supervisor mode.

一つの可能性としては、オペレーティングシステム全体をカーネル内に置くというものがある。
結果としてすべてのシステムコールの実装はスーパーバイザモードで動くことになる。

This organization is called a monolithic kernel.

この構成法はモノリシックカーネルと呼ばれる。
(注: モノリス(一枚岩)的な、という意味)

In this organization the entire operating system runs with full hardware privilege.

この構成では、オペレーティングシステム全体が完全なハードウェア特権で実行される。

This organization is convenient because the OS designer doesn’t have to decide
which part of the operating system doesn’t need full hardware privilege.

この構成法は OS 設計者がオペレーティングシステムのどの部分が全ハードウェア特権を必要としないのかを
判定する必要がないという点で、簡便である。

Furthermore, it is easier for different parts of the operating system to cooperate.

さらに、オペレーティングシステムの異なる箇所同士が協調動作するのが (マイクロカーネルよりも)
簡単になる。

For example, an operating system might have a buffer cache
that can be shared both by the file system and the virtual memory system.

例えば、オペレーティングシステムはファイルシステムと仮想メモリシステムの両方から共有される
バッファキャッシュを持つかもしれない。
(注: ファイルの読み書きをした時にその内容をカーネル内のメモリ上に残して性能を上げる仕組み。
当然ファイルシステムとメモリ管理システムの両方と密接な連携が必要。)

A downside of the monolithic organization is that the interfaces
between different parts of the operating system are often complex
(as we will see in the rest of this text),
and therefore it is easy for an operating system developer to make a mistake.

モノリシックな構成の欠点は、オペレーティングシステムの異なるパーツ間のインタフェースが
複雑になりがちなことだ(このテキストの続きで見ていくことになるだろう)。
そしてそれによってオペレーティングシステムの開発者はミスを犯しやすくなる。

In a monolithic kernel, a mistake is fatal,
because an error in supervisor mode will often cause the kernel to fail.

モノリシックカーネルでは、ミスは致命的だ。
なぜならスーパーバイザモードでの1つのエラーがカーネルを落としてしまうことも多いからだ。

If the kernel fails, the computer stops working, and thus all applications fail too.

カーネルが落ちると、コンピュータは動作を停止し、その結果すべてのアプリケーションも
止まってしまう。

The computer must reboot to start again.

そのコンピュータを再度動かすためには再起動しなければならない。

TODO:
Microkernel
shell File serveruser
space
kernel
space
Send messageFigure 2.1: A microkernel with a file-system server

To reduce the risk of mistakes in the kernel,
OS designers can minimize the amount of operating system code that runs in supervisor mode,
and execute the bulk of the operating system in usermode.

カーネル内でのミスのリスクを低減するために、OS 設計者はスーパーバイザモードで動く
オペレーティングシステムのコードを最小化し、
オペレーティングシステムの大部分をユーザモードで実行することもできる。

This kernel organization is called a microkernel.

このカーネル構成法をマイクロカーネルと呼ぶ。

Figure 2.1 illustrates this microkernel design.

図 2.1 にこのマイクロカーネルの設計を示す。

In the figure, the file system runs as a user-level process.

図中で、ファイルシステムはユーザレベルプロセスとして動いている。

OS services running as processes are called servers.

プロセスとして動いている OS のサービスはサーバと呼ばれる。

To allow applications to interact with the file server,
the kernel provides an inter-process communication mechanism
to send messages from one user-mode process to another.

アプリケーションがファイルサーバとやりとりを行えるよう、カーネルは
あるユーザモードプロセスからもう1つにメッセージを送れるような
プロセス間通信のメカニズムを提供する。

For example, if an application like the shell wants to read or write a file,
it sends a message to the file server and waits for a response.

例えば、もしシェルのようなアプリケーションがファイルを読み書きしたい場合、
ファイルサーバにメッセージを送ってその応答を待つ。

In a microkernel, the kernel interface consists of a few low-level functions
for starting applications, sending messages, accessing device hardware, etc.

マイクロカーネルでは、カーネルインタフェースはアプリケーションを起動する、
メッセージを送る、デバイスハードウェアにアクセスする、等のための
少数の低レベル機能からなる。

This organization allows the kernel to be relatively simple,
as most of the operating system resides in user-level servers.

ほとんどのオペレーティングシステムの機能がユーザレベルサーバに置かれるため、
この構成によってカーネルを比較的シンプルにできる。

In the real world, both monolithic kernels and microkernels are popular.

現実世界では、モノリシックカーネルとマイクロカーネルはどちらもポピュラーである。

Many Unix kernels are monolithic.

多くの Unix カーネルはモノリシックだ。

For example, Linux has a monolithic kernel, although some OS functions run as
user-level servers (e.g., the windowing system).

例えば、Linux はいくつかの OS 機能がユーザレベルサーバで動く (例: ウィンドウシステム) とはいえ、
モノリシックカーネルだ。

Linux delivers high performance to OS-intensive applications,
partially because the subsystems of the kernel can be tightly integrated.

Linux は OS を多用するアプリケーションに高いパフォーマンスを提供するが、
この理由の一部はカーネルのサブシステムが密に結合しているからだ。

Operating systems such as Minix, L4, and QNX are organized as a microkernel with servers,
and have seen wide deployment in embedded settings.

Minix, L4, ANX のようなオペレーティングシステムはサーバによるマイクロカーネルとして構成される。
そして広い範囲の組み込み開発環境を考慮している。

A variant of L4, seL4, is small enough that
it has been verified for memory safety and other security properties [8].

L4 の変形である seL4 は十分に小さく、メモリ安全性とその他のセキュリティ特性が
検証完了している。
(注: バグがないことの形式的証明つきらしい)

There is much debate among developers of operating systems about which organization is better,
and there is no conclusive evidence one way or the other.

どちらの構成法がよいかということに関してはオペレーティングシステム開発者の間で多くの議論がある。
そして決定的な証拠はどちらにもない。

Furthermore, it depends much on what “better” means:
faster performance, smaller code size, reliability of the kernel,
reliability of the complete operating system (including user-level services), etc.

さらに、それは「よい」が何を意味するのかに大きく依存する。
速いパフォーマンス、小さいコードサイズ、カーネルの信頼性、
完全なオペレーティングシステムとしての信頼性 (ユーザレベルサービスを含む)、等。

There are also practical considerations that may be more important than
the question of which organization.

どちらの構成法が、という問いよりももっと重要かもしれない、実用的な考慮事項もある。

Some operating systems have a microkernel but run some of the user-level services
in kernel space for performance reasons.

マイクロカーネルだがいくつかのユーザレベルサービスをパフォーマンス上の理由で
カーネル空間で動かすオペレーティングシステムも存在する。

Some operating systems have monolithic kernels because
that is how they started and there is little incentive to move to a pure microkernel organization,
because new features may be more important than rewriting the existing operating system to fit a microkernel design.

モノリシックで始まったため、そして純粋なマイクロカーネルへ移行する動機に乏しいため、
モノリシックカーネルを採用し続けているオペレーティングシステムもある。

From this book’s perspective,
microkernel and monolithic operating systems share many key ideas.

本書の観点で言うと、マイクロカーネルとモノリシックなオペレーティングシステムには
多くの鍵となるアイデアが共通している。

They implement system calls, they use page tables, they handle interrupts,
they support processes, they use locks for concurrency control,
they implement a file system, etc.

どちらもシステムコールを実装し、どちらもページテーブルを使い、どちらも割り込みをハンドルし、
どちらもプロセスをサポートし、どちらも並行性を制御するためロックを使い、
どちらもファイルシステムを実装する。

This book focuses on these core ideas.

本書はこれらのコアなアイデアにフォーカスする。

Xv6 is implemented as a monolithic kernel, like most Unix operating systems.

xv6 は多くの Unix オペレーティングシステムのように、モノリシックカーネルとして実装されている。

Thus, the xv6 kernel interface corresponds to the operating system interface,
and the kernel implements the complete operating system.

したがって、xv6 カーネルのインタフェースはオペレーティングシステムのインタフェースに相当し、
カーネルはオペレーティングシステム全体を実装する。

Since xv6 doesn’t provide many services, its kernel is smaller than some microkernels,
but conceptually xv6 is monolithic.

xv6 は多くのサービスを提供するわけではないため、そのカーネルはいくつかのマイクロカーネルよりも小さいが、
概念上は xv6 はモノリシックである。

TODO:

File Description
bio.c Disk block cache for the file system.
console.c Connect to the user keyboard and screen.
entry.S Very first boot instructions.
exec.c exec() system call.
file.c File descriptor support.
fs.c File system.
kalloc.c Physical page allocator.
kernelvec.S Handle traps from kernel, and timer interrupts.
log.c File system logging and crash recovery.
main.c Control initialization of other modules during boot.
pipe.c Pipes.
plic.c RISC-V interrupt controller.
printf.c Formatted output to the console.
proc.c Processes and scheduling.
sleeplock.c Locks that yield the CPU.
spinlock.c Locks that don’t yield the CPU.
start.c Early machine-mode boot code.
string.c C string and byte-array library.
swtch.S Thread switching.
syscall.c Dispatch system calls to handling function.
sysfile.c File-related system calls.
sysproc.c Process-related system calls.
trampoline.S Assembly code to switch between user and kernel.
trap.c C code to handle and return from traps and interrupts.
uart.c Serial-port console device driver.
virtio_disk.c Disk device driver.
vm.c Manage page tables and address spaces.
Figure 2.2: Xv6 kernel source files.

## Code: xv6 organization

コード: xv6 の構成

The xv6 kernel source is in the kernel/ sub-directory.

xv6 カーネルソースは `kernel` サブディレクトリ内にある。

The source is divided into files, following a rough notion of modularity;
Figure 2.2 lists the files.

ソースはおおまかなモジュール性の考えにしたがい、複数のファイルに分かれている。

The inter-module interfaces are defined in defs.h (kernel/defs.h).

モジュール間のインタフェースは `dehs.h` 内に定義されている。

TODO:
0
user text
and data
user stack
heap
MAXVA trampoline
trapframeFigure 2.3: Layout of a process’s virtual address space

## Process overview

プロセスの概要

The unit of isolation in xv6 (as in other Unix operating systems) is a process.

xv6 における分離の単位 (他の Unix オペレーティングシステムと同じ) はプロセスである。

The process abstraction prevents one process from wrecking or spying on
another process’s memory, CPU, file descriptors, etc.

プロセスによる抽象化はあるプロセスが他のプロセスのメモリ、CPU、ファイルディスクリプタ等を
壊したり覗き見たりすることを防ぐ。

It also prevents a process from wrecking the kernel itself,
so that a process can’t subvert the kernel’s isolation mechanisms.

また、プロセスがカーネル自体を破壊することも防ぐので、プロセスがカーネルの
分離メカニズムを破壊することはできない。

The kernel must implement the process abstraction with care because
a buggy or malicious application may trick the kernel or hardware into
doing something bad (e.g., circumventing isolation).

カーネルはプロセス抽象を注意深く実装しなければならない。
なぜならバグのある、または悪意のあるアプリケーションがカーネルやハードウェアを騙して
何かよくないことをさせてくるかもしれないからである (例: 分離を回避する)。

The mechanisms used by the kernel to implement processes include the user/supervisor mode flag,
address spaces, and time-slicing of threads.

カーネルがプロセスを実装するのに使うメカニズムには、ユーザ/スーパーバイザモードフラグ、
アドレス空間、スレッドのタイムスライシングといったものがある。

To help enforce isolation, the process abstraction provides the illusion to
a program that it has its own private machine.

分離を強力にするため、プロセス抽象はプログラムに自分だけのマシンを持っているかのような
幻想を見せる。

A process provides a program with what appears to be a private memory system,
or address space, which other processes cannot read or write.

プロセスはプライベートなメモリシステムに見えるものをプログラムに提供する。
これはアドレス空間と呼ばれ、他のプロセスは読み書きができない。

A process also provides the program with what appears to be its own CPU
to execute the program’s instructions.

また、プロセスはプログラムに自分の命令を実行する自分だけの CPU に見えるものを提供する。

Xv6 uses page tables (which are implemented by hardware) to give each process
its own address space.

xv6 はページテーブル (これはハードウェアをによって実装される) を使って
それぞれのプロセスに自分だけのアドレス空間を与える。

The RISC-V page table translates (or “maps”) a virtual address
(the address that an RISC-V instruction manipulates) to a physical address
(an address that the CPU chip sends to main memory).

RISC-V ページテーブルは仮想アドレス (RISC-V 命令が操作するアドレス) を
物理アドレス (CPU チップがメインメモリへ送るアドレス) へ変換する (「マップする」ともいう)、

Xv6 maintains a separate page table for each process that defines that process’s address space.

xv6 はそれぞれのプロセスに対してそのプロセスのアドレス空間を定義する別々のページテーブルを管理する。

As illustrated in Figure 2.3, an address space includes the process’s user memory
starting at virtual address zero.

図 2.3 に示すように、プロセスのユーザメモリを含むアドレス空間は仮想アドレスゼロから始まる。

Instructions come first, followed by global variables, then the stack, and finally a
“heap” area (for malloc) that the process can expand as needed.

まず命令、次にグローバル変数、次にスタック、最後にプロセスが必要に応じて伸長できる
「ヒープ」(malloc のため)。

There are a number of factors that limit the maximum size of a process’s address space:
pointers on the RISC-V are 64 bits wide;
the hardware only uses the low 39 bits when looking up virtual addresses in page tables;
and xv6 only uses 38 of those 39 bits.

プロセスのアドレス空間の最大サイズを制限する要素はたくさんある。
RISC-V のポインタは 64 ビット幅である。
ハードウェアはページテーブルから仮想アドレスを引くときに下位 39 ビットのみしか使わない。
そして xv6 はその 39 ビットのうち 38 ビットのみを使用する。

Thus, the maximum address is 238 − 1 = 0x3fffffffff, which is MAXVA (kernel/riscv.h:363).

したがって、最大アドレスは 2^38 - 1 = 0x3f_ffff_ffff

At the top of the address space xv6 reserves a page for a trampoline and
a page mapping the process’s trapframe.

一番上のアドレス空間で xv6 はトランポリンのためのページとプロセスのトラップフレームを
マップするページを予約している。

Xv6 uses these two pages to transition into the kernel and back;
the trampoline page contains the code to transition in and out of the kernel and
mapping the trapframe is necessary to save/restore the state of the user process,
as we will explain in Chapter 4.

xv6 はこれら2つのページをカーネルに遷移するためと戻るために使う。
トランポリンページにはカーネルへ入る・出るためのコードが入っており、
トラップフレームのマッピングはユーザプロセスの状態を退避/復元 (save/restore) するために
必要である。
これは4章で説明する。

The xv6 kernel maintains many pieces of state for each process,
which it gathers into a struct proc (kernel/proc.h:85).

xv6 カーネルはそれぞれのプロセスについてたくさんの状態の要素を管理しており、
それは `struct proc` 構造体に集約されている (`kernel/proc.h:85`)。

A process’s most important pieces of kernel state are its page table,
its kernel stack, and its run state.

プロセスの最も重要なカーネル状態の要素はページテーブル、カーネルスタック、実行状態である。

We’ll use the notation p->xxx to refer to elements of the proc structure;
for example, p->pagetable is a pointer to the process’s page table.

proc 構造体の要素を参照するのに `p->xxx` という表記を使うこととする。
例えば、`p->pagetable` はプロセスのページテーブルへのポインタである。

Each process has a thread of execution (or thread for short) that executes
the process’s instructions.

それぞれのプロセスはプロセスの命令を実行する thread of execution (略してスレッドという)
を持つ。
(注: 直訳すると「実行の脈絡」らしいが日本語ではほぼ(もしかしたら英語でも)使われない。)

A thread can be suspended and later resumed.

スレッドは中断したり後に再開したりできる (suspend - resume)。

To switch transparently between processes,
the kernel suspends the currently running thread and resumes another process’s thread.

プロセス間を透過に切り替えるため、カーネルは現在実行中のスレッドを suspend し、
他のプロセスのスレッドを resume する。

Much of the state of a thread (local variables, function call return addresses) is
stored on the thread’s stacks.

スレッドの状態の多く (ローカル変数、関数呼び出しのリターンアドレス) は
スレッドのスタックに格納されている。

Each process has two stacks: a user stack and a kernel stack (p->kstack).

それぞれのプロセスは2つのスタック持つ。
ユーザスタックとカーネルスタック (p->kstack)。

When the process is executing user instructions, only its user stack is in use,
and its kernel stack is empty.

プロセスがユーザ (モードで) 命令を実行中、ユーザスタックのみが使用され、カーネルスタックは空となる。

When the process enters the kernel (for a system call or interrupt),
the kernel code executes on the process’s kernel stack;

プロセスがカーネルへ入る時 (システムコールまたは割り込みのため)、
カーネルコードはプロセスのカーネルスタック上で実行される。

while a process is in the kernel, its user stack still contains saved data,
but isn’t actively used.

プロセスがカーネル内にいる間、そのユーザスタックは保存データを保持したままとなるが、
アクティブには使われない。

A process’s thread alternates between actively using its user stack and its kernel stack.

プロセスのスレッドはユーザスタックを使う状態とカーネルスタックを使う状態の間を切り替わる。

The kernel stack is separate (and protected from user code) so that
the kernel can execute even if a process has wrecked its user stack.

もしプロセスが自分のユーザスタックを破壊してしまったとしてもカーネルを実行できるよう、
カーネルスタックは分離されている (そしてユーザコードからは保護されている)。

A process can make a system call by executing the RISC-V ecall instruction.

プロセスは RISC-V の ecall 命令を実行することでシステムコールを行うことができる。

This instruction raises the hardware privilege level and
changes the program counter to a kernel-defined entry point.

この命令はハードウェア特権レベルを上げ、プログラムカウンタをカーネル定義の
エントリポイントへ変更する。

The code at the entry point switches to a kernel stack and executes the kernel instructions that
implement the system call.

エントリポイントのコードはカーネルスタックに切り替え、
システムコールを実装するカーネル命令を実行する。

When the system call completes, the kernel switches back to the user stack and
returns to user space by calling the sret instruction,
which lowers the hardware privilege level and resumes executing user instructions
just after the system call instruction.

システムコールが完了したら、カーネルはユーザスタックへ切り替え、
sret 命令を呼ぶことでユーザ空間へリターンする。
sret 命令はハードウェア特権レベルを下げ、システムコール命令の直後から
ユーザ命令の実行を再開する。

A process’s thread can “block” in the kernel to wait for I/O, and resume where it left off when the I/O has finished.

プロセスのスレッドは I/O を待つためにカーネル内で「ブロック」することができ、
そして I/O が完了した時に中断した場所から再開することができる。

p->state indicates whether the process is allocated, ready to run, running, waiting for I/O, or exiting.

`p->state` はプロセスが以下のどの状態なのかを示す。
確保された、実行準備完了、実行中、I/O 待ち、終了中。

p->pagetable holds the process’s page table, in the format that the RISC-V hardware expects.

`p->pagetable` は RISC-V ハードウェアが期待する形で、プロセスのページテーブルを保持する。

Xv6 causes the paging hardware to use a process’s p->pagetable when executing that
process in user space.

xv6 はプロセスをユーザ空間で実行中、ページングハードウェアにプロセスの `p->pagetable` を
使うようにさせる。

A process’s page table also serves as the record of the addresses of the physical pages
allocated to store the process’s memory.

プロセスのページテーブルは、プロセスのメモリを格納するために確保された
物理ページのアドレスの記録を提供するという働きもある。

In summary, a process bundles two design ideas:
an address space to give a process the illusion of its own memory, and,
a thread, to give the process the illusion of its own CPU.

まとめると、プロセスは2つの設計上のアイデアを含んでいる。
プロセスに自分だけのメモリの幻想を見せるためのアドレス空間と、
プロセスに自分だけの CPU の幻想を見せるためのスレッドである。

In xv6, a process consists of one address space and one thread.

xv6 では、1つのプロセスは1つのアドレス空間と1つのスレッドからなる。

In real operating systems a process may have more than one thread to take advantage of multiple CPUs.

現実のオペレーティングシステムでは、複数 CPU の利点を生かすために、
1つのプロセスは1つより多いスレッドを持つことがある。

## Code: starting xv6, the first process and system call

コード: xv6 の開始、最初のプロセスとシステムコール

To make xv6 more concrete, we’ll outline how the kernel starts and runs the first process.

xv6 をもっと明確にするために、カーネルがどのように開始するかと最初のプロセスを実行するかを
概観していく。

The subsequent chapters will describe the mechanisms that show up in this overview in more detail.

以降の章ではこの概要に出てきたメカニズムをより詳しく解説する。

When the RISC-V computer powers on, it initializes itself and runs a boot loader
which is stored in read-only memory.

RISC-V コンピュータの電源が入ると、自分自身を初期化し読み取り専用メモリ (ROM) に格納された
ブートローダを実行する。

The boot loader loads the xv6 kernel into memory.

ブートローダは xv6 カーネルをメモリにロードする。

Then, in machine mode, the CPU executes xv6 starting at _entry (kernel/entry.S:7).

次に、マシンモードで、CPU は _entry で始まる xv6 を実行する (`kernel/entry.S:7`)。

The RISC-V starts with paging hardware disabled: virtual addresses map directly to physical addresses.

RISC-V はページングハードウェア無効の状態で開始する。
仮想アドレスはダイレクトに物理アドレスにマップされている。

The loader loads the xv6 kernel into memory at physical address 0x80000000.

ローダは xv6 カーネルをメモリの物理アドレス 0x8000_0000 にロードする。

The reason it places the kernel at 0x80000000 rather than 0x0 is
because the address range 0x0:0x80000000 contains I/O devices.

カーネルを 0x0 ではなく 0x8000_0000 に置く理由は、アドレスレンジ 0x0-0x8000_0000 には
I/O デバイスが置かれているからである。

The instructions at _entry set up a stack so that xv6 can run C code.

_entry にある命令は xv6 が C コードを実行できるようスタックを設定する。

Xv6 declares space for an initial stack, stack0, in the file start.c (kernel/start.c:11).

xv6 は初期スタック stack0 の領域を start.c の中で宣言している (`kernel/start.c:11`)。

The code at _entry loads the stack pointer register sp with the address stack0+4096,
the top of the stack, because the stack on RISC-V grows down.

_entry にあるコードはスタックポインタレジスタ sp にアドレス stack0+4096、
スタックトップをロードする。
これは RISC-V のスタックがアドレスが小さくなるほうに伸びるからである。

Now that the kernel has a stack, _entry calls into C code at start (kernel/start.c:21).

これでカーネルはスタックを持ったので、_entry は start の C コードを呼び出す (`kernel/start.c`)。

The function start performs some configuration that is only allowed in machine mode,
and then switches to supervisor mode.

この start 関数はマシンモードのみで許可されたいくつかの設定を行い、
次にスーパーバイザモードへ切り替える。

To enter supervisor mode, RISC-V provides the instruction mret.

スーパーバイザモードへ入るためには、RISC-V は mret 命令を提供している。

This instruction is most often used to return from a previous call
from supervisor mode to machine mode.

この命令はスーパーバイザモードからマシンモードへの1つ前の呼び出しから戻る時に
最もよく使われる命令である。

start isn’t returning from such a call,
and instead sets things up as if there had been one:
it sets the previous privilege mode to supervisor in the register mstatus,
it sets the return address to main by writing main’s address into the register mepc,
disables virtual address translation in supervisor mode by writing 0 into the page-table register satp,
and delegates all interrupts and exceptions to supervisor mode.

start はそのような (スーパーバイザからマシンモードへの) 呼び出しから戻るわけではない。
代わりに呼び出しがあったかのように設定を行う。
レジスタ mstatus 内の前回の特権モードをスーパーバイザに、
レジスタ mepc に main のアドレスを書き込むことによってリターンアドレスを main に、
ページテーブルレジスタ satp に 0 を書き込むことによって
スーパーバイザモードでの仮想アドレス変換を無効にし、
すべての割り込みと例外をスーパバイザモードへ委譲する。
(注: satp = Supervisor Address Translation and Protection)

Before jumping into supervisor mode, start performs one more task:
it programs the clock chip to generate timer interrupts.

スーパーバイザモードへジャンプする前に、start はもう1つ仕事を行う。
タイマ割り込みを生成するようクロックチップをプログラムする。

With this housekeeping out of the way, start “returns” to supervisor mode by calling mret.
This causes the program counter to change to main (kernel/main.c:11).

このハウスキーピング処理を終えて、start は mret を呼ぶことでスーパーバイザモードへ
「リターンする」。

After main (kernel/main.c:11) initializes several devices and subsystems,
it creates the first process by calling userinit (kernel/proc.c:233).

main (`kernel/main.c:11`) がいくつかのデバイスやサブシステムを初期化した後、
userinit (`kernel/proc.c:233`) を呼ぶことで最初のプロセスを生成する。

The first process executes a small program written in RISC-V assembly,
which makes the first system call in xv6.

最初のプロセスは RISC-V アセンブリで書かれた小さなプログラムを実行し、
それは xv6 で最初のシステムコールを発行する。

initcode.S (user/initcode.S:3) loads the number for the exec system call,
SYS_EXEC (kernel/syscall.h:8), into register a7, and then calls ecall to re-enter the kernel.

initcode.S (`user/initcode.S:3`) は exec システムコールの番号
SYS_EXEC (`kernel/syscall.h:8`) をレジスタ a7 にロードし、
そして ecall を呼んで再度カーネルに入る。

The kernel uses the number in register a7 in syscall (kernel/syscall.c:132) to call the desired
system call.

カーネルは syscall (`kernel/syscall.c:132`) 内で、
要求されたシステムコールを呼び出すためにレジスタ a7 に入っている番号を使う。

The system call table (kernel/syscall.c:107) maps SYS_EXEC to sys_exec,
which the kernel invokes.

システムコールテーブル (`kernel/syscall.c:107`) は SYS_EXEC (番号) を sys_exec (関数) にマップし、
カーネルはそれを呼び出す。

As we saw in Chapter 1, exec replaces the memory and registers of the current process
with a new program (in this case, /init).

Chapter 1 で見たように、exec はカレントプロセスのメモリとレジスタを新しいプログラムで置き換える
(今回の場合、`/init`)。

Once the kernel has completed exec, it returns to user space in the /init process.

カーネルが exec を完了すると、`/init` プロセスのユーザ空間にリターンする。

init (user/init.c:15) creates a new console device file if needed and then
opens it as file descriptors 0, 1, and 2.

init (`user/init.c:15`) は必要ならば新しいコンソールデバイスを作成し、
それをファイルディスクリプタ 0, 1, 2 としてオープンする。

Then it starts a shell on the console.

これでシェルがコンソール上で動き出す。

The system is up.

システムが起動した。

## Security Model

セキュリティモデル

You may wonder how the operating system deals with buggy or malicious code.

オペレーティングシステムはどのようにしてバグのある、または悪意のあるコードに
対処するのだろうかと思うかもしれない。

Because coping with malice is strictly harder than dealing with accidental bugs,
it’s reasonable to view this topic as relating to security.

悪意に対処するのは偶発的なバグに対処するよりも確実に難しいため、
このトピックをセキュリティ関連として見ていくのが合理的だ。

Here’s a high-level view of typical security assumptions and goals in operating system design.

ここではオペレーティングシステム設計における典型的なセキュリティ上の仮定と目標についての
概観を示す。

The operating system must assume that a process’s user-level code will do
its best to wreck the kernel or other processes.

オペレーティングシステムはプロセスのユーザレベルコードはカーネルや他のプロセスを破壊するために
全力を尽くすと仮定しなければならない。

User code may try to dereference pointers outside its allowed address space;
it may attempt to execute any RISC-V instructions, even those not intended for user code;
it may try to read and write any RISC-V control register;
it may try to directly access device hardware;
and it may pass clever values to system calls in an attempt to trick the kernel
into crashing or doing something stupid.

ユーザコードは許可されたアドレス空間の外のポインタをデリファレンスするかもしれない。
あらゆる RISC-V 命令を実行しようとするかもしれない。ユーザコード用でないものも。
あらゆる RISC-V 制御レジスタを読み書きしようとするかもしれない。
デバイスハードウェアに直接アクセスしようとするかもしれない。
そしてカーネルを騙してクラッシュさせたり愚かな動作をさせるために
システムコールに巧妙な値を渡してくるかもしれない。

The kernel’s goal to restrict each user processes
so that all it can do is read/write/execute its own user memory,
use the 32 general-purpose RISC-V registers,
and affect the kernel and other processes in the ways that system calls are intended to allow.

カーネルの目標は、ユーザプロセスができるのは以下がすべてになるように
それぞれのユーザプロセスを制限することだ。
自分のユーザメモリを読む/書く/実行すること、
32 個の汎用 RISC-V レジスタを使うこと、
システムコールが意図して許可した方法でカーネルや他のプロセスに影響を与えること。

The kernel must prevent any other actions.

カーネルは他のすべてのアクションを防がなければならない。

This is typically an absolute requirement in kernel design.

これは通常、カーネル設計における絶対的な要件である。

The expectations for the kernel’s own code are quite different.

カーネル自身のコードに対する期待は全く異なる。

Kernel code is assumed to be written by well-meaning and careful programmers.

カーネルコードは善意の注意深いプログラマによって書かれていると仮定される。

Kernel code is expected to be bug-free, and certainly to contain nothing malicious.

カーネルコードにはバグがなく、悪意のあるものは一切含まれないと期待される。

This assumption affects how we analyze kernel code.

この仮定は我々がカーネルコードを解析する方法にも影響する。

For example, there are many internal kernel functions (e.g., the spin locks)
that would cause serious problems if kernel code used them incorrectly.

例えば、もしカーネルコードが誤った使い方をすれば深刻な問題を引き起こすような
たくさんの内部カーネル関数 (例: スピンロック) が存在する。
(注: まだ出てきていないがアンロックを忘れるとそのコアが割り込み不能状態のまま固まる)

When examining any specific piece of kernel code,
we’ll want to convince ourselves that it behaves correctly.

カーネルコードの特定の部分を調べるとき、それが正しく動くものと思い込みたいだろう。

We assume, however, that kernel code in general is correctly written,
and follows all the rules about use of the kernel’s own functions and data structures.

しかし、カーネルコードは一般に正しく書かれており、
カーネル自身の関数やデータ構造の使い方のルールをすべて守っていると仮定する。

At the hardware level, the RISC-V CPU, RAM, disk, etc. are assumed to operate
as advertised in the documentation, with no hardware bugs.

ハードウェアレベルでは、RISC-V CPU、RAM、ディスク等にはハードウェアバグはなく、
ドキュメントでうたわれている通りに動作するものと仮定する。

Of course in real life things are not so straightforward.

もちろん現実では物事はそう単純ではない。

It’s difficult to prevent clever user code from making a system unusable
(or causing it to panic) by consuming kernel-protected resources
– disk space, CPU time, process table slots, etc.

巧妙なユーザコードがカーネルに保護されたリソース、つまり
ディスク容量、CPU 時間、プロセステーブルスロット、等を消費して
システムを使用不能にしようとする (panic させようとする) のを防ぐのは難しい。

It’s usually impossible to write bug-free code or design bug-free hardware;
if the writers of malicious user code are aware of kernel or hardware bugs,
they will exploit them.

バグのないコードを書く、またはバグのないハードウェアを設計するというのは
通常不可能である。
悪意のあるユーザコードの作者がカーネルやハードウェアバグに気づいていれば、
そこを突かれてしまうだろう。

Even in mature, widely-used kernels, such as Linux,
people discover new vulnerabilities continuously [1].

Linux のような成熟し、広く使われているカーネルでさえ、
絶えず新たな脆弱性が発見され続けている。

It’s worthwhile to design safeguards into the kernel against the possibility that it has bugs:
assertions, type checking, stack guard pages, etc.

バグがある可能性に対してカーネルにセーフガードを設計することには価値がある。
アサーション、型チェック、スタックガードページ、等。

Finally, the distinction between user and kernel code is sometimes blurred:
some privileged user-level processes may provide essential services and
effectively be part of the operating system, and in some operating systems
privileged user code can insert new code into the kernel
(as with Linux’s loadable kernel modules).

最後に、ユーザとカーネルのコードの区別が曖昧になることがある。
特権ユーザレベルのプロセスの中には必要不可欠なサービスを提供し、
実質的にオペレーティングシステムの一部となっているものもある。
そして一部のオペレーティングシステムの特権ユーザコードはカーネルに新しいコードを挿入できる
(Linux のローダブルカーネルモジュールのように)。

## Real world

Most operating systems have adopted the process concept,
and most processes look similar to xv6’s.

ほとんどのオペレーティングシステムはプロセスの概念を採用している。

Modern operating systems, however, support several threads within a process,
to allow a single process to exploit multiple CPUs.

しかし現代のオペレーティングシステムは1つのプロセスに複数のスレッドをサポートしており、
1つのプロセスが複数の CPU を活用することができる。

Supporting multiple threads in a process involves quite a bit of machinery that xv6 doesn’t have,
including potential interface changes (e.g., Linux’s clone, a variant of fork),
to control which aspects of a process threads share.

1つのプロセスで複数のスレッドをサポートするためには
インタフェースの変更 (例: fork の亜種である Linux の clone) の可能性を含め、
スレッドがプロセスのどの部分を共有するかを制御するために、
xv6 が持っていないかなりの量の機構が必要になる。

## Exercises

1. Add a system call to xv6 that returns the amount of free memory available.

xv6 に利用可能な空きメモリの量を返すシステムコールを追加せよ。
