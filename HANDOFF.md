# MASKD 開発引き継ぎ資料

> 別セッション／別担当への引き継ぎ用。この1枚を最初に読めば、すぐに開発を再開できるようにまとめています。
> 最終更新: 2026-07-14 / 到達バージョン: **v0.45.1**

---

## 1. プロジェクト概要

- **MASKD** … 仮面舞踏会を舞台にした心理戦・推理カードゲーム（日本語UI）。**1人対戦（プレイヤー vs CPU）**。
- 実体は **`index.html` 1ファイル完結**（約 745KB）。HTML/CSS/JS すべてインライン。カード画像は base64 JPEG を `CARD_IMG` に埋め込み。
- **GitHub Pages で公開**。リポジトリ `perusonao/maskd`、**`main` ブランチ = 本番デプロイ**。
- ルール全文は README.md、変更履歴はアプリ内（ヘッダーの「MASKD」タップ）に `CHANGELOG` として保持。

## 2. デプロイ運用（重要）

- **`main` へ push すると GitHub Pages が自動デプロイ**（ワークフロー名: "pages build and deployment"）。
- このプロジェクトは一貫して **`main` に直接コミット／push** して運用してきた（feature ブランチ運用はしていない）。デプロイ確認も main のワークフロー成功で判断。
- push 後は Actions の該当 run が `completed / success` になったことを確認する。

## 3. 主要コードの場所（`index.html` 内・行番号は目安）

| 内容 | 目印 | 行(目安) |
|---|---|---|
| バージョン定数 | `const GAME_VERSION` | 3390 |
| 変更履歴 | `const CHANGELOG` | 3391 |
| 勝敗判定 | `function judge(dealer, child)` | 2955 |
| 配札 | `function dealHands()` | 1759 |
| マッチ開始（配札演出→） | `function beginMatchSetup()` | 1864 |
| ラウンド開始 | `function startRound()` | 2077 |
| ラウンド開始モーダル表示 | `function showRoundStartModal()` | 2106 |
| モーダル「はじめる」→本編 | `function beginRoundPlay()` | 2117 |
| スコア／名前描画 | `function renderScore()` | 2155 |
| 残り手札ボーナス係数 | `LEFTOVER_TOPN=2`, `AIR_LEFTOVER_FLAT=2` | 1696-1697 |
| CPU戦略 | `let cpuStrategy` | 2772 |
| CPU仮面名 | `const CPU_MASK_NAME` | 2777 |
| プレイヤー名保存(クラウド) | `window.__maskdSaveName` 呼び出し | 1884 |
| カード画像 | `const CARD_IMG` (base64・巨大行) | 1690 |

## 4. ルールの要点（実装済み）

- 全8ラウンド。毎ラウンド親（dealer）／子（child）が交代。親のみ「決戦・撤退・入替」を選べる。
- 5属性 **Sun / Dark / Moon / Kaos / Air** × 数字0〜5 ＝ 全30枚を15枚ずつ分配。
- 勝敗 `judge()`:
  - **Air** が絡む → 必ず引き分け。ただし **Kaos は Air に勝つ**。
  - **Kaos**（Airなし）→ 数字が大きい方勝ち。同値は Kaos 側敗北。特殊: Kaos 0・1 は全属性の5に勝つ／Kaos5 は5に勝つが1に負ける。
  - 同属性 → 数字勝負。異属性(Moon/Dark/Sun) → **じゃんけん**（月✊＞闇✌＞太陽✋＞月）。
  - 終盤決戦（第7・8R）の決戦は得点2倍。
- **最終得点 ＝ 獲得点 ＋ 残り戦略カウンター ＋ 残り手札ボーナス**
  - 残り手札ボーナス ＝ **非Air札の数字上位2枚の合計** ＋ **Air は数字無関係で1枚2点**（`LEFTOVER_TOPN` / `AIR_LEFTOVER_FLAT`）。
  - これは「手札温存偏重」を是正した v0.32.0 の重要修正。小数係数は不採用（ユーザー方針）。

## 5. UI/表示まわりの要点

- カード表示は 2 モード切替（初期=初心者）: `cardStyle` = `'beginner'` | `'rich'`（localStorage `maskd_cardstyle`）。`useRichCards()`。
  - 初心者: 属性色＋大きなじゃんけん記号＋数字＋英語属性名。上級者: リッチな仮面イラスト。
- 属性名は **英語表記に統一**（SUN/DARK/MOON/KAOS/AIR）。
- レイアウトは短い画面で崩れやすい。`@media (max-height: 820/750/680px)` の3段で調整済み。要素を足したら**必ず短い画面(例 402×709)で見切れ確認**すること（過去に何度も再発）。
- v0.32.6 で: 両者の手札枚数/山札枚数表示を撤去、山札は「一番上の裏イメージ」表示、盤面メッセージエリア廃止→ラウンド開始モーダル化。
- プレイヤー名（初回入力・localStorage `maskd_playername` ＋ Firestore 登録）と相手仮面名（積極的=剛力仮面／慎重=鉄壁仮面／トリッキー=技巧仮面）を表示。

## 6. Firebase / データ

- Firestore（projectId `maskd-8eafb`）にゲームログと名前を書き込み。
  - `window.__maskdSaveLog` → コレクション `logs`
  - `window.__maskdFetchLogs` / `__maskdFetchCount`
  - `window.__maskdSaveName` → コレクション `players`（名前は最大12文字）
  - `buildGameLog()` が1試合分のログを組み立て（`playerName` 含む）。
- **バランス調整の進め方**: 実ログ(JSON)を回収 → Node シミュレータ（本物の `judge`/配札/得点を移植）で戦略勝率を検証 → 係数を決める、という流れで進めてきた。

## 7. 検証方法（このリポジトリでの慣習）

- **Playwright ヘッドレス**で実ブラウザ確認:
  - Chromium: `/opt/pw-browsers/chromium-1194/chrome-linux/chrome`
  - モジュール: `/opt/node22/lib/node_modules/playwright`
  - `file:///home/user/maskd/index.html` を開く。名前モーダルで止まるので、テストでは `addInitScript` で `localStorage.setItem('maskd_playername','...')` を先に入れる。
  - 配札演出（dealerDrawOverlay）をクリックで消化 → ラウンド開始モーダル（`#roundStartModal.show`）→ `beginRoundPlay()` で本編へ。
  - 過去のテストスクリプト例（scratchpad・コンテナ揮発）: `progfull.js`(全8R通し), `rscap.js`(モーダル撮影), 各種 `sim*.js`(バランス)。※ scratchpad はセッション消滅で消えるため、必要なら作り直す。
- 変更後は **短い画面での見切れ確認** ＋ **JSエラー0** を最低ライン。

## 8. 直近の到達点（バージョン履歴の要約）

- v0.31系: UI刷新（相手手札を上部・直タップ・VS仕切り・局面ラベル・スコア強調・使用済ミニカード等）
- v0.32.0: 残り手札ボーナスの得点ルール変更（温存偏重の是正）
- v0.32.1: 場札拡大・戦略バー復活・使用済を実画像・自分の山札を一覧表示
- v0.32.2: 山札モーダルをコンパクト化・手札に「相手視点」候補アイコン・相手山札の最上段強調
- v0.32.3: 属性を英語統一・候補はじゃんけん統一・入替モーダル廃止(直タップ)・文字削減
- v0.32.4: 入替=決戦確定（札を選んだらそのまま勝敗へ）
- v0.32.5: プレイヤー名の入力/登録・相手の仮面名表示
- v0.32.6: 手札/山札の枚数表示を削除・山札を裏イメージ表示・ラウンド開始モーダル追加
- v0.33.0: **CPU AI 大幅強化**（親の1手読み・難易度スケーリング修正）。本物のjudge/配札/得点を移植したNodeシミュレータで検証し、強い参照プレイヤー相手の勝率が旧CPUの13〜34%→約49〜54%(強)へ改善。CPU主要関数は `cpuChooseCard`/`cpuDecide`/`cpuWantsForce` ＋ 新規 `cpuEps`/`cpuOppRemainingCards`/`cpuDealerLeadValue`。
- v0.34.0: 勝敗演出のジュース化（3Dめくり `flipIn`／設置 `card-place`／決着インパクト `fireImpact`＋ハプティクス）。`prefers-reduced-motion` で抑制。
- v0.35.0: リザルト/統計拡充（MVPカード・勝率・連勝＝`computeGameMVP`/`computeStreaksAndRate`）＋ 決戦/撤退の公開演出中のnull参照に安全装置。
- v0.36.0: 初心者オンボーディング（初回R1〜R3のみラウンド開始モーダルに「🔰今回のポイント」＝`onboardTip`/`ONBOARD_KEY`）。
- v0.37.0: **ミッション形式チュートリアル**（実際に操作して学ぶ）。データ＝`TUT_MISSIONS`、エンジン＝`openMissions`/`renderMission`/`tutPlayCard`/`tutDecide`/`tutFinish`、専用オーバーレイ `#tutMissionOverlay`（本編エンジンとは独立、`judge()`で答え合わせ）。タイトルの「🎓 ミッションで学ぶ」から起動。`MISSIONS_DONE_KEY`。
- v0.38.0: ①初回ウェルカム画面 `#welcomeModal`＋`welcomeChoose()`（初回はスライド自動表示をやめ、ミッション/遊び方/対戦を選択）。②ミッション追加で全8問に（role `swap`＝入替、`strict`＝温存判定を engine に追加）。③**効果音**（WebAudio合成、`sfx(name)`/`toggleSound()`、キー `maskd_sound`、フッター `#soundToggle`）。フック: `fireImpact`(win/lose)・`resolveFightJudge`(draw)・`fireKaosFlash`/`fireAirFlash`・`playYouCard`・`tutShowResult`/`tutPlayCard`。
- v0.39.0: ①**実績システム**（`ACHIEVEMENTS`・`checkAchievements(ctx)`・`openAchievements()`・キー `maskd_achievements_v1`・モーダル `#achieveModal`）。フック: `endGame`・`tutFinish`。導線: タイトル/結果画面の「🏅 実績」。②ミッション3追加で全11問（role `forceDecide`＝奇襲を追加）。
- v0.40.0: ①**上級ミッション** `TUT_ADVANCED`（本格派4問）。エンジンは `tutList`/`tutSet` で基本(`TUT_MISSIONS`)/上級を切替、`openMissions('advanced')`、`ADVANCED_DONE_KEY`/`advancedCleared()`。導線: タイトル「⚔️ 上級ミッション」＋基本修了後の誘導。②実績6追加で**全18種**（`_maxDeficit`等のヘルパー追加、`endGame`ctxに `summary`/`advancedCleared` を追加）。③対戦画面の「あなたの手札／相手の手札」見出しを削除（`handTitle` 既定空、`ohs-label` 撤去）。④**admin.html**: 対戦ログ表に「名前」列＋検索＋詳細に `playerName` を表示。
- v0.41.0: レイアウト見直し。①トップ: 上部ボタンを `top-learn-btns`（ミッション/上級）＋ `top-info-btns.mini`（実績/遊び方/ルール）の2段に整頓（更新履歴はフッター`MASKD`タップに集約）。②結果画面(`endGame`): 各ラウンド内訳/残り手札/統計を `<details class="ov-details">` で折りたたみ（要点＝スコア/MVP/連勝/実績は上部固定）。「もう一度/とじる」が初期表示内に収まるように。
- **v0.42.0（現在）**: ①CPU思考中インジケータ `#cpuThinking`（`showCpuThinking`/`hideCpuThinking`。CPU手番のsetTimeout前に表示、`cpuActAsDealer`/`cpuActAsChild`/`cpuDecide` 冒頭で解除。CPU待ちは 700/800ms）。②得点リード可視化（`renderScore` でリード側 `#youLP`/`#cpuLP` に `.leading`、`#youLeadMark`/`#cpuLeadMark` に「▲ +N」）。
- **v0.43.0（現在）**: ①盤面レイアウト切替 A(標準)/B(メリハリ)。`boardLayout`/`maskd_layout`、`applyLayout()`/`setBoardLayout()`/`toggleLayout()`、`.app.layout-b` CSS、上段スコアバー `#topScoreBar`（`renderScore`が同期、Bでは `.pinfo .pi-pt` を隠す）。導線: トップ設定 `#layoutOpts`＋フッター `#layoutToggle`。②Kaos/Air演出強化（`.kaos-flash` に紫の衝撃波リング`::before`＋ラベルpop、`.air-flash` のAIRラベルをふわっと浮上）。③**不具合修正**: `.hist-scroll > *{flex-shrink:0}` で対戦ログ一覧の行つぶれ（縦フレックスのshrink）を解消。
- **v0.44.0（現在）**: **段階的ルール開放「はじめての対戦」**。`LEARN_STAGES`(4段階)＋`stageConfig`/`deckAttrs`/`matchRounds`＋`stageAllows()`。縮小デッキ`dealHandsStage()`、`computePossibleAttrs`は`deckAttrs`基準に変更、`startGame(dealer, stage)`でステージ設定、`TOTAL_ROUNDS`→`matchRounds`(通常は8)、`scoreMultiplier`/`isLateGame`は`late2x`フラグ＋`matchRounds-1`でゲート、`renderActions`/`cpuWantsForce`/resolveFightで入替・奇襲をゲート、`endGame`でステージ時は`endLearnStage()`へ（履歴/実績/統計に残さない）。UI: `#learnOverlay`＋`openLearnLadder`/`showStageIntro`/`startLearnStage`/`endLearnStage`、`LEARN_KEY`(進捗)。実績`graduate`。導線: タイトル/ウェルカムの「🎓 はじめての対戦」。※タイトルのボタン構成も再編。

## 9. 次にやり得ること（未確定・候補）

> 外部レビュー(9.2/10)の主な指摘は v0.33〜v0.37 で対応済み（CPU個性/難易度・勝敗演出・リザルト充実・初心者ガイド・ミッション式チュートリアル）。以下は残りの候補。

- **効果音**: 意図的に未実装（自動再生の煩わしさ回避のため）。入れるならフッターにトグル（既定OFF）＋WebAudioの控えめなSFX推奨。
- さらなるバランス検証（新ログ回収 → シミュレータ再実行）。※シミュレータは scratchpad(揮発) にあった `sim.js`/`cpumod.js`/`verify5.js` 方式。必要なら再作成：本物のjudge/配札/得点＋index.htmlからCPU関数を抽出し、強い参照(greedy)と対戦させ勝率を測る。移植の忠実性は `cpumod.js`(実関数抽出)＋統一エンジンで検証できる。
- 長期リプレイ機能（実績・デイリー・ランク等）、上級者向けリッチ表示の作り込み。
- コード分割（保守性向上）※ただし単一HTML＝GitHub Pages運用の前提あり。要相談。
- ※これらは未確定。実装前にユーザーへ方向性を確認すること。

## 10. 引き継ぎ時の注意

- コミットは日本語の要約＋箇条書き（既存の履歴に倣う）。バージョンは `GAME_VERSION` と `CHANGELOG` を**必ず両方**更新。
- モデル識別子（`claude-opus-4-8` 等）はコミット/コード/PR等の成果物に**書かない**（チャット内のみ）。
- PR はユーザーが明示的に頼んだ時だけ作成。
