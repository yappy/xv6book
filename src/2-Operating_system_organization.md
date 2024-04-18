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
モノリシックカーネルは多くの Unix OS で使われている。

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
