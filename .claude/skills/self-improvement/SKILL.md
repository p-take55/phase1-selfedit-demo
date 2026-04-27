---
name: self-improvement
description: >
  エージェント自身を改善するときに使う。対象は CLAUDE.md、skills、MCP/tool、
  CLI command knowledge、memory、knowledge。自己改善、agent discovery、
  skill discovery、他エージェント調査、workflow improvement、繰り返し失敗した作業、
  既存 agent / skill / tool から学んで自分の repo を更新したいときに起動する。
---

# Self-Improvement（自己改善）

エージェントは、永続指示・task skill・MCP server・CLI command・memory・
knowledge の総合として機能する。自己改善はファイル編集だけではない。
必要なら MCP server を追加し、CLI command をインストールし、その使い方を
skill / knowledge に残して、次回から実際に使える能力にする。

## 改善フロー

1. ユーザーの要求を理解する。
   発言そのままを実装しようとしない。要求の裏にある意図を取り出す。
   - 表面の依頼: ユーザーが直接言っていること（例: 「最新の AI 情報を教えるエージェントを作って」）
   - 解こうとしている問題: なぜそれを欲しがるか（情報を追えない、判断材料が遅れる、社内に共有したい等）
   - 利用シーン: いつ、何の作業の前後で、どの粒度で助けてほしいか
   - 成功条件: 何が起きれば「期待通り」と言えるか
   情報が足りないところは推測で埋めず、ユーザーに確認する。
   ここでズレたまま進むと、以降のフローはすべて空振りする。
2. 制約を一度外して、理想挙動をコンセプトとして組み立てる。
   既存 skill に収まるか、今の tool でできるか、今の repo 構造にあるかはまだ考えない。
   ただし「自由に描く」だけでは発散するので、関心を分けて言語化する:

   - 対象・スコープ: agent は何を扱い、何を扱わないか
   - 起動・入力: いつ / 何によって動き、どんな文脈が必要か
   - 判断: 何を決めるか、どんな基準で決めるか
   - 行動・出力: 何を実行し、ユーザー / 世界に何を届けるか
   - 学習・記憶: 何を残し、次回どう再利用するか

   例1: 「最新の AI 情報を教える」
   対象（AI 業界の動向）/ 起動（毎朝・急報・ユーザー質問）/
   判断（信頼度・新規性・関心マッチ）/
   行動（要約配信・急報通知・参照リンク）/
   学習（既読・関心トピックの蓄積）。

   例2: 「コードレビューを担う」
   対象（PR の差分）/ 起動（PR open・push・依頼）/
   判断（破壊的変更・テスト網羅・規約逸脱）/
   行動（コメント・修正提案・blocker 通知）/
   学習（過去レビュー指摘の傾向・チームの好み）。
3. 現状を点検してギャップを発見する。
   まず agent の土台が埋まっているかを読む。空・placeholder・テンプレートの初期値のままなら、ユーザーが直接頼んでいなくても改善候補として記録する。
   - `CLAUDE.md`: identity / 役割 / 価値観 / 境界が具体に書かれているか。空 or 一般論だけならここを埋めることが最優先候補
   - `.claude/skills/`: 起動できる workflow が並んでいるか。platform skill しか無いなら自分の仕事に合う skill が不足している可能性が高い
   - `environment.yaml`: 仕事に必要な command / package が宣言されているか
   - `memory/` / `knowledge/`: 直近の判断・安定した参照が残っているか
   そのうえで step 2 のコンセプトを基準に、観点ごとに「理想ではこう動くはず / 現状はここで詰まる」の対でギャップを洗い出す。
   - 判断基準: 価値観・優先順位・トレードオフが agent 内に存在するか
   - 起動条件: いつ動くべきかを agent が認識できる入力経路があるか
   - 仕事の型: 繰り返し作業の手順が再利用可能な形になっているか
   - 外部サービスへの手: 必要な API / SaaS / UI / DB に直接触れる手があるか
   - ローカル処理: 必要な command / library / package が揃っているか
   - 参照知識: 用語・設定・規約・先行決定が agent から参照できる場所にあるか
   - 引き継ぎ状態: 直前の判断や保留を次 session に渡せるか
   土台の空欄もユーザー要求由来のギャップも、両方ユーザーに変更提案として提示する。ギャップが本当に見つからなければ、改善は不要。手段の検討（step 4）に進まない。
4. 実現手段を設計する。skill 作成だけを前提にしない。

   - `CLAUDE.md`: agent の役割、価値観、守るべき境界を変える
   - `.claude/skills/<name>/SKILL.md`: 繰り返す仕事の型を起動可能にする
   - MCP server 追加: 外部 service / data source / UI / DB を直接扱う手を増やす
   - CLI install: ローカル処理、検証、生成、検索を速く確実にする
   - `environment.yaml`: CLI / package の再現性を残す
   - `knowledge/<topic>.md`: 安定した事実、設定、使い分けを残す
   - `memory/<date>-<topic>.md`: 次 session への一時的な引き継ぎを残す

   どの手段を選ぶか・具体的にどの MCP / CLI / API を使うかが分からないときは、
   ここで web search を使う。理想挙動とギャップに直結する具体語で検索する:
   - 想定する MCP server / CLI / API / data source の名前
   - 似た目的の OSS / agent / skill 実装
   - 解決方式（pipeline 構成、収集間隔、評価方法など）
   先行事例から抽出するのは文言ではなく「どの能力で理想挙動を実現しているか」。
   将来は aachat 内の agent / skill DB を直接検索できるようにする想定。
5. 実装する。まず理想挙動に効く最小の能力を足す。
   - 説明を書くだけで能力が増えないなら、MCP / CLI / package 追加まで進める
   - 能力を追加したら、いつ・どう使うかを skill / knowledge / environment に残す
6. 検証する。
   - 理想挙動に近づいたか
   - 次の session が同じ能力を再現できるか
   - 起動条件は description にあり、本文は実行手順に集中しているか

## 改善の品質基準

良い自己改善:

- ユーザー要求と理想挙動から逆算している（場当たりの修正ではない）
- ギャップが言語化されており、それを埋める手段になっている
- 土台（`CLAUDE.md` / skills / environment.yaml / knowledge）の空欄を見落とさず、必要なら頼まれていなくても変更提案を上げている
- 文言追加で済まない場合は MCP / CLI / package 追加まで踏み込んでいる
- 起動条件が skill の description に表現され、本文は実行手順に集中している
- 追加した能力が次 session でも同じように再現できる
- 既存の skill / knowledge と矛盾せず、重複や競合を増やさない

避けること:

- 要求理解（step 1-2）を飛ばして手段から始める
- 土台が空のまま個別 skill だけを足す（identity が無い agent に技だけ持たせない）
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
