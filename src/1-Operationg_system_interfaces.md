# Operating system interfaces

The job of an operating system is to share a computer among multiple programs and to provide a
more useful set of services than the hardware alone supports.

OS の仕事は一台のコンピュータを複数のプログラム間に分け与えることと、
ハードウェア単体がサポートするよりも便利なサービスのセットを提供することである。

An operating system manages and abstracts the low-level hardware,
so that, for example, a word processor need not concern itself
with which type of disk hardware is being used.

OS は低レベルのハードウェアを管理し抽象化する。
その結果、例えば、ワープロ (注: 通じるのか？) はどのタイプのディスクハードウェアが
使用されているか気にする必要がない。
(注: ファイルを開いて読み書きする、という OS の機能を呼び出せば、HDD だろうが
SSD だろうが USB メモリだろうが共通のやり方でデータを読み書きできる、ということ。)

An operating system shares the hardware among multiple programs
so that they run (or appear to run) at the same time.

複数のプログラムが同時に実行される (または同時に実行されているように見える) ために
OS はハードウェアを複数のプログラム間に分け与える。
(注: 昔はシングルコアで同時に1つのプログラムしか動かなかったため、時々動作中の
プログラムを切り替えることによって同時実行しているように見せかけていたが、
最近のマルチコアシステムではコア数までなら本当に物理的に同時に動く。
それ以上の数に対しては切り替えを行う。)

Finally, operating systems provide controlled ways for programs to interact,
so that they can share data or work together.

最後に、OS はプログラム同士が通信する方法を提供する。
これにより、プログラムはデータを共有したり、協調して動いたりできる。

An operating system provides services to user programs through an interface.

OS はユーザプログラムに対してインタフェースを通してサービスを提供する。

Designing a good interface turns out to be difficult.

よいインタフェースを設計するのは難しい。

On the one hand, we would like the interface to be simple and narrow
because that makes it easier to get the implementation right.

一方では、我々はインタフェースを単純で限定されたものにしたい。
これは実装を正しいものにするのが簡単になるからだ(正しく実装するのが簡単になるからだ)。

On the other hand, we may be tempted to offer many sophisticated features to applications.

他方で、多くの高度な機能をアプリケーションに提供したいと思うかもしれない。

The trick in resolving this tension is to design interfaces that rely on a few mechanisms
that can be combined to provide much generality.

この対立を解決するコツは、高い汎用性を提供するために組み合わせて使えるような、
少数のメカニズムに基づいてインタフェースを設計することだ。
(注: 単純なことのみを行うプログラムのみを用意し、それをパイプ等で組み合わせて使うことにより
様々な状況に対応できるという UNIX 哲学のことを言っている。)

This book uses a single operating system as a concrete example to illustrate operating system
concepts.

本書では、OS の概念を説明するために、1つの OS を具体的な例として用いている。

That operating system, xv6, provides the basic interfaces introduced by Ken Thompson
and Dennis Ritchie’s Unix operating system [17], as well as mimicking Unix’s internal design.

その OS、xv6 はケン・トンプソンとデニス・リッチーによって導入された基本的なインタフェースを提供し、
Unix の内部設計を模倣している。

Unix provides a narrow interface whose mechanisms combine well, offering a surprising degree
of generality.

Unix はそのメカニズムがうまく組み合わされた限定的なインタフェースを提供し、
驚くほどの汎用性を提供する。

This interface has been so successful that modern operating systems—BSD, Linux,
macOS, Solaris, and even, to a lesser extent, Microsoft Windows—have Unix-like interfaces.

このインタフェースは大変成功していて、現代の OS - BSD, Linux, macOS, Solaris は、
そしてやや程度は低くはなるが、Microsoft Windows でさえ、
Unix ライクなインタフェースを持っている。

Understanding xv6 is a good start toward understanding any of these systems and many others.

xv6 を理解することは、これらの、またこれ以外の、あらゆるシステムを理解するための
よいとっかかりとなるだろう。

TODO: Figure 1.1

As Figure 1.1 shows, xv6 takes the traditional form of a kernel, a special program that provides
services to running programs.

図1.1 に示すように、xv6 は伝統的なカーネルという形態を取っている。
カーネルとは実行中のプログラムにサービスを提供する特別なプログラムである。

Each running program, called a process, has memory containing instructions, data, and a stack.

一つ一つの実行中のプログラムはプロセスと呼ばれ、命令、データ、スタックを含む
メモリを持っている。

The instructions implement the program’s computation.

命令列はそのプログラムの計算を実現する。

The data are the variables on which the computation acts.

データはその計算が行われる変数群である。

The stack organizes the program’s procedure calls.

スタックはそのプログラムの手続き呼び出しを構成する。
(注: 関数呼び出ししか分からなかったら関数呼び出しのことと思って OK です。)

A given computer typically has many processes but only a single kernel.

一台のコンピュータでは典型的にはたくさんのプロセスが動いているが、
カーネルは1つしかない。

When a process needs to invoke a kernel service, it invokes a system call,
one of the calls in the operating system’s interface.

あるプロセスがカーネルのサービスを呼び出す必要が出てきたとき、
そのプロセスはシステムコールを呼ぶ。
システムコールとは OS 内のインタフェース呼び出しの1つである。

The system call enters the kernel; the kernel performs the service and returns.

システムコールはカーネルへ進入する。
カーネルはサービスを実行し、そして戻る。

Thus a process alternates between executing in user space and kernel space.

このように、プロセスは実行中にユーザ空間とカーネル空間を行き来する。
(注: 空間といった場合は通例メモリ空間のことを指す。下記参照。)

As described in detail in subsequent chapters,
the kernel uses the hardware protection mechanisms provided by a CPU(*1)
to ensure that each process executing in user space can access only its own memory.

続く章で詳しく解説するように、
ユーザランドで実行中のそれぞれのプロセスが自分自身のメモリにしかアクセスできないことを保証するため、
カーネルは CPU によって提供されるハードウェア保護機構を使用する。

(*1) This text generally refers to the hardware element that executes a computation with the term CPU, an acronym

The kernel executes with the hardware privileges required to implement these
protections; user programs execute without those privileges.

カーネルはこれらの保護を実現するためにハードウェア特権と共に実行される。
ユーザプログラムはそのような特権なしで実行される。

When a user program invokes a system call,
the hardware raises the privilege level and starts executing
a pre-arranged function in the kernel.

ユーザプログラムがシステムコールを呼び出すと、
ハードウェアは特権レベルを上げ、カーネル内のあらかじめ設定された関数を実行する。

The collection of system calls that a kernel provides is the interface that user programs see.

カーネルの提供するシステムコールのセットはユーザプログラムから見たインタフェースとなる。

The xv6 kernel provides a subset of the services and system calls that Unix kernels traditionally offer.

xv6 カーネルは Unix カーネルが伝統的に提供してきたサービスやシステムコールのサブセットを提供する。

Figure 1.2 lists all of xv6’s system calls.

図 1.2 に xv6 の全システムコールを示す。

TODO: FIgure 1.2

The rest of this chapter outlines xv6’s services—processes, memory, file descriptors, pipes,
and a file system—and illustrates them with code snippets and discussions of how the shell, Unix’s
command-line user interface, uses them.

この章の残りでは、xv6 のサービス - プロセス、メモリ、ファイルディスクリプタ、パイプ、
ファイルシステム - の概略を示し、それらをコード片、およびシェル
(Unix のコマンドラインユーザインタフェース) がどのように使うかという考察で説明する。

The shell’s use of system calls illustrates how carefully they have been designed.

シェルからシステムコールがどのように使われるかは、
システムコールがどのくらい注意深く設計されているかを物語る。
(注: シェルからの OS の使い方を見れば OS の設計がどのくらい優れているかが
わかるという哲学を主張している。暗に、Unix のコマンドラインを見れば
Unix (のシステムコール)の設計がいかに優れているかが分かると言っている。)

The shell is an ordinary program that reads commands from the user and executes them.

シェルというのは、コマンドをユーザから読み込んで、それを実行するような普通のプログラムである。

The fact that the shell is a user program, and not part of the kernel,
illustrates the power of the system call interface:
there is nothing special about the shell.

シェルがユーザプログラムでありカーネル一部ではないという事実は、
システムコールインタフェースの威力を示している:
シェルに関して特別なことは何もない。

It also means that the shell is easy to replace;
as a result, modern Unix systems have a variety of shells to choose from,
each with its own user interface and scripting features.

これはまた、シェルを置き換えることが簡単であるということをも意味する。
その結果、現代の Unix システムは多くの種類のシェルの中から選択できる。
それぞれのシェルは独自のユーザインタフェースとスクリプト機能を有している。

The xv6 shell is a simple implementation of the essence of the Unix Bourne shell.

xv6 のシェルは Unix Bourne shell の要点をシンプルに実装したものになっている。
(注: Unix の由緒正しい原初のシェル。Bourne は人名。
/bin/sh はもともとこのシェルだったが、
現代では後継のシェルへのシンボリックリンクになっていることが多い。)

Its implementation can be found at (user/sh.c:1).

その実装は xv6 ソースコードの user/sh.c にある。

## Processes and memory

An xv6 process consists of user-space memory (instructions, data, and stack)
and per-process state private to the kernel.

xv6 のプロセスはユーザ空間メモリ (命令、データ、スタック) と、
カーネルに対してプライベートなプロセスごとのステートからなる。

Xv6 time-shares processes: it transparently switches the available CPUs
among the set of processes waiting to execute.

xv6 はプロセス群に時間を分け与える:
実行待ちのプロセスセットの間で利用可能な CPU を透過的に切り替える。
(注: time-sharing - OS における重要用語。
CPU 時間というシステム全体で共通のリソースをプロセス間で分配するという概念。)
(注: transparently - コンピュータ技術全般における重要用語。
透過的に。外側のユーザから違いを意識させることなく使用できること。
ここではユーザプログラムはただ自分の仕事のための命令列を実行するだけでよく、
CPU のスイッチは OS 側で行うのでそれを意識する必要が全くないことを指す。)

When a process is not executing, xv6 saves the process’s CPU registers,
restoring them when it next runs the process.

プロセスが実行中でない時、xv6 はそのプロセスの CPU レジスタをセーブしておき、
次にそのプロセスを実行するときにそれを復元する。
(注: save-restore - 退避・復元。ここではメモリを使う。)

The kernel associates a process identifier, or PID, with each process.

カーネルはそれぞれのプロセスに対して1つずつ、プロセス識別子
(process identifier - PID) を割り当てる。

A process may create a new process using the fork system call.

プロセスは fork システムコールを使って新しいプロセスを生成することができる。

fork gives the new process an exact copy of the calling process’s memory, both instructions and data.

fork は呼び出し元のプロセスメモリ (命令とデータ両方) の完全なコピーである
新しいプロセス与える。

fork returns in both the original and new processes.

fork はオリジナルと新しいプロセスの両方にリターンする。

In the original process, fork returns the new process’s PID.

オリジナルのプロセスでは、fork は新しいプロセスの PID を返す。

In the new process, fork returns zero.

新しいプロセスでは、fork はゼロを返す。

The original and new processes are often called the parent and child.

オリジナルと新しいプロセスはしばしば親と子と呼ばれる。

For example, consider the following program fragment written
in the C programming language [7]:

例えば、以下の C 言語で書かれたプログラムを考えよう。

```C
int pid = fork();
if(pid > 0){
  printf("parent: child=%d\n", pid);
  pid = wait((int *) 0);
  printf("child %d is done\n", pid);
} else if(pid == 0){
  printf("child: exiting\n");
  exit(0);
} else {
  printf("fork error\n");
}
```

The exit system call causes the calling process to stop executing and
to release resources such as memory and open files.

exit システムコールは呼んだプロセスの実行を停止し、メモリや開いているファイルのような
リソースを解放する。

Exit takes an integer status argument, conventionally 0 to indicate success
and 1 to indicate failure.

exit は 1 つの整数引数を取り、慣例的に 0 が成功を、1 が失敗を表す。

The wait system call returns the PID of an exited (or killed) child of the current process and
copies the exit status of the child to the address passed to wait;
if none of the caller’s children has exited, wait waits for one to do so.

wait システムコールは終了した (または kill された) 自分の子プロセス1つの PID を返し、
wait に渡されたアドレスにその子プロセスの終了ステータスをコピーする。
もし1つの子プロセスも終了していなかった場合、wait はどれか1つが終了するまで待つ。

If the caller has no children, wait immediately returns -1.

もし呼び出し側が子プロセスを持っていない場合、wait は直ちに -1 を返す。

If the parent doesn’t care about the exit status of a child, it can pass a 0 address to wait.

もし親が子の終了ステータスを気にしないならば、wait に 0 アドレスを渡すことができる。
(注: ヌルポインタ)

In the example, the output lines

この例では、出力行は

```text
parent: child=1234
child: exiting
```

might come out in either order (or even intermixed),
depending on whether the parent or child gets to its printf call first.

親と子のどちらが先に printf を呼ぶかに依存して、
どちらの順番にもなる可能性がある (または混ざるかもしれない)

After the child exits, the parent’s wait returns, causing the parent to print

子が終了した後、親の wait がリターンし、親は以下を出力する。

```text
parent: child 1234 is done
```

Although the child has the same memory contents as the parent initially,
the parent and child are executing with separate memory and separate registers:
changing a variable in one does not affect the other.

子は最初は親と同じメモリ内容を持つが、親と子は別々のメモリと別々のレジスタで実行される。
片方で変数を書き換えても、それはもう片方に影響を与えない。

For example, when the return value of wait is stored into pid in the parent process,
it doesn’t change the variable pid in the child.

例えば、wait の返り値が親プロセス内で pid に格納された時、
これは子プロセス内の pid 変数を変更する訳ではない。

The value of pid in the child will still be zero.

子プロセス内の pid はゼロのままとなる。

The exec system call replaces the calling process’s memory with a new memory image
loaded from a file stored in the file system.

exec システムコールは呼んだプロセスのメモリをファイルシステムに保存されている
ファイルからロードした新しいメモリイメージで置き換える。

The file must have a particular format, which specifies
which part of the file holds instructions,
which part is data, at which instruction to start, etc.

そのファイルは特定の形式である必要がある。
そのフォーマットでは以下が明記される。
そのファイルのどの部分が命令を保持しているか、
どの部分がデータなのか、どの命令からプログラムが開始するのか、等。

Xv6 uses the ELF format, which Chapter 3 discusses in more detail.

xv6 は ELF フォーマットを使う。これは3章でより詳しく議論する。

Usually the file is the result of compiling a program’s source code.

通常、この ELF ファイルはプログラムのソースコードをコンパイルした結果として得られる。

When exec succeeds, it does not return to the calling program; instead,
the instructions loaded from the file start executing at the entry point declared in the ELF header.

exec が成功した時、呼び出したプログラムへは返らない。
代わりに、ELF からロードされた命令列が、ELF ヘッダで宣言されたエントリポイントから
実行を開始する。

exec takes two arguments: the name of the file containing the executable and
an array of string arguments.

exec は2つの引数を取る。
実行可能データを含むファイル名と、文字列引数の配列である。
(注: 要はコマンド名と、コマンドラインパラメータ)

For example:

例えば、

```C
char *argv[3];
argv[0] = "echo";
argv[1] = "hello";
argv[2] = 0;
exec("/bin/echo", argv);
printf("exec error\n");
```

This fragment replaces the calling program with an instance of the program /bin/echo running
with the argument list echo hello.

このコード片は exec を呼んでいるプログラムを、引数リスト `echo hello` で実行している
`/bin/echo` プログラムのインスタンスに置き換える。

Most programs ignore the first element of the argument array,
which is conventionally the name of the program.

ほとんどのプログラムは引数配列の最初の要素を無視する。
これはプログラム名とするのが慣例である。
(注: `argv[0]`)

The xv6 shell uses the above calls to run programs on behalf of users.

xv6 シェルは上記のシステムコールを、ユーザの代わりにプログラムを実行するために使用する。
(注: ユーザというのは人間のこと。ユーザランドプログラムのことではない。)

The main structure of the shell is simple; see main (user/sh.c:146).

シェルのメイン構造はシンプルである。
main 関数を参照。`user/sh.c:146`

The main loop reads a line of input from the user with getcmd.

メインループは getcmd でユーザからの入力を1行読み取る。

Then it calls fork, which creates a copy of the shell process.

そして fork を呼び出し、fork はシェルプロセスのコピーを生成する。

The parent calls wait, while the child runs the command.

親は wait を呼び出し、その間に子はコマンドを実行する。

For example, if the user had typed “echo hello” to the shell,
runcmd would have been called with “echo hello” as the argument.

例えば、ユーザが "echo hello" とシェルにタイプしたとすると、
runcmd 関数が "echo hello" を引数として呼ばれることになる。

runcmd (user/sh.c:55) runs the actual command.

runcmd は実際のコマンドを実行する。

For “echo hello”, it would call exec (user/sh.c:79).

"echo hello" に対して、runcmd は exec を呼ぶ。

If exec succeeds then the child will execute instructions from echo instead of runcmd.

もし exec が成功したら、子プロセスが echo プログラムからの命令列を runcmd の代わりに実行する。

At some point echo will call exit, which will cause the parent to return from wait in main (user/sh.c:146).

あるところで echo は exit システムコールを呼び、
これにより親プロセスの main 関数内で wait が返ることになる。

You might wonder why fork and exec are not combined in a single call;
we will see later that the shell exploits the separation in its implementation of I/O redirection.

なぜ fork と exec は1つの呼び出しにまとめられていないのかと疑問に思うかもしれない。
シェルが I/O リダイレクションの実装において、この2つに分かれているのをうまく使っている
ところを後で見ていくことにする。

To avoid the wastefulness of creating a duplicate process and then immediately replacing it (with exec),
operating kernels optimize the implementation of fork for this use case
by using virtual memory techniques such as copy-on-write (see Section 4.6).

プロセスを複製してすぐに (exec で) 置き換えるという無駄を回避するため、
カーネルはコピーオンライトのような仮想メモリ技術を使ってこのユースケースのための
fork 実装を最適化している。

Xv6 allocates most user-space memory implicitly:
fork allocates the memory required for the child’s copy of the parent’s memory,
and exec allocates enough memory to hold the executable file.

xv6 はほとんどのユーザ空間メモリを暗黙のうちに確保する。
fork は親のメモリのコピーに必要な子のためのメモリを確保し、
exec は実行可能ファイルを置くのに十分な量のメモリを確保する。

A process that needs more memory at run-time (perhaps for malloc) can call sbrk(n) to
grow its data memory by n bytes; sbrk returns the location of the new memory.

実行時にもっとたくさんのメモリが必要になった (おそらく malloc のため) プロセスは、
sbrk(n) システムコールを呼び出してデータメモリを n バイト増やすことができる。
sbrk は新しいメモリ位置を返す。

## I/O and File descriptors

A file descriptor is a small integer representing a kernel-managed object
that a process may read from or write to.

ファイルディスクリプタはプロセスが読んだり書いたりできるカーネル管理のオブジェクトを
表す小さな整数である。

A process may obtain a file descriptor by opening a file, directory, or device,
or by creating a pipe, or by duplicating an existing descriptor.

プロセスはファイル、ディレクトリ、デバイスをオープンする、
パイプを作成する、既存のディスクリプタを複製することによって
ファイルティスクリプタを手に入れることができる。

For simplicity we’ll often refer to the object a file descriptor refers to as a “file”;
the file descriptor interface abstracts away the differences between files, pipes, and devices,
making them all look like streams of bytes.

簡単のため、ファイルディスクリプタが指すオブジェクトのことを "ファイル" と呼ぶことがしばしばある。
ファイルディスクリプタインタフェースはファイル、パイプ、デバイスの違いを隠して抽象化し、
すべてをバイトストリームのように見せる。

We’ll refer to input and output as I/O.

Input と Output (入出力) のことを I/O と呼ぶ。

Internally, the xv6 kernel uses the file descriptor as an index into a per-process table,
so that every process has a private space of file descriptors starting at zero.

内部的には、xv6 カーネルはファイルディスクリプタをプロセスごとのテーブルのインデックスとして
使用するため、すべてのプロセスはゼロから始まるファイルディスクリプタ空間を持つことになる。

By convention, a process reads from file descriptor 0 (standard input),
writes output to file descriptor 1 (standard output),
and writes error messages to file descriptor 2 (standard error).

慣例として、プロセスはファイルディスクリプタ 0 (標準入力) から読み取り、
出力をファイルディスクリプタ 1 (標準出力) に書き込み、
エラーメッセージをファイルディスクリプタ 2 (標準エラー出力) に書き込む。

As we will see, the shell exploits the convention to implement I/O redirection and pipelines.

これから見ていくように、シェルはこの慣例をうまく使って I/O リダイレクションや
パイプを実装する。

The shell ensures that it always has three file descriptors open (user/sh.c:152),
which are by default file descriptors for the console.

シェルは常に 3 つのファイルディスクリプタが開いていることを保証する。
これはデフォルトではコンソール用のファイルディスクリプタである。

The read and write system calls read bytes from and write bytes to open files named by file descriptors.

read と write システムコールはファイルディスクリプタによって指定された
開かれているファイルに対してバイト列を読んだり書いたりする。

The call read(fd, buf, n) reads at most n bytes from the file descriptor fd,
copies them into buf, and returns the number of bytes read.

`read(fd, buf, n)` 呼び出しは最大で n バイトをファイルディスクリプタ fd から読み取り、
buf へコピーし、読んだバイト数を返す。

Each file descriptor that refers to a file has an offset associated with it.

ファイルを参照しているそれぞれのファイルディスクリプタはそれに関連付けられたオフセットを持つ。

read reads data from the current file offset and then advances that offset
by the number of bytes read:
a subsequent read will return the bytes following the ones returned by the first read.

read は現在のファイルオフセットからデータを読み、読んだバイト数だけそのオフセットを進める。
次の read は最初の read が返したバイト列の次のバイト列を返すことになる。

When there are no more bytes to read, read returns zero to indicate the end of the file.

これ以上読むバイトがない場合は、read はファイルの終わり (EOF = end of file) を示すために
ゼロを返す。

The call write(fd, buf, n) writes n bytes from buf to the file descriptor fd
and returns the number of bytes written.

`write(fd, buf, n)` 呼び出しは n バイトを buf からファイルディスクリプタ fd に書き込み、
書き込んだバイト数を返す。

Fewer than n bytes are written only when an error occurs.

エラーが発生した場合のみ、n より小さいバイト数が書き込まれる。

Like read, write writes data at the current file offset and then advances that offset
by the number of bytes written:
each write picks up where the previous one left off.

read のように、write は現在のファイルオフセットにデータを書き込み、
そのオフセットを書き込んだバイト数だけ進める。

The following program fragment (which forms the essence of the program cat)
copies data from its standard input to its standard output.

次のプログラム片 (これは cat プログラムの主要な要素をなす) は標準入力からのデータを
標準出力にコピーする。

If an error occurs, it writes a message to the standard error.

エラーが発生した場合、メッセージを標準エラー出力に書き込む。

```C
char buf[512];
int n;

for(;;){
  13
  n = read(0, buf, sizeof buf);
  if(n == 0)
    break;
  if(n < 0){
    fprintf(2, "read error\n");
    exit(1);
  }
  if(write(1, buf, n) != n){
    fprintf(2, "write error\n");
    exit(1);
  }
}
```

The important thing to note in the code fragment is
that cat doesn’t know whether it is reading from a file, console, or a pipe.

このコードの中で特筆すべき点は、cat はファイルから読んでいるのか、コンソールからなのか、
パイプからなのか、を知らないということである。

Similarly cat doesn’t know whether it is printing to a console, a file, or whatever.

同様に、cat はコンソールに出力しているのか、ファイルになのか、他の何かになのかを知らないのである。

The use of file descriptors and the convention that file descriptor 0 is input and
file descriptor 1 is output allows a simple implementation of cat.

ファイルディスクリプタの使用とファイルディスクリプタ 0 が入力で 1 が出力という慣例によって
cat をシンプルに実装することができる。

The close system call releases a file descriptor, making it free for reuse by a future open,
pipe, or dup system call (see below).

close システムコールはファイルディスクリプタを解放し、将来の open, pipe, dup
システムコールで再利用できるできるようにする。(下記参照)

A newly allocated file descriptor is always the lowest-numbered unused descriptor of the current process.

新たに確保されるファイルディスクリプタは、現在のプロセスの中で使われていない番号のうち
常に一番小さいものとなる。

File descriptors and fork interact to make I/O redirection easy to implement.

ファイルディスクリプタと fork の組み合わせは I/O リダイレクションを実装しやすいものとする。

fork copies the parent’s file descriptor table along with its memory,
so that the child starts with exactly the same open files as the parent.

fork が親のファイルディスクリプタテーブルをメモリ内にコピーすることで、
子は親と完全に同じファイル群をオープンしている状態で開始する。

The system call exec replaces the calling process’s memory but preserves its file table.

exec システムコールは呼んだプロセスのメモリを置き換えるが、ファイルテーブルは
そのまま保持する。

This behavior allows the shell to implement I/O redirection by forking,
reopening chosen file descriptors in the child, and then calling exec to run the new program.

この挙動によりシェルは I/O リダイレクションを実装できる。
fork し、選択されたファイルディスクリプタを子プロセス内で reopen し、
exec を呼んで新しいプログラムを実行すればよい。

Here is a simplified version of the code a shell runs for the command cat < input.txt:

以下がシェルがコマンド `cat < input.txt` を実行する簡略化バージョンのコードである。

```C
char *argv[2];

argv[0] = "cat";
argv[1] = 0;
if(fork() == 0) {
  close(0);
  open("input.txt", O_RDONLY);
  exec("cat", argv);
}
```

After the child closes file descriptor 0, open is guaranteed to use
that file descriptor for the newly opened input.txt:
0 will be the smallest available file descriptor.

子がファイルディスクリプタ 0 を close した後の open は、
そのファイルディスクリプタ (0) を新たに open される input.txt に使うことが保証される。

cat then executes with file descriptor 0 (standard input) referring to input.txt.

cat はファイルディスクリプタ 0 (標準入力) が input.txt を指した状態で実行される。

The parent process’s file descriptors are not changed by this sequence,
since it modifies only the child’s descriptors.

親プロセスのファイルディスクリプタはこの処理の流れでは変更されない。
子のディスクリプタを変更するのみだからである。

The code for I/O redirection in the xv6 shell works in exactly this way (user/sh.c:83).

vx6 シェルの I/O リダイレクションのコードはまさにこの方法で動く(`user/sh.c:83)。

Recall that at this point in the code the shell has already forked the child shell and
that runcmd will call exec to load the new program.

コード内のこの時点でシェルは既に子を fork 完了していることと、
runcmd 関数は新しいプログラムをロードするためにこれから exec を呼ぶことを思い出そう。

The second argument to open consists of a set of flags, expressed as bits,
that control what open does.

open への2つ目の引数は、open が何をするかを制御するためのビットフラグのセットである。

The possible values are defined in the file control (fcntl) header (kernel/fcntl.h:1-5):
O_RDONLY, O_WRONLY, O_RDWR, O_CREATE, and O_TRUNC, which instruct open to open the file
for reading, or for writing, or for both reading and writing, to create the file if it doesn’t exist, and
to truncate the file to zero length.

指定可能な値は file control (fctrl) ヘッダ (`kernel/fcntl.h:1-5`) で定義されている。
O_RDONLY, O_WRONLY, O_RDWR, O_CREATE, O_TRUNC は open にそれぞれ
読み取り用、書き込み用、読み書き両用、ファイルが存在しなければ作成する、
ファイルを長さ 0 まで切り詰める、ということを指示する。

Now it should be clear why it is helpful that fork and exec are separate calls:
between the two, the shell has a chance to redirect the child’s I/O
without disturbing the I/O setup of the main shell.

これで fork と exec の呼び出しが分かれているのが役に立つ理由が明らかになったはずだ。
この2つの呼び出しの間で、シェルは親の I/O 設定を乱すことなく
子の I/O をリダイレクトする機会を得ることができる。

One could instead imagine a hypothetical combined forkexec system call,
but the options for doing I/O redirection with such a call seem awkward.

代わりに機能を合体させた forkexec システムコールを仮に考える者がいるかもしれないが、
その呼び出しに伴う I/O リダイレクションを行うためのオプションは不格好なものになるだろう。
(注: そういえば Windows の CreateProcess API の引数は見た目がまあまあひどいです。
直接そのことを言っているのかは不明。)

The shell could modify its own I/O setup before calling forkexec (and then un-do those modifications);
or forkexec could take instructions for I/O redirection as arguments;
or (least attractively) every program like cat could be taught to do its own I/O redirection.

シェルは forkexec を呼ぶ前に I/O 設定を変更する (そして呼出し後、変更を元に戻す) か、
forkexec が I/O リダイレクションの指定を引数として取るか、
(これは最も魅力的でないが) cat のようなそれぞれすべてのプログラムが
リダイレクトを自分で行うよう指示される、というような方法があり得るだろう。

Although fork copies the file descriptor table, each underlying file offset is shared between parent and child.

fork はファイルディスクリプタテーブルをコピーするが、
それぞれの内部にあるファイルオフセットは親と子の間で共有される。

Consider this example:

以下の例を考えよう

```C
if(fork() == 0) {
  write(1, "hello ", 6);
  exit(0);
} else {
  wait(0);
  write(1, "world\n", 6);
}
```

At the end of this fragment, the file attached to file descriptor 1 will contain the data hello world.

このコードの実行後、ファイルディスクリプタ 1 に関連付けられたファイルの中身は
"hello world" という内容になるだろう。

The write in the parent (which, thanks to wait, runs only after the child is done)
picks up where the child’s write left off.

親の write は (これは wait のおかげで子が完了した後にのみ実行される)、
子の write が終わった場所から行われる。

This behavior helps produce sequential output from sequences of shell commands,
like (echo hello; echo world) >output.txt.

この挙動は連続した出力を連続したシェルコマンドから生成するのに役立つ。
例えば `(echo hello; echo world) >output.txt` のようなもの。

The dup system call duplicates an existing file descriptor,
returning a new one that refers to the same underlying I/O object.

dup システムコールは既存のファイルディスクリプタを複製する(duplicate)。

Both file descriptors share an offset, just as the file descriptors
duplicated by fork do.

両方のファイルディスクリプタは、ちょうど fork で複製されたファイルディスクリプタと同じように、
オフセットを共有する。

This is another way to write hello world into a file:

以下は "hello world" をファイルに書き込むもう1つの方法である。

```C
fd = dup(1);
write(1, "hello ", 6);
write(fd, "world\n", 6);
```

Two file descriptors share an offset if they were derived from the same original file descriptor
by a sequence of fork and dup calls.

2つのファイルディスクリプタが、fork と dup の一連の呼び出しによって
同じオリジナルのファイルディスクリプタから作られた場合、
それらは1つのオフセットを共有する。

Otherwise file descriptors do not share offsets,
even if they resulted from open calls for the same file.

それ以外の場合、ファイルディスクリプタはオフセットを共有しない。
たとえ open を同じファイルに対して呼び出したとしてもである。

dup allows shells to implement commands like this:
ls existing-file non-existing-file > tmp1 2>&1.

dup はシェルでの以下のようなコマンドを可能にする。
`ls existing-file non-existing-file > tmp1 2>&1.`

The 2>&1 tells the shell to give the command a file descriptor 2
that is a duplicate of descriptor 1.

この `2>&1` はシェルに、そのコマンド (ls) に対して
ファイルディスクリプタ 1 を複製したものである ファイルディスクリプタ 2 を渡すよう指示している。
(注: close(2) して dup(1) すれば空いている最小番号である fd 2 が 1 の複製となる。
その前に `> tmp1` で close(1) の後 open("tmp1") により stdout=1 はファイルに
リダイレクトされているため、最終的に stderr=2 もファイルへ向くことになる。)

Both the name of the existing file and the error message for the non-existing file
will show up in the file tmp1.

存在するファイル名と、存在しないファイル名に対するエラーメッセージの両方が
tmp1 というファイルの中に現れることになるだろう。
(注: 存在するファイル名と存在しないファイル名の両方を渡された ls コマンドは
stdout=1 に正常な処理結果を、stderr=2 にエラーメッセージを出力するが、
`> tmp1` で stdout は `tmp1` というファイルにリダイレクトされており、
`2>&1` で stderr も同じ `tmp1` に出力されることになる。)

The xv6 shell doesn’t support I/O redirection for the error file descriptor,
but now you know how to implement it.

xv6 シェルは標準エラー出力の I/O リダイレクションはサポートしていないが、
今やその実装の仕方は分かったはずだ。
(注: わざとそのような余地を残しており、おそらく演習課題となる。)

File descriptors are a powerful abstraction,
because they hide the details of what they are connected to:
a process writing to file descriptor 1 may be writing to a file,
to a device like the console, or to a pipe.

ファイルディスクリプタは強力な抽象である。
それが具体的に何につながっているかの詳細を隠すからである。
あるプロセスがファイルディスクリプタ 1 に書いている時、
ファイルに書いているかもしれないし、コンソールのようなデバイスに書いているかもしれないし、
パイプに書いているかもしれない。

## Pipes

A pipe is a small kernel buffer exposed to processes as a pair of file descriptors, one for reading
and one for writing. Writing data to one end of the pipe makes that data available for reading from
the other end of the pipe. Pipes provide a way for processes to communicate.
The following example code runs the program wc with standard input connected to the read
end of a pipe.
int p[2];
char *argv[2];
argv[0] = "wc";
argv[1] = 0;
pipe(p);
if(fork() == 0) {
close(0);
dup(p[0]);
close(p[0]);
close(p[1]);
exec("/bin/wc", argv);
} else {
close(p[0]);
write(p[1], "hello world\n", 12);
close(p[1]);
}
The program calls pipe, which creates a new pipe and records the read and write file descriptors
in the array p. After fork, both parent and child have file descriptors referring to the pipe. The
child calls close and dup to make file descriptor zero refer to the read end of the pipe, closes the
file descriptors in p, and calls exec to run wc. When wc reads from its standard input, it reads from
the pipe. The parent closes the read side of the pipe, writes to the pipe, and then closes the write
side.
If no data is available, a read on a pipe waits for either data to be written or for all file descrip-
tors referring to the write end to be closed; in the latter case, read will return 0, just as if the end of
a data file had been reached. The fact that read blocks until it is impossible for new data to arrive
is one reason that it’s important for the child to close the write end of the pipe before executing
wc above: if one of wc ’s file descriptors referred to the write end of the pipe, wc would never see
end-of-file.
The xv6 shell implements pipelines such as grep fork sh.c | wc -l in a manner similar
to the above code (user/sh.c:101). The child process creates a pipe to connect the left end of the
pipeline with the right end. Then it calls fork and runcmd for the left end of the pipeline and
fork and runcmd for the right end, and waits for both to finish. The right end of the pipeline
may be a command that itself includes a pipe (e.g., a | b | c), which itself forks two new child
processes (one for b and one for c). Thus, the shell may create a tree of processes. The leaves
16
of this tree are commands and the interior nodes are processes that wait until the left and right
children complete.
Pipes may seem no more powerful than temporary files: the pipeline
echo hello world | wc
could be implemented without pipes as
echo hello world >/tmp/xyz; wc </tmp/xyz
Pipes have at least three advantages over temporary files in this situation. First, pipes automatically
clean themselves up; with the file redirection, a shell would have to be careful to remove /tmp/xyz
when done. Second, pipes can pass arbitrarily long streams of data, while file redirection requires
enough free space on disk to store all the data. Third, pipes allow for parallel execution of pipeline
stages, while the file approach requires the first program to finish before the second starts.

## File system

The xv6 file system provides data files, which contain uninterpreted byte arrays, and directories,
which contain named references to data files and other directories. The directories form a tree,
starting at a special directory called the root. A path like /a/b/c refers to the file or directory
named c inside the directory named b inside the directory named a in the root directory /. Paths
that don’t begin with / are evaluated relative to the calling process’s current directory, which can
be changed with the chdir system call. Both these code fragments open the same file (assuming
all the directories involved exist):
chdir("/a");
chdir("b");
open("c", O_RDONLY);
open("/a/b/c", O_RDONLY);
The first fragment changes the process’s current directory to /a/b; the second neither refers to nor
changes the process’s current directory.
There are system calls to create new files and directories: mkdir creates a new directory, open
with the O_CREATE flag creates a new data file, and mknod creates a new device file. This example
illustrates all three:
mkdir("/dir");
fd = open("/dir/file", O_CREATE|O_WRONLY);
close(fd);
mknod("/console", 1, 1);
mknod creates a special file that refers to a device. Associated with a device file are the major and
minor device numbers (the two arguments to mknod), which uniquely identify a kernel device.
When a process later opens a device file, the kernel diverts read and write system calls to the
kernel device implementation instead of passing them to the file system.
17
A file’s name is distinct from the file itself; the same underlying file, called an inode, can have
multiple names, called links. Each link consists of an entry in a directory; the entry contains a file
name and a reference to an inode. An inode holds metadata about a file, including its type (file or
directory or device), its length, the location of the file’s content on disk, and the number of links to
a file.
The fstat system call retrieves information from the inode that a file descriptor refers to. It
fills in a struct stat, defined in stat.h (kernel/stat.h) as:
#define T_DIR 1 // Directory
#define T_FILE 2 // File
#define T_DEVICE 3 // Device
struct stat {
int dev; // File system’s disk device
uint ino; // Inode number
short type; // Type of file
short nlink; // Number of links to file
uint64 size; // Size of file in bytes
};
The link system call creates another file system name referring to the same inode as an exist-
ing file. This fragment creates a new file named both a and b.
open("a", O_CREATE|O_WRONLY);
link("a", "b");
Reading from or writing to a is the same as reading from or writing to b. Each inode is identified
by a unique inode number. After the code sequence above, it is possible to determine that a and b
refer to the same underlying contents by inspecting the result of fstat: both will return the same
inode number (ino), and the nlink count will be set to 2.
The unlink system call removes a name from the file system. The file’s inode and the disk
space holding its content are only freed when the file’s link count is zero and no file descriptors
refer to it. Thus adding
unlink("a");
to the last code sequence leaves the inode and file content accessible as b. Furthermore,
fd = open("/tmp/xyz", O_CREATE|O_RDWR);
unlink("/tmp/xyz");
is an idiomatic way to create a temporary inode with no name that will be cleaned up when the
process closes fd or exits.
Unix provides file utilities callable from the shell as user-level programs, for example mkdir,
ln, and rm. This design allows anyone to extend the command-line interface by adding new user-
level programs. In hindsight this plan seems obvious, but other systems designed at the time of
Unix often built such commands into the shell (and built the shell into the kernel).
One exception is cd, which is built into the shell (user/sh.c:161). cd must change the current
working directory of the shell itself. If cd were run as a regular command, then the shell would
18
fork a child process, the child process would run cd, and cd would change the child ’s working
directory. The parent’s (i.e., the shell’s) working directory would not change.

## Real world

Unix’s combination of “standard” file descriptors, pipes, and convenient shell syntax for operations
on them was a major advance in writing general-purpose reusable programs. The idea sparked a
culture of “software tools” that was responsible for much of Unix’s power and popularity, and the
shell was the first so-called “scripting language.” The Unix system call interface persists today in
systems like BSD, Linux, and macOS.
The Unix system call interface has been standardized through the Portable Operating System
Interface (POSIX) standard. Xv6 is not POSIX compliant: it is missing many system calls (in-
cluding basic ones such as lseek), and many of the system calls it does provide differ from the
standard. Our main goals for xv6 are simplicity and clarity while providing a simple UNIX-like
system-call interface. Several people have extended xv6 with a few more system calls and a sim-
ple C library in order to run basic Unix programs. Modern kernels, however, provide many more
system calls, and many more kinds of kernel services, than xv6. For example, they support net-
working, windowing systems, user-level threads, drivers for many devices, and so on. Modern
kernels evolve continuously and rapidly, and offer many features beyond POSIX.
Unix unified access to multiple types of resources (files, directories, and devices) with a single
set of file-name and file-descriptor interfaces. This idea can be extended to more kinds of resources;
a good example is Plan 9 [16], which applied the “resources are files” concept to networks, graph-
ics, and more. However, most Unix-derived operating systems have not followed this route.
The file system and file descriptors have been powerful abstractions. Even so, there are other
models for operating system interfaces. Multics, a predecessor of Unix, abstracted file storage in a
way that made it look like memory, producing a very different flavor of interface. The complexity
of the Multics design had a direct influence on the designers of Unix, who aimed to build something
simpler.
Xv6 does not provide a notion of users or of protecting one user from another; in Unix terms,
all xv6 processes run as root.
This book examines how xv6 implements its Unix-like interface, but the ideas and concepts
apply to more than just Unix. Any operating system must multiplex processes onto the underlying
hardware, isolate processes from each other, and provide mechanisms for controlled inter-process
communication. After studying xv6, you should be able to look at other, more complex operating
systems and see the concepts underlying xv6 in those systems as well.

## Exercises

1. Write a program that uses UNIX system calls to “ping-pong” a byte between two processes
over a pair of pipes, one for each direction. Measure the program’s performance, in ex-
changes per second.
