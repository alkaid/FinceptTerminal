# FinceptTerminal 本地开发运行手册

这份手册记录本仓库在 Linux 开发机上的一次完整初始化、构建、免费数据源配置和启动过程。目标是给后续同事或 agent 一个可复用的操作基线，尤其说明当前上游文档和实际 Qt 客户端之间的差异。

当前验证环境：

- 仓库：`/home/alkaid/proj/github/alkaid/FinceptTerminal`
- 系统：Linux，桌面会话 `DISPLAY=:1`
- Qt：6.8.3，本地路径 `.qt/6.8.3/gcc_64`
- 构建预设：`linux-release`
- 日期：2026-05-25

## 0. 最短路径

新机器优先跑官方脚本，它会装系统依赖、下载 Qt 6.8.3、配置 CMake preset 并编译：

```bash
cd /home/alkaid/proj/github/alkaid/FinceptTerminal
./setup.sh
```

如果只是自动化初始化和构建，不要进入交互式启动：

```bash
./setup.sh --ci
```

普通 `./setup.sh` 构建完成后会问：

```text
Launch now? (y/n):
```

本地免登录模式建议这里选 `n`，然后用下面命令启动。原因是官方脚本最后直接执行编译产物，不会自动带上 `FINCEPT_LOCAL_ONLY=1`，直接启动会走上游默认登录流程。

```bash
cd /home/alkaid/proj/github/alkaid/FinceptTerminal

setsid env \
  FINCEPT_LOCAL_ONLY=1 \
  QT_TLS_BACKEND=openssl \
  LD_LIBRARY_PATH="$PWD/.qt/6.8.3/gcc_64/lib:${LD_LIBRARY_PATH:-}" \
  ./fincept-qt/build/linux-release/FinceptTerminal \
  >> "$HOME/.local/share/com.fincept.terminal/logs/fincept-stdout.log" \
  2>&1 < /dev/null &
```

如果要看前台日志，用：

```bash
cd /home/alkaid/proj/github/alkaid/FinceptTerminal

env \
  FINCEPT_LOCAL_ONLY=1 \
  QT_TLS_BACKEND=openssl \
  LD_LIBRARY_PATH="$PWD/.qt/6.8.3/gcc_64/lib:${LD_LIBRARY_PATH:-}" \
  ./fincept-qt/build/linux-release/FinceptTerminal
```

## 1. Git remote

本地 fork 使用 `origin`，官方仓库加为 `upstream`：

```bash
git remote add upstream https://github.com/Fincept-Corporation/FinceptTerminal.git
git fetch upstream --prune
git remote -v
```

期望输出包含：

```text
origin    git@github.com:alkaid/FinceptTerminal.git
upstream  https://github.com/Fincept-Corporation/FinceptTerminal.git
```

## 2. 依赖和构建

官方要求 Qt 6.8.3、CMake、Ninja、C++20 编译器和 Python 3.11。仓库根目录有官方 `setup.sh`，推荐新机器优先使用它：

```bash
cd /home/alkaid/proj/github/alkaid/FinceptTerminal
./setup.sh
```

`setup.sh` 会做这些事：

- 安装系统构建依赖。Linux 下会调用 `apt-get`、`pacman` 或 `dnf`；macOS 下会使用 Homebrew。
- 检查 g++/clang、CMake、Python 版本。
- 通过独立 `.aqt-venv` 安装 `aqtinstall`。
- 下载并安装 Qt 6.8.3 到 `.qt/6.8.3/<kit>`，除非已经存在。
- 设置 `CMAKE_PREFIX_PATH`。
- 运行对应平台的 CMake preset。
- 编译项目。
- 非 `--ci` 模式下最后会询问是否立即启动。

非交互或 CI 环境可用：

```bash
./setup.sh --ci
```

本次实际操作没有直接跑 `setup.sh`，原因是机器上已经有 `.qt/6.8.3/gcc_64`、Python venv 和现成构建目录；为了定位登录和数据源问题，我直接使用项目 preset 反复构建、重启验证。新机器或依赖不完整时，优先走 `setup.sh` 更省事。

已有依赖时可以手动构建：

```bash
cd /home/alkaid/proj/github/alkaid/FinceptTerminal/fincept-qt
cmake --preset linux-release
cmake --build --preset linux-release --parallel 4
```

成功后可执行文件在：

```text
fincept-qt/build/linux-release/FinceptTerminal
```

如果 CMake 找不到 Qt，确认下面路径存在，或把 `CMAKE_PREFIX_PATH` 指向实际 Qt kit：

```text
/home/alkaid/proj/github/alkaid/FinceptTerminal/.qt/6.8.3/gcc_64
```

`setup.sh` 默认会把 Qt 放在仓库根目录的 `.qt` 下，也可以用环境变量指定安装根目录：

```bash
FINCEPT_QT_ROOT=/path/to/qt-root ./setup.sh
```

## 3. Python 环境

首次启动客户端会检查并准备应用目录下的 Python/uv 环境：

```text
~/.local/share/com.fincept.terminal/
  uv/
  venv-numpy1/
  venv-numpy2/
```

启动日志中看到类似内容表示 Python 环境可用：

```text
Fast-path OK — sentinel + both hash-markers current → needs_setup=false
Using UV-managed venv-numpy2
PythonWorker Daemon ready
```

Python 包主要来自：

- `fincept-qt/resources/requirements-numpy1.txt`
- `fincept-qt/resources/requirements-numpy2.txt`

## 4. 免费数据源配置

本次在本地 SQLite 中配置了以下免 API key provider：

| Alias | Provider | 用途 |
| --- | --- | --- |
| `free_yahoo_finance` | Yahoo Finance | 股票、指数、ETF、外汇、加密行情 |
| `free_coingecko` | CoinGecko Public API | 加密市场数据 |
| `free_kraken_public` | Kraken Public REST | Kraken 公共接口连通性 |
| `free_world_bank_us_gdp` | World Bank | 宏观经济数据示例 |
| `free_dbnomics_providers` | DB.Nomics | 宏观数据 provider 列表 |
| `free_frankfurter_fx` | Frankfurter | 外汇汇率 |

可用 SQLite 检查：

```bash
sqlite3 "$HOME/.local/share/com.fincept.terminal/data/fincept.db" \
  "select alias,display_name,type,provider,category,enabled,config
   from data_sources
   where alias like 'free_%'
   order by alias;"
```

当前验证过的 Yahoo 批量行情日志：

```text
MarketData batch_all OK: quotes=87/87
```

注意：免费源本身可能限流、403、超时或短时不可用。日志里的 RSS 403、新闻源 Operation canceled、个别公共 API 超时通常不是登录问题。

## 5. 本地免登录模式

当前 Qt v4 代码和官方 `docs/GETTING_STARTED.md` 不一致：文档写了 “Continue as Guest”，但实际登录界面原本没有 guest 按钮，认证路径还会强制访问 Fincept 云端 profile/session/subscription 接口。

本地开发不使用 Fincept 服务 API 时，使用显式开关：

```bash
FINCEPT_LOCAL_ONLY=1
```

这个开关的语义：

- 创建本地 guest session：`local@fincept.local`
- 跳过 PIN gate 和 paid-plan gate
- 跳过 Fincept `/user/profile`、session pulse、subscription 刷新
- 禁用 Fincept cloud chat API 路径
- 默认 LLM 从 Fincept 切到本地 Ollama：`provider=ollama model=llama3.1`
- 不把 `local-only` 假 API key 写进 SQLite 或 SecureStorage
- 不设置该环境变量时，上游默认登录行为保持不变

### 本次实际启动方式

本次最终跑通的桌面客户端启动命令如下。它和 `setup.sh` 最后的交互式启动不同，多了 `FINCEPT_LOCAL_ONLY=1`，并把日志重定向到应用日志目录，方便后台运行和复查。

```bash
cd /home/alkaid/proj/github/alkaid/FinceptTerminal

setsid env \
  FINCEPT_LOCAL_ONLY=1 \
  QT_TLS_BACKEND=openssl \
  LD_LIBRARY_PATH="$PWD/.qt/6.8.3/gcc_64/lib:${LD_LIBRARY_PATH:-}" \
  ./fincept-qt/build/linux-release/FinceptTerminal \
  >> "$HOME/.local/share/com.fincept.terminal/logs/fincept-stdout.log" \
  2>&1 < /dev/null &
```

期望日志：

```text
FINCEPT_LOCAL_ONLY enabled — using local guest session without Fincept API authentication
LLM config loaded: provider=ollama model=llama3.1
Application ready
Terminal MCP bridge started: http://127.0.0.1:<port>
```

如果不需要后台运行，可用前台启动：

```bash
cd /home/alkaid/proj/github/alkaid/FinceptTerminal

env \
  FINCEPT_LOCAL_ONLY=1 \
  QT_TLS_BACKEND=openssl \
  LD_LIBRARY_PATH="$PWD/.qt/6.8.3/gcc_64/lib:${LD_LIBRARY_PATH:-}" \
  ./fincept-qt/build/linux-release/FinceptTerminal
```

启动后程序会自动带起 yfinance daemon。进程形态类似：

```text
./fincept-qt/build/linux-release/FinceptTerminal
.../venv-numpy2/bin/python3 .../fincept-qt/scripts/yfinance_data.py --daemon
```

如果安装了/触发了 RD-Agent，它的 Streamlit UI 可能单独运行在 `127.0.0.1:19999`。它不是主客户端，见第 6 节。

## 6. WebUI 和桌面客户端的区别

FinceptTerminal 主界面是 Qt 桌面客户端，目前没有一个和桌面端一致的官方 Web 版。

本次启动中看到的 `http://127.0.0.1:19999` 是 RD-Agent 的 Streamlit 日志/任务 UI：

```text
streamlit run .../site-packages/rdagent/log/ui/app.py --server.port=19999
```

它不是 FinceptTerminal 的 Web 客户端，所以看起来和桌面端完全不一样是正常的。

常见端口：

| 地址 | 含义 |
| --- | --- |
| `http://127.0.0.1:19999` | RD-Agent Streamlit 附属 UI |
| `http://127.0.0.1:<random>` | Terminal MCP bridge，内部 token bridge，不是浏览器 UI |

检查 RD-Agent WebUI：

```bash
curl -I --max-time 5 http://127.0.0.1:19999/
```

返回 `HTTP/1.1 200 OK` 表示它已启动。

## 7. 远程访问建议

因为主程序是 Qt 桌面应用，远程使用建议采用远程桌面方式，而不是暴露内部端口。

推荐方案：

1. Tailscale / WireGuard / ZeroTier 接入内网。
2. 使用 NoMachine、RustDesk、VNC/xrdp 或 noVNC 访问这台机器的桌面。
3. 如果必须浏览器访问，用 noVNC 包装桌面画面，并放在 VPN 或带鉴权的反向代理后面。

不要把 Terminal MCP bridge 暴露到公网。它是本机内部控制桥，不是用户界面。

## 8. 常用运行检查

查看进程：

```bash
ps -eo pid,ppid,stat,etime,cmd | rg 'FinceptTerminal|yfinance_data|streamlit'
```

查看监听端口：

```bash
ss -ltnp | rg '19999|Fincept|streamlit|python'
```

查看日志：

```bash
tail -n 200 "$HOME/.local/share/com.fincept.terminal/logs/fincept.log"
tail -n 120 "$HOME/.local/share/com.fincept.terminal/logs/fincept-stdout.log"
```

验证本地模式没有误打 Fincept 登录 API：

```bash
rg -n 'api\.fincept\.in|/user/profile|/chat/|HTTP 401|Host requires authentication|Auth error' \
  "$HOME/.local/share/com.fincept.terminal/logs/fincept.log"
```

启动后一段时间内不应出现新的登录/profile/chat 401。

## 9. 关停和重启

优先从桌面正常关闭。调试时可以按 PID 停止：

```bash
kill <FinceptTerminal_PID> <yfinance_PID>
```

如果直接 `kill`，下次启动可能触发 crash recovery 对话框。需要自动化验证时，可以写入 clean-shutdown marker：

```bash
now_ms=$(date +%s%3N)
sqlite3 "$HOME/.local/share/com.fincept.terminal/workspace.db" \
  "insert into _meta(key,value)
   values('last_clean_shutdown_at','$now_ms')
   on conflict(key) do update set value=excluded.value;"
```

这相当于跳过上一次被强杀造成的恢复提示。不要在需要恢复用户工作区时这么做。

## 10. 已遇到的坑和处理方式

### 10.1 官方文档的 “Continue as Guest” 和当前代码不一致

现象：

- 文档说打开后点击 “Continue as Guest”。
- 当前 Qt 登录界面没有这个按钮。
- 不登录会被 auth/subscription gate 挡住。

处理：

- 使用 `FINCEPT_LOCAL_ONLY=1` 作为显式本地开发开关。
- 本地模式下可以显示 `CONTINUE AS GUEST`，也可以直接进入 shell。
- 不设置开关时不改变上游默认行为，方便以后合并上游。

### 10.2 本地 guest session 被 profile 校验清掉

现象：

```text
HTTP 401: https://api.fincept.in/user/profile
Profile fetch returned 401/403 — API key invalid, clearing session
```

原因：

本地 session 使用 `local-only` 假 key，但后续 `refresh_user_data()`、`SessionGuard`、profile/subscription 校验仍然把它当云端 API key 使用。

处理：

- `AuthManager` 的初始化、profile、subscription、refresh、session recovery 在本地模式下短路。
- `SessionGuard` 本地模式不启动 pulse。
- `WindowFrame` 的周期刷新和 focus refresh 本地模式不再请求云端。

### 10.3 ChatModeService 仍访问 Fincept cloud chat

现象：

```text
Auth error 401 on /chat/sessions
Auth error 401 on /chat/agent/tasks
```

原因：

Chat Mode 是专门访问 `api.fincept.in/chat/*` 的云端服务，不等于本地 AI Chat 存储。

处理：

- 本地模式下列表类接口返回空列表或 0 credits。
- 创建、stream、agent chat、prompt optimize 等云端动作返回 disabled，不再发网络请求。

### 10.4 默认 LLM provider 指向 Fincept

现象：

启动或使用 AI 时默认 provider 可能是 `fincept`。

处理：

- 本地模式下如果没有用户配置，默认切到 Ollama。
- 如果已有 active provider 是 `fincept`，本地模式也切到 Ollama。
- 需要真正使用本地 LLM 时，请确认 Ollama 服务和模型存在：

```bash
ollama serve
ollama pull llama3.1
```

### 10.5 Python/yfinance TLS `invalid library`

现象：

```text
SSLError('Failed to perform, curl: (35) TLS connect error: ... OPENSSL_internal:invalid library')
```

原因：

桌面端启动时为了找到 bundled Qt，设置了：

```bash
LD_LIBRARY_PATH=$PWD/.qt/6.8.3/gcc_64/lib
```

Python 子进程继承该变量后，`curl_cffi`/OpenSSL 可能加载到不兼容的 Qt runtime 库。

处理：

- `PythonRunner::build_python_env()` 在 Linux 下过滤 repo-bundled Qt runtime library path。
- 主程序仍可用 `LD_LIBRARY_PATH` 启动，Python/yfinance 不再继承污染路径。

验证：

```text
PythonWorker Daemon ready
MarketData batch_all OK: quotes=87/87
```

### 10.6 Crash recovery 阻塞自动化验证

现象：

强杀客户端后，下次启动停在 crash recovery 对话框，日志只有：

```text
Recovery needed: no clean-shutdown marker
```

处理：

- 人工使用时按 UI 选择恢复或跳过。
- 自动化验证时可写入 `workspace.db` 的 `_meta.last_clean_shutdown_at`，见第 9 节。

### 10.7 免费公共源的外部错误

现象：

- RSSHub、FXStreet、Benzinga 等 RSS 403。
- 某些公共 API `Connection closed` 或 `Operation canceled`。
- Yahoo 偶发 timeout。

处理：

- 这些通常不是认证配置问题。
- 先看核心 market data 是否成功，例如 `MarketData batch_all OK`。
- 对免费源做容错预期，不要把单个外部源失败当成客户端启动失败。

## 11. 当前已验证状态

一次成功启动的关键状态：

```text
FinceptTerminal PID: 950183
yfinance daemon PID: 950240
RD-Agent Streamlit PID: 940989
RD-Agent WebUI: http://127.0.0.1:19999
Terminal MCP bridge: http://127.0.0.1:39199
Application ready
MarketData batch_all OK: quotes=87/87
```

PID 和 MCP bridge 端口每次启动都会变化，以实际 `ps` 和日志为准。
