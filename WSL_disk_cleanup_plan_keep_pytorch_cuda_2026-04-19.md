# WSL 盘清理方案（保留 PyTorch+CUDA conda 环境）

更新日期：2026-04-19  
依据会话：`.codex-snw` 会话 `019da074-2e7f-72c3-8cb3-e9de20cb62e2`（统计时间 2026-04-18）

## 1. 背景和目标

会话里的大头占用（GiB）：

- `7.80` `/home/snw/E:environmentJXL_anacondaanaconda`（旧/重复 Anaconda）
- `10.81` `/home/snw/anaconda3/pkgs`
- `5.82` `/home/snw/.cache/uv`
- `3.13` `/home/snw/.rustup`
- `2.08` `/home/snw/Codex/NextChat`
- `1.24` `/home/snw/Codex/NextChat.old.2026-01-13_180512`
- `0.49` `/home/snw/Codex/DH_digitalHub`

你的要求：

- 保留必要 conda 环境，尤其是带 `pytorch + cuda` 的环境。
- 删除旧/重复 Anaconda。
- `digitalhub`、`nextchat`、Rust、`uv` 缓存都不要。

## 2. 执行原则（避免误删）

1. 先识别和备份要保留的 conda 环境，再删除。
2. 任何删除前先做存在性检查（目录不存在就跳过）。
3. 最后做 WSL VHDX 压缩，否则 Windows 宿主盘空间可能不立刻回收。

## 3. 先做“保留环境”识别和备份

在 Ubuntu/WSL 内执行：

```bash
set -euo pipefail

source ~/anaconda3/etc/profile.d/conda.sh

TS="$(date +%F_%H%M%S)"
BK="$HOME/cleanup_backup_$TS"
mkdir -p "$BK"

conda env list > "$BK/conda_env_list.txt"
conda info --envs > "$BK/conda_info_envs.txt"

# 识别哪些环境包含 torch/cuda
for e in $(conda env list | awk 'NR>2 && $1 !~ /^#/ {print $1}'); do
  {
    echo "== $e =="
    conda run --no-capture-output -n "$e" python - <<'PY'
import importlib.util
if importlib.util.find_spec("torch") is None:
    print("torch=NO")
else:
    import torch
    print(f"torch={torch.__version__}")
    print(f"cuda={torch.version.cuda}")
    print(f"cuda_available={torch.cuda.is_available()}")
PY
    conda list -n "$e" --explicit > "$BK/${e}_explicit.txt"
  } >> "$BK/torch_cuda_probe.txt" 2>&1
done

echo "Backup dir: $BK"
echo "Check torch/cuda report: $BK/torch_cuda_probe.txt"
```

判定规则：

- `torch=NO` 的环境可优先考虑删除。
- `cuda=...` 非 `None` 的环境属于要保留候选（例如此前会话里的 `py38_gpu`）。

## 4. 分阶段清理命令（按优先级）

### 阶段 A：删旧/重复 Anaconda（高收益）

```bash
set -euo pipefail
source ~/anaconda3/etc/profile.d/conda.sh

OLD_ANACONDA="/home/snw/E:environmentJXL_anacondaanaconda"
if [ -d "$OLD_ANACONDA" ]; then
  if conda env list | rg -F "$OLD_ANACONDA" >/dev/null; then
    echo "STOP: conda env still references old anaconda path: $OLD_ANACONDA"
    exit 1
  fi
  rm -rf "$OLD_ANACONDA"
  echo "Removed: $OLD_ANACONDA"
else
  echo "Skip missing: $OLD_ANACONDA"
fi
```

### 阶段 B：只保留必要 conda 环境（保留 GPU 环境）

先手动确认保留列表，例如：

```bash
KEEP_ENVS=("base" "py38_gpu")
```

再删除其余环境：

```bash
set -euo pipefail
source ~/anaconda3/etc/profile.d/conda.sh

KEEP_ENVS=("base" "py38_gpu")

for e in $(conda env list | awk 'NR>2 && $1 !~ /^#/ {print $1}'); do
  keep=0
  for k in "${KEEP_ENVS[@]}"; do
    [ "$e" = "$k" ] && keep=1
  done
  if [ "$keep" -eq 0 ]; then
    echo "Removing env: $e"
    conda env remove -n "$e" -y
  else
    echo "Keeping env: $e"
  fi
done
```

然后清理 conda 包缓存（不会删掉已保留环境里的已安装包）：

```bash
source ~/anaconda3/etc/profile.d/conda.sh
conda clean -a -y
```

### 阶段 C：删你明确不要的项目目录

```bash
for p in \
  /home/snw/Codex/DH_digitalHub \
  /home/snw/Codex/NextChat \
  /home/snw/Codex/NextChat.old.2026-01-13_180512
do
  if [ -d "$p" ]; then
    rm -rf "$p"
    echo "Removed: $p"
  else
    echo "Skip missing: $p"
  fi
done
```

### 阶段 D：删 Rust 和 uv 缓存

```bash
if command -v rustup >/dev/null 2>&1; then
  rustup self uninstall -y || true
fi

rm -rf ~/.rustup ~/.cargo ~/.cache/uv
```

### 阶段 E：顺手清理其他缓存（可选）

```bash
rm -rf ~/.cache/yarn ~/.npm/_cacache ~/.cache/puppeteer ~/.cache/pypoetry
```

## 5. 清理前后核对（必须）

```bash
echo "=== BEFORE/AFTER CHECK ==="
df -h /

du -sh \
  /home/snw/anaconda3 \
  /home/snw/anaconda3/pkgs \
  /home/snw/anaconda3/envs \
  /home/snw/.cache/uv \
  /home/snw/.rustup \
  /home/snw/Codex/NextChat \
  /home/snw/Codex/DH_digitalHub 2>/dev/null || true
```

## 6. 让 Windows 宿主盘真正变小（关键）

WSL 里删文件后，`ext4.vhdx` 不一定自动缩小。  
需要在 Windows PowerShell（管理员）做 compact。

可以通过你 `.bashrc` 里的 `w` 进入远端 PowerShell 后执行：

```powershell
wsl --shutdown
Optimize-VHD -Path "C:\Users\surface\AppData\Local\Packages\CanonicalGroupLimited.Ubuntu20.04LTS_79rhkp1fndgsc\LocalState\ext4.vhdx" -Mode Full
```

如果 `Optimize-VHD` 不可用，用 `diskpart`：

```powershell
wsl --shutdown
diskpart
select vdisk file="C:\Users\surface\AppData\Local\Packages\CanonicalGroupLimited.Ubuntu20.04LTS_79rhkp1fndgsc\LocalState\ext4.vhdx"
attach vdisk readonly
compact vdisk
detach vdisk
exit
```

## 7. 预估回收空间

按 2026-04-18 的统计，保守估计：

- 旧 Anaconda：约 `7.8 GiB`
- uv 缓存：约 `5.8 GiB`
- Rust：约 `3.1 GiB`
- NextChat + old + digitalhub：约 `3.8 GiB`
- conda/yarn/npm 额外缓存清理：通常还能回收若干 GiB

合计常见可回收：`20+ GiB`，实际以执行时目录状态为准。
