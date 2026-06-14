# SSA 上の最適化 — 局所最適化を関数全体へ

[局所最適化の章](local-optimization.md)で作った定数伝播・値番号付けは、基本ブロック 1 個の中だけで働いた。
分岐や合流があると「表を持ち回る」素朴な方法は破綻する。これを関数全体へ持ち上げる装置が
[SSA 形式](intermediate-representation.md)だ。各変数が単一定義になるおかげで、
「`x` の値はどこから来たか」を追う手間が消え、局所的だった最適化が**大域最適化**へと自然に拡張される。

この章では、まず SSA の構築（φ をどこに置くか）を支配辺境の考え方とともに固め、
次にその上で動く二つの代表的大域最適化——疎な条件付き定数伝播 **SCCP**[](#cite:wegman1991)と
**大域値番号付け（GVN）**[](#cite:alpern1988)——を実装する。最後に、制御とデータを一枚のグラフに統合した
近代 IR「[sea of nodes](#index:sea of nodes)」[](#cite:click1995ir)へ橋渡しする。

## SSA を作る — φ はどこに置くか

[intermediate-representation.md](intermediate-representation.md) で、合流点に φ 関数を置くと述べた。
問題は「**どの**合流点に置くか」だ。全合流点に全変数の φ を置けば安全だが無駄だらけになる。
Cytron らの答えは**[支配辺境](#index:支配辺境)**（dominance frontier）を使うことだった[](#cite:cytron1991)。

ブロック X の支配辺境とは、直観的には「X が支配する領域の、ちょうど外縁」のこと。
厳密には「X が支配する（が、その先行は X に支配されない）ようなブロック」の集合だ。
変数の定義がブロック X にあるとき、その値の影響が他の経路の値とぶつかりうるのは X の支配辺境であり、
**そこにこそ φ が要る**。φ を置くとそれ自体が新たな定義になるので、その支配辺境にも φ が要る——
という反復で、過不足ない最小限の φ 配置が決まる。

支配辺境は支配木から計算できる。Cooper–Harvey–Kennedy の方法[](#cite:cooper2001)は、
各ブロックの先行をたどって idom にぶつかるまで遡る、という単純な手続きでこれを求める。

```ruby
# idom: {ブロック => 直接支配者}, preds: {ブロック => 先行ブロック配列}
def dominance_frontiers(blocks, idom, preds)
  df = Hash.new { |h, k| h[k] = Set.new }
  blocks.each do |b|
    next if preds[b].size < 2            # 合流点（先行 2 以上）でのみ φ が要る
    preds[b].each do |p|
      runner = p
      while runner != idom[b]            # idom[b] に達するまで支配木を遡り…
        df[runner] << b                  # …通ったブロックの辺境に b を加える
        runner = idom[runner]
      end
    end
  end
  df
end
```

φ を置いたら、各変数を版番号で名前替えする（`x → x1, x2, ...`）。
これは支配木を上から辿り、「いまその変数の最新版は何番か」をスタックで管理しながら、
使用箇所を最新版に、定義箇所を新版に書き換えていく。実装の詳細は紙幅の都合で
[原論文](#cite:cytron1991)に譲るが、要点は「**φ 配置は支配辺境、名前替えは支配木の走査**」
という、すべて支配関係に支えられた手続きであることだ。

> [!NOTE]
> 近年は、CFG を作りながら直接 SSA を構築する Braun らの手法（"Simple and Efficient
> Construction of SSA Form"）も広く使われ、フロントエンドから SSA を直接吐ける。
> どの方法でも、できあがる SSA の性質——単一定義・φ・支配——は同じだ。本書は性質の側に集中する。

## SCCP — 定数伝播と到達可能性を同時に解く

SSA の威力を最もよく示すのが**[SCCP](#index:SCCP)**（sparse conditional constant propagation;
疎な条件付き定数伝播）だ[](#cite:wegman1991)。これは二つの最適化を**同時に**行う。

1. **定数伝播**: 各 SSA 変数が定数か `⊤`（不明）かを求める。
2. **到達可能性**: 各ブロックが実行されうるかを求める。

なぜ「同時に」が効くのか。定数が分かると、`if x > 0` で `x` が定数 `5` なら
**真の枝しか通らない**と分かり、偽の枝は到達不能になる。逆に到達不能と分かったブロックの
代入は**合流に参加しない**ので、φ の片方の入力が消え、合流結果が定数になることがある。
[dead-code-elimination.md](dead-code-elimination.md) で触れた「楽観的に始める」構えがここでも鍵だ——
最初は**全ブロック到達不能・全変数 ⊤** と仮定し、到達可能と証明できたものだけ「点火」していく。

格子は単純で、各変数は `⊤`（まだ不明＝定数かも）・具体的な定数・`⊥`（定数でない）の三状態を取る。
合流（φ）は「同じ定数同士なら その定数、違えば `⊥`」。

```ruby
TOP = :top      # まだ何も分かっていない（最も楽観的）
BOT = :bottom   # 定数ではないと判明

def meet(a, b)                       # φ での合流
  return b if a == TOP
  return a if b == TOP
  return a if a == b                 # 同じ定数
  BOT                                # 食い違えば「定数でない」
end

def sccp(instrs, cfg, entry)
  val = Hash.new(TOP)                # SSA 変数 => 格子値
  exec = Set.new                     # 実行可能と判明したブロック
  flow = [entry]                     # 辿るべき CFG 辺のワークリスト
  ssa  = []                          # 再評価すべき SSA 定義のワークリスト

  until flow.empty? && ssa.empty?
    if (b = flow.shift)
      next if exec.include?(b)
      exec << b                       # ブロック b に点火
      cfg.instrs(b).each { |i| eval_instr(i, val, exec, flow, ssa, cfg) }
    elsif (i = ssa.shift)
      eval_instr(i, val, exec, flow, ssa, cfg) if reachable?(i, exec, cfg)
    end
  end
  val   # 各変数の確定値（定数なら畳み込み、BOT なら据え置き）
end
```

`eval_instr` は各命令を評価し、定数になったら結果を更新して、その変数の**使用箇所だけ**を
`ssa` ワークリストに積む（疎＝必要な所だけ再計算するから "sparse"）。分岐命令なら、
条件が定数のときは**通る枝だけ**を `flow` に積む。これにより、到達不能な枝が定数判定を
汚染しなくなる。SCCP は「条件分岐を考慮するから、単純な定数伝播より多くの定数を見つけられる」
——Wegman と Zadeck はこれを示した[](#cite:wegman1991)。

```ruby
# 例: 楽観的開始のおかげで「ループ変数が実は不変」を見抜ける
i1 = 1
loop do
  # i2 = phi(i1, i3)   ← 最初 i3 は TOP。i1=1 なので合流は 1。i3 も 1 と分かり…
  break if cond
  i3 = i2              # …不動点で i2 = i3 = 1 と確定。素朴な悲観解析はここで諦める
end
```

> [!TIP]
> SCCP の「楽観的に ⊤ から始める」と、攻めの DCE の「楽観的に死から始める」は同じ精神だ。
> 悲観的解析（最初から `⊥` を仮定）は安全だが、ループのように「仮定が自分を支える」構造で
> 必要以上に諦める。楽観的解析は、不動点に達するまで仮定を保ち、矛盾したものだけ降格する。
> この差が、ループ不変な定数や帰納変数の検出で効いてくる。

## 大域値番号付け（GVN）— 合流を越えて冗長を消す

[局所値番号付け](local-optimization.md)はブロック内だけだった。SSA では、変数名がそのまま値の出自を
指すので、値番号付けを**関数全体**に広げられる。これが**大域値番号付け（GVN）**だ。

最も単純な SSA-GVN は「ハッシュベース値番号付け」だ。SSA 変数を支配木の順に処理し、
各定義の右辺を「演算子＋オペランドの値番号」で正規化したキーにする。同じキーが既にあれば、
新しい変数はその先行値と**同じ値番号**を持つ——つまり冗長だ。SSA かつ支配木順なら、
先に見た定義はその使用を支配しているので、安全に置き換えられる。

Alpern–Wegman–Zadeck の GVN[](#cite:alpern1988) は、これを φ も含めて行う。
彼らは「合流（φ）が一致するなら値も等しい」という congruence（合同）関係を、
分割改良（partition refinement）で求めた。直観は「最初は全変数を一つの同値類に入れ、
オペランドの同値類が食い違うものを分けていく」——SCCP とは逆に**悲観から始めて分割**する楽しい双対だ。

実用上は、Click の **GVN-GCM**[](#cite:click1995gcm) がよく知られる。これは GVN（同じ値を一つに）と
**global code motion（GCM; 大域コード移動）**を組み合わせ、冗長を消したうえで各計算を
「使用を支配し、かつなるべくループの外」へ置き直す。値番号で計算を一意にしてから、
支配関係に従って最適な位置へ落とす——「何を計算するか」と「どこで計算するか」を分離する発想だ。
これは sea of nodes IR と相性がよく、HotSpot や Graal の中核をなす。

```ruby
# SSA・支配木順での簡易 GVN（ハッシュベース）
def gvn(instrs_in_domorder)
  vn   = {}     # 変数 => 値番号（代表変数）
  table = {}    # [op, vn(a), vn(b)] => 代表変数
  instrs_in_domorder.each do |i|
    next unless i.op == :binop
    op, a, b = i.args
    key = [op, vn.fetch(a, a), vn.fetch(b, b)].then { |o, x, y|
      commutative?(op) ? [o, *[x, y].sort_by(&:to_s)] : [o, x, y] }
    if (rep = table[key])
      vn[i.dst] = rep                    # 既出の値 → 代表に統合（冗長）
    else
      table[key] = i.dst; vn[i.dst] = i.dst
    end
  end
  vn   # vn[x] != x なら x は vn[x] に置換でき、定義は DCE で消える
end
```

GVN は[共通部分式除去](local-optimization.md)を合流越しに行うものだと言える。
局所 CSE が「同じブロック内の重複」しか消せなかったのに対し、GVN は
「異なる経路・ループをまたいだ重複」まで消す。これも SSA があってこそだ。

## sea of nodes — 制御とデータを一枚に

ここまで「CFG（基本ブロックの列）＋φ」という古典的 SSA を使ってきた。
だが Click と Paleczny は、もっと過激な IR を提案した——**sea of nodes**[](#cite:click1995ir)。

アイデアは、命令を基本ブロックに固定せず、**データ依存と制御依存だけで結ばれた一枚のグラフ**に
浮かべること。「`c = a + b`」は「`+` ノードが `a` と `b` を入力に取る」という依存辺だけで表され、
それがどのブロックに属すかは**最初は決めない**。GVN で同じ値のノードを一つに融合し、
最後に GCM[](#cite:click1995gcm)で各ノードを「使用を支配する最も外側の位置」へ落とす（スケジュールする）。

この表現の利点は、最適化が「グラフから冗長ノードを消す・融合する」だけで済み、
ブロック境界という人為的な制約に縛られないことだ。GVN・定数畳み込み・コード移動が
自然に絡み合う。欠点はメモリと実装の複雑さで、最終的にはブロックへスケジュールし直す必要がある。
HotSpot サーバコンパイラ、GraalVM、近年の V8（Turboshaft の前身 Turbofan）などが sea of nodes 系を採る。

> [!NOTE]
> sea of nodes は「**何を計算するか**（データ依存のグラフ）」と「**いつ・どこで計算するか**（スケジュール）」を
> 分離する究極の形だ。この「等価な計算をまず全部表現し、最後に最良の形を選ぶ」という発想は、
> [第 VI 部の equality saturation](equality-saturation.md) の e-graph と精神的に地続きである。
> sea of nodes は「一つの最良グラフ」を育て、e-graph は「等価なグラフ全部」を保持する、という違いはあるが。

## まとめ

- SSA 構築は「支配辺境への φ 配置」＋「支配木走査での名前替え」。すべて支配関係に支えられる。
- SCCP[](#cite:wegman1991) は定数伝播と到達可能性を楽観的に同時解決し、条件分岐を考慮するぶん素朴な定数伝播より強い。
- GVN[](#cite:alpern1988) は値番号付けを合流越しに行う大域 CSE。GVN-GCM[](#cite:click1995gcm) は「何を」と「どこで」を分離する。
- sea of nodes[](#cite:click1995ir) は制御とデータを一枚のグラフに統合した近代 IR で、多くの本格 JIT が採用する。

ここまでで関数内（イントラプロシージャル）の主要最適化はほぼ出そろった。
次章は、実行時間の大半を占める**ループ**に的を絞り、ループ専用の強力な最適化群へ進む。
