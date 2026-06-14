# 用語集

本書に登場する用語を五十音順（英字は末尾）にまとめる。各項目から、詳しく扱う章へリンクしている。
逆引き——「あの最適化の名前は何章だったか」——にも使ってほしい。

## あ行

**[インライン展開](inlining.md)**（inlining）
: 関数呼び出しを、呼び出し先の本体で置き換える変換。直接の利得（呼び出し規約のコスト削減）より、
  関数の壁を壊して他のあらゆる最適化を呼び込む効果が本体。「最適化の母」。

**[インラインキャッシュ](dynamic-dispatch.md)**（inline cache; IC）
: 動的言語のメソッド探索結果を、呼び出し点の隣に覚えておく高速化技法。
  単相 IC は一型、多相 IC（PIC）は複数型を覚える。クラスのバージョン番号で失効を管理する。

**[エスケープ解析](interprocedural-analysis.md)**（escape analysis）
: 関数内で作ったオブジェクトが外へ漏れるかを判定する解析。漏れないと分かれば、
  スカラ置換・スタック割り当て・同期除去ができ、ヒープ割り当て自体を消せる。

**[オンスタック置換](speculative-optimization.md)**（on-stack replacement; OSR）
: 実行中の関数を、途中（多くはループ内）から最適化コードへ昇格させる仕組み。脱最適化の逆向き。

## か行

**[ガード](speculative-optimization.md)**（guard）
: 投機的最適化で、特殊化の前提（「この値は Integer」など）が成り立つかを安く確認する検査。
  失敗すると脱最適化が起きる。

**[基本ブロック](intermediate-representation.md)**（basic block）
: 途中に分岐の出入りがなく、先頭から末尾まで必ず一直線に実行される命令列。CFG の節点になる。

**[基本ブロックバージョニング](speculative-optimization.md)**（basic block versioning; BBV）
: 基本ブロックを、入ってくる型ごとに遅延複製していく軽量な投機手法。型チェックが自然に消える。
  Ruby の YJIT が採用。

**[強度低減](peephole.md)**（strength reduction）
: コストの高い演算を等価で安い演算に置き換えること（`x*2 → x<<1`、ループ内の乗算を加算に）。

**[共通部分式除去](local-optimization.md)**（common subexpression elimination; CSE）
: 同じ値を計算する式が複数あるとき、一度だけ計算して使い回す最適化。値番号付けや GVN が支える。

**[グラフ彩色](register-allocation.md)**（graph coloring）
: レジスタ割り当てを「干渉する変数を別の色（レジスタ）に塗る」問題として解く定式化。Chaitin に始まる。

**[制御フローグラフ](intermediate-representation.md)**（control flow graph; CFG）
: 基本ブロックを節点、分岐を辺とした有向グラフ。最適化を「グラフ上の問い」に翻訳する基盤。

**[コールグラフ](interprocedural-analysis.md)**（call graph）
: 関数を節点、「f が g を呼ぶ」を辺とした有向グラフ。手続き間解析の出発点。間接呼び出しの解決精度が質を決める。

## さ行

**[支配関係](intermediate-representation.md)**（dominance）
: 「ブロック B に到達するには必ず A を通る」とき A は B を支配する。最適化の安全性を語る共通言語。
  直接支配者をつないだ支配木、その外縁である支配辺境が SSA 構築に使われる。

**[シェイプ／隠れクラス](dynamic-dispatch.md)**（shape / hidden class）
: 同じフィールド構成を持つ動的オブジェクトでフィールドのオフセット表を共有する仕組み。
  IC と組んでインスタンス変数アクセスを固定オフセット読みに化けさせる。

**[手続き間解析](interprocedural-analysis.md)**（interprocedural analysis）
: 関数の境界を越える解析。コールグラフ・別名解析・エスケープ解析が三種の神器。

**[投機的最適化](speculative-optimization.md)**（speculative optimization）
: 観測した型などを「取り消せる賭け」として仮定し、コードを特殊化する最適化。
  ガードで守り、外れたら脱最適化で安全に戻る。現代 JIT の心臓部。

**[スカラ置換](interprocedural-analysis.md)**（scalar replacement）
: エスケープしないオブジェクトを作らず、そのフィールドを個別のローカル変数として扱う最適化。

**[スピル](register-allocation.md)**（spill）
: レジスタが足りないとき、ある変数の値を主記憶に退避すること。メモリアクセスが増えるので極力減らす。

**[SSA 形式](intermediate-representation.md)**（static single assignment form）
: 各変数への代入をプログラム中ちょうど一箇所にする IR。合流点では φ 関数を使う。
  def-use が自明になり、局所最適化を大域化する近代最適化の標準基盤。

## た行

**[脱仮想化](whole-program-optimization.md)**（devirtualization）
: 仮想呼び出し（呼び先が実行時の型で決まる）を直接呼び出し・インラインに変える最適化。
  静的には CHA/RTA、動的には投機的に行う。

**[脱最適化](speculative-optimization.md)**（deoptimization）
: 最適化コードの実行を途中で打ち切り、同じ地点のインタプリタへ実行を移すこと。
  投機的最適化の安全網。状態の再構築が技術的核心。

**[多面体モデル](loop-optimization.md)**（polyhedral model）
: ループの反復空間を多次元の整数格子として表し、依存を満たす範囲で反復順序を変換する理論。
  自動並列化・自動ベクトル化の基盤。

**[抽象解釈](dataflow-analysis.md)**（abstract interpretation）
: プログラムを抽象的な値（符号・区間など）で評価し、安全な近似を体系的に得る理論。
  データフロー解析の正しさの源泉。

**[中間表現](intermediate-representation.md)**（intermediate representation; IR）
: ソースと機械語の間で、最適化に都合よくプログラムを表したデータ構造。三番地コード・SSA・sea of nodes など。

**[定数畳み込み／定数伝播](local-optimization.md)**（constant folding / propagation）
: 定数だけの計算をコンパイル時に済ませる（畳み込み）、定数の値を使用箇所へ持ち回る（伝播）。
  条件分岐まで考慮した強力版が SCCP。

**[データフロー解析](dataflow-analysis.md)**（data-flow analysis）
: CFG 全体を見渡して「各地点で何が成り立つか」を、格子上の不動点反復で機械的に求める枠組み。
  生存変数・到達定義・利用可能式など。最適化の安全性を供給する屋台骨。

## な行

**[値番号付け](local-optimization.md)**（value numbering）
: 各値に番号を振り「同じ番号＝必ず同じ値」を保証して冗長計算を見抜く技法。
  関数全体に広げた大域版が GVN。

## は行

**[ピープホール最適化](peephole.md)**（peephole optimization）
: 数命令の「のぞき窓」を滑らせ、局所パターンを等価な命令列に置換する最も単純な最適化。

**[不要コード除去](dead-code-elimination.md)**（dead code elimination; DCE）
: 結果がどこにも使われない計算を取り除く最適化。生存変数解析や SSA のマーキングで行う。
  実行されえない「到達不能コード」とは区別する。

**[フェーズ順序問題](equality-saturation.md)**（phase-ordering problem）
: どの最適化をどの順で適用するかで結果が変わり、最適な順序が一般に分からない問題。
  equality saturation・強化学習が挑む。

**[フタムラ投影](whole-program-optimization.md)**（Futamura projection）
: インタプリタを特定プログラムで部分評価すると、そのプログラムのコンパイル結果が得られるという原理。
  Truffle が実用化。

**[φ 関数](intermediate-representation.md)**（phi function）
: SSA で合流点に置く擬似命令。先行ブロックごとにどの値を選ぶかを表す。SSA から脱出する際にコピーへ変換して消す。

**[部分評価](whole-program-optimization.md)**（partial evaluation）
: 既知の入力について先に計算し尽くした専用プログラムを生成する技法。全プログラム最適化の極北。

**[別名解析](interprocedural-analysis.md)**（alias / points-to analysis）
: 二つのポインタ・参照が同じ場所を指しうるかを判定する解析。
  Andersen 流（精密・重い）と Steensgaard 流（粗い・ほぼ線形）が代表。

## ま・ら行

**[命令スケジューリング](instruction-scheduling.md)**（instruction scheduling）
: データ依存を保って命令を並べ替え、パイプラインのストールを埋める最適化。
  リストスケジューリング、ループ向けのソフトウェアパイプライニングなど。

**[命令選択](instruction-selection.md)**（instruction selection）
: IR の演算を機械命令で実現する際、複数 IR 演算を 1 命令に畳む機会を含めて最小コストに選ぶ最適化。
  木のタイリングと動的計画法で解く。

**[レジスタ割り当て](register-allocation.md)**（register allocation）
: 無限の仮想レジスタを有限の物理レジスタに詰める最適化。グラフ彩色と線形走査が二大流派。

**[ループ不変式の移動](loop-optimization.md)**（loop-invariant code motion; LICM）
: ループ内で毎回同じ値を計算する式を、ループの外（プレヘッダ）へ巻き上げる最適化。

## 英字

**[BBV](speculative-optimization.md)** → 基本ブロックバージョニング

**[CHA / RTA](whole-program-optimization.md)**（class hierarchy analysis / rapid type analysis）
: クラス階層（CHA）や実際に生成されるクラス（RTA）から仮想呼び出しの呼び先を絞り、脱仮想化を可能にする解析。

**[e-graph / equality saturation](equality-saturation.md)**
: 等価な式を e-class にまとめて保持するデータ構造（e-graph）と、書き換えを等価の追加として
  飽和まで適用し最良を抽出する最適化（equality saturation）。フェーズ順序問題を構造的に回避する。

**[GVN](ssa-optimization.md)**（global value numbering; 大域値番号付け）
: 値番号付けを合流をまたいで関数全体に広げた冗長除去。GVN-GCM は「何を計算するか」と「どこで計算するか」を分離する。

**[JIT](introduction.md)**（just-in-time compilation）
: 実行時に、観測した型・頻度を使ってコードをコンパイル・特殊化する方式。動的言語最適化の主舞台。

**[LTO](whole-program-optimization.md)**（link-time optimization; リンク時最適化）
: 各翻訳単位を IR のまま持ち越し、リンク時にプログラム全体を再最適化する。ThinLTO は要約でスケールさせる。

**[PGO](whole-program-optimization.md)**（profile-guided optimization）
: 一度計測実行して得たプロファイルを二度目のビルドに与え、ホットな経路を優先最適化する。

**[SCCP](ssa-optimization.md)**（sparse conditional constant propagation）
: 定数伝播と到達可能性を楽観的に同時解決する。条件分岐を考慮するぶん素朴な定数伝播より強い。

**[sea of nodes](ssa-optimization.md)**
: 制御フローとデータフローを一枚のグラフに統合した近代 IR。HotSpot サーバコンパイラや GraalVM が採用。

**[SMT ソルバ](superoptimization.md)**（satisfiability modulo theories）
: 「この論理式を真にする値はあるか」を自動判定するツール。命令列の等価性証明（superoptimization・Alive）の土台。

**[superoptimization](superoptimization.md)**
: 変換規則を捨て、仕様と等価な最短・最速の命令列を探索で見つける手法。総当たり・確率的（STOKE）など。
