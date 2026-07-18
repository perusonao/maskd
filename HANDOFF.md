# MASKD 開発引き継ぎ資料

> 別セッション／別担当への引き継ぎ用。この1枚を最初に読めば、すぐに開発を再開できるようにまとめています。
> 最終更新: 2026-07-17 / 到達バージョン: **v0.59.4**

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
- v0.59.1: **アンケート機能を追加**（Phase2「300人にプレイしてもらう」に向けたフィードバック収集基盤）。①`#surveyScreen`（設定画面から`openSurvey()`で遷移）で「面白かった度合い(1-5)」「またやりたいか(ぜひ/たぶん/あまり)」「自由記述(任意・500字)」の3問に回答できる。②Firebase Firestoreの新規`surveys`コレクションに`window.__maskdSaveSurvey()`で保存（`logs`/`players`と同じ匿名パターン、`playerId`は`getPlayerId()`を再利用）。**Firestoreセキュリティルールに`surveys`用のcreate/read許可を追加する必要がある**（index.html内のコメントに例を記載）。③送信時、`https://ntfy.sh/maskd-survey-20260717`（ユーザー個人の秘密トピック名）へブラウザから直接POSTしてスマホへプッシュ通知（ntfyアプリで同トピックを購読していれば届く。サーバー/Cloud Functions不要・失敗しても回答自体はFirestoreに保存済みなので握りつぶす設計）。④`admin.html`にアンケート集計セクションを追加（`surveySectionHTML()`、ダッシュボード最上部に表示。回答数・平均点・またやりたい内訳・個別コメント一覧。読み取り失敗時は`logs`と同様のセキュリティルール案内バナー）。テスト用に`window.__adminMock(logs,count,surveys)`の第3引数を追加。既存のFirebase設定（プロジェクト`maskd-8eafb`）をそのまま流用しているため追加のデプロイ作業は不要、Firestoreルールの追加だけで有効化できる。
- v0.59.2: **ntfy通知が届かない不具合を修正**。`fetch('https://ntfy.sh/'+NTFY_TOPIC, {headers:{'Title':'MASKD アンケート回答',...}})`の`Title`ヘッダー値に日本語（マルチバイト文字）を直接渡していたため、`Headers`が`TypeError: Cannot convert argument to a ByteString`で**同期的に例外を投げていた**（HTTPヘッダー値はASCII/Latin1のみ有効という仕様）。`fetch(...).catch(()=>{})`という書き方は非同期の**rejectionしか**捕まえられず、`fetch()`呼び出し自体が投げる同期例外は外側の`try{}catch(e){}`（中身が空）で握りつぶしていたため、画面上は「送信しました」と正常表示されるのに実際には一度もHTTPリクエストが飛んでいなかった。修正: `Title`を`'MASKD Survey'`（英語）に変更、両方のcatch節に`console.warn`を追加して今後同種の不具合がサイレントに埋もれないようにした。**教訓**: fetchのカスタムヘッダーに動的な文字列（特に日本語）を入れる時は、値がASCII範囲であることを確認するか、`encodeURIComponent`等でエスケープすること。
- v0.59.3: **通知から管理画面へワンタップ遷移**。①`index.html`に`ADMIN_URL='https://perusonao.github.io/maskd/admin.html?pass=maskd-admin'`を追加し、ntfy送信時のヘッダーに`'Click':ADMIN_URL`を付与（ntfyの標準機能。通知をタップするとこのURLが開く）。②`admin.html`に`tryAutoUnlockFromURL()`を追加：起動時に`?pass=`パラメータを見て、既存のパスフレーズ用SHA-256ハッシュ(`PASS_SHA256`)と照合し、一致すれば`sessionStorage`に解錠フラグを立てて自動的にダッシュボードを表示。一致後は`history.replaceState`でURLから`pass`パラメータを消し、アドレスバーや共有時に合言葉が平文で残らないようにしている。一致しない場合は通常通りゲート画面のまま（誤った値でバイパスされないことをPlaywrightで確認済み）。admin.html自体に「このゲートは目隠しであり、真のセキュリティではありません」という既存の注記があり、この設計とも整合している。
- **v0.59.4（現在）**: **結果画面からアンケートへ誘導**。`surveyPromptHTML()`（未回答なら`localStorage`の`maskd_survey_submitted`を見て誘導文言を返す。既に回答済みなら空文字を返し何も出さない）を新設し、本戦の結果画面(`endGame()`の`extra`、`milestoneSection`の直後)と練習モードの結果画面(`endPracticeMode()`の`learnBox`、`.tutm-controls`の直前)の両方に挿入。クリック時は`promptSurvey()`が`#overlay`/`#resultOv`/`#learnOverlay`を全て閉じてから`openSurvey()`を呼ぶ(元がどちらの結果画面でも同じ関数で遷移できる)。Playwrightで「未回答時は表示→クリックで遷移→送信後は次の対戦で非表示」の一連の流れを確認済み。
- **付随作業（コード変更なし）**: v1.0前の実機テストで溜まった`logs`(42件)・`surveys`(5件)をFirestore REST API(読み取りは`allow read:true`のため可能)経由でJSONにバックアップしてユーザーに送付。削除はセキュリティルールで`allow update, delete: if false`のため外部からは実行不可・Firebaseコンソールでのユーザー自身の手動削除が必要（手順を案内済み）。副次的に、`players`コレクションが`logs`/`surveys`のような専用ルールを持たずキャッチオール`allow read,write:if false`に該当し、名前登録機能(`window.__maskdSaveName`)が恐らく常に失敗していたことが判明（今回は対応範囲外として未修正、要望があれば`/players/{docId}`用のルール追加で直せる）。
- v0.60.0: ユーザーからの4件の要望＋UIレビューの残りに対応。①**練習モード(初級/中級/上級)もプレイ記録の対象に**：`endPracticeMode()`内で`computeMatchSummary()`の結果(`_summary`、既存の`maybeSendGameLog`と共有)を使い`saveGameToHistory(youFinal, cpuFinal, _summary, {level:levelKey, isTop:false})`を新規に呼び出す。`saveGameToHistory`のシグネチャに4件目の引数`meta={level, isTop}`を追加し、保存されるゲームオブジェクトに`level`/`isTop`タグを付与（本戦側の呼び出し=`endGame()`も`{level:levelKeyFromAttrs(deckAttrs), isTop:true}`を渡すよう更新）。これによりv0.59.0の引き継ぎメモに書いた「本戦以外は`saveGameToHistory`が呼ばれずホームの『◯戦』にカウントされない」という既知の欠落が解消。実績システム(`checkAchievements`)側への配線は今回のスコープ外（要望が「プレイ記録」のみだったため）。②**ホーム戦績バッジの表記変更**：`renderHomeStats()`の`rateEl.textContent`を`勝率${...}%`のみから`${s.wins}勝（勝率${s.winRate.toFixed(0)}%）`に変更。0戦時の`勝率 –%`表示は変更なし。バッジ幅は`.home-badge`が`display:flex`の可変幅でレイアウト崩れなし（12戦分のダミー履歴を注入してPlaywrightで実測、`.home-badge`幅165px/`.home-top`幅332pxで`.home-bell`と重ならないことを確認済み）。③**プレイヤー登録人数の可視化＋通知**：`submitPlayerName()`内の`window.__maskdSaveName(v)`呼び出し直後に、ntfy(`NTFY_TOPIC`)へ英語ヘッダー(`Title:'MASKD New Player'`)でPOSTする通知ブロックを追加（v0.59.2の教訓＝ヘッダー値は非ASCII不可、に従い日本語は本文(body)側のみで使用）。`admin.html`側は`loadData()`に`players`コレクションの`getDocs`(最大300件)＋`getCountFromServer`を追加、新設`playersSectionHTML()`でKPI件数＋直近登録者一覧を表示。**この機能が実データを表示するには、まだ未適用の`/players/{docId}`用Firestoreルール（v0.59.4の引き継ぎ事項）をユーザー側で追加する必要がある**（`surveys`と同じパターンのルール文を提示済み）。④**ホーム画面メニュー並び替え＋対戦設定画面の整理**：ユーザーの10項目UIレビューのうち、事実確認の上で妥当と判断した項目のみ実装。ホームメニューを「遊び方/ルール/実績」「ミッション/ランク/戦績」の2段に並び替え（初見ユーザーが遊ぶ前に必要な情報を上段に）。`#ruleOpts`のラベル「使う属性（むずかしさ）」→「使用するカード」、`#cpuDiffOpts`各ボタンに`.opt-sub`（CPUが手加減/CPUが本気）を追加、`updateModeSummary()`のルールチップ文言を明確化（「+KAOS（特殊）」→「+KAOS（特殊カード）」等）。既存の「このレベルのルール」「このレベルの主」(旧`step③④`)を`<details class="setup-details">`（要素名は`.ov-details`と同系統のネイティブ`<details>`パターン）で折りたたみ、初見でも①②の選択と対戦ボタンまでを一目で見せる構成に変更。`renderLevelInfo()`に「クリア条件」の見出しラベルを追加。**却下した提案**：レビュー内のカード一言説明の修正案（MOON→「SUNに勝つ」等）は`beats={moon:'dark',dark:'sun',sun:'moon'}`の実ルールと逆で不採用（v0.58.0の既存の正しい文言を維持）。「本戦」を「ランキング対象」とする表現も、ランキング機能自体が未実装(`homeRankSoon()`は「近日公開予定」)のため不採用、実態通り「勝敗を記録」のまま。CTAボタン(`対戦する`)の拡大は、ボタンが固定比率(`830/174`)の画像アセットで既に`max-width:400px`(カラム全幅)に達しており、これ以上拡大すると他要素とアスペクト比が崩れる／余白が破綻するため見送り（ユーザー自身も①について余白崩れを懸念していたため、安全側に倒した）。検証: Playwright(402×874)でホームメニュー順・折りたたみ開閉・CPUサブ文言・ルールチップ文言・クリア条件見出しの表示、および対戦設定画面が折りたたみ込みで`scrollHeight===clientHeight`(スクロール不要)のままであることを確認。4パターンのCPU設定(aggressive/strong, cautious/weak, tricky/normal, random/normal)によるフルゲーム回帰テストもJSエラー0・オーバーフローなしで通過。**なお`/players/{docId}`ルールはこの後ユーザー側で追加・適用済みを確認**（Firestore REST APIで`players`コレクションへの読み取りが`200`（空データ）になり、ルール未設定の別コレクションで試すと`403 PERMISSION_DENIED`になることと比較して確認）。
- v0.60.1: 「管理画面でプレイヤーの一覧（プレイ数）を見たい」への対応。**発見した設計上の欠落**: `players`コレクションのドキュメントは`{name, serverTs}`のみで`playerId`を保存しておらず、`logs`（対戦ログ、`playerId`/`playerName`を保存済み）と正確に突き合わせる手段がなかった。①`window.__maskdSaveName`の引数を`(name)`→`(name, playerId)`に拡張し、`players`ドキュメントに`playerId`フィールドを追加保存（呼び出し元`submitPlayerName()`も`window.__maskdSaveName(v, getPlayerId())`に変更）。②`admin.html`の`playersSectionHTML()`に`playCountFor(x)`を追加：`x.playerId`があれば`allLogs.filter(g=>g.playerId===x.playerId).length`で正確にカウント、無ければ(この変更より前の登録データ)`allLogs.filter(g=>g.playerName===x.name).length`で名前一致にフォールバック（同名の別人がいる場合は不正確になりうる旨を一覧下に注記として明示）。プレイヤー一覧の各行に「◯戦」を追加表示。Playwrightの`__adminMock`に、playerId有り/無し両方のケース(新規3件+旧データ1件+未プレイ1件)を混在させたテストデータで検証し、それぞれ正しい件数(3戦/1戦/2戦/0戦)が出ることを確認。
- v0.61.0: 管理画面(admin.html)に**論理削除(非表示)機能**を追加。ユーザーからの「履歴の論理削除機能は作れるか」への回答として2方式を提示し（①追記型＝既存の`allow create:if true; allow update,delete:if false`という追記のみのセキュリティ設計を一切変えずに済む方式、②直接更新型＝対象ドキュメントに`hidden:true`を直接書き込む方式だが`allow update`を緩める必要があり、admin.htmlのパスフレーズゲートは目隠し程度(真の認証ではない)なので誰でも同じ更新を実行できてしまう懸念がある）、ユーザーは①を選択、対象は「対戦ログ・アンケート回答・登録プレイヤー」の3種類全部を選択。**実装**: 新規コレクション`hidden_records`に`{col:'logs'|'surveys'|'players', docId, action:'hide'|'unhide', ts:serverTimestamp()}`を`addDoc`で追記するだけの設計（対象データ自体は一切書き換えない）。同じ`(col,docId)`に対する最新のイベント(タイムスタンプ順)が現在の非表示状態を表す。admin.html側は`loadData()`で`logs`/`surveys`/`players`の各ドキュメント取得時に`d.id`を`_id`として保持するよう変更（従来は`d.data()`のみでFirestoreドキュメントIDを捨てていた）、`hidden_records`も読み込んで`hiddenMap`(`col|docId`→真偽値)を構築。`isHidden(col,id)`/`visibleLogs()`/`toggleHidden(col,id,hide)`/`hideBtnHTML(col,id)`/`bindHideButtons()`を新設。**統計への反映方針**: バランス分析・KPI・プレイ数などの集計は非表示分を**常に**除外(`visibleLogs()`を使用、トグルの状態に関わらず)。一方、生データテーブル・アンケート一覧・プレイヤー一覧は既定で非表示分を除外して表示しつつ、ヘッダーの「非表示も表示」ボタン(`showHidden`フラグ)をONにすると非表示分もグレーアウト＋「元に戻す」ボタン付きで一覧に混ぜて表示し、目視確認・復元ができるようにした（完全に隠して見えなくすると誤って非表示にした時に気付けないため）。CSV/JSON書き出しも非表示分を除外(`visibleLogs()`)。各セクションに「（うち非表示 N件）」の注記を追加。**発見した副作用**: `showDetail(i)`は`allLogs.indexOf(g)`でグローバルな配列インデックスを使う設計だったため、`filtered()`の元データを`allLogs`→`(showHidden?allLogs:visibleLogs())`に変えても、返るオブジェクトは同一参照なので`indexOf`は変わらず正しく動作する(参照ベースの一致を利用)。`firestore.rules`ファイル(リポジトリに存在する、Firebaseコンソールに貼り付けて使う設計)が`logs`用のルールしか無く、これまでにユーザーが手動でコンソールに追加した`surveys`/`players`用ルールが反映されていなかったことに気づき、`isValidSurvey()`/`isValidPlayer()`/`isValidHiddenRecord()`を追加してリポジトリのルールファイルをユーザーの実際の設定に追いつかせた（今後`hidden_records`のルールを追加する際は、この更新済みファイルを丸ごと貼り直せば良い）。Playwrightで、非表示化→トグルでの表示/非表示切り替え→統計(勝率・KPI)への反映→復元、の一連の流れと、Firebase未接続時に実ボタンをクリックした際の確認ダイアログ/エラーメッセージが正しく出ることを確認。
- v0.61.1: 「プレイヤー名がFirestoreに登録されていなければ登録させてほしい」への対応。**背景**: `players`コレクションはv0.60.0リリース後もしばらくセキュリティルール未設定でキャッチオールのdeny-allに該当し(ユーザーが手動でFirebaseコンソールにルールを追加するまでの間)、その期間に名前を入力したユーザーは`window.__maskdSaveName`が常に失敗していた(黙って握りつぶされていた)。`showNameModal()`は`playerName`(localStorage)が空の時だけ表示される設計のため、既に名前を入力済みのそうしたユーザーは今後も二度と登録画面を通らず、ルールを直しても永久にクラウド未登録のまま取り残される問題があった。**実装**: 新規キー`PLAYER_NAME_SYNCED_KEY='maskd_playername_synced'`に「クラウドへの登録に成功した名前」を記録する。`syncPlayerNameToCloud(name)`(新設)が`window.__maskdSaveName(name, getPlayerId())`を呼び、成功時のみこのキーを更新(失敗時は更新しない→次回また再送を試みる、というリトライ設計)。`submitPlayerName()`は直接`window.__maskdSaveName(...)`を呼んでいた箇所をこの関数呼び出しに置き換え。`ensurePlayerNameSynced()`(新設)は「`playerName`はあるが`PLAYER_NAME_SYNCED_KEY`が現在の名前と一致しない(=未登録 or 登録未確認)」場合にのみ`syncPlayerNameToCloud`を呼ぶ救済関数で、Firebaseモジュールの初期化ブロック内、`window.__maskdLoggingReady=true`の直後(＝`window.__maskdSaveName`が実際に使える状態になった直後)から`window.ensurePlayerNameSynced()`として1回だけ呼ばれる(通常の初回登録フローとは別経路)。**テスト上の制約**: このサンドボックス環境はgstatic.com(Firebase CDN)への外部アクセスがブロックされているため、`window.__maskdLoggingReady`が実際にtrueになる場面をPlaywrightで再現できず、自動トリガー呼び出し自体はコードレビューでのみ確認。関数のロジック(未同期なら送る/既に同期済みなら送らない/名前未設定なら何もしない/送信失敗時はフラグを立てずリトライ余地を残す/`submitPlayerName()`経由でも同じ経路でフラグが立つ)はPlaywrightで`window.__maskdSaveName`を直接スタブして5パターン全て検証済み。
- v0.61.2: admin.htmlのレビュー対応中に発見した重大な既存バグの修正。**発見の経緯**: ユーザーからのadmin.htmlレビューへの回答を準備する過程で、v0.55.0で「実装済み」とされていたセッション開始/対戦開始/離脱/チュートリアル開始・完走の軽量イベント計測(`logEvent()`、`window.__maskdSaveLog`経由で`logs`コレクションへ書き込み)が、`firestore.rules`の`isValidLog()`が`winner`/`playerFinal`/`cpuFinal`を無条件で必須にしていたため、実際には一度も保存できていなかったことが判明(軽量イベントのペイロードはこれら3項目を持たない設計のため、Firestoreのルール違反で毎回create拒否されていた)。レビューが求めていた「離脱率」KPIは、そもそも元データが存在しない状態だった。**修正**: ①`isValidLog()`を「完了した対戦の記録(winner等必須)」または「軽量イベント(`type`が既知の5種類のいずれか)」のどちらかを満たせばOKとするOR条件に変更(既存の完了対戦の記録の互換性は維持)。②`admin.html`: `logs`コレクションに対戦記録とイベントが混在するようになるため、`isGameRecord(g)=(!g.type||g.type==='game')`を新設し、`visibleLogs()`(既存のゲーム統計・生データテーブル・CSV/JSON書き出し全てが依存する)をイベント除外込みに変更(混入すると勝率などの集計が壊れるため)。新設`eventStatsHTML(eventLogs)`でイベント件数(セッション開始/対戦開始/離脱/チュートリアル開始・完走)と、そこから計算した離脱率(離脱/対戦開始)・チュートリアル完走率(完走/開始)を表示するカードを追加(`render()`のKPIグリッド直後、および「対戦ログがまだ0件」の早期returnルートの両方から呼べるよう関数として切り出し)。イベントが0件の間はセクションの代わりに注記のみ表示。**ユーザーの実際のFirestoreルールとの食い違いも発覚**: ユーザーが管理コンソールに貼り付けていた実際のルールを見せてもらったところ、`hidden_records`のブロックが`isValidHiddenRecord()`を呼び出しているのに、その関数定義自体が存在しない状態だった(関数の未定義参照はルールのコンパイルエラーになり、ルールセット全体の公開に失敗している可能性が高い)。また`isValidSurvey()`は`fun is number`ではなく`fun is int`、`size()<=10`ではなく`<=20`、`players`は`isValidPlayer()`を使わず`allow create: if true`のみ、と要所要所でユーザー側の手動編集により差分があった。実害のない差分(int/number、サイズ上限、players簡略化)はユーザーの現状を尊重してそのままにし、実際に壊れていた2点(未定義の`isValidHiddenRecord`、`isValidLog`が軽量イベントを弾く)だけを修正した完全な一枚のルールファイルを新たに提示（`firestore.rules`と同一内容、丸ごと貼り替えれば良い）。テストはv0.61.0と同様のPlaywright `__adminMock`パターンで、対戦記録とイベントが混在するデータ・イベントのみのデータ・完全に空のデータの3パターンについて、統計値がイベントに汚染されないこと・離脱率/完走率が正しく計算されること・生データテーブルにイベント行が混入しないことを確認。
- v0.61.3: ユーザー提供のスマホ実機スクリーンショット(ラウンド結果画面)から「レイアウトが崩れている」不具合を特定。**調査**: 当初は`.rdetail`(結果説明文、ヒントON時は`beginnerResultNote()`による3すくみ解説などが追記され長くなる)が`.result-card`の高さを超えて`.field`からはみ出すオーバーフロー説だと仮定したが、Playwrightで402×709/402×874双方で長文ケースを再現しても`.result-card`は`.field`内に収まり(`cardOverflowsFieldTop/Bottom`とも`false`)、この説は棄却。実機スクショを詳細に拡大したところ、`.result-card`(幅82%)の左右の余白部分に、背後の盤面(スコア表示・使用済み札などの明るい色のUI)がうっすら透けて見えていることが判明。原因は`.result-overlay`の背景`rgba(13,12,20,0.72)`が72%不透明でしかなく、`.result-card`自体(96%不透明)がカバーしない左右9%ずつの余白では背後がほぼそのまま透けてしまう設計だった。Playwrightで実際に対戦を最後まで進めて`win`/`lose`の結果画面を撮影し、結果カード右端に「敗」の字が透けて見えることを実機と同様に再現・確認。**修正**: `.result-overlay`の背景を`rgba(13,12,20,0.94)`に変更(`.result-card`本体の96%不透明に近い水準へ)。再度同じ手順で撮影し、透け込みが解消されたことを確認。4パターンのフルゲーム回帰テストも通過。
- v1.0.0（正式版フリーズ宣言）: TODO.mdの「v1.0完成の定義」4項目をあらためて棚卸しし、A4(計測)がv0.61.2で・A5(Launch整備)が動画/X投稿文完成で・②(見切れなし)が今回のレイアウト修正で、いずれも既に満たされていることを確認。最後に残っていた①「初訪問ユーザーが詰まらず1戦できる」を、このセッションで初めて**本物のコールドスタート**(localStorage完全に空)でPlaywright検証：ウェルカムモーダル→「さっそく対戦する」→名前入力モーダル→ライバル登場→親決め(0カードを1枚めくる)→配札演出→初級4ラウンド完走→結果画面、まで一気通貫でエラー0・横スクロール見切れなし(402×709の短い画面)を確認。全項目達成を確認した上で`GAME_VERSION`/`CHANGELOG`を`v1.0.0`に更新し、フリーズを宣言。

  **今後のリリース運用（ユーザーとの合意）**: v1.0.0以降は用途に応じてブランチを使い分ける。
  - 不具合修正 → バージョンは`1.0.x`で加算 → `main`に直接push（従来通りの運用を継続）。
  - 仕様変更を伴う修正 → バージョンは`1.x.0`で加算 → `develop`ブランチにpush。
  - `develop`にpushした内容を実機確認できるプレビュー環境が必要、という要望があり検討中(下記参照)。

  **プレビュー環境の設計候補**: 現在のGitHub Pagesは「ブランチからデプロイ」方式(`main`固定、Settings上の設定。リポジトリに`.github/workflows/`のカスタムワークフローは存在せず、"pages build and deployment"はGitHubが自動生成する同期ワークフロー)。`develop`用のプレビューを追加する方法を2つ検討した：
  1. **GitHub Actions化 + 合成デプロイ**: Pages設定を「GitHub Actionsから配信」に切り替え、`push: [main, develop]`いずれのトリガーでも「`main`の内容をルートに、`develop`の内容を`/dev/`配下に」両方チェックアウトして1つのアーティファクトとしてデプロイするワークフローを新設する。両ブランチの履歴を汚さずに済むのが利点。ただしPages配信方式の切り替えに管理者権限が必要な可能性があり、要検証。
  2. **`main`へのボットコミット方式**: `develop`にpushされたら、その内容を`main`の`dev/`サブフォルダにコピーしてコミット・pushするワークフロー。Pages設定は変更不要ですぐ作れるが、`main`の履歴に`github-actions[bot]`名義の自動コミットが増える(`dev/`フォルダのみに限定されるが)。
  外部ホスティング(Netlify/Vercel等の「ブランチごとに自動でプレビューURLが出る」機能)も選択肢としてはあるが、ユーザーの外部アカウント連携が必要なためこのセッションでは完結できない。

  **→ 方式2を採用・実装済み**（Pages設定変更が不要ですぐ動かせるため）。`.github/workflows/dev-preview.yml`を新設：`develop`へのpushをトリガーに、`main`をチェックアウトして`develop`のランタイム資産(index.html/admin.html/manifest/画像アイコン類、計14ファイル)を`dev/`フォルダへコピーし、差分があれば`github-actions[bot]`名義でコミット・pushする。結果、同一のPagesサイトから`https://perusonao.github.io/maskd/`(本番=main)と`https://perusonao.github.io/maskd/dev/`(確認用=develop)の両方が見られる。PWAマニフェスト・画像参照は元々相対パスのため、サブフォルダ配信でもパス書き換え不要だった。ボットコミットで`main`の履歴が増える副作用はユーザーに説明済み・了承済み（「今は気になりません」）。

  **gitタグのpushで判明した制約**: `git tag`のpushだけ`HTTP 403`で失敗する(ブランチのpushは問題なく通る)。この環境のgit用ローカルプロキシがタグの新規ref作成を許可していない可能性がある(`create_trigger`/`send_later`等の一部MCPツールで見られた「承認が必要」エラーと同系統の制限とみられる)。対処として、タグ作成はユーザー自身にGitHub Web UIの「Releases → Create a new release」(タグ名入力するとreleaseと同時にtagも作成される)、またはユーザー自身の環境からの`git push origin <tag>`を案内する運用にした。`mcp__github__get_tag`で作成後のタグ存在は確認できる。

- **v1.0.1（現在）**: 「v1.0.0より前の変更履歴は管理画面のみで確認できるようにしたい」への対応。**実装**: index.htmlの`CHANGELOG`配列から、v1.0.0より前の全エントリ(v0.1.0〜v0.61.3、'旧バージョン'含む、計102件)を削除し、`v1.0.1`/`v1.0.0`の2件のみを残した(プレイヤー向けのお知らせベル`openChangelog()`・未読バッジ`unseenChangelogCount()`は元々`CHANGELOG`配列をそのまま参照する実装だったため、フィルタ処理を追加する必要はなく、データ自体を削るだけで両方に反映される)。削除した102件は一字一句そのまま`admin.html`に`PRE_V1_CHANGELOG`という新規配列として移設し、`preV1ChangelogHTML()`(新設)で「📜 開発履歴アーカイブ（v1.0.0より前）」という見出しの`<details>`(既定は折りたたみ)として表示するようにした。ダッシュボード最下部(生データテーブルの直後)、および対戦ログ0件時の早期returnルートの両方から呼ばれる。**注意点**: `admin.html`の`<script type="module">`はモジュールスコープのため、`PRE_V1_CHANGELOG`はブラウザの外部(例えばPlaywrightの`page.evaluate`からの直接アクセスや将来的な別スクリプトからの参照)からは見えない。ページ内の他の関数(`render()`等、同じモジュール内)からは通常通り参照できるため機能上は問題ない。検証時もこの点を踏まえ、DOM上に実際に描画された内容(`.rrow`要素の件数や中身)で正しさを確認する方式に切り替えた(102件・先頭v0.61.3・末尾「旧バージョン」を確認)。**既存ユーザーへの影響**: `unseenChangelogCount()`は「前回既読のバージョン」を`CHANGELOG.findIndex()`で探す実装のため、pre-1.0のバージョン文字列(例:'v0.61.3')が既読として保存されている端末では`findIndex`が`-1`を返し、`idx===-1?0:idx`のフォールバックにより未読件数は単に0になる(エラーにはならない、不自然に大きな数字も出ない)。

- **v1.0.2（現在）**: 「開発用の環境（`/dev/`）で遊んだ結果を本番用のデータに混ぜたくない」への対応。**背景**: `dev-preview.yml`ワークフローが`develop`の`index.html`/`admin.html`をそのまま`main`の`dev/`配下にコピーする設計のため、`/dev/index.html`と本番の`/index.html`は同一のFirebaseプロジェクト(`maskd-8eafb`)・同一コレクション(`logs`/`surveys`/`players`/`hidden_records`)を見に行ってしまい、動作確認中のプレイ・アンケート・名前登録がそのまま本番集計に混入する状態だった。**実装**: 両ファイルとも`location.pathname.includes('/dev/')`でランタイムに実行環境を判定し(`isDevPreview`)、真であれば全てのFirestoreコレクション名に`_dev`サフィックスを付けて読み書きする(`col(name)`/`envCol(name)`ヘルパー、Firebaseプロジェクト自体は共通のまま論理的にコレクションだけ分離)。index.html側は`window.__maskdEnv`('prod'/'dev')を新設し、`__maskdSaveLog`/`__maskdSaveName`/`__maskdSaveSurvey`/`__maskdFetchLogs`/`__maskdFetchCount`の5関数すべてを`col(...)`経由に変更。admin.html側は既存の`isHidden(col,id)`/`toggleHidden(col,id,hide)`が`col`という引数名を「非表示対象の種別」の意味で既に使っていたため、名前衝突を避けて`envCol`という別名で実装(`loadData()`内の7箇所のFirestore参照＋4箇所のルール案内エラーメッセージを`envCol(...)`に変更)。人間が一目で気づけるよう、index.htmlのホーム画面に`#devEnvBadge`(「DEV PREVIEW（開発確認用・本番とは別データ）」、`/dev/`判定時のみ表示)、admin.htmlのヘッダーに`#envBadge`(「DEV」、同条件で表示)を追加。`firestore.rules`にも`logs_dev`/`surveys_dev`/`players_dev`/`hidden_records_dev`の4コレクションを追加(各バリデーション関数は本番と共用、コレクション名のみ分離)。**要作業**: この4つの新規ルールはリポジトリのファイルを更新しただけでは有効にならず、Firebaseコンソールの「Firestore Database→ルール」タブに`firestore.rules`の全文を再度貼り付けて公開する必要がある(過去の`surveys`/`players`/`hidden_records`ルール追加時と同じ運用)。**検証の制約**: このサンドボックス環境はFirebase CDN(gstatic.com)への外部アクセスがブロックされているため、`window.__maskdEnv`や実際のFirestore書き込み先を実ネットワーク経由では確認できない(v0.61.1と同じ制約)。かわりに、Firebase呼び出しに依存しない部分(`#devEnvBadge`/`#envBadge`の表示切り替えロジック)をPlaywrightで検証：`/`相当のパスでは両バッジとも非表示、ローカルにコピーした`dev/index.html`・`dev/admin.html`(パスに`/dev/`を含む)ではそれぞれ正しく表示されることを確認。4パターンのCPU設定によるフルゲーム回帰テストも新規JSエラーなしで通過（既存のFirebase CDN到達不能によるネットワークエラーのみ、これは今回の変更と無関係の既知の制約）。

- **v1.0.3（現在）**: ユーザーが実機で「ntfyの新規プレイヤー登録通知は届いたが、admin.htmlの登録プレイヤー一覧には該当する登録が見当たらない」ことに気づき報告、その原因調査と修正。**原因**: `submitPlayerName()`が`syncPlayerNameToCloud(v)`(Firestoreの`players`への非同期書き込み)の**完了を待たずに**、直後で無条件にntfy通知(`Title:'MASKD New Player'`)を送信する作りになっていた。`syncPlayerNameToCloud`は`window.__maskdSaveName`が未定義(Firebaseモジュールの初期化がまだ終わっていない、あるいはFirebase CDNへの到達自体に失敗している等)の場合はエラーも出さず即座に何もせず返る設計のため、「書き込みは実質的に試みられてすらいないが、通知だけは届く」という食い違いが起こり得た(v0.59.2のntfyヘッダー起因の失敗、v0.61.1のplayers未登録と同系統、「通知と実処理が分離していて通知だけが独り歩きする」というこのコードベースで繰り返し出てきたバグパターン)。**修正**: `syncPlayerNameToCloud(name)`を、成功/失敗の真偽値を解決するPromiseを返す形に変更(`window.__maskdSaveName`未定義時・例外時は`Promise.resolve(false)`)。`submitPlayerName()`側は、このPromiseの`.then(ok=>{ if(!ok) return; ...ntfy送信... })`の中に通知処理を移動し、実際に書き込みが成功した場合のみ通知するようにした。一方、モーダルを閉じてオンボーディングの次のコールバック(`_afterName`)を呼ぶ処理は従来通り同期的(Firestore書き込みの完了を待たない)のままにしてあり、体験速度への影響はない。`ensurePlayerNameSynced()`(過去の未登録プレイヤーを救済する再送経路)は`syncPlayerNameToCloud`を直接呼ぶだけで通知は送らない、という既存の挙動も変更していない(通知を出すのは新規登録操作の直後だけ、という従来の意図を保った)。**検証**: Playwrightで`window.__maskdSaveName`を3パターン(成功/失敗を返す/未定義のまま)でスタブし、`fetch`をフックしてntfyへのPOST回数を計測、成功時のみ1回・それ以外は0回であることを確認。4パターンのCPU設定によるフルゲーム回帰テストも新規JSエラーなしで通過。

- **v1.0.4（現在）**: ユーザーが実機スクリーンショット(初心者向けカード表示・上級者向けカード表示の対戦画面2枚)で「自分の手札のおすすめがはみ出てます」と報告。**原因**: 手札の「おすすめ札」の印(`.rec-badge`、`{position:absolute; top:-7px; right:-6px}`でカードの右上角に少しはみ出す形で重ねて表示する設計)が、初心者向けカード(`.bcard`、数字・じゃんけん記号・属性名を大きく出す表示モード)の`overflow:hidden`によって角丸の外側で切り取られ、バッジのテキストがほぼ読めない状態になっていた。上級者向けカード(`.hcard.art`、仮面イラスト表示)は`overflow:hidden`を持たないクラス構成のため同じ問題が起きず、ユーザーが送ってくれた2枚の実機画像を比較して発見できた。`.bcard`に`overflow:hidden`が必要な本来の理由は、内部の数字(`.bc-num`)・じゃんけん記号(`.bc-glyph`/`.bc-big`)・属性名バナー(`.bc-name`、カード下端いっぱいの帯)がカードの角丸からはみ出さないようにするためで、`.rec-badge`(カードの外に少しはみ出す設計のバッジ)を巻き添えにしていたのが本質的な原因。**修正**: `.bcard`自体からは`overflow:hidden`を外し、代わりに`beginnerInner()`が返すカード内部要素(`bc-num`/`bc-glyph`/`bc-big`/`bc-role`/`bc-name`)だけを新設の`.bc-inner`(`position:absolute;inset:0;overflow:hidden;border-radius:inherit`)で包んで、そちらにoverflow:hiddenを移した。`.rec-badge`と相手からの見え方表示(`.hc-seen`)は`beginnerInner()`の外側、`.hcard.bcard`要素の直接の子のまま(`el.innerHTML=beginnerInner(c)+recBadge+seenRow`)なので、`.bc-inner`によるクリップの対象外になり、上級者向け表示と同じくカードの外に正しくはみ出して表示されるようになった。`border-radius:inherit`により、`.bc-inner`は`.hcard`(手札・9px)/`.card`(場札)/`.uc`(使用済み小札・4px)など、`.bcard`と組み合わさる文脈ごとの角丸半径をそのまま引き継ぐため、追加の個別調整は不要だった。`beginnerInner()`は`faceCardHTML()`・使用済み札プレビュー・対戦中の場札表示・チュートリアルなど複数箇所で共用されている関数のため、この修正で全ての初心者向けカード表示に一括で適用される。**検証**: Playwrightで`youHand`/`dealerCard`/`possibleAttrsByNum`を直接操作し、SUN✋がMOON✊に確実に勝てる状況を人工的に作って「おすすめ」バッジを発生させ、修正前は`.hcard`(=`.bcard`)の`overflow`が`hidden`でバッジが角丸に切り取られていたのに対し、修正後は`overflow:visible`(バッジは切れない)かつ内部の`.bc-inner`は`overflow:hidden`(数字・属性名バナーは従来通りきれいに角丸内に収まる)であることをスクリーンショットと計算スタイルの両方で確認。4パターンのCPU設定によるフルゲーム回帰テストも新規JSエラーなしで通過。

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
