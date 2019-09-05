### 前回のおさらい

- Gen: 
- Prop: 
- SGen: 

---

### 8.4 ライブラリの使用とユーザビリティの改善

- これまでに作ったライブラリの表現力・ユーザビリティを確認しよう

---

### 8.4.1 単純な例

- Listのmaxはサイズが0の時に例外を発生する
- 「サイズに基づく生成」のため、1回目のテストで失敗する(サイズ0のテストで失敗)

---

### Exercise 8.13

```scala
def listOf1[A](g: Gen[A]): SGen[List[A]]

// 前回作ったメソッド
def listOf[A](g: Gen[A]): SGen[List[A]] =
  SGen(n => g.listOfN(n))
```

---

### Exercise 8.13 解
- `SGen(n =>...)`のnは0から始まるので0の場合は1に置き換えればOK

```scala
// listOf1の定義
def listOf1[A](g: Gen[A]): SGen[List[A]] = SGen(n => g.listOfN(n max 1))

// maxの仕様を変更
val smallInt = Gen.choose(0,100)
val maxProp = forAll(listOf1(smallInt)) { ns =>
  val max = ns.max
  !ns.exists( _ > max)
}
Prop.run(maxProp)

> + OK, passed 100 tests.
```

---

### Exercise 8.14

---

### Exercise 8.14 解

```scala
val smallInt = Gen.choose(0, 100)
val sortedProp = forAll(listOf(smallInt)) { ns =>
  val nss = ns.sorted
  (nss.isEmpty || nss.tail.isEmpty || !nss.zip(nss.tail).exists {
    case (a,b) => a > b
  }) && !ns.exists(!nss.contains(_)) && !nss.exists(!ns.contains(_))
}
Prop.run(sortedProp)
```

---

### 8.4.2 並列処理のためのテストスイートの作成
- 前章で並列処理において有効でなければならない法則を発見
- これらの法則をこのライブラリで表現可能か

```scala
map(unit(1)(_ + 1) == unit(2)
```

- 表現可能だが美しくない

```scala
val ES: ExecutorService = Executors.newCachedThreadPool
val p1 = Prop.forAll(Gen.unit(Par.unit(1)))(i =>
  Par.map(i)(_ + 1)(ES).get == Par.unit(2)(ES).get)
```

---

### プロパティの証明
- forAllが汎用的すぎる
- 今回のケースはハードコーディングされた例
- それならもっと簡単に書けるはず

```scala
def check(p: => Boolean): Prop = { // 非正格であることに注意
  lazy val result = p // 再計算を避けるために結果を記憶
  forAll(unit(()))(_ => result)
}
```

---

### 「ですが、どうもしっくりきません」
- 値を1つ生成するだけのユニットジェネレータを定義しておきながら、Boolean値の評価を引き出すためにその値を無視しています
- テストケースの数を無視するPropを生成するcheckを実装

```scala
def check(p: => Boolean): Prop = Prop { (_, _, _) =>
  if (p) Passed else Falsified("()", 0)
}
```

---

### 「forAllを使用するよりマシだが」
- 相変わらず"passed 100 test"を出力
- 繰り返しテストしてもしても反証されないという意味であるとは言えない
- どうやら新しい種類のResultが必要

```scala
sealed trait Result
case object Passed extends Result
case class Falsified(failure: FailedCase,
    successes: SuccessCount) extends Result
case object Proved extends Result
```

```scala
def run(p: Prop,
        maxSize: Int = 100,
        testCases: Int = 100,
        rng: RNG = RNG.Simple(System.currentTimeMillis)): Unit =
  p.run(maxSize, testCases, rng) match {
    case Falsified(msg, n) =>
      println(s"! Falsified after $n passed tests:\n $msg")
    case Passed =>
      println(s"+ OK, passed $testCases tests.")
    case Proved =>
      println(s"+ OK, proved property.")
  }
```

- `&&`なども変更する必要がある。些細な変更

---

### Exercise 8.15

---

### Exercise 8.15 解

---

### Parのテスト
- `Par.map(Par.unit(1))(_ + 1)`が`Par.unit(2)`に等しいというプロパティの証明

```scala
val p2 = check {
  val p = Par.map(Par.unit(1))(_ + 1)
  val p2 = Par.unit(2)
  p(ES).get == p2(ES).get
}
```

---

### 「`p(ES)get`と`p2(ES).get`をどうにか」
- かなり不満の残る部分
- 等価性を比較するためにParの内部実装を認識することを強いられている
- map2を使って等価の比較をParにリフトさせれば、結果を取得するために最後に1回Parを実行するだけ

```scala
def equal[A](p: Par[A], p2: Par[A]): Par[Boolean] =
  Par.map2(p,p2)(_ == _)

val p3 = check {
  equal (
    Par.map(Par.unit(1))(_ + 1),
    Par.unit(2)
  ) (ES) get
}
```

---

### 「ですがせっかくなので」
- Parを2回実行する必要がなくなったが
- Parの実行を別の関数forAllParへ移動するのはどうか
- forAllParは様々な並列化戦略の違いを反映させる場所

```scala
// 固定サイズのスレッドプールエグゼキュータの生成確率 75%
// 固定サイズではないスレッドプールエグゼキュータの生成確率 25%
val S = weighted(
  choose(1,4).map(Executors.newFixedThreadPool) -> .75,
  unit(Executors.newCachedThreadPool) -> .25) // `a -> b` is syntax sugar for `(a,b)`
  
def forAllPar[A](g: Gen[A])(f: A => Par[Boolean]): Prop =
  forAll(S.map2(g)((_,_))) { case (s,a) => f(a)(s).get }
```

---

### 「あまり手際の良い方法ではありません」
- `S.map2(g)((_,_))`は２つのジェネレータを結合してそれらの出力をペアで生成するため、あまり手際の良い方法ではない
- 取り急ぎ、コンビネータを作成

```scala
def **[B](g: Gen[B]): Gen[(A,B)] =
  (this map2 g)((_,_))

def forAllPar[A](g: Gen[A])(f: A => Par[Boolean]): Prop =
  forAll(S ** g) { case (s,a) => f(a)(s).get }
```

- カスタム抽出しを使って、**をパターンとして使用

```scala
def forAllPar[A](g: Gen[A])(f: A => Par[Boolean]): Prop =
  forAll(S ** g) { case s ** a => f(a)(s).get }
  
object ** {
  def unapply[A,B](p: (A,B)) = Some(p)
}
```

---

### 完成
- リファクタリングとクリーンアップはライブラリのユーザビリティを大きく向上させる可能性がある

```scala
val p2 = checkPar {
  equal (
  Par.map(Par.unit(1))(_ + 1),
  Par.unit(2)
}
```

- before 

```scala
val p2 = check {
  val p = Par.map(Par.unit(1))(_ + 1)
  val p2 = Par.unit(2)
  p(ES).get == p2(ES).get
}
```

---

### 他のパターン
- 前章でテストケースを一般化したことを思い出す
- 恒等関数を計算にマッピングしても作用は発生しないという法則に簡約

```scala
map(unit(x))(f) == unit(f(x))
```

- 下記を表現できるか

```scala
map(y)(x => x) == y
```

```scala
val pint = Gen.choose(0,10) map (Par.unit(_))
val p4 =
  forAllPar(pint)(n => equal(Par.map(n)(y => y), n))
```

- このテストで十分、DoubleやStringで同じテストを生成してもあまり意味がない
- mapに影響を与える可能性があるのは、並列処理の構造

---

### Exercise 8.16
- `Par[Int]`のより高度なジェネレータを記述せよ。入れ子のレベルが深い並列計算を生成する。

```scala
// 参考
val pint = Gen.choose(0,10) map (Par.unit(_))

val pint2: Gen[Par[Int]]
```

---

### Exercise 8.16 解
- 後で
```scala
val pint2: Gen[Par[Int]] = choose(-100,100).listOfN(choose(0,20)).map(l => 
  l.foldLeft(Par.unit(0))((p,i) => 
    Par.fork { Par.map2(p, Par.unit(i))(_ + _) }))
```

---

### Exercise 8.17
- 第７章のforkに関するプロパティ `fork(x) == x`を実現せよ

```scala
val forkProp = ???
```

---

### Exercise 8.17 解
- mapの場合と同様に解ける

```scala
val forkProp = Prop.forAllPar(pint2)(i => equal(Par.fork(i), i)) tag "fork"
```

---

### 8.5 高階関数のテストと今後の展望
- 高階関数をテストするいい方法がない
- Genでデータを生成できるが関数を生成できない
- `s.takeWhile(f).forAll(f)`がtrueに評価されるプロパティを作りたい

---

### Exercise 8.18
- takeWhileが満たさなければならない他のプロパティを考えだせ。takeWhileとdropWhileの関係をうまく表すプロパティを考えつけるか

```scala
`s.takeWhile(f).forAll(f)` // これ以外に満たすべきプロパティ
```

---

### Exercise 8.18 解
- 他のプロパティ
  - 残りのリストは述語を満たしてない要素で始まるか
  
```scala
l.takeWhile(f) ++ l.dropWhile(f) == l
```

---

### 特定の引数だけを調べる
- 高階関数をテストするときに特定の引数だけを調べる方法もある
- 以下はうまくいくが、takeWhileで使用する関数をテストフレームワークに処理させるわけにはいかないのか

```scala
val isEven = (i: Int) => i%2 == 0
val takeWhileProp = Prop.forAll(Gen.listOf(int))(ns => ns.takeWhile(isEven).forAll(isEven))
```

---

### Gen[String => Int]を生成
- Stringの入力を無視し、Gen[Int]にデリゲートするString => Int関数を生成する
- しかし定数関数を生成しているだけ

```scala
def genStringIntFn(g: Gen[Int]): Gen[String => Int] = 
  g map (i => (s => i))
```

---

### Exercise 8.19
### Exercise 8.19 解
### Exercise 8.20
### Exercise 8.20 解

### 8.6 ジェネレータの法則
- Genに実装してきた関数はこれまで実装してきたPar, List, Stream, Optionは関数と似ている
- 関数が同じようなシグネチャを共有しているだけなのか、それとも同じ法則を満たしているのか
- 何か大きな力が働いている

### 8.l7 まとめ
- 
