# 4. Memcheck: a memory error detector

## 4.2. Explanation of error messages from Memcheck

### 4.2.8. Memory leak detection

<!--
Memcheck keeps track of all heap blocks issued in response to calls to malloc/new et al. So when the program exits, it knows which blocks have not been freed.
-->

Memcheckは、malloc/newなどの呼び出しに応答して発行されたすべてのヒープブロックを追跡します。したがってプログラムが終了するとき、Memcheckは解放されていないブロックを知ることができます。

<!--
If --leak-check is set appropriately, for each remaining block, Memcheck determines if the block is reachable from pointers within the root-set. The root-set consists of (a) general purpose registers of all threads, and (b) initialised, aligned, pointer-sized data words in accessible client memory, including stacks.
-->

`--leak-check`が適切に設定されている場合、開放されずに残っている各ブロックに対し、Memcheckはルートセット内のポインタからブロックに到達可能かどうかを測定します。
ルートセットは、（a）全スレッドの汎用レジスタと、（b）スタックを含むアクセス可能なクライアントメモリ内の初期化された、整列されたポインタサイズのデータ​​ワードからなります。

<!--
There are two ways a block can be reached. The first is with a "start-pointer", i.e. a pointer to the start of the block. The second is with an "interior-pointer", i.e. a pointer to the middle of the block. There are several ways we know of that an interior-pointer can occur:
-->

ブロックに到達するには2つの方法があります。
第1の方法は、**「開始ポインタ」、すなわちブロックの開始点を指すポインタ**を使うことです。
第2の方法は、**「内部ポインタ」、すなわちブロックの中間へのポインタ**を使うことです。
内部ポインタを発生させる可能性があると知られているいくつかの原因があります：

<!--
The pointer might have originally been a start-pointer and have been moved along deliberately (or not deliberately) by the program. In particular, this can happen if your program uses tagged pointers, i.e. if it uses the bottom one, two or three bits of a pointer, which are normally always zero due to alignment, in order to store extra information.

It might be a random junk value in memory, entirely unrelated, just a coincidence.

It might be a pointer to the inner char array of a C++ std::string. For example, some compilers add 3 words at the beginning of the std::string to store the length, the capacity and a reference count before the memory containing the array of characters. They return a pointer just after these 3 words, pointing at the char array.

Some code might allocate a block of memory, and use the first 8 bytes to store (block size - 8) as a 64bit number. sqlite3MemMalloc does this.

It might be a pointer to an array of C++ objects (which possess destructors) allocated with new[]. In this case, some compilers store a "magic cookie" containing the array length at the start of the allocated block, and return a pointer to just past that magic cookie, i.e. an interior-pointer. See this page for more information.

It might be a pointer to an inner part of a C++ object using multiple inheritance.
-->

* ポインタはもともとは開始ポインタであったかもしれず、故意に（あるいは故意でなく）プログラムによって動かされたかもしれない。特に、プログラムが *tagged pointer* ([Tagged pointer - Wikipedia](https://en.wikipedia.org/wiki/Tagged_pointer))を使用している場合、つまり、ポインタの下位1ビット、2ビットまたは3ビットを使用する場合、そこは追加情報を格納するために通常は常にゼロであるため、内部ポインタが発生する可能性があります。
* メモリ内のランダムなジャンク値で、まったく無関係であり、ただの偶然かもしれません。
* C++ `std::string`の内部char配列へのポインタかもしれません。たとえば、`std::string`の先頭に3ワードを追加して、長さ・容量・および参照カウントを、文字の配列を含むメモリの前に格納するコンパイラがあります。これら3つの単語の直後にポインタを返し、char配列を指します。
* コードによっては、メモリブロックを割り当てて、64ビットの数値として格納するために最初の8バイトを使用するかもしれません。`sqlite3MemMalloc`はこれを行います。
* new[]で割り当てられたC++オブジェクト（デストラクタを所有する）の配列へのポインタかもしれません。この場合、一部のコンパイラは、割り当てられたブロックの先頭に配列の長さを含む「マジッククッキー」を格納し、そのマジッククッキーのすぐ後ろへのポインタ、つまり内部ポインタを返します。詳細については、このページ※を参照してください。※リンク切れ
* 多重継承を使用するC++オブジェクトの内部部分へのポインタである可能性があります。

<!--
With that in mind, consider the nine possible cases described by the following figure.
-->

それを念頭に置いて、次の図で説明される9つの可能なケースを考えてみましょう。

```text
     Pointer chain            AAA Leak Case   BBB Leak Case
     -------------            -------------   -------------
(1)  RRR ------------> BBB                    DR
(2)  RRR ---> AAA ---> BBB    DR              IR
(3)  RRR               BBB                    DL
(4)  RRR      AAA ---> BBB    DL              IL
(5)  RRR ------?-----> BBB                    (y)DR, (n)DL
(6)  RRR ---> AAA -?-> BBB    DR              (y)IR, (n)DL
(7)  RRR -?-> AAA ---> BBB    (y)DR, (n)DL    (y)IR, (n)IL
(8)  RRR -?-> AAA -?-> BBB    (y)DR, (n)DL    (y,y)IR, (n,y)IL, (_,n)DL
(9)  RRR      AAA -?-> BBB    DL              (y)IL, (n)DL

Pointer chain legend:
- RRR: a root set node or DR block
- AAA, BBB: heap blocks
- --->: a start-pointer
- -?->: an interior-pointer

Leak Case legend:
- DR: Directly reachable
- IR: Indirectly reachable
- DL: Directly lost
- IL: Indirectly lost
- (y)XY: it's XY if the interior-pointer is a real pointer
- (n)XY: it's XY if the interior-pointer is not a real pointer
- (_)XY: it's XY in either case
```

<!--
Every possible case can be reduced to one of the above nine. Memcheck merges some of these cases in its output, resulting in the following four leak kinds.
-->

すべての可能なケースは、上記9のいずれかに帰着させることができます。Memcheckは、これらのケースのいくつかを出力中でマージし、その結果、リークの種類は次の4つになります。

<!--
"Still reachable". This covers cases 1 and 2 (for the BBB blocks) above. A start-pointer or chain of start-pointers to the block is found. Since the block is still pointed at, the programmer could, at least in principle, have freed it before program exit. "Still reachable" blocks are very common and arguably not a problem. So, by default, Memcheck won't report such blocks individually.

"Definitely lost". This covers case 3 (for the BBB blocks) above. This means that no pointer to the block can be found. The block is classified as "lost", because the programmer could not possibly have freed it at program exit, since no pointer to it exists. This is likely a symptom of having lost the pointer at some earlier point in the program. Such cases should be fixed by the programmer.

"Indirectly lost". This covers cases 4 and 9 (for the BBB blocks) above. This means that the block is lost, not because there are no pointers to it, but rather because all the blocks that point to it are themselves lost. For example, if you have a binary tree and the root node is lost, all its children nodes will be indirectly lost. Because the problem will disappear if the definitely lost block that caused the indirect leak is fixed, Memcheck won't report such blocks individually by default.

"Possibly lost". This covers cases 5--8 (for the BBB blocks) above. This means that a chain of one or more pointers to the block has been found, but at least one of the pointers is an interior-pointer. This could just be a random value in memory that happens to point into a block, and so you shouldn't consider this ok unless you know you have interior-pointers.
-->

* "Still reachable". これは、上のケース1と2（のBBBブロック）に相当します。ブロックに対する開始ポインタまたは開始ポインタからの連鎖が見つかりました。ブロックはまだ指されているので、プログラマは、少なくとも原則的には、プログラムを終了する前にそれを解放することができます。「まだ到達可能な」ブロックは非常に一般的であり、間違いなく問題にはならないでしょう。したがって、デフォルトでMemcheckはそのようなブロックを個別に報告しません。※`--show-leak-kinds=reachable`オプションを指定すると表示

* "Definitely lost". これは、上記のケース3（のBBBブロック）に相当します。これは、ブロックへのポインタが見つからないことを意味します。ブロックは「失われた」と分類されます。ブロックへのポインタが存在しないため、プログラム終了時にプログラマはブロックを解放できません。おそらくプログラムの早い時点でポインタを失ったという兆候です。本当ならプログラマにより修正されるべきです。※`--show-leak-kinds=indirect`オプションを指定すると表示

* "Indirectly lost". これは、上記のケース4と9（のBBBブロック）に相当します。これは、ブロックが「失われている」ことを意味しますが、それを指すポインタがないためというよりも、それを間接的に指すブロックがすべて失われているためです。たとえば、二分木があり、そのルートノードが失われた場合、すべての子ノードは間接的に失われます。間接リークを引き起こした（真犯人であるところの）"Definitely lost"ブロックが修正された場合に問題が解消されるため、Memcheckはこのようなブロックをデフォルトで個別に報告しません。

* "Possibly lost". これは、上記のBBBブロックのケース5〜8に相当します。これは、ブロックへ到達する単一のポインタまたは複数のポインタの連鎖が見つかったが、その鎖に含まれるポインタの少なくとも1つが内部ポインタであることを意味します。内部ポインタはたまたまブロックを指すことになったメモリ内のランダムな値である可能性があるので、内部ポインタを持っていることが分かっていない限り、これをOKと考えるべきではありません。

<!--
(Note: This mapping of the nine possible cases onto four leak kinds is not necessarily the best way that leaks could be reported; in particular, interior-pointers are treated inconsistently. It is possible the categorisation may be improved in the future.)
-->
