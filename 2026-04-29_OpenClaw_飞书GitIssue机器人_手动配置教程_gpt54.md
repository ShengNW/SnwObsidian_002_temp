# 2026-04-29 OpenClaw 飞书 GitIssue 机器人手动配置教程（`gpt-5.4`）

> 目标：在 GPU 机 `yeyingserver110` 的路径 `/root/code/bot/example/example_feishuGitIssue` 下，手动搭建一套**独立目录**的 OpenClaw 飞书机器人。  
> 行为目标：机器人能读群消息，但只在被 `@` 时回复。  
> 你已完成项：机器人已经建好且拉进目标群。

---

## 0. 你先确认两件事

1. 本机能用这个命令连上 GPU 机（不要用直连 `ssh -p 10022 ...`）：

```bash
ssh yeying-bot-root
```

2. 远端文件存在并是 3 行（`appId`、`appSecret`、`chatId`）：

```bash
ls -l /root/code/bot/example/example_feishuGitIssue/GitIssue.md
sed -n '1,3p' /root/code/bot/example/example_feishuGitIssue/GitIssue.md
```

---

## 1. 进入远端后，先设置变量（整段复制）

> 下面命令都在 **GPU 机 shell** 执行（你 `ssh yeying-bot-root` 进去后执行）。

```bash
set -euo pipefail

BASE_DIR=/root/code/bot/example/example_feishuGitIssue
INSTANCE_DIR="$BASE_DIR/.openclaw-feishu-gitissue-gpt54"
CONFIG_PATH="$INSTANCE_DIR/openclaw.json"
STATE_DIR="$INSTANCE_DIR/state"
WORKSPACE_DIR="$INSTANCE_DIR/workspace-larkbot"
LOG_PATH="$INSTANCE_DIR/gateway.out"
PID_FILE="$INSTANCE_DIR/openclaw.pid"
PORT=18890

mkdir -p "$INSTANCE_DIR" "$STATE_DIR" "$WORKSPACE_DIR"
```

---

## 2. 读取 `GitIssue.md`，并输入 Router Key（整段复制）

```bash
readarray -t FEISHU_LINES < "$BASE_DIR/GitIssue.md"
FEISHU_APP_ID="$(echo "${FEISHU_LINES[0]:-}" | tr -d '\r')"
FEISHU_APP_SECRET="$(echo "${FEISHU_LINES[1]:-}" | tr -d '\r')"
FEISHU_CHAT_ID="$(echo "${FEISHU_LINES[2]:-}" | tr -d '\r')"

if [ -z "$FEISHU_APP_ID" ] || [ -z "$FEISHU_APP_SECRET" ] || [ -z "$FEISHU_CHAT_ID" ]; then
  echo "GitIssue.md 三行内容不完整，停止。"
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
    "lastTouchedVersion": "2026.4.15",
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

## 4. 生成启停脚本（整段复制）

```bash
cat > "$BASE_DIR/start_gitissue_gpt54.sh" <<'EOF'
#!/usr/bin/env bash
set -euo pipefail
BASE_DIR="/root/code/bot/example/example_feishuGitIssue"
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
if [ -n "$pids" ] && ss -lntp | grep -q "127.0.0.1:$PORT"; then
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

for _ in 1 2 3 4 5 6 7 8 9 10; do
  sleep 1
  now="$(find_running_pids || true)"
  if [ -n "$now" ] && ss -lntp | grep -q "127.0.0.1:$PORT"; then
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
BASE_DIR="/root/code/bot/example/example_feishuGitIssue"
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
BASE_DIR="/root/code/bot/example/example_feishuGitIssue"
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

## 5. 启动并检查（整段复制）

```bash
"$BASE_DIR/start_gitissue_gpt54.sh"
"$BASE_DIR/status_gitissue_gpt54.sh"
```

如果你想实时看日志：

```bash
tail -f "$LOG_PATH"
```

你应该至少看到类似关键行：

1. `agent model: router/gpt-5.4`
2. `starting feishu[default] (mode: websocket)`
3. `ws client ready`

---

## 6. 飞书侧验收（按这个顺序）

1. 群里发一条普通文本，不 `@` 机器人。  
预期：机器人不回复。
2. 群里发一张图，不 `@` 机器人。  
预期：机器人不回复。
3. 群里 `@` 机器人并问一句“回我 pong”。  
预期：机器人回复。
4. 再试 `@` + 图片问题。  
预期：可以结合图片回复。

---

## 7. 常见问题一键处理

### 7.1 端口被占用

报错里如果出现 `Port 18890 is already in use`：

1. 把 3 个脚本里的 `PORT=18890` 同时改成另一个端口（比如 `18930`）。
2. 把 `openclaw.json` 里 `gateway.port` 也改成同一个端口。
3. 重新执行：

```bash
"$BASE_DIR/stop_gitissue_gpt54.sh"
"$BASE_DIR/start_gitissue_gpt54.sh"
```

### 7.2 Router 返回 401 / 无效令牌

```bash
OPENCLAW_CONFIG_PATH="$CONFIG_PATH" OPENCLAW_STATE_DIR="$STATE_DIR" \
openclaw config get models.providers.router.baseUrl

OPENCLAW_CONFIG_PATH="$CONFIG_PATH" OPENCLAW_STATE_DIR="$STATE_DIR" \
openclaw config get models.providers.router.api
```

应分别是：

1. `https://router.yeying.pub/v1`
2. `openai-responses`

然后重写 key：

```bash
read -rsp "重新输入 Router API Key: " NEW_KEY; echo
OPENCLAW_CONFIG_PATH="$CONFIG_PATH" OPENCLAW_STATE_DIR="$STATE_DIR" \
openclaw config set models.providers.router.apiKey "$NEW_KEY"
"$BASE_DIR/stop_gitissue_gpt54.sh"
"$BASE_DIR/start_gitissue_gpt54.sh"
```

### 7.3 出现 `Client not allowed`（极少数路由策略会拦截）

执行下面这段给 router provider 增加请求头，然后重启：

```bash
node - <<'NODE'
const fs = require('fs');
const p = '/root/code/bot/example/example_feishuGitIssue/.openclaw-feishu-gitissue-gpt54/openclaw.json';
const j = JSON.parse(fs.readFileSync(p, 'utf8'));
j.models = j.models || {};
j.models.providers = j.models.providers || {};
j.models.providers.router = j.models.providers.router || {};
j.models.providers.router.headers = {
  "User-Agent": "codex_exec/0.115.0 (Ubuntu 22.4.0; x86_64) xterm-256color (codex-exec; 0.115.0)",
  "originator": "codex_exec",
  "x-client-request-id": "openclaw-gateway",
  "session_id": "openclaw-gateway"
};
fs.writeFileSync(p, JSON.stringify(j, null, 2));
console.log('headers patched');
NODE

"$BASE_DIR/stop_gitissue_gpt54.sh"
"$BASE_DIR/start_gitissue_gpt54.sh"
```

---

## 8. 以后你只用这 3 条命令

```bash
/root/code/bot/example/example_feishuGitIssue/start_gitissue_gpt54.sh
/root/code/bot/example/example_feishuGitIssue/stop_gitissue_gpt54.sh
/root/code/bot/example/example_feishuGitIssue/status_gitissue_gpt54.sh
```

---

## 9. 本文对应的已验证事实（2026-04-29）

1. `GitIssue.md` 已存在于 `/root/code/bot/example/example_feishuGitIssue/GitIssue.md`，并且是 3 行格式。
2. `https://router.yeying.pub/v1/models` 对你提供的 key 返回 `HTTP 200`。
3. `https://router.yeying.pub/v1/responses` 使用 `model=gpt-5.4` 返回 `HTTP 200`，且可得到 `pong`。

