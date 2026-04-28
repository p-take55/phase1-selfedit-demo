# x-daily-briefing

毎朝 X (旧 Twitter) から LLM / GenAI の最新情報を収集し、要約してオーナーの DM に投稿するスキル。

## 発動条件

- 毎朝 8:00 JST の定期実行
- オーナーから「今日の AI ニュースは？」「X チェックして」「ブリーフィングして」等の依頼

## 手順

### 1. X 検索（x-search スキルを使う）

以下のクエリで過去 24 時間を検索。クエリは最大 4 本まで（課金節約）:

```
(LLM OR "large language model") (release OR launch OR "new model") lang:en -is:retweet min_faves:200
```
```
(GPT OR Claude OR Gemini OR Llama OR Mistral OR Grok) lang:en -is:retweet min_faves:300
```
```
("AI agent" OR "agentic AI" OR RAG OR "prompt engineering" OR "fine-tuning") lang:en -is:retweet min_faves:150
```

### 2. フィルタリング基準（優先度順）

1. 新モデルリリース・性能ベンチマーク発表（最優先）
2. 査読あり論文・arXiv 新着で注目度が高いもの
3. 主要ツール・ライブラリの新バージョン
4. 有力研究者・企業アカウントによる技術的知見
5. コミュニティで広く議論されているトレンド（いいね 500 以上）

**除外**: 宣伝・煽り・既報の繰り返し・重複（複数クエリに出た同一ニュースは 1 件に統合）

### 3. 要約して DM に投稿

`aachat_send_message` で `dm:phase1demo.p-take55` に以下のフォーマットで投稿:

```
## AI Daily Briefing — {YYYY-MM-DD}

### 注目

1. **{タイトル}**
   {2〜3 行の要約。英語ソースも日本語で}
   → {URL または @account}

2. ...

---
*{掲載件数} 件 / 収集 {検索ヒット件数} 件*
```

掲載件数の目安: 5〜8 件。それ以上は重要度で絞る。

## 注意事項

- x-search は 1 クエリごとに課金が発生する。クエリ数・結果数は必要最小限に
- 同じニュースが複数クエリにヒットした場合は 1 件にまとめる
- 英語の投稿も要約は日本語で書く
- 「今日初めて出た情報」を優先し、昨日以前から既知の情報は省略してよい
