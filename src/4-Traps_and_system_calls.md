# Traps and system calls

トラップとシステムコール

There are three kinds of event which cause the CPU to set aside
ordinary execution of instructions and force a transfer of control to special code
that handles the event.

CPU が通常の命令実行を一旦やめ、そのイベントをハンドルするための特別なコードに
コントロールを強制的に移すようなものとして、3つの種類のイベントがある。

One situation is a system call, when a user program executes the ecall instruction
to ask the kernel to do something for it.

1つ目の状況はシステムコールで、
ユーザプログラムがカーネルに何かをするよう依頼するために ecall 命令を実行した時である。

Another situation is an exception:
an instruction (user or kernel) does something illegal,
such as divide by zero or use an invalid virtual address.

2つ目の状況は例外である。
ある命令 (ユーザまたはカーネル) が  何か不正なことをした、
例えば 0 で除算を行った、または不正な仮想アドレスを使用したといった場合である。

The third situation is a device interrupt,
when a device signals that it needs attention,
for example when the disk hardware finishes a read or write request.

3つ目の状況はデバイス割り込みで、
あるデバイスが注意が必要であると知らせてきた場合である。
例えばディスクハードウェアが読み取りまたは書き込みリクエストを完了した、等。

This book uses trap as a generic term for these situations.

本書ではトラップをこれらの状況を表す一般的な用語として使用する。

Typically whatever code was executing at the time of the trap will later need to resume,
and shouldn’t need to be aware that anything special happened.

通常、トラップが発生した時点で実行中だったどんなコードも後で再開する必要があり、
何か特別なことが起きたことを知る必要があるべきではない。

That is, we often want traps to be transparent;
this is particularly important for device interrupts,
which the interrupted code typically doesn’t expect.

つまり、我々は通常トラップは透過的であって欲しいと考える。
これは特にデバイス割り込みに対して重要であり、
割り込まれるコードは通常デバイス割り込みを想定していない。

The usual sequence is that a trap forces a transfer of control into the kernel;
the kernel saves registers and other state so that execution can be resumed;
the kernel executes appropriate handler code (e.g., a system call implementation or device driver);
the kernel restores the saved state and returns from the trap; and the original code resumes where it left off.

通常のシーケンスではトラップはコントロールを強制的にカーネル内に移す。
カーネルは実行を後で再開できるようにレジスタとその他の状態をセーブする。
カーネルは適切なハンドラコード (例: システムコール実装やデバイスドライバ) を実行する。
カーネルはセーブしていた状態をリストアし、トラップからリターンする。
元のコードは中断した場所から再開する。

Xv6 handles all traps in the kernel; traps are not delivered to user code.

xv6 はすべてのトラップをカーネル内でハンドルする。
トラップはユーザコードには配送されない。

Handling traps in the kernel is natural for system calls.

トラップをカーネル内でハンドルするのはシステムコールに対しては自然である。

It makes sense for interrupts
since isolation demands that only the kernel be allowed to use devices,
and because the kernel is a convenient mechanism with which to share devices among multiple processes.

割り込みに対しても理にかなっている。
なぜなら分離の考え方がカーネルのみがデバイスを使えることを要求するからである。
また、カーネルはデバイスを複数のプロセス間で共有する便利な方法だからでもある。

It also makes sense for exceptions
since xv6 responds to all exceptions from user space by killing the offending program.

例外に対しても理にかなっている。
xv6 はユーザ空間からの全ての例外に対して問題を起こしているプログラムを kill することによって
応答するからだ。

Xv6 trap handling proceeds in four stages:
hardware actions taken by the RISC-V CPU,
some assembly instructions that prepare the way for kernel C code,
a C function that decides what to do with the trap,
and the system call or device-driver service routine.

Xv6 のトラップハンドリングは4つのステージで進む。
RISC-V CPU によって行われるハードウェアアクション、
カーネル C コードへの道を準備するいくつかのアセンブリ命令、
そのトラップに対して何をするかを決定する C 関数、
システムコールまたはデバイスドライバのサービスルーチン。

While commonality among the three trap types suggests that
a kernel could handle all traps with a single code path,
it turns out to be convenient to have separate code for three distinct cases:
traps from user space, traps from kernel space, and timer interrupts.

3つのタイプのトラップの共通性から、カーネルは1つのコードパスで全てのトラップを
処理することもできると考えられるが、
3つの異なるケースそれぞれに別のコードを用意した方が便利であるとわかる。
ユーザ空間からのトラップ、カーネル空間からのトラップ、タイマ割り込み。

Kernel code (assembler or C) that processes a trap is often called a handler;
the first handler instructions are usually written in assembler (rather than C) and
are sometimes called a vector.

トラップを処理するカーネルコード (アセンブラまたは C) はよく (割り込み) ハンドラと呼ばれる。
最初のハンドラ命令は通常 (C よりも) アセンブラで書かれ、(割り込み) ベクタと呼ばれることがある。

## RISC-V trap machinery

RISC-V のトラップ機構

Each RISC-V CPU has a set of control registers that the kernel writes to tell the CPU
how to handle traps, and that the kernel can read to find out about
a trap that has occurred.

各 RISC-V CPU はカーネルがどのようにトラップをハンドルするかを CPU に教えるために書き込む
コントロールレジスタのセットを持っている。

The RISC-V documents contain the full story [3].

RISC-V のドキュメントにその全容が書かれている。

riscv.h (kernel/riscv.h:1) contains definitions that xv6 uses.

riscv.h `(kernel/riscv.h:1)` に xv6 が使う定義が書かれている。

Here’s an outline of the most important registers:

以下が最も重要なレジスタ群の概要である。

* stvec: The kernel writes the address of its trap handler here;
  the RISC-V jumps to the address in stvec to handle a trap.
* stvec: カーネルはトラップハンドラのアドレスをここに書く。
  RISC-V はトラップをハンドルするために stvec 内のアドレスにジャンプする。
* sepc: When a trap occurs, RISC-V saves the program counter here
  (since the pc is then overwritten with the value in stvec).
  The sret (return from trap) instruction copies sepc to the pc.
  The kernel can write sepc to control where sret goes.
* sepc: トラップが発生した時、RISC-V はプログラムカウンタをここに保存する
  (PC は stvec 内の値で上書きされてしまうため)。
  sret (トラップから戻る) 命令は sepc を pc にコピーする。
  カーネルは sepc に書き込むことで sret が行く先を制御することができる。
* scause: RISC-V puts a number here that describes the reason for the trap.
* RISC-V はトラップの理由を示す番号をここに置く。
* sscratch: The trap handler code uses sscratch to help it
  avoid overwriting user registers before saving them.
* sscratch: トラップハンドラコードはユーザレジスタを保存する前に上書きしてしまうことを
  回避するためにsscratch を使う。
* sstatus: The SIE bit in sstatus controls whether device interrupts are enabled.
  If the kernel clears SIE, the RISC-V will defer device interrupts
  until the kernel sets SIE.
  The SPP bit indicates whether a trap came from user mode or supervisor mode,
  and controls to what mode sret returns.
* sstatus: sstatus の中の SIE ビットはデバイス割り込みが有効かどうかを制御する。
  もしカーネルが SIE をクリアしたなら、RISC-V はデバイス割り込みをカーネルが SIE を
  セットするまで遅延させる。
  SPP ビットはトラップがユーザモードかスーパーバイザモードのどちらから来たのかを示し、
  sret がどのモードに返るのかを制御する。

The above registers relate to traps handled in supervisor mode,
and they cannot be read or written in user mode.

上記のレジスタはスーパーバイザモードでハンドルされるトラップに関係しており、
ユーザモードでは読み書きができない。

There is a similar set of control registers for traps handled in machine mode;
xv6 uses them only for the special case of timer interrupts.

マシンモードでハンドルされるトラップに関しても似たレジスタセットがある。
xv6 はタイマ割り込みという特別なケースにのみ使っている。

Each CPU on a multi-core chip has its own set of these registers,
and more than one CPU may be handling a trap at any given time.

マルチコアチップの各 CPU はそれぞれ独自にこれらのレジスタを持っており、
任意の時点で複数の CPU がトラップを処理している可能性がある。

When it needs to force a trap, the RISC-V hardware does the following
for all trap types (other than timer interrupts):

トラップを発生させる必要がある時、RISC-V ハードウェアはすべてのトラップタイプ
(タイマ割り込みを除く) に対して以下を行う。

1. If the trap is a device interrupt, and the sstatus SIE bit is clear,
  don’t do any of the following.
1. Disable interrupts by clearing the SIE bit in sstatus.
1. Copy the pc to sepc.
1. Save the current mode (user or supervisor) in the SPP bit in sstatus.
1. Set scause to reflect the trap’s cause.
1. Set the mode to supervisor.
1. Copy stvec to the pc.
1. Start executing at the new pc.

訳

1. トラップがデバイス割り込みの場合、かつ sstatus SIE ビットがクリアされている場合、
  以下は一切行わない。
1. sstatus 内の SIE ビットをクリアすることによって割り込みを無効にする。
1. pc を sepc にコピーする。
1. 現在のモード (ユーザまたはスーパーバイザ) を sstatus の SPP ビットに保存する。
1. scause にトラップの原因を反映させる。
1. モードをスーパーバイザに設定する。
1. stvec を pc にコピーする。
1. 新しい pc から実行を開始する。

Note that the CPU doesn’t switch to the kernel page table,
doesn’t switch to a stack in the kernel,
and doesn’t save any registers other than the pc.

CPU はカーネルページテーブルを切り替えないこと、カーネルスタックに切り替えないこと、
pc 以外の一切のレジスタを保存しないことに注意が必要である。

Kernel software must perform these tasks.

カーネルソフトウェアがこれらの仕事を行わなければならない。

One reason that the CPU does minimal work during a trap is
to provide flexibility to software;
for example, some operating systems omit a page table switch in some situations
to increase trap performance.

CPU がトラップ中に最小限の仕事しか行わない一つの理由は、
ソフトウェアに柔軟性を提供することである。
例えば、オペレーティングシステムの中にはある状況下でトラップのパフォーマンスを上げるために
ページテーブルのスイッチを省略するものもある。

It’s worth thinking about whether any of the steps listed above could be omitted,
perhaps in search of faster traps.

より高速なトラップ処置を求めて
上に挙げたステップのうちどれかを省略できないか考えてみる価値はあるだろう。

Though there are situations in which a simpler sequence can work,
many of the steps would be dangerous to omit in general.

もっと簡単なシーケンスでうまくいく状況もあるが、
一般的には省略するのは危険なステップが多い。

For example, suppose that the CPU didn’t switch program counters.

例えば、CPU がプログラムカウンタを切り替えなかったとしよう。

Then a trap from user space could switch to supervisor mode while still running user instructions.

その場合、ユーザ空間からのトラップはユーザ命令を実行中にスーパーバイザモードに
切り替えられることになる。

Those user instructions could break user/kernel isolation,
for example by modifying the satp register to point to a page table
that allowed accessing all of physical memory.

そのようなユーザ命令はユーザ/カーネルの分離を破壊できてしまう。
例えば satp レジスタを全ての物理メモリへのアクセス許すページテーブルを指すように
変更することによって。

It is thus important that the CPU switch to a kernel-specified instruction address,
namely stvec.

したがって CPU がカーネルの指定した命令アドレス、つまり stvec に切り替えるのは
重要なことである。

## Traps from user space

ユーザ空間からのトラップ

Xv6 handles traps differently depending on whether the trap occurs while executing in the kernel
or in user code.

xv6 はトラップがカーネルまたはユーザコードのどちらを実行中に起きたかに応じて
ハンドルの仕方を変える。

Here is the story for traps from user code;
Section 4.5 describes traps from kernel code.

ここではユーザコードのトラップについて説明する。
4.5 節でカーネルコードからのトラップを説明する。

A trap may occur while executing in user space
if the user program makes a system call (ecall instruction),
or does something illegal, or if a device interrupts.

ユーザ空間で実行中にトラップが発生する可能性があるのは、
ユーザプログラムがシステムコールを呼んだ (ecall 命令) 場合、
または何か不正なことをした場合、
またはデバイスが割り込みをかけてきた場合である。

The high-level path of a trap from user space is uservec (kernel/trampoline.S:21),
then usertrap (kernel/trap.c:37);
and when returning,
usertrapret (kernel/trap.c:90) and then userret (kernel/trampoline.S:101).

ユーザ空間からのトラップの大まかな経路は
uservec (`kernel/trampoline.S:21`)、usertrap (`kernel/trap.c:37`)、
そして戻る時は
usertrapret (`kernel/trap.c:90`) そして userret (`kernel/trampoline.S:101`) である。

A major constraint on the design of xv6’s trap handling is
the fact that the RISC-V hardware does not switch page tables when it forces a trap.

xv6 のトラップハンドリング設計上の大きな制約は、RISC-V ハードウェアがトラップする時に
ページテーブルを切り替えないという事実である。

This means that the trap handler address in stvec must have a valid mapping
in the user page table, since that’s the page table in force
when the trap handling code starts executing.

これは stvec 内のトラップハンドラアドレスがユーザページテーブルで有効なマッピングを
持っていなければならないということを意味する。
これはトラップハンドリングコードが実行を開始した時に有効なページテーブルだからだ。

Furthermore, xv6’s trap handling code needs to switch to
the kernel page table;
in order to be able to continue executing after that switch,
the kernel page table must also have a mapping for the handler pointed to by stvec.

さらに、xv6 のトラップハンドリングコードはカーネルページテーブルに切り替える必要がある。
その切り替え後に実行を続けられるようにするために、カーネルページテーブルにも
stvec が指すハンドラへのマッピングが必要である。

Xv6 satisfies these requirements using a trampoline page.

xv6 はこれらの要件をトランポリンページを使うことで満たしている。

The trampoline page contains uservec, the xv6 trap handling code that stvec points to.

トランポリンページには uservec が含まれており、
uservec は stvec が指す先の、xv6 のトラップハンドリングコードである。

The trampoline page is mapped in every process’s page table at address TRAMPOLINE,
which is at the top of the virtual address space so that
it will be above memory that programs use for themselves.

このトランポリンページは全てのプロセスのページテーブルのアドレス TRAMPOLINE にマップされており、
それはプログラムが自分のために使うメモリより上位になるよう、仮想アドレス空間の最上位にある。

The trampoline page is also mapped at address TRAMPOLINE in the kernel page table.

このトランポリンページはカーネルページテーブルのアドレス TRANPOLINE にもマップされている。

See Figure 2.3 and Figure 3.3.

図 2.3 と図3.3 を見よ。

Because the trampoline page is mapped in the user page table,
without the PTE_U flag, traps can start executing there in supervisor mode.

トランポリンページはユーザページテーブルにマップされているため (PTE_U フラグはなし)、
トラップはスーパバイザモードでそこから実行を開始することができる。

Because the trampoline page is mapped at the same address in the kernel address space,
the trap handler can continue to execute after it switches to the kernel page table.

トランポリンページはカーネルアドレス空間の同じアドレスにマップされているので、
トラップハンドラはカーネルページテーブルに切り替えた後も実行を続けることができる。

The code for the uservec trap handler is in trampoline.S (kernel/trampoline.S:21).

uservec トラップハンドラのためのコードは trampoline.S (`kernel/trampoline.S:21`) にある。

When uservec starts, all 32 registers contain values owned by the interrupted user code.

uservec の開始時、全ての 32 個のレジスタには割り込まれたユーザコードが持っていた値が入っている。

These 32 values need to be saved somewhere in memory,
so that they can be restored when the trap returns to user space.

これらの 32 個の値はメモリのどこかにセーブしておく必要があり、
そうすることでトラップがユーザ空間に戻る時にリストアすることができる。

Storing to memory requires use of a register to hold the address,
but at this point there are no general-purpose registers available!

メモリへの格納にはアドレスを保持するレジスタを使用する必要性があるが、
この時点では使用可能な汎用レジスタがない！

Luckily RISC-V provides a helping hand in the form of the sscratch register.

幸運にも RISC-V は sscratch レジスタという形で救いの手を差し伸べてくれる。

The csrw instruction at the start of uservec saves a0 in sscratch.

uservec の始まりにある scrw 命令が a0 を sscratch にセーブする。

Now uservec has one register (a0) to play with.

これで uservec には1つのレジスタ (a0) の遊びができた。

uservec’s next task is to save the 32 user registers.

uservec の次の仕事は 32 個のユーザレジスタをセーブすることだ。

The kernel allocates, for each process, a page of memory for a trapframe structure
that (among other things) has space to save the 32 user registers (kernel/proc.h:43).

カーネルはプロセスごとに、32 個のユーザレジスタをセーブするための領域を含む
トラップフレーム構造体のための1ページのメモリを確保している。

Because satp still refers to the user page table,
uservec needs the trapframe to be mapped in the user address space.

satp はまだユーザページテーブルを指しているため、
uservec にとってはトラップフレームはユーザアドレス空間にマップされていなければならない。

Xv6 maps each process’s trapframe at virtual address TRAPFRAME
in that process’s user page table;
TRAPFRAME is just below TRAMPOLINE.

xv6 は各プロセスのトラップフレームをそのプロセスのユーザページテーブルの
仮想アドレス TRAPFRAM にマップする。
TRAPFRAME は TRAMPOLINE のちょうどすぐ下である。

The process’s p->trapframe also points to the trapframe,
though at its physical address so the kernel can use it through the kernel page table.

プロセスの `p->trapframe` も物理アドレスではあるがトラップフレームを指すので、
カーネルはカーネルページテーブルからもそれを使うことができる。

Thus uservec loads address TRAPFRAME into a0 and saves all the user registers there,
including the user’s a0, read back from sscratch.

uservec はアドレス TRAPFRAME を a0 にロードし、そこに全てのユーザレジスタをセーブする。
これにはユーザの a0 を含む。これは sscratch から読み戻せばよい。

The trapframe contains the address of the current process’s kernel stack,
the current CPU’s hartid, the address of the usertrap function,
and the address of the kernel page table.

トラップフレームには現在のプロセスのカーネルスタックのアドレス、
現在の CPU の hartid、ユーザトラップ関数のアドレス、
カーネルページテーブルのアドレスが含まれている。

uservec retrieves these values, switches satp to the kernel page table,
and calls usertrap.

uservec はそれらの値を取得し、satp をカーネルページテーブルに切り替え、
usertrap を呼ぶ。

The job of usertrap is to determine the cause of the trap, process it,
and return (kernel/trap.c:37).

usertrap の仕事はトラップの原因を判定し、それを処理し、リターンすることである
(`kernel/trap.c:3`)。

It first changes stvec so that a trap while in the kernel
will be handled by kernelvec rather than uservec.

usertrap はまず stvec を、カーネル中でのトラップが uservec ではなく kernelvec で
ハンドルされるように変更する。

It saves the sepc register (the saved user program counter),
because usertrap might call yield to switch to another process’s kernel thread,
and that process might return to user space,
in the process of which it will modify sepc.

usertrap は sepc (割り込み前のプログラムカウンタ) レジスタをセーブする。
これは usertrap は yield を呼び出して他のプロセスのカーネルスレッドに切り替える可能性があり、
そのプロセスはユーザ空間にリターンする可能性があるからである。
その過程で sepc は変更されることになる。

If the trap is a system call, usertrap calls syscall to handle it;
if a device interrupt, devintr;
otherwise it’s an exception, and the kernel kills the faulting process.

もしトラップがシステムコールなら、usertrap はそれを処理するために syscall を呼び出す。
もしデバイス割り込みなら、devintr を呼び出す。
それ以外なら例外であり、カーネルは問題を起こしているプロセスを kill する。

The system call path adds four to the saved user program counter
because RISC-V, in the case of a system call, leaves the program pointer
pointing to the ecall instruction
but user code needs to resume executing at the subsequent instruction.

システムコールの経路ではセーブされているユーザのプログラムカウンタに 4 を足す。
これは RISC-V はシステムコールの場合、プログラムカウンタを ecall 命令を指したままにするが、
ユーザコードは次の命令から実行を再開する必要があるからだ。

On the way out, usertrap checks if the process has been killed or
should yield the CPU (if this trap is a timer interrupt).

出口で、usertrap はプロセスが kill されていないか、または CPU を明け渡すべきでないか
(もしこのトラップがタイマ割り込みだった場合)、を確認する。

The first step in returning to user space is the call to usertrapret (kernel/trap.c:90).

ユーザ空間へ戻る時の最初のステップは usertrapret (`kernel/trap.c:90`) を呼び出すことだ。

This function sets up the RISC-V control registers to prepare for a future trap from user space.

この関数はこれから先のユーザ空間からのトラップに備えるために、
RISC-V のコントロールレジスタを設定する。

This involves changing stvec to refer to uservec,
preparing the trapframe fields that uservec relies on,
and setting sepc to the previously saved user program counter.

これには stvec を uservec を指すように切り替えること、
uservec が頼っているトラップフレームのフィールドを準備すること、
sepc を以前にセーブしたユーザプログラムカウンタの値に設定すること、が含まれる。

At the end, usertrapret calls userret on the trampoline page
that is mapped in both user and kernel page tables;
the reason is that assembly code in userret will switch page tables.

最後に、usertrapret はユーザとカーネル両方のページテーブルにマップされている
トランポリンページ上の userret を呼ぶ。
その理由は userret 内のアセンブリコードがページテーブルを切り替えるからである。

usertrapret’s call to userret passes a pointer to the process’s user page table
in a0 (kernel/trampoline.S:101).

usertrapret による userret の呼び出しでは、プロセスのユーザページテーブルへのポインタを
a0 で渡す (`kernel/trampoline.S:101`)。

userret switches satp to the process’s user page table.

userret は sapt をプロセスのユーザページテーブルへ切り替える。

Recall that the user page table maps both the trampoline page and TRAPFRAME,
but nothing else from the kernel.

ユーザページテーブルはトランポリンページとトラップフレームの両方をマップしているが、
その他には何もカーネルからマップしていないことを思い出そう。

The trampoline page mapping at the same virtual address in user and kernel page tables
allows userret to keep executing after changing satp.

トランポリンページがユーザとカーネルページテーブルで同じ仮想アドレスにマップされていることで、
userret は satp を変更した後もそのまま実行を続けることができる。

From this point on, the only data userret can use is
the register contents and the content of the trapframe.

この時点から、userret が使えるデータはレジスタの内容とトラップフレームの内容だけになる。

userret loads the TRAPFRAME address into a0,
restores saved user registers from the trapframe via a0,
restores the saved user a0, and executes sret to return to user space.

userret は TRAPFRAME アドレスを a0 にロードし、
セーブされていたユーザレジスタをトラップフレームから a0 経由でリストアし、
セーブされていた a0 をリストアし、そして sret を実行してユーザ空間に戻る。

## Code: Calling system calls

Chapter 2 ended with initcode.S invoking the exec system call (user/initcode.S:11). Let’s look
at how the user call makes its way to the exec system call’s implementation in the kernel.
initcode.S places the arguments for exec in registers a0 and a1, and puts the system call
number in a7. System call numbers match the entries in the syscalls array, a table of function
pointers (kernel/syscall.c:107). The ecall instruction traps into the kernel and causes uservec,
usertrap, and then syscall to execute, as we saw above.
syscall (kernel/syscall.c:132) retrieves the system call number from the saved a7 in the trapframe
and uses it to index into syscalls. For the first system call, a7 contains SYS_exec (ker-
nel/syscall.h:8), resulting in a call to the system call implementation function sys_exec.
When sys_exec returns, syscall records its return value in p->trapframe->a0. This will
cause the original user-space call to exec() to return that value, since the C calling convention
on RISC-V places return values in a0. System calls conventionally return negative numbers to
indicate errors, and zero or positive numbers for success. If the system call number is invalid,
syscall prints an error and returns −1.
4.4 Code: System call arguments
System call implementations in the kernel need to find the arguments passed by user code. Because
user code calls system call wrapper functions, the arguments are initially where the RISC-V C
calling convention places them: in registers. The kernel trap code saves user registers to the current
process’s trap frame, where kernel code can find them. The kernel functions argint, argaddr,
and argfd retrieve the n ’th system call argument from the trap frame as an integer, pointer, or a file
descriptor. They all call argraw to retrieve the appropriate saved user register (kernel/syscall.c:34).
Some system calls pass pointers as arguments, and the kernel must use those pointers to read
or write user memory. The exec system call, for example, passes the kernel an array of pointers
referring to string arguments in user space. These pointers pose two challenges. First, the user pro-
gram may be buggy or malicious, and may pass the kernel an invalid pointer or a pointer intended
to trick the kernel into accessing kernel memory instead of user memory. Second, the xv6 kernel
page table mappings are not the same as the user page table mappings, so the kernel cannot use
ordinary instructions to load or store from user-supplied addresses.
The kernel implements functions that safely transfer data to and from user-supplied addresses.
fetchstr is an example (kernel/syscall.c:25). File system calls such as exec use fetchstr to
retrieve string file-name arguments from user space. fetchstr calls copyinstr to do the hard
work.
copyinstr (kernel/vm.c:403) copies up to max bytes to dst from virtual address srcva in
the user page table pagetable. Since pagetable is not the current page table, copyinstr
uses walkaddr (which calls walk) to look up srcva in pagetable, yielding physical address
pa0. The kernel maps each physical RAM address to the corresponding kernel virtual address,
so copyinstr can directly copy string bytes from pa0 to dst. walkaddr (kernel/vm.c:109)
checks that the user-supplied virtual address is part of the process’s user address space, so programs
47
cannot trick the kernel into reading other memory. A similar function, copyout, copies data from
the kernel to a user-supplied address.
4.5 Traps from kernel space
Xv6 configures the CPU trap registers somewhat differently depending on whether user or kernel
code is executing. When the kernel is executing on a CPU, the kernel points stvec to the assembly
code at kernelvec (kernel/kernelvec.S:12). Since xv6 is already in the kernel, kernelvec can
rely on satp being set to the kernel page table, and on the stack pointer referring to a valid kernel
stack. kernelvec pushes all 32 registers onto the stack, from which it will later restore them so
that the interrupted kernel code can resume without disturbance.
kernelvec saves the registers on the stack of the interrupted kernel thread, which makes
sense because the register values belong to that thread. This is particularly important if the trap
causes a switch to a different thread – in that case the trap will actually return from the stack of the
new thread, leaving the interrupted thread’s saved registers safely on its stack.
kernelvec jumps to kerneltrap (kernel/trap.c:135) after saving registers. kerneltrap
is prepared for two types of traps: device interrupts and exceptions. It calls devintr (kernel/-
trap.c:178) to check for and handle the former. If the trap isn’t a device interrupt, it must be an
exception, and that is always a fatal error if it occurs in the xv6 kernel; the kernel calls panic and
stops executing.
If kerneltrap was called due to a timer interrupt, and a process’s kernel thread is running
(as opposed to a scheduler thread), kerneltrap calls yield to give other threads a chance to
run. At some point one of those threads will yield, and let our thread and its kerneltrap resume
again. Chapter 7 explains what happens in yield.
When kerneltrap’s work is done, it needs to return to whatever code was interrupted
by the trap. Because a yield may have disturbed sepc and the previous mode in sstatus,
kerneltrap saves them when it starts. It now restores those control registers and returns to
kernelvec (kernel/kernelvec.S:50). kernelvec pops the saved registers from the stack and ex-
ecutes sret, which copies sepc to pc and resumes the interrupted kernel code.
It’s worth thinking through how the trap return happens if kerneltrap called yield due to
a timer interrupt.
Xv6 sets a CPU’s stvec to kernelvec when that CPU enters the kernel from user space;
you can see this in usertrap (kernel/trap.c:29). There’s a window of time when the kernel has
started executing but stvec is still set to uservec, and it’s crucial that no device interrupt occur
during that window. Luckily the RISC-V always disables interrupts when it starts to take a trap,
and xv6 doesn’t enable them again until after it sets stvec.

4.6 Page-fault exceptions
Xv6’s response to exceptions is quite boring: if an exception happens in user space, the kernel
kills the faulting process. If an exception happens in the kernel, the kernel panics. Real operating
48
systems often respond in much more interesting ways.
As an example, many kernels use page faults to implement copy-on-write (COW) fork. To
explain copy-on-write fork, consider xv6’s fork, described in Chapter 3. fork causes the child’s
initial memory content to be the same as the parent’s at the time of the fork. Xv6 implements fork
with uvmcopy (kernel/vm.c:306), which allocates physical memory for the child and copies the
parent’s memory into it. It would be more efficient if the child and parent could share the parent’s
physical memory. A straightforward implementation of this would not work, however, since it
would cause the parent and child to disrupt each other’s execution with their writes to the shared
stack and heap.
Parent and child can safely share physical memory by appropriate use of page-table permissions
and page faults. The CPU raises a page-fault exception when a virtual address is used that has no
mapping in the page table, or has a mapping whose PTE_V flag is clear, or a mapping whose
permission bits (PTE_R, PTE_W, PTE_X, PTE_U) forbid the operation being attempted. RISC-V
distinguishes three kinds of page fault: load page faults (when a load instruction cannot translate its
virtual address), store page faults (when a store instruction cannot translate its virtual address), and
instruction page faults (when the address in the program counter doesn’t translate). The scause
register indicates the type of the page fault and the stval register contains the address that couldn’t
be translated.
The basic plan in COW fork is for the parent and child to initially share all physical pages,
but for each to map them read-only (with the PTE_W flag clear). Parent and child can read from
the shared physical memory. If either writes a given page, the RISC-V CPU raises a page-fault
exception. The kernel’s trap handler responds by allocating a new page of physical memory and
copying into it the physical page that the faulted address maps to. The kernel changes the relevant
PTE in the faulting process’s page table to point to the copy and to allow writes as well as reads,
and then resumes the faulting process at the instruction that caused the fault. Because the PTE
allows writes, the re-executed instruction will now execute without a fault. Copy-on-write requires
book-keeping to help decide when physical pages can be freed, since each page can be referenced
by a varying number of page tables depending on the history of forks, page faults, execs, and exits.
This book-keeping allows an important optimization: if a process incurs a store page fault and the
physical page is only referred to from that process’s page table, no copy is needed.
Copy-on-write makes fork faster, since fork need not copy memory. Some of the memory
will have to be copied later, when written, but it’s often the case that most of the memory never
has to be copied. A common example is fork followed by exec: a few pages may be written after
the fork, but then the child’s exec releases the bulk of the memory inherited from the parent.
Copy-on-write fork eliminates the need to ever copy this memory. Furthermore, COW fork is
transparent: no modifications to applications are necessary for them to benefit.
The combination of page tables and page faults opens up a wide range of interesting possibil-
ities in addition to COW fork. Another widely-used feature is called lazy allocation, which has
two parts. First, when an application asks for more memory by calling sbrk, the kernel notes the
increase in size, but does not allocate physical memory and does not create PTEs for the new range
of virtual addresses. Second, on a page fault on one of those new addresses, the kernel allocates a
page of physical memory and maps it into the page table. Like COW fork, the kernel can implement
49
lazy allocation transparently to applications.
Since applications often ask for more memory than they need, lazy allocation is a win: the
kernel doesn’t have to do any work at all for pages that the application never uses. Furthermore,
if the application is asking to grow the address space by a lot, then sbrk without lazy allocation
is expensive: if an application asks for a gigabyte of memory, the kernel has to allocate and zero
262,144 4096-byte pages. Lazy allocation allows this cost to be spread over time. On the other
hand, lazy allocation incurs the extra overhead of page faults, which involve a kernel/user transi-
tion. Operating systems can reduce this cost by allocating a batch of consecutive pages per page
fault instead of one page and by specializing the kernel entry/exit code for such page-faults.
Yet another widely-used feature that exploits page faults is demand paging. In exec, xv6 loads
all text and data of an application eagerly into memory. Since applications can be large and reading
from disk is expensive, this startup cost may be noticeable to users: when the user starts a large
application from the shell, it may take a long time before user sees a response. To improve response
time, a modern kernel creates the page table for the user address space, but marks the PTEs for the
pages invalid. On a page fault, the kernel reads the content of the page from disk and maps it into
the user address space. Like COW fork and lazy allocation, the kernel can implement this feature
transparently to applications.
The programs running on a computer may need more memory than the computer has RAM.
To cope gracefully, the operating system may implement paging to disk. The idea is to store only
a fraction of user pages in RAM, and to store the rest on disk in a paging area. The kernel marks
PTEs that correspond to memory stored in the paging area (and thus not in RAM) as invalid. If an
application tries to use one of the pages that has been paged out to disk, the application will incur
a page fault, and the page must be paged in: the kernel trap handler will allocate a page of physical
RAM, read the page from disk into the RAM, and modify the relevant PTE to point to the RAM.
What happens if a page needs to be paged in, but there is no free physical RAM? In that case,
the kernel must first free a physical page by paging it out or evicting it to the paging area on disk,
and marking the PTEs referring to that physical page as invalid. Eviction is expensive, so paging
performs best if it’s infrequent: if applications use only a subset of their memory pages and the
union of the subsets fits in RAM. This property is often referred to as having good locality of
reference. As with many virtual memory techniques, kernels usually implement paging to disk in
a way that’s transparent to applications.
Computers often operate with little or no free physical memory, regardless of how much RAM
the hardware provides. For example, cloud providers multiplex many customers on a single ma-
chine to use their hardware cost-effectively. As another example, users run many applications on
smart phones in a small amount of physical memory. In such settings allocating a page may require
first evicting an existing page. Thus, when free physical memory is scarce, allocation is expensive.
Lazy allocation and demand paging are particularly advantageous when free memory is scarce.
Eagerly allocating memory in sbrk or exec incurs the extra cost of eviction to make memory
available. Furthermore, there is a risk that the eager work is wasted, because before the application
uses the page, the operating system may have evicted it.
Other features that combine paging and page-fault exceptions include automatically extending
stacks and memory-mapped files.

4.7 Real world
The trampoline and trapframe may seem excessively complex. A driving force is that the RISC-
V intentionally does as little as it can when forcing a trap, to allow the possibility of very fast
trap handling, which turns out to be important. As a result, the first few instructions of the kernel
trap handler effectively have to execute in the user environment: the user page table, and user
register contents. And the trap handler is initially ignorant of useful facts such as the identity of
the process that’s running or the address of the kernel page table. A solution is possible because
RISC-V provides protected places in which the kernel can stash away information before entering
user space: the sscratch register, and user page table entries that point to kernel memory but
are protected by lack of PTE_U. Xv6’s trampoline and trapframe exploit these RISC-V features.
The need for special trampoline pages could be eliminated if kernel memory were mapped
into every process’s user page table (with appropriate PTE permission flags). That would also
eliminate the need for a page table switch when trapping from user space into the kernel. That
in turn would allow system call implementations in the kernel to take advantage of the current
process’s user memory being mapped, allowing kernel code to directly dereference user pointers.
Many operating systems have used these ideas to increase efficiency. Xv6 avoids them in order to
reduce the chances of security bugs in the kernel due to inadvertent use of user pointers, and to
reduce some complexity that would be required to ensure that user and kernel virtual addresses
don’t overlap.
Production operating systems implement copy-on-write fork, lazy allocation, demand paging,
paging to disk, memory-mapped files, etc. Furthermore, production operating systems will try to
use all of physical memory, either for applications or caches (e.g., the buffer cache of the file
system, which we will cover later in Section 8.2). Xv6 is naïve in this regard: you want your
operating system to use the physical memory you paid for, but xv6 doesn’t. Furthermore, if xv6
runs out of memory, it returns an error to the running application or kills it, instead of, for example,
evicting a page of another application.
4.8 Exercises
1. The functions copyin and copyinstr walk the user page table in software. Set up
the kernel page table so that the kernel has the user program mapped, and copyin and
copyinstr can use memcpy to copy system call arguments into kernel space, relying on
the hardware to do the page table walk.
2. Implement lazy memory allocation.
3. Implement COW fork.
4. Is there a way to eliminate the special TRAPFRAME page mapping in every user address
space? For example, could uservec be modified to simply push the 32 user registers onto
the kernel stack, or store them in the proc structure?
5. Could xv6 be modified to eliminate the special TRAMPOLINE page mapping?
