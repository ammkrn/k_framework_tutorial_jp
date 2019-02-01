# もっと複雑な構文を定義する

第二章で、LAMBDAより複雑Cと似たような構文を持つIMP言語を定義します。

このレッスンで学ぶことは：
+ カプセル化を守るため、言語の定義をいくつかのモジュールに分割する方法
+ kのビルトイン構文的リスト構造の使い方

kツールは定義している言語のフィーチャーをまとめるための装置を提供しています。モジュールのまとめ方・構成を制御するルールや基準があまりないんですが、大半の時言語の構文と意味論を別のモジュールとして定義することがよく行われます。

今回は、モジュールを二つ作って、それぞれIMP-SYNTAXとIMPを名乗っていきましょう。`imports`キーワードを使って、あるモジュールから他のモジュールを引き入れられます。名前の通り、IMPの構文をIMP-SYNTAXに定義し、意味論をIMPに定義します。

kは輸入しているモジュールの内容を他のモジュールへ含めてくれるだけです（そうする内に、複数のモジュールを輸入する時にも各定義が一回以上定義されていないってことを保証します）。kツールズのモジュールシステムはそれ以上何も特別なことをやってくれません。

IMPは六つの構文カテゴリを持っています。`imp.k`で全部見えます。数学表現の`AExp`、真偽値表現の`BExp`、ブロックの`Block`, 文の`Stmt`,プログラムの`Pgm`と`,`で区別される変数名を持つリストの`Ids`です。ブロックは条件式の枝とwhileループ文のボディーを限るための特別な文としてあります。数学表現と真偽値表現は何も特別な特徴を持っていません。

既に気づいたかもしれませんが、`<=`と`&&`は少しだけ皮をかぶせております。`<=`は`[seqstrict]`アトリビュートが付いてありますね。`seqstrict`という飾りは引「数が左から右へ順番に評価するべき」という意味です。`strict`と同様に、`seqstrict`に対して微調整ができます（どれの引数に当てはまるか、カスタム評価順番を定義することも可能）。デフォールトで`seqstrict`は全ての引数対当てはまります。`&&`に短絡・ショートサーケットの意味論を持たせたいので、第一引数だけに正格評価を適用していきます。つまり、第一引数が`false`に評価されたら、残りは評価される必要がなくなるので、やりません。

ブロック(Block)は波括弧で作られて、何かの内容を持っていも、空いているようにも存在可能です。

「文」(Stmt)は何も特別な行動を持ちません。しかし気をつけていけばいいところは`;`文を分裂するための記号ではなく、代入文を停止するための記号だけです。BlockはStmtの部分ソートとしてあります。

構文より意味論に集中出来るため、既にテストしたパーサーの優先を使って続けます。実際に言語を定義する時、望みの行動を行うパーサーを手に入れるために優先との実験や微調整する必要があるはずです。

IMPプログラムの形はCと同じく`int`キーワード後、変数入り「,」で区別されているリストを宣言し、あとでいくつかの文を含むものです。この構文のきっかけは、与えられたIMPプログラムを`main(){..}`で囲んで行くと、有効なCプログラムになる。ここまでのIMP言語なら、変数は初めに宣言されるリスト以外宣言する方法がありません。第4章で定義するIMP++はこの限りを解禁します。

`,`に区別されるリストは`List{Id,","}`で定義されることを注意してください。これはkがサポートしているビルトイン汎用的なリストです。広域的にというなら、

```
syntax B ::= List{A,T}
```

この定義は`B`という新たな非終端記号を宣言し、`A`はリストの内容のソートを判明し、`T`はリストの区別記号を表します。他の言語のリストと同じく、nil/何も入っていないリストも存在可能です。IMPなら、何も入ってないリストは変数がないプログラムを表しますね。他のkフィーチャーと同様に、リストの行動を詳しくコントロールする方法があります。

正格評価アトリビュートがちゃんと役に立つため、全てのあり得る計算から、結果にしたいことを指定しなければなりません。この仕事はLAMBDA章で経験を積みましたね。IMPモジュールの中に`integer`と`Boolean`は結果にしましょう。

`imp.k`をコンパイルして、プログラムを実行することで発生されたパーサーと実験して見て。IMPはCの部分で、テキストエディターをCモードにすればちゃんとハイライトされるはずです。

`sum.imp`という例は`n`までの数の合計を`sum`変数と束縛します。
```
    int n, sum;
    n = 100;
    sum=0;
    while (!(n <= 0)) {
      sum = sum + n;
      n = n + -1;
    }
```

`krun sum.imp`をやってみて、パースーの出力を保持するkセルを見てください。

`collatz.imp`は`m`までの数に対して「コラッツの問題」を計算して、ステップの数を`s`に記録します。

```
    int m, n, q, r, s;
    m = 10;
    while (!(m<=2)) {
      n = m;
      m = m + -1;
      while (!(n<=1)) {
        s = s+1;
        q = n/2;
        r = q+q+1;
        if (r<=n) {
          n = n+n+n+1;         // n becomes 3*n+1 if odd
        } else {n=q;}          //        of   n/2 if even
      }
    }
```

`m`まで、素数がいくつあるってことを計算する`primes.imp`もあります。
```
    int i, m, n, q, r, s, t, x, y, z;
    m = 10;  n = 2;
    while (n <= m) {
      // checking primality of n and writing t to 1 or 0
      i = 2;  q = n/i;  t = 1;
      while (i<=q && 1<=t) {
        x = i;
        y = q;
        // fast multiplication (base 2) algorithm
        z = 0;
        while (!(x <= 0)) {
          q = x/2;
          r = q+q+1;
          if (r <= x) { z = z+y; } else {}
          x = q;
          y = y+y;
        } // end fast multiplication
        if (n <= z) { t = 0; } else { i = i+1;  q = n/i; }
      } // end checking primality
      if (1 <= t) { s = s+1; } else {}
      n = n+1;
    }
```

IMPの意味論が定義された後、これらのプログラムは全部実行可能になります。前といったように、これらのプログラムを`main(){..}`で囲んでいくと、好きなCコンパイラーでコンパイル・実行できます。IMPの意味論を定義する前に、`kast`と呼ばれるkのビルトインパーサーについていろんなことを明らかにしたいんです。我ながらかなり火力を持つ道具としてありますが、奇跡を起こせるものではありません。kのパーサーはいろんな些細ではない言語をパースできますが（例えばこのチュートリアルにある2_languages/KOOL言語）本物のパーサーの代わりに使用されるようなものとしてデザインされなかったです。

kで定義される構文はすごく複雑なプログラミング言語の具体的な構文を完全に定義するために書かれたものではなく、意味論を定義する時の便利な記号法として書かれたものです。なので、kをよく*the syntax of semantics*と呼びます（意味論の構文って）。具体的な構文向けの外部パーサーをkツールと接続することの例示は公式チュートリアルのリポジトリの/samples/にあるKERNELC言語で見えます。

それが言われても、是非自分の言語をビルトインパーサーと合わせて見ることがおすすめです！構文ついての細かい問題処理したくないってこと言いながら自分の言語構文を諦めて行くことはやらないでください。本当にツールの短所のせいで言語を定義できないなら、Kチームに知らせてください。

このレッスンまで我々はデフォールトコンフェィギュレーションだけ見たことあります。次回、kのカスタムコンフェィギュレーションを定義する方法を見ていきます。