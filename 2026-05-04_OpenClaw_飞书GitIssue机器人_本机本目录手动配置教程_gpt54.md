# 2026-05-04 OpenClaw 飞书 GitIssue 机器人本机本目录手动配置教程（`gpt-5.4`）

> 目标：在**本机**目录 `/root/Codex/Law` 启动一个独立 OpenClaw 飞书机器人实例。  
> 行为目标：机器人能读群消息，但只在被 `@` 时回复。  
> 你已准备：`/root/Codex/Law/Law.md`（3 行：`appId`、`appSecret`、`chatId`）。

---

## 0. 相对 2026-04-29 原文的最小改动（必须改）

1. 删除“SSH 到远端机器”步骤，全部改为本机执行。  
2. `BASE_DIR` 固定改为 `/root/Codex/Law`。  
3. 凭据文件从 `GitIssue.md` 改为 `Law.md`。  
4. `start` 脚本等待启动时间从 `10s` 改为 `60s`。  
   - 本机实测 OpenClaw `2026.4.25` 首次启动飞书实例需要约 `17~25s` 才到 `ready`，10 秒会误报失败。  
5. 端口就绪检测改为同时兼容 `127.0.0.1` 和 `[::1]`。

其余流程保持原思路，不改核心逻辑。

---

## 1. 先做 30 秒体检（整段复制）

```bash
set -euo pipefail
cd /root/Codex/Law

pwd
ls -l Law.md
sed -n '1,3p' Law.md

openclaw --version
node -v
openssl version
```

你应该看到：

1. 当前目录是 `/root/Codex/Law`
2. `Law.md` 存在且正好 3 行
3. `openclaw` 可执行

---

## 2. 设置变量并读取 `Law.md`（整段复制）

```bash
set -euo pipefail

BASE_DIR=/root/Codex/Law
INSTANCE_DIR="$BASE_DIR/.openclaw-feishu-gitissue-gpt54"
CONFIG_PATH="$INSTANCE_DIR/openclaw.json"
STATE_DIR="$INSTANCE_DIR/state"
WORKSPACE_DIR="$INSTANCE_DIR/workspace-larkbot"
LOG_PATH="$INSTANCE_DIR/gateway.out"
PID_FILE="$INSTANCE_DIR/openclaw.pid"
PORT=18890

mkdir -p "$INSTANCE_DIR" "$STATE_DIR" "$WORKSPACE_DIR"

readarray -t FEISHU_LINES < "$BASE_DIR/Law.md"
FEISHU_APP_ID="$(echo "${FEISHU_LINES[0]:-}" | tr -d '\r')"
FEISHU_APP_SECRET="$(echo "${FEISHU_LINES[1]:-}" | tr -d '\r')"
FEISHU_CHAT_ID="$(echo "${FEISHU_LINES[2]:-}" | tr -d '\r')"

if [ -z "$FEISHU_APP_ID" ] || [ -z "$FEISHU_APP_SECRET" ] || [ -z "$FEISHU_CHAT_ID" ]; then
  echo "Law.md 三行内容不完整，停止。"
  exit 1
fi

read -rsp "粘贴 Router API Key（输入不显示）: " ROUTER_API_KEY
echo
if [ -z "$ROUTER_API_KEY" ]; then
  echo "Router key 为空，停止。"
  exit 1
fi

GATEWAY_TOKEN="$(openssl rand -hex 24)"

echo "FEISHU_APP_ID=$FEISHU_APP_ID"
echo "FEISHU_CHAT_ID=$FEISHU_CHAT_ID"
echo "PORT=$PORT"
```

---

## 3. 生成 `openclaw.json`（整段复制）

```bash
cat > "$CONFIG_PATH" <<EOF
{
  "meta": {
    "lastTouchedVersion": "2026.4.25",
    "lastTouchedAt": "$(date -u +%Y-%m-%dT%H:%M:%SZ)"
  },
  "models": {
    "providers": {
      "router": {
        "baseUrl": "https://router.yeying.pub/v1",
        "apiKey": "$ROUTER_API_KEY",
        "auth": "api-key",
        "api": "openai-responses",
        "models": [
          {
            "id": "gpt-5.4",
            "name": "GPT-5.4",
            "api": "openai-responses",
            "reasoning": false,
            "input": ["text", "image"]
          }
        ]
      }
    }
  },
  "agents": {
    "defaults": {
      "model": {
        "primary": "router/gpt-5.4"
      },
      "workspace": "$WORKSPACE_DIR",
      "compaction": {
        "mode": "safeguard"
      }
    }
  },
  "messages": {
    "groupChat": {
      "mentionPatterns": [
        ".*"
      ]
    }
  },
  "commands": {
    "native": "auto",
    "nativeSkills": "auto",
    "restart": true,
    "ownerDisplay": "raw"
  },
  "channels": {
    "feishu": {
      "enabled": true,
      "connectionMode": "websocket",
      "appId": "$FEISHU_APP_ID",
      "appSecret": "$FEISHU_APP_SECRET",
      "groupPolicy": "allowlist",
      "groupAllowFrom": [
        "$FEISHU_CHAT_ID"
      ],
      "requireMention": true,
      "groups": {
        "$FEISHU_CHAT_ID": {
          "requireMention": true
        }
      },
      "mediaMaxMb": 50,
      "historyLimit": 20
    }
  },
  "gateway": {
    "port": $PORT,
    "mode": "local",
    "auth": {
      "mode": "token",
      "token": "$GATEWAY_TOKEN"
    }
  },
  "plugins": {
    "entries": {
      "feishu": {
        "enabled": true
      }
    }
  }
}
EOF

chmod 600 "$CONFIG_PATH"
```

---

## 4. 先做配置校验（整段复制）

```bash
OPENCLAW_CONFIG_PATH="$CONFIG_PATH" OPENCLAW_STATE_DIR="$STATE_DIR" openclaw config validate
```

看到 `Config valid` 再继续。

---

## 5. 生成启停脚本（整段复制）

```bash
cat > "$BASE_DIR/start_gitissue_gpt54.sh" <<'EOF'
#!/usr/bin/env bash
set -euo pipefail
BASE_DIR="/root/Codex/Law"
INSTANCE_DIR="$BASE_DIR/.openclaw-feishu-gitissue-gpt54"
CONFIG_PATH="$INSTANCE_DIR/openclaw.json"
STATE_DIR="$INSTANCE_DIR/state"
LOG_PATH="$INSTANCE_DIR/gateway.out"
PID_FILE="$INSTANCE_DIR/openclaw.pid"
PORT=18890

find_running_pids() {
  for p in $(pgrep -x openclaw 2>/dev/null || true); do
    if [ -r "/proc/$p/environ" ] && tr '\0' '\n' < "/proc/$p/environ" | grep -q "^OPENCLAW_CONFIG_PATH=$CONFIG_PATH$"; then
      echo "$p"
    fi
  done
}

pids="$(find_running_pids || true)"
if [ -n "$pids" ] && ss -lntp | grep -qE "127.0.0.1:$PORT|\[::1\]:$PORT"; then
  echo "already running: pids=$pids"
  exit 0
fi

if [ -n "$pids" ]; then
  for p in $pids; do kill "$p" 2>/dev/null || true; done
  sleep 1
fi

mkdir -p "$STATE_DIR"
nohup env OPENCLAW_CONFIG_PATH="$CONFIG_PATH" OPENCLAW_STATE_DIR="$STATE_DIR" \
  openclaw gateway run --port "$PORT" >> "$LOG_PATH" 2>&1 &

echo $! > "$PID_FILE"

for _ in $(seq 1 60); do
  sleep 1
  now="$(find_running_pids || true)"
  if [ -n "$now" ] && ss -lntp | grep -qE "127.0.0.1:$PORT|\[::1\]:$PORT"; then
    echo "started: pids=$now"
    exit 0
  fi
done

echo "start failed, tail log:"
tail -n 120 "$LOG_PATH" || true
exit 1
EOF

cat > "$BASE_DIR/stop_gitissue_gpt54.sh" <<'EOF'
#!/usr/bin/env bash
set -euo pipefail
BASE_DIR="/root/Codex/Law"
INSTANCE_DIR="$BASE_DIR/.openclaw-feishu-gitissue-gpt54"
CONFIG_PATH="$INSTANCE_DIR/openclaw.json"
PID_FILE="$INSTANCE_DIR/openclaw.pid"
PORT=18890

find_running_pids() {
  for p in $(pgrep -x openclaw 2>/dev/null || true); do
    if [ -r "/proc/$p/environ" ] && tr '\0' '\n' < "/proc/$p/environ" | grep -q "^OPENCLAW_CONFIG_PATH=$CONFIG_PATH$"; then
      echo "$p"
    fi
  done
}

pids="$(find_running_pids || true)"
if [ -z "$pids" ] && [ ! -f "$PID_FILE" ]; then
  echo "not running"
  exit 0
fi

for p in $pids; do kill "$p" 2>/dev/null || true; done
sleep 2
for p in $pids; do kill -9 "$p" 2>/dev/null || true; done

for gp in $(ss -lntp 2>/dev/null | awk -v p=":$PORT" '$4 ~ p && /openclaw-gatewa/ {gsub(/.*pid=/,"",$NF); gsub(/,.*/,"",$NF); print $NF}'); do
  kill "$gp" 2>/dev/null || true
  kill -9 "$gp" 2>/dev/null || true
done

rm -f "$PID_FILE"
echo "stopped"
EOF

cat > "$BASE_DIR/status_gitissue_gpt54.sh" <<'EOF'
#!/usr/bin/env bash
set -euo pipefail
BASE_DIR="/root/Codex/Law"
INSTANCE_DIR="$BASE_DIR/.openclaw-feishu-gitissue-gpt54"
CONFIG_PATH="$INSTANCE_DIR/openclaw.json"
STATE_DIR="$INSTANCE_DIR/state"
LOG_PATH="$INSTANCE_DIR/gateway.out"
PORT=18890

echo "instance_dir=$INSTANCE_DIR"
echo "config_path=$CONFIG_PATH"
echo "state_dir=$STATE_DIR"

echo "== process (by config path) =="
found=0
for p in $(pgrep -x openclaw 2>/dev/null || true); do
  if [ -r "/proc/$p/environ" ] && tr '\0' '\n' < "/proc/$p/environ" | grep -q "^OPENCLAW_CONFIG_PATH=$CONFIG_PATH$"; then
    found=1
    echo "openclaw pid=$p"
    ps -fp "$p" || true
  fi
done
if [ "$found" -eq 0 ]; then
  echo "openclaw not running"
fi

echo "== port =="
ss -lntp | grep -E "127.0.0.1:$PORT|\[::1\]:$PORT" || true

echo "== key config =="
OPENCLAW_CONFIG_PATH="$CONFIG_PATH" OPENCLAW_STATE_DIR="$STATE_DIR" openclaw config get agents.defaults.model.primary || true
OPENCLAW_CONFIG_PATH="$CONFIG_PATH" OPENCLAW_STATE_DIR="$STATE_DIR" openclaw config get channels.feishu.requireMention || true
OPENCLAW_CONFIG_PATH="$CONFIG_PATH" OPENCLAW_STATE_DIR="$STATE_DIR" openclaw config get channels.feishu.groupAllowFrom || true

echo "== recent log =="
tail -n 120 "$LOG_PATH" 2>/dev/null || true
EOF

chmod +x "$BASE_DIR/start_gitissue_gpt54.sh" "$BASE_DIR/stop_gitissue_gpt54.sh" "$BASE_DIR/status_gitissue_gpt54.sh"
```

---

## 6. 启动与检查（整段复制）

```bash
"$BASE_DIR/start_gitissue_gpt54.sh"
"$BASE_DIR/status_gitissue_gpt54.sh"
```

实时日志：

```bash
tail -f /root/Codex/Law/.openclaw-feishu-gitissue-gpt54/gateway.out
```

至少应出现：

1. `agent model: router/gpt-5.4`
2. `ready (... feishu ...)`
3. `starting channels and sidecars...`

---

## 7. 飞书侧验收顺序

1. 群里发普通文本，不 `@` 机器人。  
预期：不回复。  
2. 群里发图片，不 `@` 机器人。  
预期：不回复。  
3. 群里 `@` 机器人并问“回我 pong”。  
预期：回复。  
4. 再试 `@` + 图片问题。  
预期：能结合图片回复。

---

## 8. 常见问题一键处理

### 8.1 端口冲突

出现 `Port 18890 is already in use`：

1. 把 `openclaw.json` 的 `gateway.port` 改新端口（如 `18930`）。  
2. 把 3 个脚本中的 `PORT=18890` 同步改成同一端口。  
3. 执行：

```bash
/root/Codex/Law/stop_gitissue_gpt54.sh
/root/Codex/Law/start_gitissue_gpt54.sh
```

### 8.2 Router 401 / 无效令牌

检查当前配置：

```bash
OPENCLAW_CONFIG_PATH="/root/Codex/Law/.openclaw-feishu-gitissue-gpt54/openclaw.json" \
OPENCLAW_STATE_DIR="/root/Codex/Law/.openclaw-feishu-gitissue-gpt54/state" \
openclaw config get models.providers.router.baseUrl

OPENCLAW_CONFIG_PATH="/root/Codex/Law/.openclaw-feishu-gitissue-gpt54/openclaw.json" \
OPENCLAW_STATE_DIR="/root/Codex/Law/.openclaw-feishu-gitissue-gpt54/state" \
openclaw config get models.providers.router.api
```

应分别是：

1. `https://router.yeying.pub/v1`
2. `openai-responses`

重写 key 并重启：

```bash
read -rsp "重新输入 Router API Key: " NEW_KEY; echo
OPENCLAW_CONFIG_PATH="/root/Codex/Law/.openclaw-feishu-gitissue-gpt54/openclaw.json" \
OPENCLAW_STATE_DIR="/root/Codex/Law/.openclaw-feishu-gitissue-gpt54/state" \
openclaw config set models.providers.router.apiKey "$NEW_KEY"

/root/Codex/Law/stop_gitissue_gpt54.sh
/root/Codex/Law/start_gitissue_gpt54.sh
```

### 8.3 启动慢导致误判失败

如果你自行改过 `start` 脚本，把等待时间改回了 10 秒，会出现“`start failed` 但其实后面自己起来了”。  
把等待循环恢复为 `60` 秒版本即可（本文脚本已处理）。

### 8.4 飞书鉴权超时排查（网络）

先测飞书 OpenAPI 是否可达：

```bash
curl -sS -m 15 -X POST 'https://open.feishu.cn/open-apis/auth/v3/tenant_access_token/internal' \
  -H 'Content-Type: application/json' \
  -d '{"app_id":"x","app_secret":"y"}'
```

只要返回 JSON（哪怕是参数错误）就说明链路基本可达。

---

## 9. 你以后只用这 3 条命令

```bash
/root/Codex/Law/start_gitissue_gpt54.sh
/root/Codex/Law/stop_gitissue_gpt54.sh
/root/Codex/Law/status_gitissue_gpt54.sh
```

---

## 10. 本机已验证事实（2026-05-04）

1. `openclaw --version`：`2026.4.25`。  
2. `openclaw config validate` 对本文生成的实例配置通过。  
3. 用本文脚本可在本机将实例启动到 `agent model: router/gpt-5.4` 与 `ready`。  
4. `Law.md` 的飞书 `appId/appSecret` 可获取 `tenant_access_token`（飞书鉴权接口返回 `code=0`）。  
5. `Law.md` 三行都不是 Router API Key；Router Key 需单独输入。

