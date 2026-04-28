# discord-weekly-briefing

毎週月曜朝に、参加している Discord サーバーの過去 7 日間の動きをまとめてオーナーの DM に投稿するスキル。

## 発動条件

- 毎週月曜 8:00 JST の定期実行（他のウィークリーブリーフィングと同じタイミング。**月曜のみ実行**、それ以外の日は何もしない）
- オーナーから「Discord まとめて」「サーバーの動き教えて」「今週の Discord は？」等の依頼

## 前提環境変数

以下が設定されていることを確認する。未設定なら DM に「DISCORD_BOT_TOKEN が未設定のため Discord ブリーフィングをスキップしました」と通知して終了する。

- `DISCORD_BOT_TOKEN` — Discord Bot Token（必要 Intent: `GUILD_MESSAGES`, `MESSAGE_CONTENT`, `GUILDS`）

## セットアップ（初回のみ）

1. https://discord.com/developers/applications でアプリを作成
2. Bot タブ → Token をコピー → `.zprofile` に `export DISCORD_BOT_TOKEN=...` を追加
3. Bot タブ → Privileged Gateway Intents → **Message Content Intent** を ON にする
4. OAuth2 → URL Generator → Scopes: `bot` → Bot Permissions: `Read Message History`, `View Channels` → 生成 URL で各サーバーに招待

## 手順

### 1. 参加サーバー一覧を取得

```bash
GUILDS=$(curl -s \
  -H "Authorization: Bot ${DISCORD_BOT_TOKEN}" \
  "https://discord.com/api/v10/users/@me/guilds")

GUILD_COUNT=$(echo "$GUILDS" | jq 'length')
```

取得失敗（HTTP 401）なら「Discord Bot Token が無効です」と DM して終了する。

### 2. 対象期間を設定

```bash
# macOS
WEEK_AGO_EPOCH=$(date -v-7d +%s)
WEEK_AGO_DATE=$(date -v-7d +%Y-%m-%d)
TODAY=$(date +%Y-%m-%d)

# Discord Snowflake: epoch ms → snowflake (左シフト 22 bit)
# 2015-01-01 を Discord epoch とする
DISCORD_EPOCH=1420070400000
WEEK_AGO_MS=$(( WEEK_AGO_EPOCH * 1000 ))
AFTER_SNOWFLAKE=$(( (WEEK_AGO_MS - DISCORD_EPOCH) << 22 ))
```

### 3. サーバーごとにチャンネルのメッセージを収集

各サーバー（最大 10 件）のテキストチャンネルを確認し、過去 7 日間のメッセージ数を集計する:

```bash
SUMMARY=""

for GUILD_ID in $(echo "$GUILDS" | jq -r '.[0:10] | .[].id'); do
  GUILD_NAME=$(echo "$GUILDS" | jq -r --arg id "$GUILD_ID" '.[] | select(.id == $id) | .name')

  # チャンネル一覧（テキストチャンネルのみ: type=0）
  CHANNELS=$(curl -s \
    -H "Authorization: Bot ${DISCORD_BOT_TOKEN}" \
    "https://discord.com/api/v10/guilds/${GUILD_ID}/channels" \
    | jq '[.[] | select(.type == 0)]')

  GUILD_MSG_COUNT=0
  ACTIVE_CHANNELS=""

  for CHANNEL_ID in $(echo "$CHANNELS" | jq -r '.[0:5] | .[].id'); do
    CHANNEL_NAME=$(echo "$CHANNELS" | jq -r --arg id "$CHANNEL_ID" '.[] | select(.id == $id) | .name')

    # 過去 7 日間のメッセージ取得（after スノーフレーク使用）
    MESSAGES=$(curl -s \
      -H "Authorization: Bot ${DISCORD_BOT_TOKEN}" \
      "https://discord.com/api/v10/channels/${CHANNEL_ID}/messages?after=${AFTER_SNOWFLAKE}&limit=100")

    MSG_COUNT=$(echo "$MESSAGES" | jq 'if type == "array" then length else 0 end')

    if [ "$MSG_COUNT" -gt 0 ]; then
      GUILD_MSG_COUNT=$(( GUILD_MSG_COUNT + MSG_COUNT ))
      ACTIVE_CHANNELS="${ACTIVE_CHANNELS}\n  - #${CHANNEL_NAME}: ${MSG_COUNT} 件"
    fi
  done

  if [ "$GUILD_MSG_COUNT" -gt 0 ]; then
    SUMMARY="${SUMMARY}\n### ${GUILD_NAME}\n- 合計 ${GUILD_MSG_COUNT} メッセージ${ACTIVE_CHANNELS}"
  fi
done
```

### 4. メンション（@mention）を抽出

オーナーの Discord ユーザー ID が `DISCORD_USER_ID` 環境変数に設定されている場合、各チャンネルのメッセージからメンションを抽出してハイライトする:

```bash
if [ -n "${DISCORD_USER_ID}" ]; then
  # mention_everyone または自分への mention を検索
  MENTIONS=$(echo "$MESSAGES" | jq -r --arg uid "$DISCORD_USER_ID" '
    .[] | select(
      (.mention_everyone == true) or
      (.mentions | map(.id) | any(. == $uid))
    ) |
    "- \(.author.username): \(.content[:80]) (\(.timestamp[:10]))"
  ')
fi
```

### 5. 先週との比較（memory/ を使う）

`memory/discord-last-week.md` が存在すれば読み込み、先週と比較:
- 新たに活発になったサーバー・チャンネル
- メッセージ数の増減

集計後、今週のデータで `memory/discord-last-week.md` を上書き保存（次週の比較用）:

```markdown
---
date: {YYYY-MM-DD}
---
## サーバー別メッセージ数
- {server_name}: {count} 件
  - #{channel}: {count} 件
...
```

### 6. 要約して DM に投稿

全サーバーでメッセージ 0 件の場合は「今週の Discord に目立った動きはありませんでした」と投稿して終了する。

`aachat_send_message` で `dm:phase1demo.p-take55` に投稿:

```
## Discord Weekly Briefing — {YYYY-MM-DD} 週

### サーバー別ハイライト
#### {server_name}
- 合計 {n} メッセージ
- 活発なチャンネル: #{channel1}（{n}件）, #{channel2}（{n}件）

（メンションがある場合）
### あなたへのメンション
- {username}: {message_excerpt} ({date})

### 先週からの変化
- 増加: {server/channel}（+{n}件）
- 減少: {server/channel}（{−n}件）
（先週データなし → 初回のため比較なし）

---
*{guild_count} サーバー / 過去 7 日間を集計*
```

## 注意事項

- Discord Bot API のレート制限: 50 req/s グローバル、チャンネルごとに 5 req/5s。サーバー数・チャンネル数が多い場合は `sleep 1` を挟む
- `MESSAGE_CONTENT` Intent が OFF だとメッセージ本文が空になる。その場合は件数のみ報告する
- 過去 7 日以前のメッセージは Snowflake after で自動的に除外される（Discord 側でフィルタリング）
- プライベートチャンネルはボットが参加していなければ取得不可。取得エラー（HTTP 403）は無視して続行する
- 環境変数は repo に書かず、ローカルの `.zprofile` / `.env` で管理する
- HTTP 429 はレート制限超過。`retry_after` 秒待ってリトライするか、DM に「Discord API レート制限に達したため一部データが欠損している可能性があります」と追記する
