# セルを動的に追加・削除すること:コンフィギュレーション抽象化レッスン２

今回は、構成が進化出可能なコンフィギュレーションの使い方を調べながら、動的スレッド作成・スレッド停止をIMP++に加えます。

`spawn S` 構文構造の意味論を「与えられたタスク `S` を実行する並行スレッドを作る」ことにしたいんです。`spawn S` で作ったスレッドは作成時の親スレッドの環境のスナップショットを受けるので、親スレッドと子供スレッドはその瞬間で同じメモリが見えます。それ以上メモリ共有装置がないのでその時点から各スレッドは独立に進行していけます。

それらのことを検討したら、各スレッドは自分の環境・計算セルなど持つことが必須という含意がありますね。ｋならそういうことは簡単に出来ます。`<k>` セルも `<env>` セルも包の `<thread>` セルに入れて行くだけでいいんです。これなら実行中プログラムは `<thread>` を0個以上どの数も持てるので `<thread>` セルを `multiplicity` アトリビュートと一緒に定義します。`multiplicity` ラベルの右にある限量子は正規表現のクリーンスターを同じ意味があって「0以上」の個数とマッチすることです。

```
    <thread multiplicity="*" color="blue">
      <k color="green"> $PGM:Stmt </k>
      <env color="LightSkyBlue"> .Map </env>
    </thread>
```

`multiplicity` 宣言は必要ではないんですが、すごくいい慣例です。

1. スレッドの複数性が言語の明示的な部分になるので、静的検査道具やコンパイラーなどのことが前よりよく効く。

2. コンフィギュレーションが前より明示的、理解やすくなる。

3. コンフィギュレーションの処理・規則を書くことを容易にする

キャプセル化を守るため、全ての `<thread>` セルを一つの `<threads>` セルにまとめていくのがおすすめですけど必要ではありません。

ここまで、言語の定義をずいぶん変えましたね。実は、ここまでの定義をコンパイルしてプログラムを実行すれば、問題なく実行します。不思議なんですね。

前にも言ったことありますが、コンフィギュレーション抽象化のおかげで現在書いている規則と直接関係ある情報について考えてそれ以上のことをコンパイラーに任せていけるための概念であります。新たな定義によると、検索などの規則をちゃんと行う方法がただ一つあって、それは `<k/>`, `<env/>` セルを `<thread/>` セルに入れて、全部 `<threads/>`セルに包んでいくことです。

検索ルールのコンフィギュレーション文脈を可能な限り簡潔に完成するコードはこれ：

```
    rule <threads>...
           <thread>...
             <k> X:Id => I ...</k>
             <env>... X |-> N ...</env>
           ...</thread>
         ...<threads/>
         <store>... N |-> I ...</store>  [lookup]
```

何かの`A`スレッドの`<k/>`セルを他の`B`スレッドの環境と繋ぐことも出来ます:

```
    rule <thread>...
             <k> X:Id => I ...</k>
         ...</thread>
         <thread>...
             <env>... X |-> N ...</env>
         ...</thread>
         <store>... N |-> I ...</store>  [lookup]
```

kはデフォールトで「貪欲」スタイルで規則を完成します。「規則をコンフィギュレーションを合わせるため、最短な道は？」というような手続きとして考えられます。

コンフィギュレーション抽象化は必要な概念ではないんですが、だんだん使って慣れていけば、大事な友達になるはずです。

お待ちかねの`spawn`規則を定義する時点に来ました：

```
    rule <k> spawn S => . ...</k> <env> Rho </env>
         (. => <thread>... <k> S </k> <env> Rho </env> ...</thread>)
```

ここにもコンフィギュレーション抽象化が働いています。現在のコンフィギュレーションについて考えたら（特に`<thread/>`セルのmultiplicity宣言）意味をなすようにこの規則を完成する方法がただ一つあって、それは`<k>`セルと`<env/>`セルを`<thread>`に囲んで、子スレッドにある`...`を`<thread>`のデフォールト内容と交換することです。この事情で他のセルが存在しないので`...`は要りませんが、規則の汎用さを上げているものであるので捨てません。

論理の立場からスレッドの意味論をそれよりコンパクト、汎用的に定義できるはずなんですが:

    rule <k> spawn S => . ...</k> <env> Rho </env>
         (. => <k> S </k> <env> Rho </env>)

この書き方は `thread` という名前に依存がないので前より汎用版であります。例えばコンフィギュレーションに `thread` の名前を `agent` に変更したくなったら、後で定義される規則にある `<thread/>` を `<agent/>` に変更することが必要ではないんです。あいにく、現在のコンフ抽象化を処理するアルゴリズムはそのような定義を受けられません。しかし、ｋ チームはその規制を解除するように働いています。

では、スレッドの意味論を終えましょう。子供スレッドの停止状態は `<k/>` セルが殻になった状態ですね。なら、その `<thread/>` セルを捨てられます。

```
    rule <thread>... <k> . </k> ...</thread> => .  [structural]
```

それで、ここまでの出来栄えを調べてみましょう。`imp` の定義をコンパイルして、`spawn.imp` プログラムを見たら、実行し見て下さい。

`spawn.imp` の出力から、以下のことをちゃんと見なさい：

- `<threads/>` セルは何も入っていないので、プログラムが問題なく停止したってこと分かります。
- 印字された値は `<store>` にある値ではありません。`<store>` にある値は連続実行からもらうはずの値でもありません。

面白い行動が出てきました。可能な行動全て見たいんですね。

今回も`krun --search` オプションを使います。レッスン３にも言ったが、 `kompile` がデフォルトで作り出す言語モデルは実行向けの物です。虱潰し式の検査を有効にする検査するためのバックトラッキングマーカーを入れません。したがって、レッスン３と同様にどれの規則が「遷移・transition」としてみなすべきかってことを明示的に指定する必要があります。プログラムが大きくなると検査空間も即座に爆発に拡張するのでプログラムが非常に小さい場合以外直感的に全規則を遷移として指定することはいけません。2スレッド150文を持っているプログラムは観測可能な宇宙に存在している粒子の個数より行動を持っています。

大雑把に言うなら、*行動を競合する*規則を「遷移」としてマークすることがいい戦略です。つまり、複数の規則が同時に一致して、規則がどのように適用されることの順序によってプログラムの結果になにかの影響を与えられる規則です。足し算は何も行動を競合していない規則の典型例ですね。`3+7` をいつ評価しても `10` になります。一方、変数検索は遷移としてマークした方がいいんです。ある変数 `x` の検索を遅延すれば他のスレッドはその遅延期間内に `x` を変更して、プログラムの行動を変えるかもしれない。

`spawn.imp` に対して `--search` を正しく使えるため、考えものがもう一つあります。k は `--search` で虱潰し式の検査をしている場合、ストリームとのやり取りを無効にしないとなりません。そうしないと行動空間が無限になってしまいます。しかし、`spawn.imp` は標準入力から文字を読む対話プログラムです。このようなプログラムを検査している時に、何かの入力を `krun` へパイプすることがおすすめです。そうすると、k は標準入力バファーをフラッシュして、パイプされた入力をコンフィギュレーションにある `<in/>` セルに入れてくれます。

例えば以下のような命令は `100` を標準入力に入れてプログラムのコンフィギュレーションが初期化される時点で標準入力バファーにある `100` が `<in/>`セルにフラッシュされて、最後に`krun`は虱潰し式検査を開始します。

```
    echo 100 | krun spawn.imp --search
```

「遷移・transition」をマークして検索を真剣でやりましょう。レッスン 3 でタグした規則は今回もタグする必要があります (lookup, increment, assignment)。それ以上 read は他のスレッドにある read と競合する可能性があるので、それも遷移にします。assignment も print の最初規則も遷移としてマークすることが必要です。j

じゃ、それが終わったら遷移を判明するようにコンパイルしましょう。

```
    kompile imp.k --transition "lookup increment assignment read print"
```

改めて入力をパイプで渡して実行すると `spawn.imp` の行動が全部がちゃんと露出されるはずです。独自な情動が 12 あります。

```
`echo 23 | krun spawn.imp --search` 
```

ここまでのIMP++はスレッドを同期する機能を何も持っていません。次回は他のスレッドの終結を待たせる`join`文のくわえ方を見ます。
