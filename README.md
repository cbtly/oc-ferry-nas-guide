# OC Ferry NAS 实战：从源码到 24 小时在线

> 实战记录：2026 年 10 月 15 日，OC Ferry 0.3.0 从 Windows 迁移到 NAS（J3160 / fnOS），systemd 常驻运行，9 个免费模型全部可用，自研中文 WebUI 上线。
>
> **兼容性说明**：本教程在 fnOS（J3160 小主机）上实测通过。核心步骤（Node v22 + systemd 服务 + 环境变量）是标准 Linux 操作，任何带 systemd 的 NAS（如群晖 DSM 7+、威联通 QTS、OpenMediaVault）照着调整路径即可用；TrueNAS（FreeBSD 无 systemd）需改用其自身的服务管理机制。文中 `/vol1/1000/` 是 fnOS 的存储路径，其他系统请替换为实际路径。

---

## 目录

- [1. 简介](#1-简介)
- [2. 环境准备](#2-环境准备)
- [3. 迁移步骤](#3-迁移步骤)
- [4. 避坑指南](#4-避坑指南)
- [5. 进阶](#5-进阶)
- [6. 参考](#6-参考)
- [7. 兼容性](#7-兼容性)

---

## 1. 简介

### OC Ferry 是什么

OC Ferry 是一个桥接服务，把 [OpenCode](https://github.com/sst/opencode) 的免费模型通过 OpenAI 兼容的 API 暴露出来。任何支持自定义 base URL 的客户端（WorkBuddy、Hermes、Codex 等）都可以接入使用。

核心能力：

- OpenAI 兼容的 `/v1` 转发接口（chat completions）
- 启动时自动探测模型可用性（含工具调用能力）
- API Key 鉴权
- 自研中文 WebUI（`/ui`）：状态监控、模型列表、一键检测、代理开关

### 为什么搬到 NAS

| 场景 | Windows 跑 | NAS 跑 |
|------|-----------|--------|
| 开机要求 | 电脑必须开着 | 7x24 常年在线 |
| 局域网访问 | 电脑休眠就断 | 随时可用 |
| 多设备共享 | 得逐台配置 | 一个地址全家用 |
| 功耗 | 几十瓦 | J3160 十几瓦 |

一句话：NAS 版是一次部署、长期受益。手机、电脑、平板，局域网内任何设备填一个地址就能用。

---

## 2. 环境准备

### 2.1 NAS 要求

- 能 SSH 登录（root 或有 sudo 权限的用户）
- 有 systemd（fnOS / Debian / Ubuntu 系都自带）
- 至少 2GB 可用磁盘空间（含 OpenCode 二进制约 100MB + Node 运行时）

本次实战环境：

```
设备：J3160 小主机（4GB 内存）
系统：fnOS
主机名：hiyori
IP：192.168.31.53
部署目录：/vol1/1000/oc-ferry
数据目录：/vol1/1000/oc-ferry/data
服务端口：41980
```

### 2.2 Node.js v22+（关键）

**这是最重要的前置条件。** NAS 自带的 Node 往往是 v18，OC Ferry 依赖的 `undici` 在 Node 18 下加载失败，服务会直接退出（详见[避坑指南](#41-node-18-vs-22)）。

检查当前版本：

```bash
node -v
# 如果输出 v18.x，必须升级
```

安装 Node v22 二进制版（以 x64 为例，ARM 机型请下载对应的 arm64 包）：

```bash
cd /tmp
curl -sL https://nodejs.org/dist/v22.22.0/node-v22.22.0-linux-x64.tar.xz -o node.tar.xz
tar -xf node.tar.xz
cp -r node-v22.22.0-linux-x64/bin/* /usr/local/bin/
cp -r node-v22.22.0-linux-x64/lib/* /usr/local/lib/ 2>/dev/null
cp -r node-v22.22.0-linux-x64/include/* /usr/local/include/ 2>/dev/null
cp -r node-v22.22.0-linux-x64/share/* /usr/local/share/ 2>/dev/null
/usr/local/bin/node -v
# 应输出 v22.22.0
```

> 注意：不要覆盖系统自带的 `/usr/bin/node`，把 v22 装到 `/usr/local` 即可。`/usr/local/bin` 在 PATH 中的优先级高于 `/usr/bin`，且 systemd 服务里我们会写绝对路径。

### 2.3 源码准备（在开发机上）

假设你已经有一份能跑的 OC Ferry 源码（含 `node_modules` 已安装）：

```bash
cd ~/workspace/oc-ferry-build
tar --exclude='RuiC-oc-ferry/node_modules' \
    --exclude='RuiC-oc-ferry/release' \
    --exclude='RuiC-oc-ferry/.git' \
    -czf oc-ferry-nas.tar.gz RuiC-oc-ferry
ls -lh oc-ferry-nas.tar.gz
# 约 3-7MB（不含 node_modules）
```

> `node_modules` 不打包，到 NAS 上重新 `npm install --production`，避免架构不一致（比如开发机是 Windows，NAS 是 Linux）。

---

## 3. 迁移步骤

### 3.1 传输到 NAS

用你顺手的方式传上去（scp、rsync、网盘、catbox 等）：

```bash
# 示例：scp
scp oc-ferry-nas.tar.gz root@192.168.31.53:/vol1/1000/oc-ferry/
```

### 3.2 解压并安装依赖

```bash
ssh root@192.168.31.53
mkdir -p /vol1/1000/oc-ferry
cd /vol1/1000/oc-ferry
tar -xzf oc-ferry-nas.tar.gz
cd RuiC-oc-ferry
npm install --production
```

### 3.3 创建 systemd 服务

```bash
cat > /etc/systemd/system/oc-ferry.service << 'EOF'
[Unit]
Description=OC Ferry - OpenCode free models bridge
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=root
WorkingDirectory=/vol1/1000/oc-ferry/RuiC-oc-ferry
Environment=FERRY_HOST=0.0.0.0
Environment=FERRY_PORT=41980
Environment=FERRY_DATA_DIR=/vol1/1000/oc-ferry/data
Environment=FERRY_NO_SYNC=1
ExecStart=/usr/local/bin/node src/main.js
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
EOF
```

关键配置说明：

| 配置项 | 作用 |
|--------|------|
| `FERRY_HOST=0.0.0.0` | 监听所有网卡，局域网可访问（默认 127.0.0.1 只能本机） |
| `FERRY_PORT=41980` | 服务端口 |
| `FERRY_DATA_DIR` | 数据目录（API Key、模型探测缓存都放这里） |
| `FERRY_NO_SYNC=1` | 跳过 WorkBuddy 同步（NAS 上不需要） |
| `ExecStart=/usr/local/bin/node` | **必须指向 Node v22**，不能用系统自带的 v18 |
| `Restart=always` | 崩溃自动重启 |

### 3.4 启动服务

```bash
systemctl daemon-reload
systemctl enable oc-ferry
systemctl start oc-ferry
sleep 10
systemctl status oc-ferry --no-pager | head -12
```

正常应看到 `Active: active (running)`。

### 3.5 确认启动完成

首次启动会自动下载 OpenCode 二进制，需要几分钟。看日志确认：

```bash
journalctl -u oc-ferry --no-pager | grep -i "ready\|opencode" | tail -5
```

看到类似下面的输出就是就绪了：

```
OC Ferry ready at http://127.0.0.1:41980/v1; 9 free models; probe cache hit 0, probing 9
```

也可以直接查健康检查接口：

```bash
curl -s http://127.0.0.1:41980/health \
  -H "Authorization: Bearer $(cat /vol1/1000/oc-ferry/data/api-key)" | head -c 300
```

`phase` 为 `ready` 即正常。

### 3.6 获取 API Key 并打开 WebUI

```bash
cat /vol1/1000/oc-ferry/data/api-key
```

复制输出的密钥，在浏览器打开：

```
http://192.168.31.53:41980/ui
```

粘贴密钥后，可以看到模型列表、逐个检测状态、一键重新检测、系统代理开关等。

转发接口地址（给客户端用）：

```
http://192.168.31.53:41980/v1
```

---

## 4. 避坑指南

### 4.1 Node 18 vs 22

**现象：**

```bash
journalctl -u oc-ferry --no-pager | tail -20
# 看到 undici 加载失败，Node.js v18.20.4，服务状态 1/FAILURE
```

**根因：**

NAS 系统自带的 Node 是 v18，OC Ferry 依赖的 `undici` 库在 Node 18 下无法加载。服务启动即崩溃，systemd 反复重启也无济于事。

**解法：**

按 [2.2](#22-nodejs-v22关键) 安装 Node v22 到 `/usr/local`，并确保 service 文件的 `ExecStart` 写的是绝对路径：

```ini
# 错误
ExecStart=/usr/bin/node src/main.js
# 正确
ExecStart=/usr/local/bin/node src/main.js
```

验证：

```bash
grep ExecStart /etc/systemd/system/oc-ferry.service
/usr/local/bin/node -v   # 应输出 v22.x
```

### 4.2 修改 service 文件后必须 daemon-reload

**现象：**

改了 `ExecStart` 路径，重启服务，日志里还是旧的 Node 版本。

**根因：**

systemd 缓存了服务文件，直接 `systemctl restart` 不会重新读取。

**解法：**

```bash
systemctl daemon-reload
systemctl restart oc-ferry
```

每次改完 `/etc/systemd/system/oc-ferry.service` 都要跑一遍 `daemon-reload`。

### 4.3 FERRY_HOST 绑定

**现象：**

NAS 本机 `curl http://127.0.0.1:41980/health` 正常，但局域网其他设备打不开 `http://192.168.31.53:41980/ui`。

**根因：**

默认 `FERRY_HOST=127.0.0.1` 只监听回环地址。

**解法：**

```ini
Environment=FERRY_HOST=0.0.0.0
```

改完记得 `daemon-reload` + `restart`。

> 安全提示：`0.0.0.0` 意味着局域网内任何设备可访问。API Key 是唯一的鉴权手段，请妥善保管，不要暴露到公网。

### 4.4 API Key 位置

**现象：**

找不到密钥，WebUI 提示未授权。

**根因：**

密钥不在配置文件里，是首次启动时随机生成的，存放在数据目录。

**解法：**

```bash
cat $FERRY_DATA_DIR/api-key
# 即 /vol1/1000/oc-ferry/data/api-key
```

### 4.5 首次启动下载 OpenCode

**现象：**

服务显示 running，但 `/health` 返回 `phase: starting`，或日志里没有 "OC Ferry ready"。

**根因：**

首次启动要从网络下载 OpenCode 二进制（约 100MB），下载完成前服务处于 starting 阶段。

**解法：**

耐心等几分钟，看日志：

```bash
journalctl -u oc-ferry -f
```

看到 `OC Ferry ready at ...` 即完成。如果长时间卡住，检查 NAS 网络是否正常。

---

## 5. 进阶

### 5.1 textMode：解决部分模型的工具调用不兼容

**背景：**

OC Ferry 通过 OpenCode 的 `format: { type: 'json_schema' }` 机制约束模型输出结构化 JSON。但某些模型的上游端点不兼容这个机制，探测时会报：

```
only 'auto' is supported for 'tool_choice'
```

探测程序会误判该模型"不支持工具"，将其标记为"仅对话"（chatOnly）。实测 `opencode/muse-spark-1.3-contributor-free` 就遇到这个问题——而模型本身是支持工具调用的。

**解法（textMode 降级路径）：**

当探测到特定的 `tool_choice` 不兼容错误时，自动用纯文本模式重试：请求时省略 `format` 字段，模型返回纯文本中的 JSON 信封，由桥接层现有的 `decode()` 解析。

WebUI 中这类模型显示为 `可用·文本模式`，工具调用功能完整。`textMode` 标记会持久化到探测缓存，重启后依然有效。

### 5.2 推理档位自动降级

**背景：**

免费模型的推理强度（reasoning effort）通常有上限。例如 Muse Spark 1.3 免费档最高只到 `high`，客户端如果传 `reasoning_effort: max`，会直接报 400：

```
Requested reasoning effort is not available for this model
```

然后客户端掉到备用模型，用户体验很差。

**解法：**

桥接层自动降级：请求档位超出模型支持范围时，自动使用"不超过请求档位的最高可用档"（如 `max` → `high`），而不是报错。档位顺序为：

```
none < minimal < low < medium < high < xhigh < max
```

特殊规则：

- 请求 `none`（关闭推理）但模型不支持时，**不降级**，仍然 400——悄悄打开推理是错误行为
- 降级信息记录在请求对象上，可供日志追踪

### 5.3 接入客户端

OC Ferry 是标准的 OpenAI 兼容接口，接入三件套：

| 项 | 值 |
|---|---|
| Base URL | `http://192.168.31.53:41980/v1` |
| API Key | `$FERRY_DATA_DIR/api-key` 的内容 |
| 模型 ID | `opencode/muse-spark-1.3-contributor-free` 等 |

获取完整模型列表：

```bash
curl -s http://192.168.31.53:41980/v1/models \
  -H "Authorization: Bearer YOUR_API_KEY" | python3 -m json.tool
```

**WorkBuddy 批量导入示例：**

WorkBuddy 的自定义模型存在 `%USERPROFILE%\.workbuddy\models.json`（注意是带点的隐藏目录）。可以直接编辑 JSON 批量添加：

```json
{
  "id": "opencode/muse-spark-1.3-contributor-free",
  "name": "opencode/muse-spark-1.3-contributor-free",
  "vendor": "Custom",
  "url": "http://192.168.31.53:41980/v1",
  "apiKey": "YOUR_API_KEY",
  "supportsToolCall": true,
  "supportsImages": true,
  "supportsReasoning": true,
  "useCustomProtocol": false,
  "reasoning": {
    "supportedEfforts": ["high", "medium", "low"]
  },
  "maxInputTokens": 1000000
}
```

改完重启 WorkBuddy 即可。

---

## 6. 参考

### 6.1 环境变量

| 变量 | 默认值 | 说明 |
|------|--------|------|
| `FERRY_HOST` | `127.0.0.1` | 监听地址，局域网访问设为 `0.0.0.0` |
| `FERRY_PORT` | `41980` | 服务端口 |
| `FERRY_DATA_DIR` | `./data` | 数据目录（API Key、缓存） |
| `FERRY_NO_SYNC` | `0` | 设为 `1` 跳过 WorkBuddy 同步 |

### 6.2 API 端点

| 端点 | 说明 |
|------|------|
| `GET /health` | 健康检查（需鉴权），返回 phase、模型数、版本 |
| `GET /v1/models` | 模型列表（OpenAI 格式） |
| `POST /v1/chat/completions` | 对话补全（OpenAI 格式） |
| `GET /ui` | 中文 WebUI 管理面板 |
| `GET /ui/api/status` | WebUI 状态接口（需 Bearer Key） |
| `POST /ui/api/probe` | WebUI 触发模型检测（需 Bearer Key） |

### 6.3 文件位置

| 路径 | 说明 |
|------|------|
| `/vol1/1000/oc-ferry/RuiC-oc-ferry` | 程序目录 |
| `/vol1/1000/oc-ferry/data/api-key` | API 密钥 |
| `/vol1/1000/oc-ferry/data/` | 探测缓存等运行时数据 |
| `/etc/systemd/system/oc-ferry.service` | systemd 服务文件 |
| `/usr/local/bin/node` | Node v22 二进制 |

### 6.4 常用运维命令

```bash
# 查看状态
systemctl status oc-ferry --no-pager | head -12

# 实时日志
journalctl -u oc-ferry -f

# 重启
systemctl restart oc-ferry

# 开机自启
systemctl enable oc-ferry

# 健康检查
curl -s http://127.0.0.1:41980/health \
  -H "Authorization: Bearer $(cat /vol1/1000/oc-ferry/data/api-key)"
```

---

## 7. 兼容性

### 7.1 实测环境

| 项 | 值 |
|---|---|
| 设备 | J3160 小主机（4GB 内存） |
| 系统 | fnOS |
| Node | v22.22.0（二进制安装到 `/usr/local`） |
| 服务管理 | systemd |
| OC Ferry | 0.3.0 |

### 7.2 其他 NAS 系统

| 系统 | 可行性 | 说明 |
|------|--------|------|
| 群晖 DSM 7+ | 可行 | 有 systemd，照本教程调整存储路径即可 |
| 威联通 QTS | 可行 | 基于 Linux，有 systemd，注意权限配置 |
| OpenMediaVault | 可行 | Debian 系，原生 systemd |
| TrueNAS | 需调整 | FreeBSD 内核，无 systemd，需改用其服务管理机制 |
| Unraid | 可行 | 可用 Docker 或手动配置服务 |

核心要求只有三条：**Linux 内核 + Node v22 + 能常驻运行进程的方式**（systemd / Docker / 其他）。满足这三条，本教程的思路都适用，具体命令按各家系统的习惯调整。

### 7.3 模型数量说明

教程中提到的"9 个免费模型"是 2026 年 10 月 15 日实测时的数量。OpenCode 的免费模型目录会随时间变化（增减、更名），实际以 `/v1/models` 返回为准，不必纠结数字是否对得上。

---

*英文版见 [README_EN.md](README_EN.md)*
