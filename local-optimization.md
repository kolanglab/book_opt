# 局所最適化 — 定数伝播・コピー伝播・値番号付け

ピープホールが数命令の窓だったのに対し、この章は**基本ブロック 1 個**に視野を広げる。
ブロック内は分岐の出入りがなく一直線に実行されるので、命令を上から順に「解釈」していけば、
各時点で何が定数で、何と何が等しいかを正確に追える。この「直線コードを賢く読む」という
土俵で、最適化の定番——[定数畳み込み](#index:定数畳み込み)・[定数伝播](#index:定数伝播)・
[コピー伝播](#index:コピー伝播)・[共通部分式除去](#index:共通部分式除去)——を一通りそろえる
[](#cite:aho2006)[](#cite:muchnick1997)。

これらは古典中の古典だが、Kildall が 1973 年に「すべて同じデータフロー枠組みの実例だ」と
示した[](#cite:kildall1973)とおり、互いに深く関係している。本章では基本ブロック内（局所版）で
直観を作り、関数全体への拡張（大域版）は [ssa-optimization](ssa-optimization.md) に渡す。

## 定数伝播と定数畳み込み — 値を持ち回る

[前章](peephole.md)の定数畳み込みは「`2 * 3` を `6` にする」だけだった。
だが実際のコードでは、定数は変数を経由して間接的に現れる。

```ruby
x = 3
y = x * 2     # x が 3 と分かれば、これは 3*2 = 6
z = y + 1     # y が 6 と分かれば、これは 7
```

`x = 3` という事実をブロック内に**持ち回り**、各使用箇所で `x` を `3` に置き換える——
これが**定数伝播**だ。置き換えた結果 `3 * 2` のような定数式ができたら、
畳み込み（定数畳み込み）で `6` にする。両者は手と手を取り合って働く。

実装は、ブロックを上からたどりながら「いま定数と分かっている変数 → その値」の表
（環境）を更新していくだけだ。代入の右辺を環境で評価し、定数になればその変数も表に入れる。

```ruby
def local_const_prop(block)
  env = {}                       # 変数名 => 確定した定数値
  block.map do |i|
    case i.op
    when :assign
      v = eval_operand(i.args[0], env)
      if v.is_a?(Integer)
        env[i.dst] = v                              # 定数なら記録
      else
        env.delete(i.dst)                           # 定数でなくなったら表から外す
      end
      Instr.new(:assign, i.dst, [(v || i.args[0]).to_s])
    when :binop
      op, a, b = i.args
      va, vb = eval_operand(a, env), eval_operand(b, env)
      if va.is_a?(Integer) && vb.is_a?(Integer)
        folded = va.send(op, vb)                    # 定数畳み込み
        env[i.dst] = folded
        Instr.new(:assign, i.dst, [folded.to_s])
      else
        env.delete(i.dst)                           # 定数でなくなったら表から外す
        Instr.new(:binop, i.dst, [op, va&.to_s || a, vb&.to_s || b])
      end
    else i
    end
  end
end

def eval_operand(s, env)
  return Integer(s) rescue nil if s =~ /\A-?\d+\z/  # リテラル
  env[s]                                            # 変数なら環境を引く（無ければ nil）
end
```

`env.delete(i.dst)` の一行が地味に重要だ。`x` がいったん定数 `3` でも、後で `x = foo()` と
再代入されたら `x` はもう定数ではない。表から消し忘れると、古い値で誤って置換してしまう。
**事実は更新と失効を正しく管理しなければならない**——最適化の正しさは、こういう細部に宿る。

> [!NOTE]
> 分岐のある関数全体では、`if` の二経路から `1` と `2` が合流して「定数でない」となる場合がある
> （[前章のデータフロー解析](dataflow-analysis.md)で見た合流）。
> 局所版はブロック内だけなので合流が起きず単純だが、大域版は格子上の合流（meet）が要る。
> さらに「定数になった条件分岐は決して通らない枝を生む」ことに気づくと、
> 解析と最適化を同時に回す **SCCP**[](#cite:wegman1991) に到達する——[ssa-optimization](ssa-optimization.md) で扱う。

## コピー伝播 — 余計な別名を消す

`x = y` のような**コピー代入**は、最適化の副産物として大量に生まれる
（ピープホールの `x+0 → x` も、SSA からの脱出時の φ 除去もコピーを作る）。
コピーがあると、本来 `y` を使えば済むところで `x` を経由してしまう。
**コピー伝播**は、`x = y` の後で `x` を使っている箇所を `y` で置き換える。

```ruby
a = b
c = a + 1     # → c = b + 1（a を b に置換）
```

これ単独では命令は減らないが、`a` を使う箇所が全部 `b` に置き換われば、
`a = b` 自体が[不要コード除去](dead-code-elimination.md)で消せるようになる。
最適化は単独では効かず、**互いの機会を作り合って連鎖する**——コピー伝播はその典型だ。

実装は定数伝播とそっくりで、表が「定数値」ではなく「コピー元の変数」になるだけだ。
正しさの条件も同じで、`x = y` の後に `y` が再代入されたらコピー関係は失効する。

```ruby
def copy_propagation(block)
  copy = {}                      # x => y （x は y のコピー）
  block.map do |i|
    args = i.args.map { |a| copy.fetch(a, a) }       # オペランドを代表元に置換
    ni = i.class.new(i.op, i.dst, args)
    copy.reject! { |k, v| k == i.dst || v == i.dst } # dst が絡む関係は失効
    copy[i.dst] = args[0] if ni.op == :assign && variable?(args[0])
    ni
  end
end

def variable?(s) = s.is_a?(String) && s !~ /\A-?\d+\z/
```

`copy.reject!` で「代入先 `dst` をキーまたは値に含む関係」を消しているのがポイント。
`x` を上書きしたら `x = y` は無効だし、`y` を上書きしたら `... = y` だったコピーも
（古い `y` を指していたので）無効になる。

## 値番号付け — 「同じ計算」を見抜く

最後に、共通部分式除去（CSE）を支える**[値番号付け](#index:値番号付け)**（value numbering）を扱う。
アイデアは、各値に**番号**を振り、「同じ番号 ＝ 必ず同じ値」を保証すること。
`a + b` を計算したら、その結果に番号を割り当て、`(+, #a, #b)` という式を番号に対応づけて覚える。
次に同じオペランドで `+` が現れたら、再計算せず前の結果を使い回せる[](#cite:aho2006)。

```ruby
a = x + y
b = x + y     # x, y が不変なら a と同じ値 → b = a に置換できる
```

局所版（ブロック内）の値番号付けは、ハッシュ表一つで書ける。
キーは「演算子と、オペランドの値番号の組」、値は「その式を最初に計算した変数」だ。

```ruby
def value_numbering(block)
  num   = {}    # 変数名 => 値番号
  expr  = {}    # [op, vn1, vn2] => 既にその値を持つ変数名
  next_vn = 0
  result = []

  block.each do |i|
    if i.op == :binop
      op, a, b = i.args
      va = (num[a] ||= (next_vn += 1))    # オペランドに値番号を（無ければ新規に）
      vb = (num[b] ||= (next_vn += 1))
      va, vb = vb, va if commutative?(op) && vb < va   # 可換演算は正規化（a+b と b+a を同一視）
      key = [op, va, vb]
      if (prev = expr[key])               # 同じ式を計算済み？
        result << Instr.new(:assign, i.dst, [prev])    # → 再利用
        num[i.dst] = num[prev]
      else
        result << i
        num[i.dst] = (next_vn += 1)       # 新しい値を生んだ
        expr[key] = i.dst
      end
    else
      result << i
      num[i.dst] = (next_vn += 1) if i.dst
    end
  end
  result
end

def commutative?(op) = %i[+ * & | ^].include?(op)
```

可換演算の**正規化**（`a+b` と `b+a` の値番号を揃える）を入れておくと、検出率が上がる。
これは小さな工夫だが、「等価なものを同じ表現に正規化してから比較する」という発想は、
[第 VI 部の equality saturation](equality-saturation.md) で大規模に再登場する重要な考え方だ。

> [!TIP]
> 値番号付けは「式の構造が同じ」を見るが、`a*2` と `a+a` のような**意味は同じだが構造が違う**
> 式は別物と判定する。これを揃えるには代数的な書き換え（再結合・分配則）が要り、
> どこまでやるかが難しい。局所値番号付けは安価で堅実な下限、
> equality saturation は「等価な書き換えを総当たりする」上限、と位置づけられる。

## 局所から大域へ

本章の三つ——定数伝播・コピー伝播・値番号付け——はすべて「ブロックを上から読み、
表を更新しながら置換する」同じ骨格でできている。違いは表が持つ事実
（定数値・コピー元・値番号）だけだ。

だが視野をブロックの外、関数全体に広げると、合流点で事実がぶつかる問題が出る。
そこで効くのが [intermediate-representation.md](intermediate-representation.md) で導入した **SSA 形式**だ。
SSA では各変数が単一定義なので、「`x` の値は何か」を表で持ち回る必要すらなくなり、
これらの局所最適化が自然に関数全体へ拡張される。次章 [ssa-optimization](ssa-optimization.md) で、
その威力——疎な定数伝播 SCCP[](#cite:wegman1991) と大域値番号付け[](#cite:alpern1988)——を見る。

その前に、本章までで「冗長な計算」をたくさん作った（コピー代入や使われない一時変数）。
それらを安全に掃除する[不要コード除去](dead-code-elimination.md)を、先に片付けておこう。
