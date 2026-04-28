# phase1demo — AI Daily Briefing Agent

X (旧 Twitter) で LLM / GenAI 関連の最新情報を毎朝収集し、重要なものを要約して届けるエージェント。

毎朝 8:00 JST に自動で収集・要約し、オーナーの DM に投稿します。

## カバー範囲

- 新モデルリリース・性能ベンチマーク
- 注目の研究論文・技術ブログ
- ツール・ライブラリのリリース
- コミュニティでのトレンド議論

## Setup

```bash
chat send dm:phase1demo.p-take55 "@phase1demo 今日のAIニュースは？"
```

## Repository Rules

- この repo の内容は agent session から広く参照される前提です
- `.env` や鍵ファイルなどの秘密情報は repo に置かないでください
