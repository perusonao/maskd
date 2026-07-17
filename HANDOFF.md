# MASKD 開発引き継ぎ資料

> 別セッション／別担当への引き継ぎ用。この1枚を最初に読めば、すぐに開発を再開できるようにまとめています。
> 最終更新: 2026-07-17 / 到達バージョン: **v0.59.1**

---

## 1. プロジェクト概要

- **MASKD** … 仮面をかぶった騎士たちの戦いを描く心理戦・推理カードゲーム（日本語UI）。**1人対戦（プレイヤー vs CPU）**。
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
- v0.44.0: **段階的ルール開放「はじめての対戦」**。`LEARN_STAGES`(4段階)＋`stageConfig`/`deckAttrs`/`matchRounds`＋`stageAllows()`。縮小デッキ`dealHandsStage()`、`computePossibleAttrs`は`deckAttrs`基準に変更、`startGame(dealer, stage)`でステージ設定、`TOTAL_ROUNDS`→`matchRounds`(通常は8)、`scoreMultiplier`/`isLateGame`は`late2x`フラグ＋`matchRounds-1`でゲート、`renderActions`/`cpuWantsForce`/resolveFightで入替・奇襲をゲート、`endGame`でステージ時は`endLearnStage()`へ（履歴/実績/統計に残さない）。UI: `#learnOverlay`＋`openLearnLadder`/`showStageIntro`/`startLearnStage`/`endLearnStage`、`LEARN_KEY`(進捗)。実績`graduate`。導線: タイトル/ウェルカムの「🎓 はじめての対戦」。※タイトルのボタン構成も再編。
- v0.45〜v0.55: 表示改善の細部修正、ホーム画面のロビー化、ゲームモード選択（初級/中級/上級/本戦）とレベル別ライバル・★実績・仮面あて予想、進行度に応じたオンボーディング、プレイログ計測強化、テーマ文言を「仮面舞踏会」から「仮面騎士の戦い」へ統一。詳細はアプリ内 `CHANGELOG` を参照。
- v0.56.0: **v1.0前の画面整理**。①`openLearnLadder`（「はじめての対戦」4段ラダー）への導線をホーム/ウェルカムから撤去（新しいレベル制と内容が重複していたため）。`LEARN_STAGES`関連の関数・DOM(`#learnOverlay`は練習モード結果画面と共用のため存置)は到達不能なコードとして残置＝**復活させる場合はホームの`.home-learn-link`とウェルカムの選択肢を戻すだけでよい**。②結果画面の下部ボタンを「🏅実績」「📊戦績」の2つに集約（`openHistory`/`openMatchLog`は結果画面から除去、`openMatchLog`は`#statsModal`内に残置）。③`#statsModal`から「みんなの統計（クラウド）」ボタンを除去（`openGlobalStats`/`#globalStatsModal`は到達不能のまま残置）。④`#welcomeModal`を「▶さっそく対戦する」中心の2択に簡素化。⑤`#tutorialOverlay`（遊び方）に「⚔️上級ミッションに挑戦」を常設リンクとして追加（従来は基本ミッション完走後のみ到達可能だった）。⑥①の影響で解除不能になった実績`graduate`（仮面騎士デビュー）を`ACHIEVEMENTS`から削除（19種→18種）。
- v0.57.0: **対戦設定画面(`#instructions`)の再構成**。ユーザー提案のモックアップを元に、番号ステップ`.setup-step`（①レベル→②CPU強さ→③ルール→④ライバル→⑤アクション）に再編。旧`.setup-details`（折りたたみ「詳細設定」）は廃止しCSSごと削除、アクション設定は常時表示のステップ⑤に格上げ。③`updateModeSummary()`は長文パラグラフをやめ`.rule-chips`（属性/アクション/記録/終盤2倍のアイコンチップ）＋`.rule-rounds`（ラウンド数）に変更。④`renderLevelInfo()`に`.lv-note`を追加し、「CPUの強さはライバル自体を変えない（同じ相手の手加減のみ変える）」ことを正確に説明（旧モックアップ案にあった「対戦相手はこの強さのライバルになる」という誤った表現は採用せず修正）。「詳細を見る」ボタン（飛び先未定義）とモックアップの下部タブバー（ホーム/デッキ/対戦/成績/設定＝未実装のナビ構造）は範囲外として見送り。
- v0.57.1: **対戦設定画面を1画面(スクロールなし)に圧縮**。`#instructions`は`.app{height:min(920px,97vh)}`の枠に`inset:0`で収まる構造(`overflow-y:auto`)なので、実際の可視高はビューポートではなく`.app`の高さで決まる点に注意。文言短縮（レベルボタンラベル・ルールチップ・★目標=`LEVEL_META.stars[].short`を新設し1行連結表示・注記1行化）と余白圧縮（`.setup-step`padding/margin・`.lv-*`各要素）で実施。`.lv-name`/`.lv-quote`/`.lv-goals-line`/`.lv-note`に`white-space:nowrap`＋`text-overflow:ellipsis`を付与し、内容量に関わらず行数を固定(高さの予測可能性を担保)。検証は幅402px×高さ820/844px系で全6通り(r1/r2/r3×basic/full)ともscrollHeight===clientHeight(スクロール不要)を確認。ただしiPhone SE等の小型端末(高さ667px前後)では`.app`高が97vh制約で縮むため約130px分の内部スクロールが残る(既知の制約、フォント読み込みタイミングにより実測値は多少変動しうる)。
- v0.58.0: **ホーム画面の整理**（ユーザーの10項目レビューのうち、事実誤認や範囲外(TODO.md B/フリーズ対象C)を除いた7項目を実装。カルーセル化とEXPバー化は保留、デイリーチャレンジ機能は明確に見送り＝TODO.md カテゴリB）。①`.home-cta-img`（対戦する）を`.mask-fan`より上に移動。②メニュー2段(`.home-menu`×2)を`.home-menu-group`(gap6px)でまとめ、`.home-set-img`との間に`.home-menu-divider`（細い金の区切り線）を追加。④ホームロゴの`#versionTagTop`（バージョン表記）を撤去し、`#appSettings`末尾に`#versionTagSettings`として移設（JS側は`document.getElementById('versionTagSettings').textContent=GAME_VERSION`に変更）。⑥お知らせベルに**正直な未読バッジ**（`unseenChangelogCount()`/`markChangelogSeen()`/`updateBellBadge()`、キー`maskd_changelog_seen_version`）。初回(未設定時)はその場でGAME_VERSIONを既読扱いにして0件表示＝導入前の履歴を「未読」として水増ししない設計。以後はバージョンが上がるたびに`CHANGELOG`配列中の位置で件数を算出、`openChangelog()`で既読化。⑦`.home-title`のfont-size 60px→44px（ロゴ縮小）。⑧`MASK_FAN`各要素に`d`(一言説明)を追加：MOON="闇に勝つ"/AIR="多くは引分"/SUN="月に勝つ"/KAOS="特殊ルール"/DARK="太陽に勝つ"（`beats={moon:'dark',dark:'sun',sun:'moon'}`の実ルールから正確に算出。ユーザー提示のサンプル文言＝「推理/入替/得点」等はゲーム内容と不一致のため不採用）。⑨設定ボタンは実在7導線（ミッション/実績/ランク/遊び方/ルール/戦績/設定）のみで統一、架空の「ショップ」等は追加せず。加えて`.home-screen{justify-content:flex-start→center}`に変更（要素縮小に伴い縦長画面で下部に不自然な空白ができたための微調整、7項目の指示範囲内の軽微な追随修正）。検証: 402×874/390×844/375×667の3サイズでホーム画面のスクロール無し・バッジ表示/既読化・設定画面のバージョン表示を確認。
- v0.58.1: v0.58.0の未実装分だった「⑤Lv表示の正直化」を実施。`#hbLv`→`#hbTotal`にリネームし、`1+Math.floor(total/2)`の疑似Lv算出をやめて実対戦数をそのまま「◯戦」（未対戦時は「対戦前」）と表示。勝率バー(`#hbBar`)自体の仕様は変更なし（表示中の「Lv」という誤解を招く枠組みだけ外した）。③カルーセル化は見送りのまま据え置き。
- v0.59.0: **対戦設定画面のレベル選択を再構成＋仮面あての位置づけ変更**。①`#ruleOpts`を「初級／中級／上級／本戦」の4択に統合（`data-val="top"`が本戦、クリック時に`selectedRule='r3'; selectedAction='full'`を同時セット）。旧`#actionOpts`(step⑤「アクション設定」)は完全に削除し、選択中の状態表示は新設の`currentLevelChoice()`(rule+actionから'r1'/'r2'/'r3'/'top'を逆算)で行う。**発見した重要な仕様**: `beginMatchSetup()`は`isTop`(=r3&&full)でないと`_lastPracticeCfg`経由の「練習モード」(`stageConfig.practice=true`)になり、`endPracticeMode()`(`#learnOverlay`使用)へ分岐して**`saveGameToHistory()`が呼ばれない＝ホームの「◯戦」に一切カウントされない**。つまり本戦以外は全部ローカル統計に残らない実装だった(テレメトリ送信`maybeSendGameLog`は練習モードでも呼ばれるので計測自体はできている)。②「🔰初心者ではじめる」ボタン(`startBeginnerPreset()`)を削除。`onHomePlay()`は`isFirstTimer()`なら対戦設定画面自体をスキップして直接初級を開始するため、この画面に到達した時点で見ている人は初心者ルート対象外＝完全に重複した導線だった(v0.56.0で削除した「はじめての対戦」ラダーと同じパターン)。③**仮面あて(読み)を得点計算から分離**：`endGame()`の`youFinal`から`+youReadBonus`を削除。CPU側に対応する仕組みが無く非対称だった点をユーザーが指摘、「独立したミニゲームならアリ、ルールの一部なら不要」との判断で「勝敗に影響しない成績」に格下げ。結果画面の得点内訳テーブルから読みボーナス行を削除し、代わりに`milestoneSection`(MVP/通算成績と同じ場所)に「🎯 仮面あてN回的中（勝敗には影響しません）」を追加。`youReadBonus`変数自体はテレメトリ用に残置(`readBonus:`ログ)、`youReadHits`ベースの実績「看破者」・★目標判定は無変更。UIの誘導文言(`+2pt`を含む予想プロンプト・的中トースト)も全て「勝敗に影響しない」トーンに書き換え。README「特徴」も同様に修正。
- **v0.59.1（現在）**: **アンケート機能を追加**（Phase2「300人にプレイしてもらう」に向けたフィードバック収集基盤）。①`#surveyScreen`（設定画面から`openSurvey()`で遷移）で「面白かった度合い(1-5)」「またやりたいか(ぜひ/たぶん/あまり)」「自由記述(任意・500字)」の3問に回答できる。②Firebase Firestoreの新規`surveys`コレクションに`window.__maskdSaveSurvey()`で保存（`logs`/`players`と同じ匿名パターン、`playerId`は`getPlayerId()`を再利用）。**Firestoreセキュリティルールに`surveys`用のcreate/read許可を追加する必要がある**（index.html内のコメントに例を記載）。③送信時、`https://ntfy.sh/maskd-survey-20260717`（ユーザー個人の秘密トピック名）へブラウザから直接POSTしてスマホへプッシュ通知（ntfyアプリで同トピックを購読していれば届く。サーバー/Cloud Functions不要・失敗しても回答自体はFirestoreに保存済みなので握りつぶす設計）。④`admin.html`にアンケート集計セクションを追加（`surveySectionHTML()`、ダッシュボード最上部に表示。回答数・平均点・またやりたい内訳・個別コメント一覧。読み取り失敗時は`logs`と同様のセキュリティルール案内バナー）。テスト用に`window.__adminMock(logs,count,surveys)`の第3引数を追加。既存のFirebase設定（プロジェクト`maskd-8eafb`）をそのまま流用しているため追加のデプロイ作業は不要、Firestoreルールの追加だけで有効化できる。

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
