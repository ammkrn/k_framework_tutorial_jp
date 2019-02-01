# 意味的リスト；入出力ストリーミング

このレッスンで、`read` と `print` 構造をIMP++に追加します。その上、意味的リストの使い方を調べながらリストを標準入出力ストリームと繋げるように k での意味論を対話インタープリターに変化する方法も見ます。

最初に、入出力ストリームを表すセルをコンフィギュレーションに加えていきます:

```
    <in color="magenta"> .List </in>
    <out color="Orchid"> .List </out>
```

これらのセルは両方空のリストが入っているように初期化されています。意味的リストは空白で区切られる項の連続です(普通の関数プログラミング言語のリスト型と同じようなものです)。リスト入りの要素は `K` ソートの `t` という項であって、書き方は `ListItem(t)`です。`ListItem` という包はパーサーに曖昧さを避けるために必要です。対照的に、意味的マッピング (状態・state、環境・environment、蓄積・store 用に物) は `t1 |-> t2` というようなペアの集合としてあります。

`print` 文は `AExp` だけではなく、文字列も印字可能ように定義したいので、K に文字列が結果・result であることを K に伝えないとなりません。より面白くするため、`+` 記号を文字列結合としてオーバロードします。`+` はもはや正確評価を使うので、`+` 記号が文字列対使用されたことを `+String` というビルトインに縮小するだけでいいんです。

`read` 演算の意味論は直接であって、`<in/>` セルにある最初の要素を消費します。今回の `read` は整数しか受け取りません。

`print` 演算の意味論は考えものになりますね。`print` はAExpを任意の個数を受け取って、左から右へ評価して出力します。例えば `print("Hello", 3/0, "Bye");` は "Hello," を画面に印字して、後で違反の０割で詰まっているようになってしまいます。引数を全部評価して画面に印字するように定義すれば、項が全体詰まっているようになってどれの項が悪かったかっていうことが識別できなくなるので望ましくない行動であります。だから、最初の一歩は`print`の引数を評価する。なんか、`print` は`strict(1)`というような評価戦略があるということを言いたいけど、正確さのアトリビュートは独自の言語構造対使えます。必要な物は *２つ* の構造対使える評価戦略です (`print` と `AExp` 入りのリスト)。無邪気に `print` を `strict(1)` にすれば、最初の引数としてある `AExps` は全体処理されているようになってしまって、`AExps` 項を評価するための規則がないのでまた詰まっているようになります。`AExps` のリスト構造を `strict` にすれば、`AExps` リストが全体評価されて同じ問題になります。実は、このような問題を正しく解けるために `context・文脈` 宣言を使います。このように、`print` は与えられたリストの最初要素しか評価しないように定義出来るようになります。

```
    context print(HOLE:AExp, _);
```

`AExp` ソートとしてある `HOLE` をよく見なさい。`strict` アトリビュートと比べたら、`context` はより細かい評価戦略・複数の言語構造対のコントロールを定義できます。`HOLE` 部分は評価されるべきの要素を判明するためにあります。例えば除算の`strict` アトリビュートは以下の文脈と対応します:

```
    context HOLE / _
    context _ / HOLE
```

最大限に一般化すれば、ちょうど一つの `HOLE` を持って変数も`HOLE` 対任意のサイドコンディションが定義されている項は全部 `context` であります。第六章で例示が他にもあります。

評価されたら、`print` の第一引数が整数・文字列の一つになるはずです。文字列も整数も印字したいんだけどただひとつの規則を書きたいので整数と文字列の 和・union である `Printable` 構文カテゴリを新たに定義します。

`kompile` して `io.imp` を `krun` で実行しましょう。思ったとおり、`read` 構造が最上であって空になった `<in/>` セルを持っているように詰まっているんですね。ちゃんと実行するため、`read` ルールが一致可能なアイテムを `<in/>` セルに入れていかければなりません。このようなリストを入れましょう:

```
    <in> ListItem(3) ListItem(5) ListItem(7) </in>
```

改めて `io.imp` を `krun` すれば、`<k/>` セルが空になって問題なく終了するはずです。`read` のおかげで、渡されたアイテムが `<in/>` から取られて `<out/>` に置かれたんです。

セマンティクリスト入りのセルを標準入出力ストリームと繋いだら、`k` はあとの処理を自動的にやります。`stream="stdin"` と `stream="stdout"` アトリビュートをそれぞれ `<in/>` `</out/>` セルに付けていくことでその機能を試してみましょう。`stdin` と繋がっているストリームは本物の標準入力からアイテムを取ります。実行中のプログラムは入力アイテムが必要けど既になければ、アイテムが来るまでプロセスをブロックします。`stdout` と繋がっているセルは自分のアイテムを順番で標準出力ストリームへ送ります。

`io.imp`をもう一回 `kompile` して実行しましょう。ちゃんとメッセージをプリントしてユーザーからの数字を待ちます。数字を２つ入力して `Enter` キーを押して見て下さい。与えられた数字の合計を持つメッセージとプログラムの最後コンフィギュレーションが印字されるはずです。コンフィギュレーションが結果と一緒に見たくない場合 `krun` を `--output none` オプションで呼んで下さい。

```
    krun io.imp --output none
```

与えられた数までを数えての数の合計を計算してプリントする `sum-io.imp` プログラムも実行してみましょう。飽きたら `0` を試して見て。プログラムが終了しましたが、`<k/>` セルにゴミが残っているんですね。なぜならば、`halt` 文の意味論はまで定義したことがないんだからです。

従って、次のレッスンで`halt` の意味論を定義しブロックの局所的変数宣言対の意味論を修理します。
