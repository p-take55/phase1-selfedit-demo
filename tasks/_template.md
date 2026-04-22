---
# --- aachat tasks template ---
# ファイル名 = タスク ID。リネームは ID 変更と同義。
# 作成 / 更新日時は DB 側で管理されるため、ここには書かない。
#
# status:       proposed / open / input_required / failed / completed / cancelled
# mode:         plan / code / review 等、プロジェクト固有
# assigned_agent_name: 担当エージェント
# depends_on:   他エンティティの <kind>/<id> 形式（例: tasks/investigate-turn-state）
#               拡張子は書かない。同種別内でも <kind> を省略しない
# turns:        Turn の配列
# current_turn: 0..turns.len()
# pre_commands / post_commands: 実行前後のコマンド
# artifacts:    name / template_id / delivered
# parent:       親エンティティの <kind>/<id> 形式（オプション）
# new_session:  true で context 完全リセット
#
# 本文（この下）がタスクの prompt です。
# 他エンティティへの言及は [[<kind>/<id>]] 記法で書く（例: [[tasks/fix-login]]）。
# 参照先の存在は強制されない（dead link は WebUI で灰色表示になるのみ）。
# 推奨構造:
#   ## ゴール
#   ## 問い
#   ## 情報源
#   ## 成果物
#   ## 今回やらないこと
---

## ゴール

（一行で意味を書く。成果物ではなく、プロジェクトにとっての価値）

## 問い

（価値ある問いを 1 つ）

## 情報源

- （取りに行くファイル / ドキュメント / MCP）

## 成果物

（何がどこに書かれたら完了か）

## 今回やらないこと

- （除外項目。フォーカスは除外で作る）
