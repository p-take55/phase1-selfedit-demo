# github-weekly-briefing

毎週月曜朝に、GitHub で watching している repo の最新リリースと重要 Issue をまとめてオーナーの DM に投稿するスキル。

## 発動条件

- 毎週月曜 8:00 JST の定期実行（Spotify ブリーフィングと同じタイミング。**月曜のみ実行**、それ以外の日は何もしない）
- オーナーから「GitHub まとめて」「watching repo の動き教えて」「今週のリリース教えて」等の依頼

## 前提環境変数

以下が設定されていることを確認する。未設定なら DM に「GITHUB_TOKEN が未設定のため GitHub ブリーフィングをスキップしました」と通知して終了する。

- `GITHUB_TOKEN` — GitHub Personal Access Token（スコープ: `repo`, `notifications`）

## 手順

### 1. 対象期間を設定

```bash
# macOS
WEEK_AGO=$(date -v-7d -u +%Y-%m-%dT%H:%M:%SZ)
WEEK_AGO_DATE=$(date -v-7d +%Y-%m-%d)
TODAY=$(date +%Y-%m-%d)
```

### 2. Watching repo のリストを取得

最大 300 件（3 ページ）取得し、過去 7 日間に push があった repo に絞る:

```bash
WATCHING_REPOS=""
for PAGE in 1 2 3; do
  PAGE_REPOS=$(curl -s -H "Authorization: token ${GITHUB_TOKEN}" \
    "https://api.github.com/user/subscriptions?per_page=100&page=${PAGE}" \
    | jq -r --arg since "$WEEK_AGO" \
      '.[] | select(.pushed_at >= $since) | .full_name')
  [ -z "$PAGE_REPOS" ] && break
  WATCHING_REPOS="${WATCHING_REPOS} ${PAGE_REPOS}"
done

# 上位 30 件に絞る（API 呼び出し数を抑制）
ACTIVE_REPOS=$(echo "$WATCHING_REPOS" | tr ' ' '\n' | grep -v '^$' | head -30)
REPO_TOTAL=$(echo "$WATCHING_REPOS" | tr ' ' '\n' | grep -v '^$' | wc -l | tr -d ' ')
```

### 3. 新着リリースを収集

各 repo の直近リリースを確認し、過去 7 日間のものを抽出:

```bash
RELEASES=""
for REPO in $ACTIVE_REPOS; do
  LATEST=$(curl -s -H "Authorization: token ${GITHUB_TOKEN}" \
    "https://api.github.com/repos/${REPO}/releases?per_page=5" \
    | jq -r --arg since "$WEEK_AGO" \
      '.[] | select(.published_at >= $since and (.draft == false)) |
       "- **\(.repository.full_name // "'"$REPO"'")** \(.tag_name) — \(.name) (\(.published_at[:10])) \(.html_url)"' 2>/dev/null)
  # full_name が releases に含まれない場合に repo 名を補完
  if [ -n "$LATEST" ]; then
    RELEASES="${RELEASES}
$(echo "$LATEST" | sed "s|full_name //|$REPO //|g")"
  fi
done
```

リリース情報が取れない repo は無視する（プライベート repo や権限外）。

### 4. 重要 Issue を収集

Watching repo の Issue から重要度の高いものを抽出する。2 つのアプローチを組み合わせる:

**4a. ラベル付き Issue（バグ・セキュリティ・重大変更）**

```bash
IMPORTANT_ISSUES=$(curl -s -H "Authorization: token ${GITHUB_TOKEN}" \
  "https://api.github.com/issues?filter=subscribed&state=open&sort=updated&since=${WEEK_AGO}&per_page=100" \
  | jq -r '
    .[] |
    select(
      .labels | map(.name | ascii_downcase) |
      any(test("bug|critical|security|breaking|urgent|blocker|p0|p1|high.priority"))
    ) |
    "- **\(.repository.full_name // "unknown")#\(.number)** \(.title) [\(.labels | map(.name) | join(", "))] 👍\(.reactions["+1"] // 0) (\(.updated_at[:10])) \(.html_url)"
  ')
```

**4b. 高リアクション Issue（ラベル問わず注目度が高いもの）**

```bash
HIGH_REACTION_ISSUES=$(curl -s -H "Authorization: token ${GITHUB_TOKEN}" \
  "https://api.github.com/issues?filter=subscribed&state=open&sort=updated&since=${WEEK_AGO}&per_page=100" \
  | jq -r '
    .[] |
    select((.reactions["+1"] // 0) + (.reactions["heart"] // 0) + (.reactions["rocket"] // 0) >= 10) |
    "- **\(.repository.full_name // "unknown")#\(.number)** \(.title) 👍\(.reactions["+1"] // 0)❤️\(.reactions["heart"] // 0) (\(.updated_at[:10])) \(.html_url)"
  ')
```

重複 Issue が両方に含まれる場合は「重要ラベル」側を優先し、高リアクション側は除去する。

### 5. 先週との比較（memory/ を使う）

`memory/github-last-week.md` が存在すれば読み込み、先週のリリース・Issue と比較:
- 新たにリリースされた repo
- 先週から引き続き注目の Issue（継続注目）

集計後、今週のデータで `memory/github-last-week.md` を上書き保存（次週の比較用）:

```markdown
---
date: {YYYY-MM-DD}
---
## リリース
- {repo}: {tag}
...

## 重要 Issue
- {repo}#{number}: {title}
...
```

### 6. 要約して DM に投稿

リリース・Issue がともに 0 件の場合は「今週の watching repo に目立った動きはありませんでした」と投稿して終了する。

`aachat_send_message` で `dm:phase1demo.p-take55` に投稿:

```
## GitHub Weekly Briefing — {YYYY-MM-DD} 週

### 新着リリース
- **{owner/repo}** {tag} — {release name} ({date})
  {url}
（なければ「今週のリリースなし」）

### 重要 Issue
#### バグ・セキュリティ・重大変更
- **{owner/repo}#{number}** {title} [{labels}] 👍{reactions} ({date})
  {url}

#### 注目 Issue（高リアクション）
- **{owner/repo}#{number}** {title} 👍{+1}❤️{heart} ({date})
  {url}
（なければ省略）

### 先週からの変化
- 新着: {repo/issue}
- 継続注目: {repo/issue}
（先週データなし → 初回のため比較なし）

---
*watching {repo_total} repos → 過去 7 日間に push あり {active_count} repos を確認*
```

## 注意事項

- GitHub REST API の認証済みレート制限は 5,000 req/hr。1 回の実行で最大 ~35 リクエスト（30 repo 分のリリース + 2 回の Issues 取得）に収まるよう設計している
- `per_page=100` の Issues クエリで 100 件を超える場合はページネーションせず上位 100 件のみ処理する（週次ブリーフィングとして十分）
- プライベート repo や権限外 repo の API 呼び出しはエラーを無視して続行する
- 環境変数 `GITHUB_TOKEN` は repo に書かず、ローカルの `.zprofile` / `.env` で管理する
- HTTP 403 / 429 が返ったらレート制限超過。DM に「GitHub API レート制限に達したため一部データが欠損している可能性があります」と追記する
