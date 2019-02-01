# タグを添付する；`--transition` コンパイラーオプション

_役者からのノート２つあります：_ 

1. `--search` が問題なく使えるようになるため、kがデフォールトで使用する Ocaml バックエンドではなく、前バージョンの Java バックエンドでコンパイルした方がいいと思います。もし `krun div.imp --search` を実行する内に何かの `illegal operation` というエラーが返ってきたら、`imp-kompiled` ダイレクトリーを削除して、`kompile imp.k --backend java` でもう一回やってみて下さい。

2. レッスンを読みながら元動画を見ているのなら、動画で使用される `superheat` と `supercool` オプションがその時から `--transition` にまとめて変更されました。下にあるレッスンに詳しく説明されます。

このレッスンで、変数の値を 1 で増やす演算の意味論を定義します。それをやりながら、虱潰し検索向けの言語モデルを作るための構文や意味論の構造をタグする方法も見ます。

変数をインクリメントする規則は自明的と思います：

```
    rule <k> ++X => I +Int 1 ...</k>
         <env>... X |-> N ...</env>
         <store>... N |-> (I => I +Int 1) ...</store>
```

出来たら、レッスン 1 からのプログラムをいろいろ実行してみて下さい(特に `div.imp`)。

インクリメント演算の副作用を起こせる機能や `strict` アトリビュートと宣言される構造の評価順序、それらのことのせいでプログラムは非決定的な行動を示します。

`div.imp` を見れば `y` の位置に `1` が割り当てられた行動がありますが、他の起こり得る行動が複数あります。`krun` を `--search` オプションと呼ぶことで、全行動の最終状態を露出できます。例えば：

```
    krun div.imp --search
```

あっ、一つだけ出てきた...

`--search` と呼ばれた時に、`krun` は言語モデルに入っているバックトラッキングマーカを使ってプログラムと対応する状態遷移システムを検査してみます。しかし、その状態遷移システムの詳細レベル・マーカーがどれくらい入っている量などのことは `kompile` に制御されます。大半の k ユーザーはデフォールトで効率が高い実行可モデルを手に入れたいのでそれはデフォールトで発生される物です。特に、`kompile`はデフォールトでバックトラッキングマーカーを何も入りません。だから今回、出てきた行動が一つしかなかったです。

ならば、望みの発生されたモデルを k からもらう方法を見ていきましょう。元の言語定義から、なにが独自の「遷移」にしたいことを `kompile` の `--transition` オプションで明示的に指定できます。今回、言語構造による非決定性だけ検査したいんです。将来のレッスンで、並行性による非決定性などのことも調べます。

***
非決定的評価戦略による行動空間を全部検査したい場合、言語の構造を全部`--transition`オプションへ渡せます。これはいつでもやりたいオプションに聞こえるかもしれませんが、大きいプログラムか言語でやってみれば、重すぎって悟るようになるはずです。状態空間が多すぎになって、k は多分クラッシュしてしまいます。一つだけの正格評価を使う文論を10行しか持ってないプログラムはもはや 1000 以上の調査しなければならない行動を持ちます。ユーザーの望みや実用さに迫られていて、Kツールは言語モデルの微調整を許すために`--transition`を提供しています。

言語構造の中からどれが「遷移」になることを指定するために構文の構造、意味論の規則、どれにもタグを付けます。他のタグと同じく、これは右側にある大括弧中のタグとして書かれます。一つの構造・規則は複数のタグを持てて、同じタグは複数の言語構造に付けてもいいんです。例示として除算の構文構造、検索規則、値を 1 で増やす規則をそれぞれ `division`, `lookup`, `increment`というタグを付けていきましょう。規則に付けたタグは今回使用しないけど、慣れるためにやります。

現在の言語定義なら、除算の正確評価から発生する非決定行動を調べることの最もやりやすい方法は以下の命令で `kompile` することです。

    `kompile imp.k --transition "division"`

除算なら、非決定的な行動を起こせる言語規則は `lookup` と `increment` だけです。なぜならば、除算項が温められている内にそれ以上適用可能性ある演算がありません。

実を言うと、任意の正確に評価される言語構造は非決定的な行動を起こすか起こさないかを判断することは絶対簡単な仕事ではありません。例えばIMPは副作用を起こす構造がなかったら、除算から発生する非決定性もなくなります。特定の与えられたプログラムは非決定性があるかどうかはそれ以上もっと判断しづらいです。それに伴って、k は言語からなにが「遷移」としてあるべきっていうことを自動的に決めようとしません。代わりに、そのことを自分で判明する機能を与えてくれます。

`krun div.imp --search` 命令はこのプログラム全行動(５つある)を見せます。出てきた行動から、ゼロ割をするやつが一つありますね。

`--transition` オプションは言語のデザインとの実験したり、プログラムを形式的に検査したりすることに時にかなり便利な機能です。言語モデルの非決定性対より細かいコントロールが必要になった時があれば、k チームに連絡して下さい。

レッスンが終わる前、秘訣をもう一つ教えていきたいんですけど、使い過ぎないように言って下さい。`kompile` のオプションを言語定義中の要素にタグとして付けられます。そうすると、`kompile` は定義対呼ばれたら、それらのオプションは付けた構造に適用されます。例えば除算のプロダクション規則の側に `transition` タグを付けていくと、 `kompile imp` は `kompile imp.k --transition "division"` と同じ物になります。

確かに便利そうな近道ですが、大半の時に使わないほうがいいと思います。`transition` などのオプションをいろんな物に付けていって自然に忘れてしまう時があったら、言語モデルがどんどん遅くなってしまうチャンスが高いんだからです。しかし、非決定性と早く実験したいけど各言語構造対の一意的な名前を想像したりコンパイラーに全てのフラグを渡したりするなどのことがやりたくないときに便利なものです。　

じゃ、次のレッスンで、入出力を言語に追加することで対話インタープリターのようなものも作ります。

