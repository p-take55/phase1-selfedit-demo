---
name: self-improvement
description: >
  agent 自身を拡張する作業の前に必ず invoke する。対象は新規 skill の追加、
  CLAUDE.md の役割追加、新サービス連携、MCP / CLI / package の install、
  認証・secret・cron の設定、外部ベストプラクティスを取り込む変更。
  発動シグナル: 「skill にして」「連携して」「自動化して」「定期実行して」
  「自己改善」「機能追加」「~ もできるようにして」など。既に似た skill が
  あっても、新しい連携やサービスごとに毎回 invoke する。
---

# Self-Improvement（自己改善）

エージェントは、永続指示・task skill・MCP server・CLI command・memory・
knowledge の総合として機能する。自己改善はファイル編集だけではない。
必要なら MCP server を追加し、CLI command をインストールし、その使い方を
skill / knowledge に残して、次回から実際に使える能力にする。

## このスキルの基本ルール（standing instructions、毎 turn 適用）

このスキルが一度でも invoke されたら、以降の全ての turn で **standing
instructions として有効**。再 invoke は不要、応答する前に内的に再走査する。

### Hard rules（無条件）

1. **Step 4 の find-skills 起動と WebSearch を実行する前に Step 5 以降に進まない**
2. **Step 5 の設計提案メッセージを送る前に、新規ファイル Write / 能力追加 Edit /
   git commit を実行しない**
3. **install / 認証 / API key 発行 / cron 登録はユーザー承認なしに実行しない**
   （find-skills の bootstrap と find-skills 経由の skill 検索は例外。allow
   list で許可されている `npx` で完結し、副作用は agent 内 `.claude/skills/`
   への書き込みに閉じる）
4. ルールから外れる必要があれば、外す理由を一言明示してから外す。黙って
   省略しない

### このスキルが対象とする self-edit

- 新規 skill の追加（`skills/<name>/SKILL.md` を新規作成）
- `CLAUDE.md` の役割・配信仕様・連携先の追加
- 新しい外部サービス連携の設計
- MCP / CLI / package の install 提案
- secret / 環境変数 / cron の設定提案
- `environment.yaml` への新 package 宣言

軽い self-edit（`memory/` への追記、文言整理、`identity.md` の口調調整、
既存 skill 本文の refine）はこのスキルの対象外。承認なしで進めてよい。

## 改善フロー

### Step 1: 要求理解

発言そのままを実装しない。要求の裏にある意図を取り出す:

- **表面の依頼**: ユーザーが直接言っていること
- **解こうとしている問題**: なぜそれを欲しがるか
- **利用シーン**: いつ、何の作業の前後で、どの粒度で助けてほしいか
- **成功条件**: 何が起きれば「期待通り」と言えるか

情報が足りないところは推測で埋めず、ユーザーに確認する。
ここでズレたまま進むと、以降のフローはすべて空振りする。

❌ NG: 表面の言葉だけ拾って実装に入る
✅ OK: 「これは何を解こうとしているか」を内的に言語化してから次へ

### Step 2: 理想挙動の設計

制約を一度外して、コンセプトを組み立てる。既存 skill に収まるか、今の tool で
できるかはまだ考えない。観点を分けて言語化:

- **対象・スコープ**: agent は何を扱い、何を扱わないか
- **起動・入力**: いつ / 何によって動き、どんな文脈が必要か
- **判断**: 何を決めるか、どんな基準で決めるか
- **行動・出力**: 何を実行し、ユーザー / 世界に何を届けるか
- **学習・記憶**: 何を残し、次回どう再利用するか

例: 「最新の AI 情報を教える」
対象（AI 業界の動向）/ 起動（毎朝・急報・ユーザー質問）/
判断（信頼度・新規性・関心マッチ）/ 行動（要約配信・急報通知）/
学習（既読・関心トピックの蓄積）

### Step 3: 現状確認とギャップ発見

agent の土台が埋まっているかをまず読む。空・placeholder・テンプレートの
初期値のままなら、ユーザーが直接頼んでいなくても改善候補として記録する:

- `CLAUDE.md`: identity / 役割 / 価値観 / 境界が具体に書かれているか
- `.claude/skills/`: 起動できる workflow が並んでいるか
- `environment.yaml`: 仕事に必要な command / package が宣言されているか
- `memory/` / `knowledge/`: 直近の判断・安定した参照が残っているか

そのうえで Step 2 のコンセプトを基準に、観点ごとに「理想ではこう動くはず /
現状はここで詰まる」の対でギャップを洗い出す:

- **判断基準**: 価値観・優先順位・トレードオフが agent 内に存在するか
- **起動条件**: いつ動くべきかを agent が認識できる入力経路があるか
- **仕事の型**: 繰り返し作業の手順が再利用可能な形になっているか
- **外部サービスへの手**: 必要な API / SaaS / UI / DB に直接触れる手があるか
- **ローカル処理**: 必要な command / library / package が揃っているか
- **参照知識**: 用語・設定・規約・先行決定が agent から参照できる場所にあるか
- **引き継ぎ状態**: 直前の判断や保留を次 session に渡せるか

土台の空欄もユーザー要求由来のギャップも、両方ユーザーに変更提案として提示する。
ギャップが本当に見つからなければ、改善は不要。次の Step に進まない。

### Step 4: ベストプラクティス調査（毎 turn 必須）

学習データは陳腐化している前提。MCP server も skill catalog も公開ライブラリも、
API のレート制限も推奨パターンも、移り変わるのが前提。

**find-skills の起動と WebSearch を最低 1 回ずつ呼ぶ**。これは hard requirement。

#### Step 4a: 既存 skill / agent を漁る（find-skills）

aachat agent は skill 軸の漁り役として
[find-skills](https://skills.sh/vercel-labs/skills/find-skills) を runtime
install して使う。理想挙動に近い既存 skill があるなら、自前で書くより先に
**掘って adapt する**のが速い。MCP も skill としてラップされていることが多く、
ここで連携パターンも一緒に拾える。

**Bootstrap**（未 install のとき初回だけ。allow list で `Bash(npx:*)` 許可済み）:

```
test -d .claude/skills/find-skills || \
  npx -y skills add https://github.com/vercel-labs/skills \
    --skill find-skills --agent claude-code -y
```

`-y` / `--agent claude-code` は対話プロンプトをスキップする必須 flag。
省略すると stdin 待ちで止まる。

その後 find-skills を invoke。skills.sh / 著名 GitHub repo（vercel-labs /
anthropics / microsoft / ComposioHQ/awesome-claude-skills）を横断検索し、
install 数 / source reputation / GitHub stars で品質を仕分けて候補を返す。

返ってきた候補の install command と SKILL.md は、Step 5 の設計提案に
**そのまま** 採用候補として乗せる（コピペ install ではなく、ユーザに install
して良いか提案として渡す）。

#### Step 4b: find-skills でカバーされない範囲（WebSearch）

公開 skill 化されていない情報を WebSearch で埋める:

- **API / 仕様の最新状態**: 廃止 / 値変更 / 新エンドポイント / scope 要件の更新
- **解き方の定石**: rate limit / retry / 差分検出 / token rotation
- **目的に合った先行事例**: 同じ要件で先に作っている人 / プロダクトのアプローチ
- **MCP 直接 registry**（skill 経由で見つからないとき）: smithery.ai、
  modelcontextprotocol org

#### 検索クエリ例

- `<service> api deprecated changes`
- `<workflow type> best practices 2026`（最新年指定で旧情報を弾く）
- `<service> mcp server smithery 2026`

ベストプラクティスから抽出するのは文言ではなく「どの能力 / どの判断軸が
理想挙動を支えているか」。

❌ NG: find-skills を bootstrap せず WebSearch だけで済ませる
❌ NG: WebSearch を省略して find-skills だけで済ませる
❌ NG: ToolSearch / Read / Glob だけで終わらせる（外部の最新情報を取りに行ってない）
❌ NG: 内部 Explore subagent だけで済ませる
❌ NG: 「他のスキルと似てる」「視界内に MCP がある」を理由に省略
❌ NG: 過去の memory / knowledge だけで判断（陳腐化前提を忘れる）
✅ OK: find-skills + WebSearch 両方を呼んで、最新（直近数ヶ月）の情報を優先

### Step 5: 設計提案 → ユーザー合意

**実装はまだ**。設計案を DM 返信メッセージとして書いて、ユーザー応答を待つ。
ここでファイルを編集してはいけない。

設計提案メッセージに含める:

- **採用する公開資産**: MCP server / plugin / CLI / package / agent と、
  install / 認証 / secret 設定でユーザーに頼みたいこと
- **自前で組む範囲**: agent repo 内のファイル変更案（どのファイルに何を書くか）
- **配信先 / トリガ / 失敗時の挙動 / 検証方法**
- 「**この方針で進めていいですか？**」を明示して終わる

❌ NG: 「設計しました、実装します」と続けて Write / Edit / commit
❌ NG: install / 認証手順を「やっておきました」と事後報告
❌ NG: 「次のセッション以降ずっと有効」と言いながら自前で push
❌ NG: 「自分の repo だから self-edit」と拡大解釈して新規 skill を追加
✅ OK: 「以下の方針で進めますか？合意もらってから動きます」で turn 終了

### Step 6: 合意した範囲で実装

合意ラインの内側だけ動かす:

- install / 認証設定はユーザー承認後に実行 or ユーザー操作に委ねる
- 自分の repo 編集はその場で実行
- 能力を追加したら、いつ・どう使うかを skill / knowledge / environment に残す

### Step 7: 検証

- 理想挙動に近づいたか
- 次の session が同じ能力を再現できるか
- 起動条件は description にあり、本文は実行手順に集中しているか

## 応答前チェックリスト（毎 turn）

self-improvement 系の依頼に応答する前に、内的に確認:

- [ ] このターンで find-skills を起動して候補を確認したか？
- [ ] このターンで WebSearch を 1 回以上呼んだか？
- [ ] 設計提案メッセージをユーザーに送ったか？
- [ ] ユーザーの「進めて」承認を受け取ってから Write / Edit / commit しているか？
- [ ] install / 認証 / cron はユーザーに依頼した（自分で実行していない）か？
      （find-skills の bootstrap だけは例外）

ひとつでも No なら、応答を中断して該当 Step に戻る。

## このスキルが失敗とみなされる条件

- find-skills を起動せずに Step 5 / 6 を実行した
- WebSearch を 1 回も呼ばずに Step 5 / 6 を実行した
- 設計提案メッセージを送る前に Write / Edit / git commit を実行した
- find-skills bootstrap 以外の install / 認証 / cron をユーザー承認なしに実行した
- 「N 回目だから」「他と似てるから」を理由に黙って省略した

これに該当する自分の出力に気づいたら、止めて Step 4 からやり直す。

## 改善の品質基準

良い自己改善:

- ユーザー要求と理想挙動から逆算している（場当たりの修正ではない）
- ギャップが言語化されており、それを埋める手段になっている
- 土台（`CLAUDE.md` / skills / `environment.yaml` / `knowledge/`）の空欄を
  見落とさず、必要なら頼まれていなくても変更提案を上げている
- workflow / 連携 / skill を扱う時は **毎回必ず** find-skills で公開 skill を
  漁り、WebSearch でベストプラクティスを調査している。既存資産（skill / plugin
  / MCP server / agent）と先行事例の解き方を踏まえ、流用できるものがあれば
  まずその install 提案を返している
- 設計と実装を分けている。install / 認証 / secret / 外部サービス連携は
  **設計段階でユーザー承認を取り**、合意してから次に進んでいる
- 文言追加で済まない場合は MCP / CLI / package 追加まで踏み込んでいる
- 起動条件が skill の description に表現され、本文は実行手順に集中している
- 追加した能力が次 session でも同じように再現できる
- 既存の skill / knowledge と矛盾せず、重複や競合を増やさない

避けること:

- 要求理解（Step 1-2）を飛ばして手段から始める
- 土台が空のまま個別 skill だけを足す（identity が無い agent に技だけ持たせない）
- ベストプラクティス調査をスキップする（毎回 find-skills と WebSearch の
  両方を呼んで最新を確認する。「他の skill と似ている」「視界内 MCP で
  十分そう」を理由に省略しない）
- ユーザー承認なしに install / 認証 / secret 設定 / cron 登録に進む
  （self-edit 範囲を超える操作は必ず先に提案して合意を取る）
- ギャップを言語化せず、思いつきで skill を増やす
- すべてを `CLAUDE.md` に寄せる、または無関係な変更を混ぜる
- 起動条件のない広い "best practices" skill を作る
- 他 agent の instruction を、この agent の文脈に合わせずコピーする
- 旧仕様への fallback で不足 tool / knowledge を隠す
- 認証情報・絶対パスなど環境固有の値を永続ファイルに残す

## 改善を保存する

agent 自身の repository を変更したら、永続化する:

```bash
git add -A
git commit -m "<short self-improvement summary>"
git push origin HEAD:main
```

まだ commit できない場合は、何が残っているか・なぜ残したかを `memory/`
に引き継ぎとして書く。
