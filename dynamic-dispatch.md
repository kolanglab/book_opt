# 動的ディスパッチを支えるインラインキャッシュと隠れクラス

動的言語の `obj.method` は、見た目より遥かに高くつく。`obj` の型は実行時まで分からず、
クラスは継承され、`method_missing` のようなフックもあり、クラス自体が実行中に書き換わる。
だから素朴な実装は、呼び出しのたびにメソッド表を辿る**[メソッド探索](#index:メソッド探索)**
（method lookup）を行う。ループの中なら、これが何百万回も繰り返される。なお、インタプリタが命令を一つずつ読んで分岐するディスパッチ自体にもコストがあり、スレッデッドコード[](#cite:bell1973)や効率的なインタプリタ設計[](#cite:ertl2003)で削られてきたが、それは本書が扱う最適化とは層が異なる話だ。本章が狙うのは、その上に乗る言語機能（メソッド探索とインスタンス変数アクセス）のコストである。

この章では、動的言語最大のコストであるメソッド探索とインスタンス変数アクセスを、
**[インラインキャッシュ](#index:インラインキャッシュ)**[](#cite:deutsch1984)と
**[隠れクラス](#index:隠れクラス)**（shape）で削り取る方法を扱う。
鍵となる発想はただ一つ、「前回ここを通ったときの結果を覚えておく」である。
この単純な観測が、毎回の辞書引きを一回のポインタ比較に化けさせる。

## メソッド探索はなぜ遅いのか

Ruby のメソッド呼び出しを思い浮かべよう。`obj.foo` は概ねこう解決される。

1. `obj` のクラスを得る。
2. そのクラスのメソッド表に `foo` があるか調べる。
3. なければスーパークラスへ。Object、BasicObject まで遡る。
4. それでもなければ `method_missing`。

継承が深いと探索は何段にもなる。Ruby を素朴に書くとこんなイメージだ。

```ruby
def lookup(klass, name)
  while klass
    m = klass.method_table[name]
    return m if m
    klass = klass.superclass     # 見つかるまでスーパークラスを遡る
  end
  nil
end
```

問題は、**同じ呼び出し点 `obj.foo` は、たいてい毎回同じクラスの `obj` に対して実行される**こと。
ループで配列をなめれば `obj` は毎回 `Integer` かもしれない。なのに素朴な実装は、
毎回ゼロから探索をやり直す。この「同じ問いを繰り返している」性質を突くのがキャッシュである。

## 単相インラインキャッシュ

**インラインキャッシュ**（inline cache; IC）は、Deutsch と Schiffman が Smalltalk-80 で導入した
古典技法だ[](#cite:deutsch1984)。発想は「探索結果を、その呼び出し点の隣に（inline に）覚えておく」ことである。
各呼び出し点に「前回の `obj` のクラス」と「そのとき解決したメソッド」をキャッシュとして持たせる。

次にその呼び出し点を通ったとき、`obj` のクラスが**前回と同じ**なら、探索をスキップして
キャッシュしたメソッドを直に呼ぶ。違ったら探索し直してキャッシュを更新する。

```ruby
class CallSite
  def initialize = (@cached_class = nil; @cached_method = nil)

  def call(obj, name, *args)
    klass = obj.class
    if klass == @cached_class            # キャッシュヒット: クラス比較 1 回だけ
      @cached_method.bind(obj).call(*args)
    else                                 # ミス: 探索してキャッシュ更新
      @cached_method = lookup(klass, name)
      @cached_class  = klass
      @cached_method.bind(obj).call(*args)
    end
  end
end
```

ヒット時のコストは「クラスへのポインタ 1 個の比較」にまで縮む。これが**[単相インラインキャッシュ](#index:単相インラインキャッシュ)**
（monomorphic inline cache; 単相 IC）だ。「単相」とは、その呼び出し点が一種類の型だけを見る状態を指す。
実測上、ほとんどの呼び出し点は単相であることが知られていて[](#cite:deutsch1984)、だからこそ IC は劇的に効く。

> [!NOTE]
> キャッシュは「クラスが前回と同じ」を仮定する。だが Ruby ではクラスにメソッドを後付けできるので、
> 「同じクラスだが、メソッド定義が変わった」場合にキャッシュが古くなる。これを検出するため、
> 各クラス（または大域）に**バージョン番号**（serial/global method state）を持たせ、
> メソッド定義が変わるたびに番号を増やす。キャッシュはクラスに加えてこの番号も照合し、
> 番号が変わっていたら無効化する。キャッシュは必ず失効の仕組みとセットになる。
> [局所最適化の章](local-optimization.md)で見た「事実は更新と失効を管理せよ」の再来だ。

## 多相インラインキャッシュ（PIC）

呼び出し点が**複数の型**を見ることもある。`shapes.each { |s| s.area }` で `shapes` に
`Circle` も `Square` も入っていれば、`s.area` は二つの型を交互に見る。これを**多相**（polymorphic）と呼ぶ。
単相 IC はこの場合、毎回キャッシュミスして探索し直す。キャッシュが役に立たない最悪状態（**スラッシング**）になる。

Hölzle、Chambers、Ungar は Self の研究で、これを**[多相インラインキャッシュ](#index:多相インラインキャッシュ)**
（polymorphic inline cache; PIC）に拡張した[](#cite:holzle1991)。一つの呼び出し点に
**複数の(クラス, メソッド)対**を覚え、観測した型を順に増やしていく。

```ruby
class PolymorphicCallSite
  def initialize(limit = 4) = (@cache = {}; @limit = limit)

  def call(obj, name, *args)
    klass = obj.class
    method = @cache[klass]
    unless method                        # この型は初見
      method = lookup(klass, name)
      @cache[klass] = method if @cache.size < @limit   # 上限まで覚える
    end
    method.bind(obj).call(*args)
  end
  # @cache が limit を超えたら「メガモーフィック」: キャッシュを諦め大域メソッドキャッシュへ
end
```

PIC は「この呼び出し点が見た型の集合」そのものを記録する。型が 2〜数種なら全部キャッシュして高速だ。
だが種類が多すぎる（上限超過）と**メガモーフィック**（megamorphic）状態になり、
キャッシュを諦めて共有のメソッドキャッシュに頼る。単相 → 多相 → メガモーフィックという
呼び出し点の「型の温度」は、次章の[投機的最適化](speculative-optimization.md)でも重要な情報になる。
単相な呼び出し点は「型をほぼ確信できる」ので、投機的インラインの絶好の標的だ。

> [!TIP]
> PIC は「キャッシュ」であると同時に、**プロファイラ**でもある点が深い。
> 「この呼び出し点はどの型を、どんな頻度で見たか」という情報が PIC に貯まる。
> Hölzle と Ungar はこれを**型フィードバック**[](#cite:holzle1994)として最適化コンパイラに渡し、
> 「観測された型を仮定してインラインする」投機的最適化の土台にした。
> IC は高速化の道具であると同時に、JIT に「現実」を教える情報源なのだ。

## 隠れクラス（shape）によるインスタンス変数アクセス

メソッド探索と並ぶ動的言語のコストが、**インスタンス変数アクセス**だ。
`obj.x` を読むとき、`obj` がハッシュ表（`{"x" => 1, "y" => 2}`）でフィールドを持っていたら、
毎回ハッシュ引きが要る。静的言語なら「`x` はオブジェクト先頭から 8 バイト目」と固定オフセットで
読めるのに、動的言語はオブジェクトごとにフィールド構成が違いうるので、素朴にはそれができない。

ここで効くのが**[隠れクラス](#index:隠れクラス)**（hidden class）あるいは**シェイプ**（shape）だ。
V8 が "hidden class"、Self/JavaScriptCore が "maps"、Ruby 3.2 以降が "Object Shapes" と呼ぶ、同じ発想である。
アイデアは「同じフィールド構成を持つオブジェクトをまとめ、その構成（shape）を共有する」ことだ。
shape は「どのフィールドが何番目のオフセットにあるか」を表で持つ。各オブジェクトは
自分の値の配列と、shape へのポインタだけを持つ。

```
オブジェクト              Shape（共有）
+----------+            +------------------+
| shape* ──┼──────────▶ | x → offset 0     |
| [1, 2]   |            | y → offset 1     |
+----------+            +------------------+
```

フィールドを追加すると（`obj.z = 3`）、新しい shape に**遷移**する。
shape は「フィールド追加の履歴」を木として共有するので、同じ順序でフィールドを足したオブジェクトは
同じ shape に行き着く。Ruby 風に書くとこうだ。

```ruby
class Shape
  attr_reader :offsets
  def initialize(offsets = {}) = @offsets = offsets
  def transition(field)                    # field を追加した新 shape を返す（キャッシュ）
    @children ||= {}
    @children[field] ||= Shape.new(@offsets.merge(field => @offsets.size))
  end
end

class DynObject
  def initialize = (@shape = ROOT_SHAPE; @values = [])
  def set(field, v)
    off = @shape.offsets[field]
    if off then @values[off] = v
    else
      @shape = @shape.transition(field)    # 新フィールド: shape を遷移
      @values << v
    end
  end
  def get(field) = @values[@shape.offsets[field]]
end
```

shape の威力は IC と組んだときに出る。`obj.x` の呼び出し点に「前回の shape と、そのときの x のオフセット」を
キャッシュすれば、shape が一致する限り `obj.x` は「shape ポインタ比較 1 回＋固定オフセットの配列読み 1 回」になる。
静的言語のフィールドアクセスと同じ速さだ。動的なのに静的並みを実現するのが、この shape ＋ IC である。

> [!NOTE]
> shape を活かすには、**フィールドを毎回同じ順序で初期化する**コードが速い。
> JavaScript で「コンストラクタによってフィールドの追加順が違う」オブジェクトは、別々の shape に分かれ、
> 呼び出し点が多相化して遅くなる。「最適化に優しいコードの書き方」が処理系の内部構造から導かれる、
> よい例だ。Ruby の Object Shapes 導入後も、`initialize` でのインスタンス変数定義順が性能に効くようになった。

## まとめ

- 動的言語のメソッド探索とインスタンス変数アクセスは素朴には毎回辞書引きで遅い。だが「同じ呼び出し点は同じ型を繰り返し見る」性質を突ける。
- 単相インラインキャッシュ[](#cite:deutsch1984)は探索結果を呼び出し点に覚え、クラス比較 1 回に縮める。失効はバージョン番号で管理する。
- 多相インラインキャッシュ（PIC）[](#cite:holzle1991)は複数型を覚え、メガモーフィックなら諦める。PIC は型フィードバック[](#cite:holzle1994)の情報源でもある。
- 隠れクラス（shape）はフィールド構成を共有し、IC と組んでインスタンス変数アクセスを固定オフセット読みに化けさせる。

IC と shape は「観測した型を覚えて速くする」キャッシュだった。次章は一歩進めて、
観測した型を**仮定してコードを特殊化し、外れたら逃げる**。投機的最適化と脱最適化の世界へ入る。
