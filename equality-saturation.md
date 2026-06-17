# equality saturation

ここまでの最適化器は、人間が決めた変換を人間が決めた順序で適用してきた。
だがこの設計には、本書で何度も顔を出した**[フェーズ順序問題](#index:フェーズ順序問題)**がつきまとう。
A を先にやるか B を先にやるかで結果が変わり、最適な順序は一般に分からない。
さらに悪いことに、変換を破壊的に適用すると、もう片方の機会を永久に潰してしまう。

この章で扱う**[equality saturation](#index:equality saturation)**（等価飽和）[](#cite:tate2009)は、
この問題に構造的に挑む。発想は大胆だ。「変換を順に適用する」のをやめ、
等価な書き換えをすべて同時に一つのデータ構造に詰め込み、最後に最良の一つを取り出す。
順序を選ばない（全部試す）ことで、フェーズ順序問題そのものを回避する。

## 破壊的書き換えの何が問題か

具体例で問題を体感しよう。`x * 2` という式を最適化したい。二つの書き換え規則があるとする。

- 規則 A: `x * 2 → x << 1`（[強度低減](peephole.md)：乗算をシフトに）
- 規則 B: `(x * 2) / 2 → x`（約分）

> [!NOTE]
> 規則 B は数学的整数の上での話だ。固定幅整数では `x * 2` がオーバーフローしうるし、
> 符号付き除算は 0 方向への切り捨てを伴うため、`(x * 2) / 2` は一般には `x` と等しくない。
> ここでは順序問題の説明に集中するため数学的整数を仮定するが、実際の処理系では各書き換えに
> 「成り立つ意味領域」を必ず添える必要がある（本章末でも改めて触れる）。

いま式が `(x * 2) / 2` だったとする。規則 A を先に適用すると `(x << 1) / 2` になり、
規則 B のパターン `(x * 2) / 2` がもうマッチしなくなり、`x` まで簡約する好機を逃す。
逆に規則 B を先に使えば `x` になり、こちらが正解だ。

これが破壊的書き換えの罠だ。「いま得に見える変換（A）」が、「あとで大きな得になる変換（B）」を
潰す。どちらを先にやるべきかは、式によって違い、一般には**先読みしないと分からない**。
従来のコンパイラはヒューリスティックな順序とパスの繰り返しでしのいできたが、根本解決ではなかった。

> [!NOTE]
> この問題は[命令選択](instruction-selection.md)の「まとめ技タイル」や、
> [値番号付け](local-optimization.md)の「構造が違う等価式」でも見た。
> 「等価な表現が複数あり、どれが最良かは文脈次第」。これは最適化の至るところに潜む難しさだ。
> equality saturation は、この難しさに「全部の表現を同時に保持する」という力技で応える。

## e-graph

equality saturation の心臓は**[e-graph](#index:e-graph)**（equivalence graph; 等価グラフ）という
データ構造だ[](#cite:tate2009)[](#cite:willsey2021)。e-graph は「等価だと判明した式を、一つの箱に
まとめて」保持する。この箱を **e-class**（等価クラス）と呼ぶ。

通常の式の木では、各ノードは「自分が何の演算で、子は誰か」を持つ。e-graph では、
ノード（**e-node**）の子は、個別の式ではなく e-class を指す。そして「`x * 2` と `x << 1` は等価」と
分かったら、両者を同じ e-class に入れる。一つの e-class は「互いに等価な式の集合」を表す。

```
e-class C0 = { x }
e-class C1 = { 2 }
e-class C2 = { (* C0 C1), (<< C0 1) }   ← x*2 と x<<1 が同じクラス（等価）に同居
```

ここが核心だ。`x * 2` を `x << 1` に置き換えるのではなく、e-graph に
「両方とも C2 の住人だ」と**記録するだけ**だ。元の `x * 2` は消えない。だから規則 A を適用しても、
規則 B が必要とする `x * 2` の表現は e-graph に残り続ける。破壊しないから、順序を選ぶ必要がない。

e-graph は、互いに等価なたくさんの式を指数的に圧縮して持てる。
n 個の独立な等価書き換えがあれば表現しうる式は 2ⁿ 通りにもなるが、e-graph は共有のおかげで
コンパクトに保持できる。「等価な全表現を一度に持つ」という一見無理な要求が、現実的になる。

## 飽和

e-graph の上で、書き換え規則を等価の追加として適用する。規則 `l → r` は、
「e-graph 内で `l` にマッチする箇所を見つけ、`r` を作り、`l` の e-class と `r` の e-class を
併合（union）する」という操作になる。これを規則の適用で新しい等価が一つも増えなくなるまで
繰り返す。この状態を**[飽和](#index:飽和)**（saturation）と呼ぶ[](#cite:tate2009)。

```ruby
# 概念的な equality saturation のループ
def equality_saturation(egraph, rules)
  loop do
    matches = rules.flat_map { |rule| egraph.search(rule.lhs) }   # 全規則のマッチを探す
    changed = false
    matches.each do |m|
      new_id = egraph.add(m.rule.rhs, m.subst)   # 右辺を e-graph に追加
      changed |= egraph.union(m.eclass, new_id)  # 左辺の e-class と併合
    end
    egraph.rebuild                                # 併合を伝播（合同閉包を保つ）
    break unless changed                          # もう増えない = 飽和
  end
end
```

ここで一つ理論的な仕掛けがある。e-class を併合すると、それを子に持つ式どうしも等価になる
（`a` と `b` が等価なら `f(a)` と `f(b)` も等価）。この「等価の伝播」を**合同閉包**
（congruence closure）と呼び、`rebuild` がそれを保つ。egg[](#cite:willsey2021)の大きな貢献は、
この rebuild を毎回でなくまとめて行う遅延戦略で、equality saturation を桁違いに高速化したことだ。
これにより equality saturation が研究室の理論から実用ツールへと近づいた。

飽和に達した e-graph は、「適用可能なすべての書き換えを尽くした、等価式の宇宙」だ。
**飽和まで到達できれば**、その時点の等価クラスは規則の適用順序によらず一意に定まる。
だから「どの順で適用するか」というフェーズ順序問題を、構造的に避けられる。
ただしこれは飽和に届く範囲での話だ。規則集合によっては e-graph が際限なく膨らんで飽和しないため、
実際には反復回数やサイズで打ち切る。打ち切りが入ると、最終的にどの式が取り出せるかは
規則の適用順や打ち切り点に左右されうるので、完全な順序非依存（合流性）を無条件に保証できるわけではない。

> [!WARNING]
> 規則によっては飽和に達しないことがある。`x → x + 0 → x + 0 + 0 → ...` のように
> 無限に式を生む規則があると、e-graph は際限なく膨れる。実際の equality saturation は、
> 飽和しなければ「ノード数や時間の上限で打ち切る」。だから厳密には「飽和または打ち切りまで」だ。
> どんな規則集合なら飽和するか、膨張をどう抑えるかは、この分野の実践的な課題である。

## 抽出

飽和した e-graph は「等価式の宇宙」だが、最終的に欲しいのは**一つの最良な式**だ。
e-graph から、各 e-class につき一つの e-node を選んで、コストが最小の式を取り出す。
これを**[抽出](#index:抽出)**（extraction）と呼ぶ。

各 e-node にコスト（命令数、遅延など）を与え、「各 e-class の最小コスト」を
葉から根へボトムアップに求める。これは[命令選択の DP](instruction-selection.md)とそっくりだ。

```ruby
# 各 e-class の最小コストと、それを実現する e-node を求める
def extract(egraph, cost_fn)
  best = {}            # eclass => [cost, enode]
  loop do
    changed = false
    egraph.eclasses.each do |ec, enodes|
      enodes.each do |node|
        # この e-node のコスト = 自身のコスト + 子 e-class の最小コストの和
        next unless node.children.all? { |c| best.key?(c) }
        c = cost_fn[node.op] + node.children.sum { |ch| best[ch][0] }
        if !best[ec] || c < best[ec][0]
          best[ec] = [c, node]; changed = true
        end
      end
    end
    break unless changed   # 不動点
  end
  build_term(egraph.root, best)   # 根から選択を辿って式を組み立てる
end
```

`(x * 2) / 2` の例では、飽和後の e-graph に `x` も `(x<<1)/2` も `(x*2)/2` もすべて
等価として同居している。抽出は「最小コスト」を選ぶので、迷わず `x` を取り出す。
規則の適用順序を一切気にせず、最良の結果が得られた。これが equality saturation の威力だ。

> [!NOTE]
> 抽出も実は奥が深い。共有のある e-graph で「真に最小コスト」を選ぶのは
> [DAG の命令選択](instruction-selection.md)と同様に難しく（一般に NP 困難）、
> 貪欲な DP は近似になる。厳密抽出には ILP ソルバを使う研究もある。
> 「等価式を全部持つのは飽和、最良を選ぶのは抽出」。この二段が equality saturation の骨格だ。

## どこで使われているか

equality saturation は研究段階を脱しつつある。egg ライブラリ[](#cite:willsey2021)の登場で実装が容易になり、
応用が広がった。

- **浮動小数点の精度最適化**（Herbie）: 実数として等価な書き換えの中から、浮動小数点で評価したときの誤差が小さい式を探す（ビット等価な変形ではなく、丸め誤差を減らす近似的な書き換えだ）。
- **ハードウェア設計とテンソル計算**: 深層学習のテンソルグラフを最適化する。Tensat[](#cite:yang2021tensat)は
  テンソルグラフ超最適化を equality saturation で行い、逐次探索の TASO 比で最適化時間を 1/48 に縮めた。
- **CAD、記号計算、型推論**: 等式的書き換えが効く領域全般。
- **従来コンパイラの最適化**: Tate らの原論文[](#cite:tate2009)は、LLVM 的な最適化を
  equality saturation で行い、人手のパス順序を上回る例を示した。

そして egg の登場後、equality saturation は**プロダクションのコンパイラ**にも届いた。WebAssembly ランタイム
Wasmtime のバックエンド Cranelift は、中間最適化器に ægraph（acyclic e-graph）[](#cite:fallin2023aegraph)を
採用している。ただし egg 流の飽和は JIT の速度制約には重すぎたため、SSA の非循環性を活かして
飽和を一回走査に畳み込み、ノード構築時に即座に書き換える形へ作り替えられた。
「等価な式を全部持ってから選ぶ」という理論的な発想が、実用コンパイラの速度制約と折り合った好例だ。

> [!NOTE]
> **発展**：egg 以後、equality saturation は単なるツールから研究プログラムへ育った。
> e-graph 上のパターン照合を関係データベースの結合として定式化し高速化した relational e-matching[](#cite:zhang2022relational)、
> equality saturation と [Datalog](dataflow-analysis.md) を統合して書き換えと解析を一体化した egglog[](#cite:zhang2023egglog)、
> 書き換え規則そのものを自動で見つける Ruler[](#cite:nandi2021ruler)など、理論と道具の両面で発展が続いている。

equality saturation は[第 II 部の値番号付け](local-optimization.md)や
[SSA の GVN](ssa-optimization.md)の正統な後継だと言える。値番号付けが「構造が同じ式を一つにまとめる」のに対し、
equality saturation は「**書き換え規則で等価と証明できる式すべてを一つにまとめる**」。
等価性の扱いを、構文的一致から意味的書き換えへと、徹底的に推し進めたものだ。
そして[sea of nodes](ssa-optimization.md)が「一つの最良グラフを育てる」のに対し、e-graph は
「等価なグラフ全部を保持してから選ぶ」。同じ問題意識の、対照的な二つの答えになっている。

## まとめ

- 破壊的書き換えは「今の得が後の得を潰す」フェーズ順序問題を生む。equality saturation はこれを構造的に回避する。
- e-graph[](#cite:tate2009)は等価な式を e-class にまとめて保持し、書き換えを「置換」でなく「等価の追加（union）」として扱う。だから何も壊れない。
- 規則を増えなくなるまで適用するのが飽和、最小コストの式を取り出すのが抽出。egg[](#cite:willsey2021)は rebuild の遅延で実用速度を実現した。
- 数値精度、テンソル、コンパイラ最適化で実用化が進む。値番号付けと sea of nodes の正統な後継。

equality saturation が「等価な変換を全部試す」なら、次章はさらに過激だ。
「等価な命令列を探索で見つけ出す」superoptimization と、その正しさを機械に証明させる検証へ進む。
