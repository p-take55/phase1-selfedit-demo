---
name: pr-review
description: >
  TypeScript web プロジェクトの PR をレビューする。
  「PR レビューして」「review this PR」「#123 を見て」「差分を確認して」
  「この PR どう思う」などで起動する。
  対象は TypeScript / TSX を含む web プロジェクト（React, Next.js, Node.js 等）。
---

# PR Review（TypeScript Web）

## 実行手順

### 1. PR / 差分を特定する

引数に PR 番号があれば `gh pr diff <num>` を使う。
なければ現在の branch と main の差分を使う:

```bash
# PR 番号指定
gh pr view <num> --json title,body,files
gh pr diff <num>

# branch 差分
git diff main...HEAD --name-only   # 変更ファイル一覧
git diff main...HEAD               # 差分全文
```

### 2. テストファイルを先に読む

変更ファイルから以下のパターンに一致するものを優先して読む:

```
*.test.ts   *.test.tsx
*.spec.ts   *.spec.tsx
tests/      __tests__/
e2e/        cypress/   playwright/
```

テストを先に読む理由: 実装コードを読む前に「何が期待されているか」を把握できる。

### 3. 実装ファイルを読む

テストと対応する実装ファイルを読む。優先順:

1. ビジネスロジック (hooks, services, utils, lib/)
2. API ルート / server-side コード (pages/api/, app/api/, routes/)
3. UI コンポーネント (components/, pages/, app/)
4. 設定ファイル (tsconfig, package.json 等)

### 4. TypeScript Web 観点でチェックする

**型安全性**
- `any` の使用 → 型が書けるはずの箇所は指摘
- non-null assertion (`!`) の乱用
- 型引数が省略されて推論が崩れている箇所
- 外部入力 (req.body, query params) に型がついているか

**テスト**
- 変更された実装に対応するテストが追加・更新されているか
- エッジケースとエラーケースが網羅されているか
- テストが実装の内部ではなく振る舞いをテストしているか

**React / Web フロント**
- hooks のルール違反（条件分岐内での呼び出し等）
- key prop の欠落または index キー
- 不必要な `useEffect` や依存配列の抜け
- `useMemo` / `useCallback` の過剰使用・不足
- アクセシビリティ (alt, aria-*, role)

**API / サーバーサイド**
- 入力バリデーションの有無（zod, valibot, yup 等）
- 認証・認可チェックの漏れ
- エラーハンドリングと適切な HTTP ステータスコード
- 機密情報のレスポンス漏れ

**パフォーマンス**
- N+1 クエリ相当の処理
- 無限ループの可能性（useEffect の依存配列）
- 重い処理の同期実行

### 5. 構造化レビューを出力する

```markdown
## PR レビュー: <タイトル or ブランチ名>

### サマリー
<1-2 行で全体感>

### Blocker（マージ前に直すべき）
- [ ] ...

### Minor（できれば直したい）
- [ ] ...

### 良い点
- ...

### テストについて
<テストの網羅度・質への所感>
```

ブロッカーがなければ「LGTM」と明記する。

## やってはいけない

- テストファイルより先に実装ファイルを読む
- 変更と無関係なファイルまで指摘の対象にする
- 「こうした方が良い」だけで理由を書かない
- スタイルの好みを Blocker に分類する
