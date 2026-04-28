# spotify-weekly-briefing

毎週月曜朝に、過去 7 日間の Spotify リスニングデータを収集し、曲の傾向をまとめてオーナーの DM に投稿するスキル。

## 発動条件

- 毎週月曜 8:00 JST の定期実行（他のブリーフィングと同じタイミング。**月曜のみ実行**、それ以外の日は何もしない）
- オーナーから「Spotify まとめて」「今週何聴いてた？」「リスニング傾向は？」等の依頼

## 前提環境変数

以下が設定されていることを確認する。未設定なら DM に「Spotify 環境変数が未設定のため実行をスキップしました」と通知して終了する。

- `SPOTIFY_CLIENT_ID` — Spotify アプリのクライアント ID
- `SPOTIFY_CLIENT_SECRET` — Spotify アプリのクライアントシークレット
- `SPOTIFY_REFRESH_TOKEN` — Authorization Code フローで取得したリフレッシュトークン（スコープ: `user-read-recently-played user-top-read`）

## 手順

### 1. アクセストークンを取得

リフレッシュトークンを使って新しいアクセストークンを発行する:

```bash
ACCESS_TOKEN=$(curl -s -X POST "https://accounts.spotify.com/api/token" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "grant_type=refresh_token&refresh_token=${SPOTIFY_REFRESH_TOKEN}&client_id=${SPOTIFY_CLIENT_ID}&client_secret=${SPOTIFY_CLIENT_SECRET}" \
  | jq -r '.access_token')
```

取得失敗（`null` または空）なら DM に「Spotify アクセストークン取得失敗」と通知して終了する。

### 2. トップトラック・アーティストを取得（過去 4 週間）

`short_term`（≈ 過去 4 週間）のデータが最も「今週の傾向」に近い:

```bash
# トップトラック TOP 50
TOP_TRACKS=$(curl -s "https://api.spotify.com/v1/me/top/tracks?time_range=short_term&limit=50" \
  -H "Authorization: Bearer ${ACCESS_TOKEN}")

# トップアーティスト TOP 50
TOP_ARTISTS=$(curl -s "https://api.spotify.com/v1/me/top/artists?time_range=short_term&limit=50" \
  -H "Authorization: Bearer ${ACCESS_TOKEN}")
```

### 3. 最近の再生履歴を取得（過去 7 日間）

`recently-played` は最大 50 件まで取得可能。1 週間分をカバーするため `before` カーソルで複数回ページネーションする:

```bash
# Unix ミリ秒タイムスタンプ: 7 日前
BEFORE=$(date -v-7d +%s)000   # macOS
# BEFORE=$(date -d '7 days ago' +%s)000  # Linux

RECENT=$(curl -s "https://api.spotify.com/v1/me/player/recently-played?limit=50&before=${BEFORE}" \
  -H "Authorization: Bearer ${ACCESS_TOKEN}")
# さらに古いページが必要なら .cursors.before を次の before に使って繰り返す
```

### 4. データを集計・分析

取得した JSON を jq で集計する:

```bash
# トップトラック TOP 5（名前 / アーティスト名）
echo "$TOP_TRACKS" | jq -r '.items[:5] | .[] | "\(.name) — \(.artists[0].name)"'

# トップアーティスト TOP 5（名前 / ジャンル）
echo "$TOP_ARTISTS" | jq -r '.items[:5] | .[] | "\(.name) [\(.genres[:2] | join(", "))]"'

# ジャンル傾向（上位アーティストのジャンルを集計）
echo "$TOP_ARTISTS" | jq -r '[.items[].genres[]] | group_by(.) | map({genre: .[0], count: length}) | sort_by(-.count) | .[:5] | .[] | "\(.genre): \(.count)"'
```

### 5. 先週との比較（memory/ を使う）

`memory/spotify-last-week.md` が存在すれば読み込み、先週の TOP 5 と比較して変化を検出する:
- 新しくランクインしたアーティスト・トラック
- ランクアウトしたアーティスト・トラック

集計後、今週のデータを `memory/spotify-last-week.md` に上書き保存する（次週の比較用）:

```markdown
---
date: {YYYY-MM-DD}
---
## TOP アーティスト
1. {name}
2. ...

## TOP トラック
1. {name} — {artist}
2. ...
```

### 6. 要約して DM に投稿

`aachat_send_message` で `dm:phase1demo.p-take55` に投稿:

```
## Spotify Weekly Briefing — {YYYY-MM-DD} 週

### よく聴いたアーティスト TOP 5
1. {artist} [{genres}]
2. ...

### よく聴いたトラック TOP 5
1. {track} — {artist}
2. ...

### ジャンル傾向
- {genre}: 高め / 中 / 低め（先週比）

### 先週からの変化
- 新登場: {artist / track}
- 圏外: {artist / track}
（先週データなし → 初回のため比較なし）

---
*Spotify short_term データ（過去 ≈ 4 週間）を元に集計*
```

## 注意事項

- `recently-played` は最大 50 件・30 日以内の制約がある。細かい再生回数カウントは取れないため、トップ判定は `/me/top/` エンドポイントに頼る
- rate limit（HTTP 429）が返ってきたら `Retry-After` ヘッダの秒数待ってリトライする
- アクセストークンは有効期限 3,600 秒。1 セッション内で使い回してよい（再発行しない）
- 環境変数は repo に書かず、ローカルの `.zprofile` / `.env` で管理する
- Premium サブスクリプションが必要（Spotify Developer の Development Mode は 2026 年 2 月以降 Premium 必須）
