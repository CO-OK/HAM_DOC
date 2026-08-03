# AIS Receiver 工程实施计划

> 编写日期：2026-08-03
> 目标平台：macOS（已在其他平台用 SDR 工具链）
> 硬件：SDR Blog V4 (R828D tuner)
> 接收位置：上海普陀区光复西路 1145 弄（隆德小区，朝南阳台对苏州河）
> 计划交付：可独立运行的 Git 工程

---

## 0. 用户需求清单（需求基线）

| # | 需求 | 说明 |
|---|------|------|
| 1 | 工程化 + git 版本控制 | git init、`.gitignore`、初始 commit |
| 2 | README 完整 | 工程原理 + 数据流向说明 |
| 3 | Make 能力 | `make start` 启动 + 自动打开浏览器 |
| 4 | 前置依赖检查 | 检查 `AIS-catcher`、Python 等是否安装（homebrew 除外） |
| 5 | 数据规模限制 + 回收策略 | NMEA 录制文件、AIS-Catcher 进程日志都需限制 |
| 6 | 不需要开机自启动 | 不写 launchd plist |
| 7 | 配置项可暴露 | 端口、无线电参数（增益、采样率、PPM 等）都能改 |

---

## 1. 关键设计决策

### 1.1 架构：单进程方案（最简）

用 **AIS-Catcher 自带的 Web UI** 直接替代 OpenCPN。理由：
- AIS-Catcher v0.42+ 内置 Web 服务器（Leaflet + Chart.js），`http://localhost:8080` 直接看实时船位
- 少一个进程、少一份配置
- 用户明确要求"自动打开浏览器"——OpenCPN 是桌面应用，Web UI 才符合

### 1.2 最终链路

```
   162 MHz 射频（船台 12.5W VHF）
        │
   ┌────┴────┐
   │  天线   │  ← 偶极立起来（垂直极化）
   └────┬────┘
        │ 同轴
   ┌────┴────┐
   │ RTL-SDR │  ← R828D tuner, 24-1766 MHz
   │ Blog V4 │  ← USB
   └────┬────┘
        │ librtlsdr（直接驱动）
   ┌────┴────────────────┐
   │   AIS-Catcher 进程  │
   │  ┌──────────────┐   │
   │  │ GMSK 解调    │   │  ← 物理层 → 链路层
   │  │ HDLC 帧解析  │   │
   │  │ CRC 校验     │   │
   │  └──────┬───────┘   │
   │         │ NMEA 0183 │
   │  ┌──────┴───────┐   │
   │  │ Web Server   │   │  ← 应用层
   │  │ (Leaflet)    │   │
   │  └──────┬───────┘   │
   └─────────┼───────────┘
             │ HTTP
        ┌────┴────┐
        │ 浏览器  │
        │ (用户)  │
        └─────────┘
```

AIS-Catcher 一个进程干三件事：**RF → 解码 → Web**。

### 1.3 关键选型

| 选择 | 决定 | 理由 |
|------|------|------|
| 部署形态 | **单进程 + Make 包装** | 简单、易调试、易自动化 |
| 显示 | **AIS-Catcher 内置 Web** | 不需要 OpenCPN |
| 配置格式 | **YAML** | 可读性好、支持注释 |
| 进程管理 | **PID 文件 + 锁文件** | 避免重复启动 |
| 日志 | **stdout/stderr 重定向到文件** | 简单可靠 |
| 数据回收 | **后台轮询脚本 + 大小阈值** | 不需要外部定时器 |
| 依赖检查 | **shell 脚本独立检查** | 不依赖额外工具 |

### 1.4 执行模式：Goal 模式

落地 agent 必须**用 goal 模式**运行，按以下规则：

#### 1.4.1 Goal Statement（复制即用）

> **目标**：在 `~/HAM/ais-receiver/` 完成 PLAN-AIS-RECEIVER.md 描述的 AIS Receiver 工程的实施。
>
> **验收（全部满足才算 complete）**：
> 1. git 仓库已初始化，至少 8 个语义清晰的 commit
> 2. `make help` 列出所有目标
> 3. `make check` 报告所有依赖 [✓]（或清晰报错）
> 4. `make start` 成功启动 AIS-Catcher，自动打开浏览器到 `http://localhost:8080`
> 5. 浏览器看到地图，**至少 1 艘船的标记**（说明 RF → 解码 → Web 全链路通）
> 6. `make stop` 干净退出（PID 文件清掉、`pgrep` 找不到进程）
> 7. `make status`、`make logs`、`make rotate`、`make clean` 各自行为正确
> 8. `data/nmea/` 下有当天日期的 `.nmea` 文件且在增长
> 9. README.md 包含：项目简介、架构图、数据流向图、make 目标说明、调参指南
> 10. `.gitignore` 正确忽略 `data/nmea/*.nmea`、`logs/*.log`、`.run/*.pid`

#### 1.4.2 行为规则

| 状态 | 何时调用 | 怎么调用 |
|------|----------|----------|
| **complete** | 全部 10 项验收通过 | `update_goal status=complete` |
| **blocked** | 同一障碍卡 ≥ 3 个 turn | `update_goal status=blocked` + 简短说明卡在哪 |
| **不要做** | 中途遇到小问题 | 自己修，不要问用户（homebrew 软件失败这种已知问题先重试一次） |

#### 1.4.3 Agent 必须自己处理的"小事"（不要问用户）

- YAML 缩进错误 → 自己看报错
- shell 脚本语法错误 → 自己 `bash -n` 检查
- commit message 不够清晰 → 自己重写
- `make start` 启动失败 → 自己看 `logs/ais-catcher.out.log` 排查
- 端口被占用 → 自己换端口（修改 `config.yaml`）
- `PyYAML` 没装 → 自己 `pip3 install pyyaml --user` 或提示用户
- AIS-Catcher brew 装不上 → 自己尝试 `brew tap abcd567a/tap && brew install --verbose`

#### 1.4.4 真正需要 mark blocked 的情况（≥ 3 turn 解决不了）

- AIS-Catcher 装不上且 brew 没有 mirror
- RTL-SDR V4 设备不被识别（macOS USB 权限问题）
- 162 MHz 频段完全收不到任何 AIS 信号（连续 1 小时 0 msg/s）
- macOS 系统版本与 librtlsdr 不兼容

> 这 4 种都**先自行重试 1 次**（如 `brew reinstall`、`AIS-catcher -l`），重试失败再继续观察，**累积 3 turn 仍然没进展**才 `blocked`。

#### 1.4.5 完成时的自检清单

在 `update_goal status=complete` 之前，agent 必须**逐项验证**：

```bash
# 1. git 历史
cd ~/HAM/ais-receiver && git log --oneline | wc -l   # ≥ 8

# 2. make help
make help   # 输出表格

# 3. make check
make check  # 全 [✓]

# 4-5. 启动并验证
make start
sleep 30
curl -s http://localhost:8080/geojson | python3 -c "import json,sys;d=json.load(sys.stdin);print(f'{len(d[\"features\"])} ships')"   # ≥ 1
# 或者：直接访问 http://localhost:8080 截图

# 6. 停止
make stop
pgrep -f AIS-catcher   # 空输出

# 7. 其他目标
make status     # "not running"
make logs       # tail 显示历史
make rotate     # 报告清理结果
make clean      # 不报 data/nmea/ 内容

# 8. 数据生成
ls -la data/nmea/   # 看到当天日期的 .nmea 文件

# 10. gitignore
git check-ignore data/nmea/test.nmea   # 输出路径（被忽略）
```

全部通过 → `update_goal status=complete`

---

## 2. 工程结构

```
ais-receiver/                                    # 工程根目录
├── .git/                                        # git 仓库
├── .gitignore                                   # 忽略 data/、logs/、.run/、*.log
├── README.md                                    # 用户/开发者文档
├── Makefile                                     # 入口
├── config.yaml                                  # 主配置（可编辑）
├── scripts/
│   ├── check-deps.sh                            # 依赖检查
│   ├── start.sh                                 # 启动逻辑
│   ├── stop.sh                                  # 停止
│   ├── status.sh                                # 状态
│   ├── rotate.sh                                # 数据/日志回收
│   ├── open-browser.sh                          # 打开浏览器
│   └── lib.sh                                   # 公共函数
├── data/
│   └── nmea/
│       └── .gitkeep
├── logs/
│   └── .gitkeep
├── .run/
│   └── .gitkeep                                 # PID 文件目录
└── docs/
    ├── architecture.md                           # 详细架构（可选）
    └── troubleshooting.md                        # 故障排除（可选）
```

---

## 3. 配置文件（config.yaml）

**位置**：`config.yaml`（工程根目录）

```yaml
# ========================================
# ⚠️ 天线极化决定 90% 接收效果
# ========================================
# VHF AIS 必须用 **垂直极化**（偶极子立起来）
# 平放 = 损失 ~20 dB = 几乎收不到信号
# 详见 SDR-AIS上海船舶监听指南.md 第 8 节"天线方案"
# ========================================
# AIS 接收工程配置
# 修改后需要重启（make restart）生效
# ========================================

ais_catcher:
  # 可执行文件路径（一般是 PATH 里能找到的，不用改）
  binary: "AIS-catcher"
  
  # Web UI 端口（浏览器访问 http://localhost:<port>）
  web_port: 8080
  
  # ============================================
  # 无线电参数（你可能想调的都在这）
  # ============================================
  
  # Tuner 增益（0-49 dB，0=自动）
  # 上海信号强，建议 25-35
  gain: 30
  
  # 采样率（Hz）
  # 可选：288000、1536000、2304000
  # 默认 1536000（1.536M）最佳平衡
  sample_rate: 1536000
  
  # 频率校正（PPM）
  # V4 通常 ±1PPM 内，无需校正
  # 如果信号差，设为 0.5 或 -0.5 试
  ppm: 0
  
  # RTL AGC（自动增益控制）— **必须关**
  rtlagc: "off"
  
  # Tuner 带宽（Hz）— AIS 频道 25 kHz
  # 设 192000 = 192 kHz，覆盖两频道有富余
  bandwidth: 192000
  
  # Bias tee（V4 5V 输出）
  # 不接外部 LNA 时设为 off
  biastee: "off"
  
  # Verbose 级别：0-10
  # 0 = 静默
  # 5 = 每 5 秒打印一次统计
  # 10 = 实时详细
  verbose: 5

station:
  # 站点名称（Web UI 上显示）
  name: "Shanghai Putuo"
  
  # 站点位置（用于显示在 Web 地图上）
  lat: 31.2400
  lon: 121.4300
  
  # 是否在 Web UI 上分享位置（off/on）
  share_loc: "off"

# ============================================
# 数据管理（防撑爆磁盘）
# ============================================
data:
  # NMEA 录制目录
  nmea_dir: "data/nmea"
  
  # 单个 NMEA 文件最大字节数
  # 超过会自动 rotate 到 nmea_2026-08-03_001.nmea
  max_file_size_mb: 50
  
  # 总 NMEA 数据最大占用
  max_total_size_mb: 500
  
  # NMEA 文件保留天数
  retention_days: 7
  
  # 回收检查间隔（分钟）
  rotate_interval_min: 30

# ============================================
# 进程日志管理
# ============================================
log:
  # AIS-Catcher 进程日志目录
  dir: "logs"
  
  # 进程日志单文件最大字节数
  max_file_size_mb: 20
  
  # 进程日志保留天数
  retention_days: 14

# ============================================
# 进程管理
# ============================================
process:
  # PID 文件路径
  pid_file: ".run/ais-catcher.pid"
  
  # 进程 stdout/stderr 输出文件
  out_log: "logs/ais-catcher.out.log"
```

---

## 4. Makefile 接口

```makefile
# 顶层 Makefile（草图）

# 默认目标
.DEFAULT_GOAL := help

# 配置
CONFIG_FILE := config.yaml
PYTHON := python3
SHELL := /bin/bash

.PHONY: help check install start stop restart status logs clean config device-list version

help:                   ## 显示帮助
	@grep -E '^[a-zA-Z_-]+:.*?## .*$$' $(MAKEFILE_LIST) | \
	 awk 'BEGIN {FS = ":.*?## "}; {printf "  \033[36m%-20s\033[0m %s\n", $$1, $$2}'

check:                  ## 检查依赖（homebrew 之外）
	@bash scripts/check-deps.sh

install:                ## 提示安装 homebrew 软件（不自动装）
	@echo "请手动运行："
	@echo "  brew install librtlsdr"
	@echo "  brew install abcd567a/tap/ais-catcher"
	@echo "完成后再次运行 make check"

start:                  ## 启动 + 打开浏览器
	@bash scripts/start.sh

stop:                   ## 停止 AIS-Catcher 进程
	@bash scripts/stop.sh

restart: stop start     ## 重启

status:                 ## 查看进程状态
	@bash scripts/status.sh

logs:                   ## tail 当前日志
	@bash scripts/lib.sh tail_log

device-list:            ## 列出 SDR 设备
	@AIS-catcher -l

config:                 ## 打开配置文件
	@open -t $(CONFIG_FILE) || vim $(CONFIG_FILE)

version:                ## 显示版本
	@AIS-catcher -h 2>&1 | head -3 || echo "AIS-catcher 未安装"

clean:                  ## 清理临时文件
	@bash scripts/lib.sh clean
```

**预期输出**：
```bash
$ make help
  help                 显示帮助
  check                检查依赖
  install              提示安装 homebrew 软件
  start                启动 + 打开浏览器
  stop                 停止 AIS-Catcher 进程
  restart              重启
  status               查看进程状态
  logs                 tail 当前日志
  clean                清理临时文件
  config               打开配置文件
  device-list          列出 SDR 设备
  version              显示版本
```

---

## 5. 脚本规范

### 5.1 通用约定

- 用 `bash` 写，可移植
- 严格使用 `set -euo pipefail`
- 公共函数放 `scripts/lib.sh`
- 通过 `yq`（Python 写也行）解析 YAML 配置
- 所有路径基于工程根目录（用 `$(cd $(dirname $0)/.. && pwd)` 推断）
- 日志用 timestamp 格式

### 5.2 scripts/lib.sh 核心函数

```bash
# 工程根目录
project_root() {
    cd "$(dirname "${BASH_SOURCE[0]}")/.." && pwd
}

# 读 YAML 配置
config_get() {
    local key="$1"
    # 用 python 解析，避免依赖 yq
    python3 -c "
import yaml, sys
with open('${CONFIG_FILE}') as f:
    cfg = yaml.safe_load(f)
keys = '$key'.split('.')
v = cfg
for k in keys:
    v = v.get(k, '')
print(v)
"
}

# 检查进程是否在跑
is_running() {
    local pid_file
    pid_file="$(project_root)/$(config_get process.pid_file)"
    [[ -f "$pid_file" ]] && kill -0 "$(cat "$pid_file")" 2>/dev/null
}

# 写日志
log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*" | tee -a "$(project_root)/logs/operations.log"
}
```

### 5.3 scripts/check-deps.sh（关键）

**职责**：检查以下项，告诉用户缺啥、怎么装。

| 依赖 | 检查方法 | 装法（不替你装） |
|------|----------|------------------|
| macOS | `uname -s` == "Darwin" | — |
| Homebrew | `which brew` | 提示装 |
| Python 3 | `which python3` && version >= 3.6 | 提示装 |
| PyYAML | `python3 -c "import yaml"` | `pip3 install pyyaml` |
| librtlsdr | `brew list librtlsdr` | `brew install librtlsdr` |
| AIS-catcher | `which AIS-catcher` && `-l` 能跑 | `brew install abcd567a/tap/ais-catcher` |
| RTL-SDR V4 设备 | `AIS-catcher -l` 输出含 "RTL2838UHIDIR" | 提示插设备 |
| USB 权限（macOS 通常 OK） | 不需要 | — |

**输出格式**（彩色）：
```
[✓] macOS
[✓] Homebrew
[✓] Python 3.11.0
[✓] PyYAML
[✓] librtlsdr (homebrew)
[✓] AIS-catcher (v0.57)
[✓] RTL-SDR V4 已连接 (SN: 00000001)

所有依赖已就绪！可以运行 `make start`
```

### 5.4 scripts/start.sh（核心）

**流程**：
1. 调 `check-deps.sh`（失败直接退出）
2. 检查是否已在跑（PID 文件存在且进程活着 → 提示先 stop）
3. 创建必要目录（`data/nmea/`、`logs/`、`.run/`）
4. 从 YAML 构造 AIS-Catcher 命令行参数
5. 用 `nohup` 启动到后台
6. 写 PID 文件
7. 等待 3 秒看是否还在跑（启动失败排查）
8. 打开浏览器到 `http://localhost:<web_port>`
9. 打印 "启动成功 → 浏览器已打开"

**命令行构造**（从 YAML 拼出来）：

> ⚠️ **实施前必做**：先 `AIS-catcher -h` 验证下列参数！下面是基于 v0.57 文档的草图，参数名/大小写/用法可能随版本变化。
>
> **关键参数速查**（按 `--help` 实际输出调整）：
> - `-d:N` — RTL-SDR 设备索引（**带冒号**！`-d 0` 是序列号会找不到设备）
> - `-v N` — verbose 级别
> - `-s Hz` — 采样率
> - `-a Hz` — 接收带宽
> - `-gr tuner <dB>` — Tuner 增益（**小写 tuner**）
> - `-gr rtlagc on/off` — RTL AGC（**必须 off**）
> - `-gr biastee on/off` — Bias tee（V4 有 5V 输出）
> - `-p PPM` — 频率校正
> - `-M D|T|M` — Meta 数据：`D` = signalpower+ppm（调参时打开），`T` = 时间戳，`M` = 国家
> - `-N PORT` — Web UI 端口
> - `-N STATION "name"` / `-N LAT lat` / `-N LON lon` / `-N SHARE_LOC on/off`
> - `-X on/off` — aiscatcher.org 社区共享（**默认 ON**，不需要要显式 off）
> - `-f file.nmea` — NMEA 输出文件

```bash
AIS-catcher \
    -d:0 \
    -v $(config_get ais_catcher.verbose) \
    -s $(config_get ais_catcher.sample_rate) \
    -a $(config_get ais_catcher.bandwidth) \
    -gr tuner $(config_get ais_catcher.gain) \
    -gr rtlagc $(config_get ais_catcher.rtlagc) \
    -gr biastee $(config_get ais_catcher.biastee) \
    -p $(config_get ais_catcher.ppm) \
    -M $(config_get ais_catcher.meta) \
    -X $(config_get ais_catcher.community_share) \
    -N $(config_get ais_catcher.web_port) \
    -N STATION "$(config_get station.name)" \
    -N LAT $(config_get station.lat) \
    -N LON $(config_get station.lon) \
    -N SHARE_LOC $(config_get station.share_loc) \
    -f "$(date +%Y-%m-%d).nmea"
```

### 5.5 scripts/stop.sh

**流程**：
1. 读 PID 文件
2. `kill -TERM` 给 5 秒退出
3. 还活着 → `kill -9`
4. 清 PID 文件

### 5.6 scripts/rotate.sh（数据回收）

**两类回收**：

#### A. NMEA 文件回收

```bash
# 1. 删除超过 retention_days 的文件
find "$nmea_dir" -name "*.nmea" -mtime +$retention_days -delete

# 2. 如果总大小 > max_total_size_mb，删最老的
total_size=$(du -sm "$nmea_dir" | cut -f1)
if (( total_size > max_total_size_mb )); then
    find "$nmea_dir" -name "*.nmea" -printf '%T@ %p\n' | \
        sort -n | head -n 5 | awk '{print $2}' | xargs rm
fi
```

#### B. 进程日志回收

```bash
# 1. 超过 max_file_size_mb → rotate
if [[ -f "$out_log" ]] && (( $(stat -f%z "$out_log") > max_size * 1024 * 1024 )); then
    mv "$out_log" "$out_log.$(date +%Y%m%d-%H%M%S).old"
    # 重启进程来切新日志（用 SIGHUP 不行，因为 AIS-catcher 不支持 reopen log）
    make restart  # Makefile 已有 restart: stop start
fi

# 2. 删老的日志
find "$log_dir" -name "*.old" -mtime +$retention_days -delete
```

### 5.7 回收触发方式

- `make start` 时**自动跑一次** rotate
- **不**写 launchd（用户要求）
- 内部用一个**轻量级轮询**机制：
  - 在 `start.sh` 里启动一个**辅助进程**（也是 nohup），每 `rotate_interval_min` 分钟跑一次 rotate.sh
  - 或者把 rotate 集成到 `start.sh` 启动后**永远 loop** 的子 shell
  - **最简方案**：rotate 是 **start 时** + **用户手动** 触发

**采用方案**：start 时跑一次 + 用户手动 `make rotate`，**不开后台轮询**。理由：
- 用户没要求后台轮询
- 加后台轮询 = 多一个进程、复杂度上升
- 手动 `make rotate` 用户可以接 cron/launchd（如果以后想）

---

## 6. README 结构

```markdown
# AIS Receiver

> 实时接收上海黄浦江/苏州河流域船舶 AIS 信号，基于 RTL-SDR V4 + AIS-Catcher

## 快速开始（5 行内）

\`\`\`bash
brew install librtlsdr abcd567a/tap/ais-catcher  python3 pyyaml
git clone <repo> ~/HAM/ais-receiver
cd ~/HAM/ais-receiver
make check      # 验证依赖
make start      # 启动 + 打开浏览器
\`\`\`

浏览器访问 http://localhost:8080 看到实时船位地图。

## 目录

1. 系统架构
2. 数据流向
3. 安装
4. 使用
5. 配置
6. 调参
7. 数据管理
8. 故障排除
9. 扩展

## 1. 系统架构

\`\`\`
[示意图：射频 → RTL-SDR → AIS-Catcher → Web 浏览器]
\`\`\`

## 2. 数据流向

详细描述：162 MHz → 偶极天线 → V4 tuner → IQ 数据 → AIS-Catcher 解调
→ 内存 NMEA 句子 → 同时 (a) WebSocket 推前端 (b) 写文件到 data/nmea/

## 3. 安装

`brew install ...`

## 4. 使用

- make help
- make start
- make stop
- make status
- make logs
- make clean
- make rotate

## 5. 配置

完整 config.yaml 字段说明

## 6. 调参

增益、采样率、PPM、天线极化的影响

## 7. 数据管理

data/nmea/ 目录、自动 rotate 策略、手动 rotate

## 8. 故障排除

常见错误、怎么排查

## 9. 扩展方向

- 多 SDR 并行
- 1090 MHz ADS-B
- MarineTraffic 上传
```

**关键**：第 1、2 节是用户特别要求的"工程上的原理以及数据流向说明"，要详细。

---

## 7. 实施步骤（分阶段）

### 阶段 1：脚手架（30 分钟）

1. `mkdir -p ~/HAM/ais-receiver && cd ~/HAM/ais-receiver`
2. `git init`
3. 写 `.gitignore`：
   ```
   data/nmea/*.nmea
   data/nmea/*.nmea.*
   logs/*.log
   logs/*.log.*
   .run/*.pid
   ```
4. 创建目录：`mkdir -p scripts data/nmea logs .run docs`
5. 写 `data/nmea/.gitkeep`、`logs/.gitkeep`、`.run/.gitkeep`
6. **第一个 commit**：`chore: initial scaffold`

### 阶段 2：配置 + lib（30 分钟）

1. 写 `config.yaml`（按上面第 3 节内容）
2. 写 `scripts/lib.sh`（公共函数 + YAML 解析）
3. **commit**：`feat: add config and lib`

### 阶段 3：依赖检查（20 分钟）

1. 写 `scripts/check-deps.sh`
2. `make check` 跑通
3. **commit**：`feat: add dependency checker`

### 阶段 4：启动 / 停止（45 分钟）

1. 写 `scripts/start.sh`
2. 写 `scripts/stop.sh`
3. 写 `scripts/open-browser.sh`
4. 写 `scripts/status.sh`
5. **commit**：`feat: add process lifecycle scripts`

### 阶段 5：Makefile（15 分钟）

1. 写 `Makefile`
2. `make help` 验证
3. **commit**：`feat: add Makefile`

### 阶段 6：第一次端到端（30 分钟）

1. `make start`
2. 验证：浏览器打开、看到地图、看到船
3. `make stop` 验证停止
4. 修 bug

### 阶段 7：数据回收（30 分钟）

1. 写 `scripts/rotate.sh`
2. 在 `start.sh` 里加 rotate 调用
3. `make rotate` 验证
4. **commit**：`feat: add data rotation`

### 阶段 8：README（45 分钟）

1. 写 `README.md`（按第 6 节结构）
2. 写 `docs/architecture.md`（详细架构图、信号链路图）
3. **commit**：`docs: add README and architecture`

### 阶段 9：打磨（30 分钟）

1. 跑 `make check && make start && 验证` 全流程
2. 故意制造 2 个错误场景，看错误信息是否友好
3. **commit**：`chore: polish error messages`

### 阶段 10：交付（10 分钟）

1. `git log` 检查 commit 历史清晰
2. 推送到 GitHub（如果用户有）

**总时间**：约 4-5 小时（含调试）

---

## 8. 验收标准

### 8.1 自动化测试项

| 测试 | 预期 | 怎么验 |
|------|------|--------|
| `make help` | 列出所有目标 | 跑 |
| `make check` | 全部 [✓] 或清晰报错 | 跑 |
| `make start` | 浏览器自动打开、看到地图 | 跑 + 截图 |
| `make status` | 显示 PID、运行时间 | 跑 |
| `make logs` | tail 日志、按 Ctrl+C 退出 | 跑 |
| `make stop` | 进程干净退出 | 跑 + `ps` 验证 |
| `make start`（重复） | 提示已在跑 | 跑 |
| `make rotate` | 清掉过期文件、报告大小 | 跑 + `du` |
| `make clean` | 清临时文件、不动 data/nmea/ | 跑 + `ls` |

### 8.2 端到端测试

- **链路测试**：启动后 30 秒内，Web UI 上能看到至少 1 艘船（黄浦江或苏州河）
- **错误恢复**：故意拔掉 RTL-SDR → 进程退出 → 重新插上 → `make start` 又能起来
- **数据回收**：造一个 100 MB 大小的 fake `.nmea` → `make rotate` → 应该被删

### 8.3 用户可验证

- 浏览器 `http://localhost:8080` 显示 AIS-Catcher web UI
- 地图上能看到黄浦江/苏州河的船
- 终端输出 `decoded: N msg/s` 且 N > 0
- `ls -la data/nmea/` 能看到当天日期的 NMEA 文件在持续写入

---

## 9. 关键技术细节

### 9.1 YAML 解析的 Python 方案

```python
# 在 lib.sh 里的 config_get 函数
python3 -c "
import yaml, sys
with open('config.yaml') as f:
    cfg = yaml.safe_load(f)
keys = 'ais_catcher.gain'.split('.')
v = cfg
for k in keys:
    v = v.get(k, '')
print(v)
"
```

不引入 yq 依赖，Python 标准库 + PyYAML 够了。

### 9.2 macOS 文件大小检查

```bash
file_size_bytes() {
    stat -f%z "$1"  # macOS
}
# 或用 wc -c
file_size_bytes() {
    wc -c < "$1" | tr -d ' '
}
```

### 9.3 后台进程 + 跨父进程存活

```bash
# 用 nohup + disown
nohup "$cmd" > "$out_log" 2>&1 &
echo $! > "$pid_file"
disown
```

### 9.4 启动后等待端口可访问

```bash
# 启动后等 3 秒，再 curl 探一下端口
sleep 3
if ! curl -s "http://localhost:$port" > /dev/null; then
    log "ERROR: web server didn't start in 3s"
    exit 1
fi
```

---

## 10. 风险与限制

| 风险 | 影响 | 缓解 |
|------|------|------|
| PyYAML 未装 | config 读不出来 | check-deps 提示装 |
| AIS-Catcher 不支持 V4 | 找不到设备 | 用 brew tap 的最新版（v0.57+） |
| macOS 防火墙拦截端口 | 浏览器打不开 | 提示用户允许 |
| 数据涨爆 | 磁盘满 | rotate 策略 + 周期性检查 |
| 进程假死 | stop 不响应 | `kill -9` 兜底 |
| Python 版本过老 | 不支持 | check-deps 报清楚 |

---

## 11. 后续扩展（plan 不实现，留口子）

- `make stream` 把 NMEA 转发到 MarineTraffic（`make config` 加一个 `upload:` 节）
- `make adsb-start` 启动 1090 MHz ADS-B（独立进程）
- `make all-stop` 停所有相关进程
- 把脚本包装成 Homebrew Formula
- 写一个 GitHub Action 跑 `make check`

---

## 12. 交付物清单

- [ ] git 仓库（`~/HAM/ais-receiver/`）
- [ ] README.md
- [ ] Makefile
- [ ] config.yaml
- [ ] scripts/（6 个 .sh）
- [ ] docs/architecture.md
- [ ] .gitignore
- [ ] 至少 8 个清晰的 git commit
- [ ] `make start` 端到端跑通
- [ ] `make check && make status && make stop` 都干净

---

## 13. 关键参考

- AIS-Catcher GitHub: <https://github.com/jvde-github/AIS-catcher>
- AIS-Catcher v0.42+ Web Server: README 中 "Web interface" 节
- 配置文件格式：README 中 "Configuration file" 节
- 用户档案：`/Users/quanwei1/.minimax/memory/user.md`
- 已有 AIS 文档：`/Users/quanwei1/HAM/SDR-AIS上海船舶监听指南.md`
- 已有 SDR++ 指南：`/Users/quanwei1/HAM/SDR++上海监听指南.md`

---

*本计划由 Mavis 编写，等待另一个 agent 落地实施。*

---

## 14. 实施后补记（v0.1.0 → v0.1.1 的踩坑）

> 落地实施时发现、计划未覆盖的 4 条关键问题。新人按本计划实施前**必读**。

### 14.1 macOS 安装没有 homebrew tap

❌ **计划原文**：`brew tap abcd567a/tap && brew install ais-catcher`

✅ **实际情况**：`abcd567a/tap` 是某用户给 Raspberry Pi 写 systemd 服务的脚本仓库，**不是 AIS-catcher 官方 homebrew 仓库**，clone 会报 404。

**正确做法（macOS）**：
```bash
# 1. 装系统依赖
brew install librtlsdr pkg-config

# 2. 从 GitHub 源码编
git clone --depth 1 https://github.com/jvde-github/AIS-catcher.git
cd AIS-catcher
make CFLAGS="-DHASWEBVIEWER"    # ⚠️ 必须显式加这个 flag！
cp AIS-catcher /opt/homebrew/bin/
chmod +x /opt/homebrew/bin/AIS-catcher
```

⚠️ **WebViewer 默认未编译**：AIS-catcher 的 `Makefile` **不会**自动加 `-DHASWEBVIEWER`（只有 CMakeLists.txt 才会）。用默认 `make` 或 `make rtl-only` 出来的 binary 启动时报 `WebViewer support not compiled in.` 然后退出。**必须** `make CFLAGS="-DHASWEBVIEWER"`。

### 14.2 设备索引语法

❌ **计划原文**：`AIS-catcher -d 0`

✅ **实际情况**：`AIS-catcher` 把 `-d 0` 解释成"序列号 0"（找不到设备，进程启动后报 `Receiver: cannot set up device.`）。**正确语法是 `-d:0`（带冒号）**。

```bash
# 错（当成 SN）
AIS-catcher -d 0

# 对（当成 index）
AIS-catcher -d:0

# 选 SN 才是
AIS-catcher -d 00000001
```

实施时 `start.sh` 必须用 `-d:"$dev_idx"` 而不是 `-d "$dev_idx"`。

### 14.3 社区共享默认 ON（隐私风险）

AIS-catcher **默认** `-X on`，自动把收到的 AIS 数据上传到 `aiscatcher.org:4242` 社区 hub。每次启动日志会看到：

```
TCP feed: open socket for host: 185.77.96.227, port: 4242
```

**如果不想共享**，在 `config.yaml` 加：
```yaml
ais_catcher:
  community_share: "off"    # 显式关闭
```

并在 `start.sh` 里把 `-X $(config_get ais_catcher.community_share)` 拼进命令行。**默认建议 off**（除非你想参与社区）。

### 14.4 调参用 `-M D` + `verbose: 10`

光看 `verbose: 5` 只能知道"收到 N 条"，但不知道**信号质量**。调参关键指标是 `signalpower` 和 `ppm`：

✅ **推荐配置**（config.yaml）：
```yaml
ais_catcher:
  verbose: 10                # 每 1 秒打统计（不是 5 秒）
  meta: "D"                  # 关键！输出 signalpower + ppm
```

调参时从日志找 `signalpower: -XX.X dB`：
| signalpower | 含义 |
|-------------|------|
| -20 ~ -35 dB | 极强信号（船 < 5 km） |
| -35 ~ -50 dB | 良好（典型 5-20 km） |
| -50 ~ -60 dB | 弱信号（> 20 km 或天线差） |
| < -60 dB | 几乎无法解码，调增益/天线 |

`ppm` 如果稳定在 ±3 之外，写到 `config.yaml` 的 `ppm` 字段修正（提升 ~5% 解码率）。

### 14.5 增益起步值

❌ **计划建议**：`gain: 30`（上海信号强 25-35）

✅ **实测**（上海普陀，0:00 凌晨，V4 偶极立式）：
- `gain: 30` → 0 msgs（夜里船少，阈值不够）
- `gain: 42` → 1 msg/10s，signalpower -46 dB

**建议起步值 38-42**（不是 25-35）。等白天船多了再降增益避免饱和。

### 14.6 验收时间窗口

❌ **计划原文**："启动后 30 秒内至少 1 艘船"（1.4.5 自检清单第 5 项）

✅ **实际情况**：
- 白天 8-22 点：满足（黄浦江/苏州河随时 10+ 艘）
- 凌晨 0-5 点：**可能 0 艘**（船都停泊了）
- 建议把验收门槛改成"白天"或加时间窗口

---

*补记 2026-08-04 · 实施 agent：Mavis*
