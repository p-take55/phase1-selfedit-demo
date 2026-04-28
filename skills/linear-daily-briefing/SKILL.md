# linear-daily-briefing

毎朝 Linear から自分担当チケットを取得し、ブロッカーになりそうなものをオーナーの DM に投稿するスキル。

## 発動条件

- 毎朝 8:00 JST の定期実行
- オーナーから「Linear 確認して」「ブロッカーある？」「チケットの状況は？」等の依頼

## 前提

- 環境変数 `LINEAR_API_KEY` が設定されていること（リポジトリには置かない）
- Linear の API エンドポイント: `https://api.linear.app/graphql`

## 手順

### 1. 自分担当の未完了チケットを取得

以下の GraphQL クエリを `WebFetch` で POST して取得する:

```graphql
query MyIssues {
  viewer {
    id
    name
    assignedIssues(
      filter: {
        state: { type: { nin: ["completed", "cancelled"] } }
      }
      first: 50
    ) {
      nodes {
        id
        identifier
        title
        priority
        priorityLabel
        state {
          name
          type
        }
        dueDate
        updatedAt
        url
        relations {
          nodes {
            type
            relatedIssue {
              identifier
              title
              state { type }
            }
          }
        }
        labels {
          nodes { name }
        }
      }
    }
  }
}
```

リクエスト例（WebFetch で実行）:
- URL: `https://api.linear.app/graphql`
- Method: POST
- Headers: `Authorization: Bearer $LINEAR_API_KEY`, `Content-Type: application/json`
- Body: `{"query": "<上記クエリ>"}`

### 2. ブロッカー判定基準（優先度順）

以下のいずれかに該当するチケットを「ブロッカー候補」として報告する:

| 判定条件 | 重要度 |
|---|---|
| 状態が "Blocked" または labels に "blocked" を含む | 🔴 緊急 |
| 期日 (dueDate) が今日以前で未完了 | 🔴 緊急 |
| 期日が今後 2 日以内かつ priority が Urgent/High | 🟡 要注意 |
| relations に "blocks" 型があり、関連先が進行中 | 🟡 要注意 |
| priority が Urgent で 3 日以上更新なし | 🟡 要注意 |
| priority が High で 7 日以上更新なし | 🔵 確認推奨 |

該当なしの場合は「今日の担当チケットに緊急のブロッカーはありません」と報告する。

### 3. 要約して DM に投稿

`aachat_send_message` で `dm:phase1demo.p-take55` に以下のフォーマットで投稿:

```
## Linear Daily Briefing — {YYYY-MM-DD}

### 🔴 緊急（今すぐ対応）

- **{IDENTIFIER}** [{state}] {title}
  理由: {判定条件}
  → {url}

### 🟡 要注意（今日中に確認）

- **{IDENTIFIER}** [{state}] {title}
  理由: {判定条件}
  → {url}

### 🔵 確認推奨

- **{IDENTIFIER}** [{state}] {title}
  理由: {判定条件}
  → {url}

---
*担当 {総件数} 件中 {ブロッカー件数} 件を報告*
```

ブロッカーが 0 件の場合:

```
## Linear Daily Briefing — {YYYY-MM-DD}

担当チケット {総件数} 件を確認。今日の緊急ブロッカーはありません。
```

## 注意事項

- `LINEAR_API_KEY` は環境変数から読む。ログや投稿に含めない
- API 呼び出しは 1 回のみ（課金・レート制限の節約）
- チケットのタイトルや本文は要約せず、identifier と title をそのまま使う
- "blocks" 型 relation は「自分のチケットが他を止めている」状態なので特に重要視する
