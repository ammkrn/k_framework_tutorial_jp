# コンフィギュレーション抽象化第一課：規則の種類

今回、IMP言語の定義を完成します。その内*コンフィギュレーション抽象化*の最初の一歩を踏み出しながら「構成規則」と「計算規則」の相違を調べます。

# IMPの構文規則

それで、規則の残りを加えていきましょう。数学表現、真偽値表現を支配する規則は自明的と思います。これらの規則に使用されてる変数は全部ビルトイン演算(\_+Int\_ などの演算)の引数として渡されているので、ｋツールは変数のソートを自動的に推理します。その上に、推理されたソートは動的に強制されます。確かに、整数ではないものが渡されたら足し算の規則は適用したくありません。今回やらないんですけど、`&&` の規則中の `B` と `_` を `B:K` で切り替えるように動的確認を省略することも出来ます。その規則が一致・当て嵌る境遇で `B` あるいは `_` は必ず `BExp` としてあることを証明出来ますから。従って、`B` が `BExp` としてあることが事前に分かって、ソートを確かめる必要が無くなります。これは些細なことに見えるかもしれませんが、言語・プログラムがだんだん拡大していくとそのような動的確認を計算する時間も盛り上がります。次のブロックを支配する規則も自明的と思います。

代入の規則は `=>` (書き換えを表す太い矢印) が二つあります。最初なのは`k`セルにあって、代入の文を溶けています。二番目なのは `state` セルにあって、代入されている変数の値をアプデートしています。`state`にある書き換え文は丸括弧に囲まれていることは大事なことを示しているので注意してください。`=>` は「greedy・貪欲的」であります。つまり、`=>`記号は小なり大なり記号で表されているセルの境界に達するまで左右、双方向へ一致します。書き換え矢印の範囲を限るため、丸括弧を使うように出来ます。

連続合成の規則(sequential composition rule)の構文糖衣を脱いだら、`S1 S2`から`S1 ~> S2`に変化されるんだけです。確かにそれらの文は意味論が同じであります。文は結局`.`(無)まで評価されるので`S1 ~> S2`中の`S1`が処理されたら後、次のタスクは自動的に`S2`になります。

条件文の規則と`while`文の規則は明らかであります。Kのコンフィギュレーション抽象化ということのおかげで、whileの「真」ケースと対応する規則は意外と繰り返し適用されません。そのポイントは近いうちに調べます。

IMPプログラムはまず変数宣言の集合から始めます。宣言文に含まれる変数が０として初期化する後、残っている文を実行していくものです。
プログラムを制御する規則は、宣言された変数を初期化しながら、重複している要素がないことも確認します。今回、重複要素を確認する仕事はｋのビルトインマップ対定義されている「マップの全鍵を返す」`keys`関数と「要素`x`が集合`S`の元であるかどうか」を確認する`in`関数でやります。実際に(産業界とかで)、ランタイムじゃなく、コンパイルタイムで無効なプログラムを正格的に却下出来るために、「静的型確認する装置・type checker」を作ります。ｋで定義される言語向けのtype checkerの作り方は第五章で見ますけど、今回ｋのマップ型と仲良くしましょう。

kはプログラムと意味論に対して同じパーサーを使用するので、二番目の規則にある`int .Ids; S`の代わりに`int; S`しか書かれていなかれば、問題なくパースされるはず。しかし、明確にするため、*無*を表す記号を明示的に書くことが慣用的であります。

出来たら、IMPの意味論は完成です! しかし、習って行くべきの知識がもう少し残っています。

# コンフィギュレーション抽象化

kで、全ての意味論規則は「あるコンフィギュレーションから他のコンフィギュレーションまでの遷移」を定義します。コンフィギュレーション抽象化の真髄は、「コンフィグ間の遷移を正しく実行するために必要ではないことを省略したい」ということです。省略を正しく出来るようになるため、ｋは元のコンフィギュレーション(意味論の上に`configuration`で宣言されたやつ)のセルの構成を参照します。例えば、kソートの項しか処理しない規則はセルを何も入ってませんね。IMPの意味論なら、セルを処理する規則が３つだけありました。そのような規則を`kompile`すると、コンパイラーが元のコンフィグにあるけど規則にかかれていない構成を完成してくれる。元例えば我々が定義した`while`ループが元のコンフィグとどこが違うっていうなら、`<k>`セルに包んでないので、コンパイラーは我の規則を以下のようなものにしてくれます:

```
    rule <k> while (B) S => if (B) {S while (B) S} else {} ...</k>
```

で、`=>`で表されている遷移は、書き直されている項は準備が良いまで行われません。これは`while`ループの「真」ケースが繰り返し適用されないことの理由；内なるループはまだプロセス出来る状態にならなかったです。規則をこの不完成のように書くことはコンフィギュレーションの構成を抽象化して、前より読みやすくするので*コンフィギュレーション抽象化*と呼びます。

IMP++でよく見るはずですがコンフィギュレーション抽象化は便利だけではなく、言語の定義のモジュレリティーをかなり増えてくれる概念です。Kのセルを完成すること以上機能も持っています。

# 構成規則と計算規則の比較

kには、規則がに種類あって、「構成的」と「計算的」と呼びます。直感的に、構成的規則は、計算的規則が適用可能になれるためにコンフィギュレーションを並び替えるものです。したがって、構成的規則は計算ステップとして見なされません。kに定義される意味論は「遷移システム」の発生器として見なすことができ、各々のプログラムに遷移システムを割り当てます。遷移システムとして見たら、遷移を表すステップは計算的規則だけ作れる、構成的規則は観測できない物としてあります。kで、全ての規則はデフォールトで計算的だと考えられます(暗黙的な温め・冷めルール以外)。とある規則を構成的にしたければ、`strucutral`というアトリビュートを添付したほうがいいんです。

その`structural`アトリビュートの使い方をみてみましょう。検索、数学、真偽値の規則が適用された時に、計算的な進歩がするので「計算的」に放っておきたいね。だが、ブロックの規則は構文を集める物だけですから、構成的にしてもいいんですね。言語の定義から作った遷移システムが早くて効率的になるため、行動を失わずに計算的な規則を少なくしたいんです。今回、ブロックの規則、`while`ループの拡張と対応する規則、変数宣言リストが空になった場合の規則も構成的にすればいい。

それで、`kompile`して、レッスン1でパースしたプログラムを`krun`で実行して。計画通りに実行するはずです。`<state>`セルはプログラムの最終状態を表します。`<k>`セルはコードの最終内容表します（プログラムが問題なく実行する時に何も入っていないようになるはずです）。