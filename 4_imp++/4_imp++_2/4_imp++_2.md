# コンフィギュレーションを洗練すること； 鮮度 (Freshness)


今回は、スレッドと局所変数の構文の準備をします。第三章のLAMBDA++と同様に状態を表す`<state/>`セルを環境と蓄積(`<env/>`・`<store/>`)に分割していきます。もちろん、そのような変更はコンフィギュレーションの修正になるはずですね。

プログラムの変数を値に束縛する状態マッピングを分割するため、コンフィギュレーション宣言を以下のように分けていきます:

現在：

```
    <state color="red"> .Map </state>
```

修正後:

```
    <env color="LightSkyBlue"> .Map </env>
    <store color="red"> .Map </store>
```


セルをこのように分割することは構成の立場からかなり大きい変更と見なされるんだから、`<state/>`セルを用いる規則を新しいコンフィギュレーションとの互換性を確認・修理していく必要があります。

`state`セルを用いた最初の規則は変数検索ルールですね。

```
    rule <k> X:Id => I ...</k> <state>... X |-> I ...</state>
```

変更後版はこれ：

```
    rule <k> X:Id => I ...</k>
         <env>... X |-> N ...</env>
         <store>... N |-> I ...</store>
```

つまり、マッチを二回していて書き換えるような手順になります。`X`の`N`一致とのマッピングを環境で検索して、`store`の`N`位置にある`I`を見て、最後に`store`で見つけた`I`値を`k`セルにある書き換えの結果として置いていきます。`X`と`N`は単一の規則で二回一致されているので、面白い例示になると思います。

代入演算の規則は検索と同様に変更します。

```
  rule <k> X = I:Int; => . ...</k>
       <env>... X |-> N ...</env>
<store>... N |-> (_ => I) ...</store>
```

変数宣言を処理する規則は前より少し複雑です。`store`にある位置をアロケートして、新たに宣言されている変数をその位置と束縛する必要がありますから。第三章のLAMBDA++と似たような技術を使います：

```
    rule <k> int (X,Xs => Xs); ...</k>
         <env> Rho => Rho[X <- !N:Int] </env>
         <store>... .Map => !N |-> 0 ...</store>
```

ここにもフレッシュ変数を手に入れる構文 (`!N`) を使います。覚えないかもしれないけど、(`!`) を使用する規則がランタイムで適用される度に、`!`と指定されたソートの一意的で他の所に使用されたことがない変数作って用いられます。しかし、その行動を実装するための技術のせいで、変数の値は非決定的に作られるので、`!` で発生される値がフレッシュであること(プログラムの他の変数に比べたら一意的である) 以上なにも推定しないで下さい。

この `!` の本当の行動が見えるため、IMP++の定義を`kompile`して`sum.imp`を実行してみて下さい。プログラム内の２つの変数のため、フレッシュ変数が２つ作られるはずです。カウンターを保持しているセルも追加されたことを注意して下さい。

次のレッスンで変数を位置で増やす演算の意味論を定義して、それから発生する非決定性を見ながら、Kの非決定性を露出したり検査したりするための道具を見ます。

