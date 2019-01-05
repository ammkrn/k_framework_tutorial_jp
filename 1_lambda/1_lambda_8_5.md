# mu についての補助レッスン


このレッスンは翻訳者に書かれました。公式版の`mu`についての説明はかなり簡明だったんですから、分からない方々がいるかも知らないんじゃないかって思っていた。この`mu`バインダーはMichel Parigot先生の`lambda-mu calculus`の部品として最初に紹介されました。
https://link.springer.com/chapter/10.1007%2FBFb0013061

実は、`fix`/`letrec`より`mu`の方が再起を分かりやすくて美しく表現すると思います。

kで`mu`を表現する方法は：

```
// 構文
syntax Val ::= "mu" Id "." Exp  [binder]

// 意味論
rule mu X . E => E[(mu X . E) / X]
 ```

意味論を定義する規則の読み方は「`mu X . E`の書き直し規則を適用する方法は、すべて」
「`mu X . E` の書き直し規則が適用された後、 E の中にある X は全て (mu X . E)に取り替えました」です。
「mu X E から、E の中にある X を全て (mu X . E) と取り替えます。

`mu`の作動を仲良くするため、foldl を`mu`で実装しましょう。foldlを見たことがなければ、

普通の整数入りリスト(nil/consから作られるやつ)を足し算と畳んで、sumを計算していきます。普通の関数言語で、これはパターン一致を使って直感的に書けますね。

ここのfoldlはKで書かれていないんですが、必要となる知識はfoldlのやることだけです。
```
foldl (l : list nat) (g: nat -> nat -> nat) (acc : nat) : nat :=
  match l with
  | [] => acc
  | hd :: tl => foldl' tl g (g acc hd)
```

この関数は引数を三つ受け取って、(l)自然数入りのリスト、(g)自然数受け二項演算と(acc)累算器です。引数として受けたリストは中身があったら、二項演算をヘッド要素と累算器に対して適用して、その演算の結果を新しい累算器としてfoldlの繰り返しへ渡します。リストは中身がなければ、累算器をそのまま結果として返されます。

例えば foldl へ [1;2;3] (+) 0 渡すと、
foldl [1;2;3] (+) (0 + 1) ...
foldl [2;3] (+) (1 + 2) ...
foldl [3] (+) (3+3) ...
foldl [] (+) (6) -> 6

のように評価されます。


で、こういうことを`mu`で作りたいんです。この定義は新しいもので、

1. foldl のボディーの行動を似る不再起ラムダを作る。

```
// ここまで、else にある foldl' は意味がない。
(λ l g acc . if l = [] then acc else (foldl'' tl (g acc hd)))
```

2. 上の関数を `mu` の形式定義の E として入れて

```
mu X . (λ l g acc . (if l = [] then acc else (rec_fold tl g (g acc hd)))) 
    => if l = [] then acc else (fold tl g (g acc hd)) [(mu X . E) / X]
```

3. `mu`規則がバインド・書き直すことを`foldl'`として判明する。
```
mu foldl' . (λ l g acc . (if l = [] then acc else (foldl' tl g (g acc hd)))) 
    => if l = [] then acc else (foldl' tl g (g acc hd)) [(mu foldl' . E) / foldl']
```

4. これを`mu`の書き直し規則の意味論と合わせます。

リストは残りが入ってるなら、`else`ケースになって、その中にfoldl'が書かれていますので、muの書き直し規則は新しいfoldl'召喚を発生してくれます。リストが空になったら、then ケースになって、foldl'書かれていなくて、`mu`の規則が何もしないから、再起が停止します。

確かに長たらしいけど、上にある [1;2;3] (+) 0 を明示的に`mu`で評価すると、

```
mu X . (λ l g acc . (if l = [] then acc else (foldl' tl g (g acc hd)))) => if l = [] then acc else (foldl' tl g (g acc hd)) [(mu X . E) / X]  ([1;2;3], (+), 0)

else ケースと一致するので、

foldl' [2;3] (+) (0 + 1) [(mu X . E) / X]

になって；mu X の代替すれば。。。

mu X . (λ l g acc . (if l = [] then acc else (foldl' tl g (g acc hd)))) => if l = [] then acc else (foldl' tl g (g acc hd)) [(mu X . E) / X]  ([2;3], (+), 1)

foldl' [3] (+) (1 + 2) [(mu X . E) / X]

mu X . (λ l g acc . (if l = [] then acc else (foldl' tl g (g acc hd)))) => if l = [] then acc else (foldl' tl g (g acc hd)) [(mu X . E) / X]  ([3], (+), 3)

foldl' [] (+) (3 + 3) [(mu X . E) / X]

mu X . (λ l g acc . (if l = [] then acc else (foldl' tl g (g acc hd)))) => if l = [] then acc else (foldl' tl g (g acc hd)) [(mu X . E) / X]  ([3], (+), 3)

if l = [] then acc と一致するので、acc だけ返され、acc に foldl' が何もないので mu は停止しました。

```
