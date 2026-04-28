# strava-weekly-briefing

毎週月曜朝に、過去 7 日間の Strava アクティビティを収集し、運動量の傾向をまとめてオーナーの DM に投稿するスキル。

## 発動条件

- 毎週月曜 8:00 JST の定期実行（他のウィークリーブリーフィングと同じタイミング。**月曜のみ実行**、それ以外の日は何もしない）
- オーナーから「Strava まとめて」「今週の運動は？」「アクティビティ教えて」等の依頼

## 前提環境変数

以下が設定されていることを確認する。未設定なら DM に「Strava 環境変数が未設定のため実行をスキップしました」と通知して終了する。

- `STRAVA_CLIENT_ID` — Strava アプリのクライアント ID
- `STRAVA_CLIENT_SECRET` — Strava アプリのクライアントシークレット
- `STRAVA_REFRESH_TOKEN` — Authorization Code フローで取得したリフレッシュトークン（スコープ: `activity:read_all`）

## 手順

### 1. アクセストークンを取得

リフレッシュトークンを使って新しいアクセストークンを発行する:

```bash
TOKEN_RESPONSE=$(curl -s -X POST "https://www.strava.com/oauth/token" \
  -d "client_id=${STRAVA_CLIENT_ID}" \
  -d "client_secret=${STRAVA_CLIENT_SECRET}" \
  -d "grant_type=refresh_token" \
  -d "refresh_token=${STRAVA_REFRESH_TOKEN}")

ACCESS_TOKEN=$(echo "$TOKEN_RESPONSE" | jq -r '.access_token')
```

取得失敗（`null` または空）なら DM に「Strava アクセストークン取得失敗」と通知して終了する。

### 2. 過去 7 日間のアクティビティを取得

```bash
# Unix タイムスタンプ: 7 日前（macOS）
AFTER=$(date -v-7d +%s)
# AFTER=$(date -d '7 days ago' +%s)  # Linux

ACTIVITIES=$(curl -s \
  "https://www.strava.com/api/v3/athlete/activities?after=${AFTER}&per_page=100" \
  -H "Authorization: Bearer ${ACCESS_TOKEN}")
```

### 3. データを集計・分析

```bash
# 件数
TOTAL_COUNT=$(echo "$ACTIVITIES" | jq 'length')

# 種別ごとの件数と距離（m → km）
echo "$ACTIVITIES" | jq -r '
  group_by(.sport_type) |
  map({
    type: .[0].sport_type,
    count: length,
    distance_km: (map(.distance) | add) / 1000 | round,
    moving_time_min: (map(.moving_time) | add) / 60 | round,
    elevation_m: (map(.total_elevation_gain) | add) | round
  }) |
  sort_by(-.distance_km) |
  .[] |
  "\(.type): \(.count)回, \(.distance_km)km, \(.moving_time_min)分, 獲得標高\(.elevation_m)m"
'

# 週合計
TOTAL_DISTANCE=$(echo "$ACTIVITIES" | jq '[.[].distance] | add // 0 | . / 1000 | round')
TOTAL_TIME=$(echo "$ACTIVITIES" | jq '[.[].moving_time] | add // 0 | . / 60 | round')
TOTAL_ELEVATION=$(echo "$ACTIVITIES" | jq '[.[].total_elevation_gain] | add // 0 | round')
TOTAL_CALORIES=$(echo "$ACTIVITIES" | jq '[.[].calories // 0] | add | round')

# 最長アクティビティ
LONGEST=$(echo "$ACTIVITIES" | jq -r 'sort_by(-.distance) | .[0] | "\(.name) (\(.sport_type)) \(.distance / 1000 | round)km \(.start_date_local[:10])"')

# 高強度アクティビティ（suffer_score が高いもの）
HARDEST=$(echo "$ACTIVITIES" | jq -r 'sort_by(-.suffer_score) | .[0] | "\(.name) (\(.sport_type)) suffer:\(.suffer_score // "N/A") \(.start_date_local[:10])"')
```

### 4. 先週との比較（memory/ を使う）

`memory/strava-last-week.md` が存在すれば読み込み、先週のデータと比較して変化を検出する:
- 総距離・総時間の増減
- 新しく登場したスポーツ種別
- アクティビティ回数の変化

集計後、今週のデータを `memory/strava-last-week.md` に上書き保存（次週の比較用）:

```markdown
---
date: {YYYY-MM-DD}
---
## 週合計
- 件数: {n}回
- 距離: {n}km
- 時間: {n}分
- 獲得標高: {n}m
- カロリー: {n}kcal

## 種別内訳
- {type}: {count}回, {distance}km
...
```

### 5. 要約して DM に投稿

アクティビティが 0 件の場合は「今週の Strava アクティビティはありませんでした」と投稿して終了する。

`aachat_send_message` で `dm:phase1demo.p-take55` に投稿:

```
## Strava Weekly Briefing — {YYYY-MM-DD} 週

### 週合計
- アクティビティ: {n}回
- 総距離: {n}km
- 総時間: {n}時間{m}分
- 獲得標高: {n}m
- 消費カロリー: {n}kcal（記録あり分のみ）

### 種別内訳
- {type}: {count}回, {distance}km, {time}分
（距離順）

### ハイライト
- 最長: {name} ({sport}) {distance}km ({date})
- 最高強度: {name} ({sport}) suffer:{score} ({date})

### 先週からの変化
- 距離: {±n}km（先週比）
- 時間: {±n}分（先週比）
- アクティビティ数: {±n}回
（先週データなし → 初回のため比較なし）

---
*Strava API（過去 7 日間）を元に集計*
```

## 注意事項

- Strava API のレート制限: 15 分あたり 100 リクエスト / 1 日あたり 1,000 リクエスト。週次ブリーフィングは数リクエストで完結するため問題ない
- `per_page=100` で取得するため、週 100 件を超えるアクティビティがある場合はページネーションが必要だが、通常は不要
- `calories` フィールドは心拍計付きデバイスや特定のスポーツでのみ記録される。`null` の場合は集計から除外し「記録あり分のみ」と注記する
- `suffer_score` も同様にデバイス依存。`null` の場合はハイライトから除外する
- HTTP 401 はアクセストークン失効。手順 1 からやり直す
- HTTP 429 はレート制限超過。DM に「Strava API レート制限に達したためスキップしました」と通知して終了する
- 環境変数は repo に書かず、ローカルの `.zprofile` / `.env` で管理する
- STRAVA_REFRESH_TOKEN は Strava がローテーションする場合がある。`TOKEN_RESPONSE` の `refresh_token` フィールドが現在のものと異なる場合は `.zprofile` の値を更新する必要がある
