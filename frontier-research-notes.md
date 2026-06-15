# 最前線リサーチ統合メモ（2015–2025）

> 本書「最適化入門」の最前線を更新するための調査メモ。deep-research を16パス回し、
> すべての主張を3票の反証検証（2/3で棄却）にかけ、一次資料に照合した検証済みの成果のみを記載する。
> 文献は `references.bib` に追加済み（合計163エントリ、キー重複なし）。
> これは**本書の一部ではなく作業用メモ**。本文反映後は削除してよい。

調査の動機: 本書の最前線（第VI部）の引用は **2021年で止まっており**、しかも本文で言及する
Ithemal・CompilerGym・LLM・MLIR 周辺の多くが無引用だった。ここ10年、特に2021年以降に
本書が取りこぼしている軸を、出典付きで固めた。

---

## A. 既存3章を更新すべき進展

### A1. equality saturation — 1ライブラリ(egg)から研究プログラム＋プロダクションへ
- **Relational E-matching**（Zhang, Wang, Willsey, Tatlock, POPL 2022）`zhang2022relational` — e-matching を最悪計算量最適な関係結合に再定式化。初の計算量結果、桁違いに高速。
- **egglog**（Zhang ら, PLDI 2023）`zhang2023egglog` — Datalog と equality saturation を統合。増分実行・協調解析・束。
- **Tensat**（Yang, Phothilimthana ら, MLSys 2021）`yang2021tensat` — EqSat をテンソルグラフ超最適化へ。TASO比 最大16%高速・最適化時間1/48。
- **ægraph（acyclic e-graph）**（Chris Fallin, EGRAPHS 2023）`fallin2023aegraph` — e-graph が **Cranelift/Wasmtime に実装**された。egg のままでは JIT 速度に耐えず再設計。「最前線が実務に降りた」事例。
- **Ruler**（Nandi ら, OOPSLA 2021）`nandi2021ruler` — 書き換え規則そのものを equality saturation で自動合成。

### A2. superoptimization と検証 — Alive→Alive2、合成系の実用化
- **Alive2**（Lopes, Lee, Hur, Liu, Regehr, PLDI 2021 Distinguished）`lopes2021alive2` — 実LLVM向け有界翻訳検証、誤警報なし、実バグ47件。本書の Alive 記述の直接後継。
- **Souper**（Sasnauskas, Chen, ..., Regehr, 2017）`sasnauskas2017souper` — LLVM 合成超最適化器。
- 検証の広がり: **CompCert**（Leroy CACM/JAR 2009）`leroy2009compcert` `leroy2009backend`、**CakeML**（Kumar ら POPL 2014, Most Influential 2024）`kumar2014cakeml`、**Vellvm/VIR**（Zhao POPL 2012 / Zakowski ICFP 2021）`zhao2012vellvm` `zakowski2021vir`（土台は Interaction Trees `xia2020itrees`）、**Crellvm**（PLDI 2018）`kang2018crellvm`、**Peek**（PLDI 2016）`mullen2016peek`、翻訳検証 `pnueli1998translation` `rideau2010regalloc`、命令選択の検証 **Crocus**（VanHattum ら ASPLOS 2024）`vanhattum2024crocus`。

### A3. ML/LLM — MLGOの先、本書が無引用で言及していた箇所
- 学習コストモデル **Ithemal**（Mendis ら ICML 2019）`mendis2019ithemal`、**Ansor**（Zheng ら OSDI 2020）`zheng2020ansor`。
- RLパス順序 **AutoPhase**（Haj-Ali ら MLSys 2020）`hajali2020autophase`、**CompilerGym**（Cummins ら CGO 2022 Distinguished）`cummins2022compilergym`。
- **LLM×コンパイラ**: Cummins ら 2023 `cummins2023llm` → **Meta LLM Compiler** 2024 `cummins2024llmcompiler`（7B、LLVM-IR/asm で継続事前学習）。本書のLLM段落の具体的な錨。

---

## B. 本書に丸ごと欠けている新しい軸（新節/新章が要る）

### B1. テンソル/ドメイン特化コンパイラ ★この10年最大の潮流
- **基盤**: TVM（Chen ら OSDI 2018）`chen2018tvm`、MLIR（既出 `lattner2021`）+ Linalg方言、Glow（`rotem2018glow`）。
- **グラフ超最適化**: TASO（Jia ら SOSP 2019）`jia2019taso`、PET（Wang ら OSDI 2021, 部分等価変換）`wang2021pet`。
- **構築/メモリ中心カーネル生成**: Roller（OSDI 2022）`zhu2022roller`、Welder（OSDI 2023, tile-graph）`shi2023welder`、Hidet（ASPLOS 2023, task-mapping）`ding2023hidet`。
- **スケジュール言語**: Halide（Ragan-Kelley ら PLDI 2013 / CACM 2018）`ragankelley2013halide` `ragankelley2018halidecacm` — アルゴリズムとスケジュールの分離。
- **学習autoscheduler**: Adams ら（SIGGRAPH 2019, 初めて人間を平均で上回る）`adams2019autoscheduler`、GPU版（OOPSLA 2021）`anderson2021gpuauto`、FlexTensor（ASPLOS 2020）`zheng2020flextensor`。
- **ユーザ記述/検証スケジュール**: Exo（Ikarashi ら PLDI 2022）`ikarashi2022exo`。
- **GPUカーネルDSL**: Triton（Tillet ら MAPL 2019）`tillet2019triton`。
- **polyhedral復権**: Tiramisu（CGO 2019）`baghdadi2019tiramisu`、Polygeist（PACT 2021）`moses2021polygeist`、TACO 疎テンソル（Kjolstad ら OOPSLA 2017）`kjolstad2017taco`。

### B2. セキュア/定数時間コンパイル ★第三の軸（速さ・正しさに加えて機密性）
- **問題**: 最適化が定数時間対策を黙って壊す。Simon-Chisnall-Anderson（EuroS&P 2018）`simon2018whatyouget`、Binsec/Rel（S&P 2020）`daniel2020binsecrel`。
- **解**: 定数時間保存コンパイル（Barthe ら POPL 2020, CompCert拡張）`barthe2020consttime`、Jasmin（CCS 2017）`almeida2017jasmin`、FaCT（PLDI 2019）`cauligi2019fact`、CT-wasm（POPL 2019）`watt2019ctwasm`、Blade（投機リーク除去, POPL 2021）`vassena2021blade`。
- **理論**: secure compilation サーベイ（Patrignani-Ahmed-Clarke, CSUR 2019）`patrignani2019securecomp`。

### B3. ポストリンク/PGO機構 ★本書はPGOを概念だけ、機構は未踏、BOLT系は不在
- サンプリングPGO: AutoFDO（Chen ら CGO 2016）`chen2016autofdo`、CSSPGO/pseudo-probe（CGO 2024）`he2024csspgo`。
- プロファイル整合性: profi 最小費用流（He ら POPL 2022）`he2022profi`。
- 配置: Pettis-Hansen（PLDI 1990）`pettis1990positioning` → ext-TSP（Newell-Pupyrev, IEEE TC 2020）`newell2020exttsp`。
- ポストリンク: BOLT（Panchenko ら CGO 2019）`panchenko2019bolt`、Lightning BOLT（CC 2021）`panchenko2021lightningbolt`、Propeller（ASPLOS 2023）`shen2023propeller`、ThinLTO（CGO 2017）`johnson2017thinlto`。

### B4. 高速JIT / WebAssembly ★BBVの隣に置くべき
- copy-and-patch（Xu-Kjolstad, OOPSLA 2021、CPython 3.13 JIT に採用）`xu2021copyandpatch`。
- Wasm実行階層: in-place interpreter（Titzer OOPSLA 2022）`titzer2022inplace`、baseline解析（Titzer CGO 2024）`titzer2024baseline`。
- 検証付き脱最適化: Sourir（POPL 2018）`fluckiger2018sourir` → CoreJIT（Coq検証, POPL 2021）`barriere2021corejit`。

### B5. IR設計 ★MLIRだけが新世代ではない
- sea-of-nodes（Click-Paleczny IR'95）`click1995ir` — C2/TurboFan/Graal で使用。**2025年 V8 が脱SoN→CFGベース Turboshaft/Maglev へ**（`v8t2025leavingson`）。
- PDG（Ferrante ら TOPLAS 1987）`ferrante1987pdg`、RVSDG（Reissmann ら TECS 2020）`reissmann2020rvsdg`、需要依存グラフからのCFG再構成（Bahmann ら TACO 2015）`bahmann2015reconstruct`。

### B6. メモリアクセス/局所性/データ配置 ★本書で手薄
- polyhedral局所性: Pluto（Bondhugula ら PLDI 2008）`bondhugula2008pluto`、isl（Verdoolaege ICMS 2010）`verdoolaege2010isl`、PPCG（TACO 2013）`verdoolaege2013ppcg`、cache-oblivious（Frigo ら FOCS 1999）`frigo1999cacheoblivious`。
- データ配置: Chilimbi 構造体分割/配置（PLDI 1999×2）`chilimbi1999definition` `chilimbi1999layout`、SIMD/GPU向け（Henretty CC 2011 `henretty2011layout`、Sung IJPP 2012 `sung2012layout`）。
- プリフェッチ: Mowry-Lam-Gupta（ASPLOS 1992）`mowry1992prefetch`。

### B7. レジスタ割り当ての進展 ★本書はグラフ彩色/線形走査止まり
- SSAベース: 干渉グラフが弦グラフ→多項式時間彩色（Hack-Grund-Goos CC 2006 `hack2006ssa`、Hack-Goos IPL 2006 `hackgoos2006ipl`、Braun-Hack spilling CC 2009 `braun2009spilling`）。
- puzzle solving（Pereira-Palsberg PLDI 2008）`pereira2008puzzle`。
- 実装: LLVM greedy（Olesen 2011）`olesen2011greedy`、Cranelift regalloc2（Fallin 2022）`fallin2022regalloc2`。
- 学習: RL4ReAl（CC 2023）`venkatakeerthy2023rl4real`。

### B8. 部分評価 / 自己最適化インタプリタ ★動的言語章の隣に
- Futamura射影（HOSC 1999再録）`futamura1999`、Jones-Gomard-Sestoft（1993）`jones1993`。
- Truffle/GraalVM: 自己最適化AST（DLS 2012）`wurthinger2012selfopt`、One VM（Onward! 2013）`wurthinger2013`、**Practical Partial Evaluation（第一Futamura射影をJITで実用化, PLDI 2017）**`wurthinger2017`。Jones最適性（Brown-Palsberg POPL 2018）`brown2018jones`。

### B9. 自動微分をコンパイラ問題として
- **Enzyme**（Moses-Churavy NeurIPS 2020）`moses2020enzyme` — **最適化後**のLLVM IRを微分（前に微分するより約4.2倍速い）。GPU版（SC 2021）`moses2021enzymegpu`。
- 源流: Zygote（SSA微分）`innes2018zygote`、Tapenade（TOMS 2013）`hascoet2013tapenade`、理論 Elliott（ICFP 2018）`elliott2018ad`。

### B10. クエリ/データ解析コンパイル
- Neumann HyPer（push型LLVMコンパイル, VLDB 2011）`neumann2011query`、LMS（GPCE 2010）`rompf2010lms`、LegoBase（VLDB 2014）`klonatos2014legobase`、How to Architect a Query Compiler Revisited（Futamura視点, SIGMOD 2018）`tahboub2018revisited`、Weld（CIDR 2017 / VLDB 2018）`palkar2017weld` `palkar2018weld`、Spark SQL codegen（SIGMOD 2015）`armbrust2015sparksql`、compiled-vs-vectorized（Kersten VLDB 2018）`kersten2018everything`。

### B11. 高位合成 / ハードウェアコンパイル
- Calyx（ASPLOS 2021）`nigam2021calyx`、Dahlia（時間感応アフィン型, PLDI 2020）`nigam2020dahlia`、Spatial（PLDI 2018）`koeplinger2018spatial`、検証HLS Vericert（OOPSLA 2021）`herklotz2021vericert`、CIRCT `circt`。

### B12. 並列化/GPU/ヘテロ＋関数型fusion
- Lift（CGO 2017）`steuwer2017lift` / RISE-Elevate（ICFP 2020）`hagedorn2020rise`、Polly-ACC（ICS 2016）`grosser2016pollyacc`、Futhark（PLDI 2017）`henriksen2017futhark`。
- fusion: 短絡fusion（Gill ら FPCA 1993）`gill1993deforestation`、stream fusion（ICFP 2007）`coutts2007streamfusion`、GHC規則（Haskell 2001）`peytonjones2001rules`。

---

## C. 本文反映の方針（Task 7）
1. **既存章の外科的更新**: equality-saturation / superoptimization / ml-and-future / whole-program-optimization / intermediate-representation / register-allocation / loop-optimization に、該当する検証済み引用と1〜数文を追加。
2. **新節/新章**: B1（テンソル/DSL）と B2（セキュア）は本書に軸ごと欠けているため、第VI部に独立した扱いを新設。
3. part-frontier.md の導入も、広がった軸を予告する形に更新。

## D. デュアルトピックで枯れた教訓
deep-research は1パスで2トピック束ねると後半が検証段で枯れる（レート/予算）。単一トピックに割ると安定して3-0で通る。広域は「狭く深く×多数」で回すのが正解だった。
