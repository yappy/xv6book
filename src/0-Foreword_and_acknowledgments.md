# Foreword and acknowledgments

This is a draft text intended for a class on operating systems.

これはオペレーティングシステムの授業を目的としたドラフトテキストである。

It explains the main concepts of operating systems by studying an example kernel, named xv6.

このテキストは OS の主要な概念を、xv6 という名前のカーネルを例に説明している。
(注: カーネルは OS の中核部分のこと。)

Xv6 is modeled on Dennis Ritchie’s and Ken Thompson’s Unix Version 6 (v6) [17].

xv6 はデニス・リッチーとケン・トンプソンの Unix Version6 (v6) をモデルにしている。
(注: C 言語の発明者。C 言語 は Unix を書くために作られた。)

Xv6 loosely follows the structure and style of v6,
but is implemented in ANSI C [7] for a multi-core RISC-V [15].

xv6 はおおまかには v6 の構成とスタイルを踏襲しているが、
ANSI C で、マルチコア RISC-V 向けに、実装されている。
(注: ANSI C: C89 のこと。現代語として普通に読める C 言語。)
(注: RISC-V: りすくふぁいぶ。大学で広く教えられてきた MIPS の後継。~~はっきり言ってモダナイズされた MIPS。~~)

This text should be read along with the source code for xv6, an approach inspired by John
Lions’ Commentary on UNIX 6th Edition [11].

このテキストは xv6 のソースコードと共に読み進めてほしい。
これは "John Lions’ Commentary on UNIX 6th Edition" に触発されたアプローチである。
(注: v6 の解説本だが、K&R 以前の古文 C であり、現代人にはまともに読めない。
~~なので古文の勉強は大切である。~~)

See <https://pdos.csail.mit.edu/6.1810> for pointers to on-line resources for v6 and xv6,
including several lab assignments using xv6.

<https://pdos.csail.mit.edu/6.1810> に v6 および xv6 へのオンラインリソースへのリンクがある。
また、xv6 を用いた演習課題も含まれる。
(注: 海外の大学に行ったことが無いので lab assignments の定訳が分からない。)

We have used this text in 6.828 and 6.1810, the operating system classes at MIT.

我々はこのテキストを 6.828 および 6.1810 という MIT での
オペレーティングシステムの授業で使用した。

We thank the faculty, teaching assistants, and students of those classes
who have all directly or indirectly contributed to xv6.
In particular, we would like to thank Adam Belay, Austin Clements, and Nickolai
Zeldovich. Finally, we would like to thank people who emailed us bugs in the text or sugges-
tions for improvements: Abutalib Aghayev, Sebastian Boehm, brandb97, Anton Burtsev, Raphael
Carvalho, Tej Chajed, Rasit Eskicioglu, Color Fuzzy, Wojciech Gac, Giuseppe, Tao Guo, Haibo
Hao, Naoki Hayama, Chris Henderson, Robert Hilderman, Eden Hochbaum, Wolfgang Keller,
Henry Laih, Jin Li, Austin Liew, Pavan Maddamsetti, Jacek Masiulaniec, Michael McConville,
m3hm00d, miguelgvieira, Mark Morrissey, Muhammed Mourad, Harry Pan, Harry Porter, Siyuan
Qian, Askar Safin, Salman Shah, Huang Sha, Vikram Shenoy, Adeodato Simó, Ruslan Savchenko,
Pawel Szczurko, Warren Toomey, tyfkda, tzerbib, Vanush Vaswani, Xi Wang, and Zou Chang Wei,
Sam Whitlock, LucyShawYang, and Meng Zhou

If you spot errors or have suggestions for improvement, please send email to Frans Kaashoek
and Robert Morris.

もし誤りを発見した、または改善の提案がある場合には、Frans Kaashoek とRobert Morris に
メールしてほしい。
