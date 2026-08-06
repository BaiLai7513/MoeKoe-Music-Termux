# MoeKoe Music 在 Termux 中部署完整指南
## ARM64设备都可以 / ZeroTermux（Termux） / Node.js 直跑

---

## 零、前置条件

- **设备**：Android 手机 / 平板，ARM64 架构（绝大多数现代设备均满足）
- **终端**：ZeroTermux 或标准 Termux（任选其一；建议允许 Termux 访问存储，方便读写文件，小白建议选Zerotermux部署AI阅读文档进行部署）
- **网络**：能访问 GitHub 的网络即可。GitHub 源在国内可能较慢或受限，若下载慢/失败，可任选一种加速方式：
  - 代理软件（Clash / v2ray / sing-box 等均可）开 TUN 模式并放行 Termux
  - 或命令行设环境变量 `export https_proxy=http://127.0.0.1:<代理端口>`
  - 或单独配置 git：`git config --global https.proxy http://127.0.0.1:<代理端口>`
  - 只要能正常 `git clone` / 下载到源码即可，**不限制具体软件与方式**
- **存储**：建议剩余空间 2GB 以上（`node_modules` 占用较大）
- **保活（可选但推荐）**：将 Termux 加入系统后台白名单 / 电池策略改为无限制，服务需常驻不被杀
- **Root（可选）**：非必需，仅第十四章「极客方案」需要

---

## 一、安装基础依赖

Termux / ZeroTermux 新装后没有任何编程工具，全部从头装(路径中非询问一律输入Y回车)：

```bash
pkg update && pkg upgrade -y
pkg install nodejs git make python3 -y
```

| 包名 | 用途 |
|------|------|
| nodejs | 跑 API 后端 + Vite 前端 |
| git | 从 GitHub 拉项目代码 |
| make | npm install 时编译原生模块（不装会报错） |
| python3 | node-gyp 编译依赖（不装会报错） |

验证：

```bash
node -v && npm -v && git --version && make --version && python3 --version
```

---

## 二、克隆项目

```bash
cd ~
git clone --recurse-submodules https://github.com/MoeKoeMusic/MoeKoeMusic.git
```

如果 submodule 没自动拉取，手动补：

```bash
cd ~/MoeKoeMusic
git submodule init
git submodule update --remote --merge
```

---

## 三、安装依赖

### 3.1 主项目依赖（跳过 Electron 二进制下载）

```bash
cd ~/MoeKoeMusic
npm install --ignore-scripts
```

> `--ignore-scripts` 跳过 Electron 的 postinstall，因为 Electron 没有 Android 二进制，装了也用不上。

### 3.2 API 子模块依赖

```bash
cd ~/MoeKoeMusic/api
npm install
```

---

## 四、移动端适配（关键步骤）（此操作需要根据手机调整不建议使用，建议用步骤八-方案B）

编辑 `~/MoeKoeMusic/index.html`，在 `</head>` 前插入：

```html
<style>
    @media screen and (max-width: 768px) {
        .side-navigation { width: 60px !important; }
        .side-top-actions { left: 60px !important; padding: 0 10px 0 6px !important; }
        .side-navigation .side-profile-info,
        .side-navigation .side-section-title,
        .side-navigation .side-link span,
        .side-navigation .side-search input,
        .side-navigation .side-playlist-tabs button,
        .side-navigation .side-playlist-tabs span,
        .side-navigation .side-playlist-empty { display: none !important; }
        .side-navigation .side-link { justify-content: center !important; padding: 0 !important; gap: 0 !important; }
        .side-navigation .side-search { justify-content: center !important; padding: 0 !important; }
        .side-navigation .side-playlist-tabs { justify-content: center !important; }
        .side-navigation .side-playlist-list { align-items: center !important; }
        .side-navigation .side-navigation-main { padding: 8px 6px !important; }
    }
</style>
```

也可以用 sed 一键注入：

```bash
cd ~/MoeKoeMusic
sed -i '/<link href="\/assets\/font-awesome\/css\/all.min.css" rel="stylesheet">/a\
    <style>@media screen and (max-width:768px){.side-navigation{width:60px!important}.side-top-actions{left:60px!important;padding:0 10px 0 6px!important}.side-navigation .side-profile-info,.side-navigation .side-section-title,.side-navigation .side-link span,.side-navigation .side-search input,.side-navigation .side-playlist-tabs button,.side-navigation .side-playlist-tabs span,.side-navigation .side-playlist-empty{display:none!important}.side-navigation .side-link{justify-content:center!important;padding:0!important;gap:0!important}.side-navigation .side-search{justify-content:center!important;padding:0!important}.side-navigation .side-playlist-tabs{justify-content:center!important}.side-navigation .side-playlist-list{align-items:center!important}.side-navigation .side-navigation-main{padding:8px 6px!important}}</style>' index.html
```

---

## 五、创建一键启动脚本 `~/moekoe`（非 Root 版，主方案）

> **一个文件搞定**：IP 检测与端口探测逻辑已用 Node.js 内联进脚本，不再需要 `.getip.js` / `.check.js` 两个辅助文件。Termux 非 root 下 `ip addr show` 读不到热点接口、`ss -tlnp` 看不到端口，Node.js 方案实测可用。

```bash
cat > ~/moekoe << 'EOF'
#!/data/data/com.termux/files/usr/bin/bash
# ============================================
# MoeKoe Music 启停脚本（非 Root 版，主方案）
# 内置 IP 检测 + 端口探测，无需 .getip.js/.check.js
# ~/moekoe          → 手机 PWA 模式（API: 127.0.0.1）
# ~/moekoe start    → 电脑浏览器模式（自动检测热点 IP）
# ~/moekoe stop     → 停止服务
# ~/moekoe ip       → 输出热点 IP
# ~/moekoe check    → 输出 API/VITE 端口状态
# ============================================

export PATH="$PATH:/system/bin"
MOEKOE_DIR="$HOME/MoeKoeMusic"
CMD=${1:-local}

# 端口探测：输出 API:UP/DOWN + VITE:UP/DOWN
chk() {
    node -e '
const net = require("net");
function c(p, cb) {
    const s = net.connect({ host: "127.0.0.1", port: p, timeout: 1000 });
    s.on("connect", () => { s.destroy(); cb(true); });
    s.on("error", () => { s.destroy(); cb(false); });
    s.on("timeout", () => { s.destroy(); cb(false); });
}
c(6521, a => c(8080, v => {
    console.log("API:" + (a ? "UP" : "DOWN"));
    console.log("VITE:" + (v ? "UP" : "DOWN"));
}));
' 2>/dev/null
}

# 热点 IP 检测（优先 wlan2/ap0/usb0，兜底任意局域网 IPv4）
getip() {
    node -e '
const os = require("os");
const n = os.networkInterfaces();
const P = ["wlan2", "ap0", "usb0", "wlan1", "softap0"];
const B = /^(rmnet|r_rmnet|meta|tun|ppp)/;
const R = /^(127\.|10\.|100\.|198\.18)/;
let ip = null;
for (const name of P) {
    if (!n[name]) continue;
    for (const a of n[name]) {
        if (a.family === "IPv4" && !a.internal && !R.test(a.address)) { ip = a.address; break; }
    }
    if (ip) break;
}
if (!ip) {
    for (const name of Object.keys(n)) {
        if (B.test(name)) continue;
        for (const a of n[name]) {
            if (a.family === "IPv4" && !a.internal && !R.test(a.address)) { ip = a.address; break; }
        }
        if (ip) break;
    }
}
if (ip) console.log(ip);
' 2>/dev/null
}

api_running()  { chk | grep -q "API:UP"; }
vite_running() { chk | grep -q "VITE:UP"; }

stop_all() {
    pkill -f "node app\.js" 2>/dev/null
    pkill -f "$MOEKOE_DIR.*vite" 2>/dev/null
    pkill -f "$MOEKOE_DIR.*sass" 2>/dev/null
    sleep 1
}

wait_api() {
    local i=0
    while [ $i -lt 10 ]; do
        chk | grep -q "API:UP" && return 0
        sleep 1
        i=$((i+1))
    done
    return 1
}

case "$CMD" in
    ip)
        getip
        ;;
    check)
        chk
        ;;
    stop)
        echo "→ 停止 MoeKoe..."
        if ! api_running && ! vite_running; then
            echo "没有运行中的 MoeKoe 服务"
            exit 0
        fi
        stop_all
        if api_running || vite_running; then
            echo "⚠ 端口仍被占用："
            chk
        else
            echo "→ MoeKoe 已停止"
        fi
        exit 0
        ;;
    start)
        if api_running || vite_running; then
            echo "MoeKoe 已在运行（先 ~/moekoe stop）"
            exit 0
        fi
        stop_all
        echo "→ 启动 MoeKoe Music（电脑模式）..."
        cd "$MOEKOE_DIR/api" && nohup node app.js --platform=lite --port=6521 >/dev/null 2>&1 &
        if wait_api; then
            echo "→ API 已启动 (6521)"
        else
            echo "⚠ API 启动失败，手动检查："
            echo "  cd $MOEKOE_DIR/api && node app.js --platform=lite --port=6521"
            exit 1
        fi
        HOTSPOT_IP=$(getip)
        if [ -n "$HOTSPOT_IP" ]; then
            export VITE_APP_API_URL="http://${HOTSPOT_IP}:6521"
            echo "→ 热点 IP: $HOTSPOT_IP"
            echo "→ 电脑访问 http://${HOTSPOT_IP}:8080"
        else
            echo "⚠ 未检测到热点 IP，请手动执行："
            echo "  export VITE_APP_API_URL=http://<热点IP>:6521 && npx vite --host 0.0.0.0"
        fi
        cd "$MOEKOE_DIR" && npx vite --host 0.0.0.0
        ;;
    *)
        if api_running || vite_running; then
            echo "MoeKoe 已在运行（先 ~/moekoe stop）"
            exit 0
        fi
        stop_all
        echo "→ 启动 MoeKoe Music（手机 PWA）..."
        cd "$MOEKOE_DIR/api" && nohup node app.js --platform=lite --port=6521 >/dev/null 2>&1 &
        if wait_api; then
            echo "→ API 已启动 (6521)"
        else
            echo "⚠ API 启动失败，手动检查："
            echo "  cd $MOEKOE_DIR/api && node app.js --platform=lite --port=6521"
            exit 1
        fi
        echo "→ 手机 PWA http://localhost:8080"
        cd "$MOEKOE_DIR" && npx vite --host 0.0.0.0
        ;;
esac
EOF
chmod +x ~/moekoe
```

验证：

```bash
bash -n ~/moekoe && echo "语法 OK"
```

> 脚本状态判断用内置端口探测，IP 检测用内置 `getip`，均不需要 root。**注意**：脚本中 `node -e` 内的 JS 必须用单引号包裹（bash 不展开 `$`），勿改为双引号。

---
## 六、启动服务

```bash
~/moekoe          # 手机 PWA 模式（API 走 127.0.0.1）
~/moekoe start    # 电脑浏览器模式（自动检测热点 IP）
~/moekoe ip       # 查询热点 IP（电脑访问地址用）
~/moekoe check    # 查询端口状态（API:UP/DOWN + VITE:UP/DOWN）
```

看到以下输出表示成功：

```
→ API 已启动 (6521)
→ 热点 IP: 192.168.48.180
→ 电脑访问 http://192.168.48.180:8080
VITE v7.x.x ready in xxx ms
➜  Local:   http://localhost:8080/
➜  Network: http://192.168.48.180:8080/
```

> **注意**：Android 热点每次开启 DHCP 网段会变（6.x / 48.x / …），电脑访问地址**以每次启动时脚本打印的 `→ 电脑访问 http://xxx:8080` 为准**，不要用旧地址。

## 七、停止服务

```bash
~/moekoe stop
```

---

## 八、浏览器访问

手机浏览器打开：

```
http://localhost:8080
```

---

## 九、添加到主屏幕（PWA，类原生 App）

### 方案 A：Chrome / Edge / Kiwi（推荐）

1. 打开 `http://localhost:8080`
2. 菜单 `⋮` → **"添加到主屏幕"** / **"安装应用"**
3. 桌面出现 MoeKoe Music 图标，点开即全屏独立窗口

### 方案 B：Edge 桌面模式（保留完整侧边栏）

1. Edge 打开 `http://localhost:8080`
2. 菜单 → **"桌面版网站"**（勾选）
3. 再添加到主屏幕，布局即桌面完整版

### 方案 C：同 WiFi 下电脑访问

电脑浏览器打开 `http://<手机IP>:8080`（启动日志中 Network 地址），同界面可直接使用。

> **常见踩坑**：部分路由器默认开启 **AP 隔离/客户端隔离**，WiFi 设备之间互不可见，即使同子网也 ping 不通。
>
> **根治方案**：进路由器后台（浏览器访问 `192.168.1.1` 或 `192.168.0.1`），在「WiFi 设置」「无线设置」「高级设置」中找到 **「AP 隔离」「客户端隔离」「无线隔离」** 开关，关闭即可。关闭后无需切换热点，同 WiFi 下直接 `http://<手机WiFi IP>:8080` 访问。若无法登录路由器（如校园网/公司网络），则用下方方案 D 热点直连绕过。

---

### 方案 D：手机热点直连（绕过 AP 隔离）

当路由器开了 AP 隔离导致电脑 ping 不通手机时，用手机热点建立直连局域网。

#### D.1 确认连通

电脑连上手机热点后，查手机热点 IP（Termux 内）：

```bash
~/moekoe ip
```

示例输出：`192.168.48.180`，这就是热点子网中手机的 IP。

电脑 ping 这个 IP 确认连通。

#### D.2 为什么默认启动脚本不行

前端 `src/utils/apiBaseUrl.js` 默认 API 地址为 `http://127.0.0.1:6521`。电脑浏览器访问时，`127.0.0.1` 指向的是**电脑自己**，而非手机。必须通过环境变量 `VITE_APP_API_URL` 覆写。

#### D.3 启动（电脑模式）

直接用快捷命令：

```bash
~/moekoe start
```

会自动检测热点 IP 并注入 `VITE_APP_API_URL=http://<热点IP>:6521`，无需手动设置。

手动方式（备选）：

```bash
cd ~/MoeKoeMusic
VITE_APP_API_URL=http://<热点IP>:6521 npx vite --host 0.0.0.0 --port 8080
```

> 热点 IP 每次开启会变，以 `~/moekoe ip` 输出或 `~/moekoe start` 打印的地址为准。

#### D.4 电脑访问

```
http://<手机热点IP>:8080
```

#### D.5 排查表

| 现象 | 原因 | 解决 |
|------|------|------|
| 电脑 ping 不通手机 | AP 隔离或不同子网 | 切手机热点（本方案） |
| 页面白屏/一直加载 | API 地址指向 127.0.0.1 | 设置 `VITE_APP_API_URL` |
| 能看到页面但数据加载失败 | API 端口不对 | `~/moekoe check` 确认 API:UP |
| 电脑浏览器访问超时 | API 进程未启动（`~/moekoe` 中 `&` 后台启动可能失败） | `~/moekoe check`，若 API:DOWN 则手动启动 API |
| F12 Console 报 CORS | Vite 未监听所有接口 | 确认 `--host 0.0.0.0` |

> **不支持**：Firefox / 狐猴浏览器（Android 版 Firefox 不支持 PWA 安装）。

---
注：后续使用便捷方案建议给ZeroTermux（Termux）锁定后台+电池省电策略改为无限制
此保活机制依旧存在冻结后台的情况，可以添加入游戏助手为利用游戏白名单保活有后台挂机可启用用此保活，极客用户直接用软件固化为系统应用保活，例如：爱玩机工具箱/Scene
## 十、架构说明

```
┌─────────────────────────────────────┐
│  浏览器 / PWA                        │
│  http://localhost:8080               │
│  (Vite 前端，Vue.js + VitePWA)       │
└──────────────┬──────────────────────┘
               │ API 请求
               ▼
┌─────────────────────────────────────┐
│  Node.js 进程 (Termux)               │
│  node app.js --platform=lite        │
│  http://localhost:6521               │
│  (酷狗音乐 API 后端)                  │
└──────────────┬──────────────────────┘
               │
               ▼
          酷狗音乐服务器
```

- **前端**：Vite 开发服务器，端口 8080
- **后端**：Node.js 直跑 `app.js`，端口 6521
- **为什么不用 pkg 二进制**：`pkg` 编译出的 ARM64 二进制依赖 glibc，Termux 使用 Android Bionic libc，无法运行
- **为什么不用 Electron**：Electron 没有 Android ARM64 二进制，`--ignore-scripts` 跳过
- **这和 Docker 本质一样**：两个进程（API + 前端），只是没打容器包

---

## 十一、后续更新

```bash
cd ~/MoeKoeMusic
git stash                      # 暂存移动端 CSS 修改
git pull
git submodule update --remote --merge
npm install --ignore-scripts   # 如有新依赖
cd api && npm install && cd ..
git stash pop                  # 恢复移动端 CSS
```

> 如 stash pop 有冲突，手动将第四步的 `<style>` 块重新插入 `index.html` 即可。

重启：

```bash
# 先停掉旧进程（停用zerotermux/termux），然后重启启动即可
~/moekoe
```

---

## 十二、常见问题

| 问题 | 原因 | 解决 |
|------|------|------|
| `./bin/app_linux: cannot execute` | pkg 二进制依赖 glibc | 改用 `node app.js`，不跑二进制 |
| `npm run serve` 无输出 | 命令不对 | 用 `npx vite --host 0.0.0.0` |
| localhost 拒绝连接 | 前端未启动 | 确保 API 和前端两个进程都在跑 |
| API 返回错误 | 未加参数 | 确认 `--platform=lite --port=6521` |
| 电脑浏览器访问超时 | API 进程未启动（`~/moekoe` 的 `&` 后台可能失败） | `~/moekoe check` 确认 API:UP，没跑则手动启动 |
| 侧边栏太宽挡住内容 | 未注入移动端 CSS | 检查 `index.html` 是否含第四步的 `<style>` |

---

## 十三、端口与进程管理

### 停用服务

```bash
~/moekoe stop
```

或手动杀：

```bash
# 查看运行状态（端口探测，非 root 可用）
~/moekoe check

# 手动杀进程
pkill -f "node app.js"
pkill -f "$HOME/MoeKoeMusic.*vite"
pkill -f "$HOME/MoeKoeMusic.*sass"
```

> 非 root 下 `ss -tlnp` 无端口读取权限，运行状态一律用 `~/moekoe check` 判断。

---

## 十四、极客可选方案：Root 版脚本（替代步骤五&六）

> 主方案 `~/moekoe` 为非 Root 版（纯 Termux 权限），已满足日常使用。
> 以下 Root 版仅推荐给已 root 的极客用户，用于解决特殊场景。

### 适用场景

| 场景 | 说明 |
|------|------|
| 清理 root 残留进程 | 之前用 root 启动过 MoeKoe，残留进程非 Root 版杀不掉（端口 8080/6521 被占） |
| 端口被顽固占用 | root 级 `pkill -9` 强制清理任意 uid 进程 |
| 状态检查更准确 | root 下 `ss -tlnp` 完整可见，能看进程 pid |

### 安装

```bash
cat > ~/moekoe_root << 'EOF'
#!/data/data/com.termux/files/usr/bin/bash
# MoeKoe Music Root 版（极客可选方案）
export PATH="$PATH:/system/bin"
MOEKOE_DIR="$HOME/MoeKoeMusic"
S() { su -c "$*" 2>/dev/null; }
getip() {
    node -e '
const os = require("os");
const n = os.networkInterfaces();
const P = ["wlan2", "ap0", "usb0", "wlan1", "softap0"];
const B = /^(rmnet|r_rmnet|meta|tun|ppp)/;
const R = /^(127\.|10\.|100\.|198\.18)/;
let ip = null;
for (const name of P) {
    if (!n[name]) continue;
    for (const a of n[name]) {
        if (a.family === "IPv4" && !a.internal && !R.test(a.address)) { ip = a.address; break; }
    }
    if (ip) break;
}
if (!ip) {
    for (const name of Object.keys(n)) {
        if (B.test(name)) continue;
        for (const a of n[name]) {
            if (a.family === "IPv4" && !a.internal && !R.test(a.address)) { ip = a.address; break; }
        }
        if (ip) break;
    }
}
if (ip) console.log(ip);
' 2>/dev/null
}
api_up()  { S "ss -tln | grep -q 6521"; }
vite_up() { S "ss -tln | grep -q 8080"; }
cleanup() {
    S "pkill -9 -f 'node app.js' 2>/dev/null; \
       pkill -9 -f '$MOEKOE_DIR.*vite' 2>/dev/null; \
       pkill -9 -f 'npm exec vite' 2>/dev/null; \
       pkill -9 -f '$MOEKOE_DIR.*sass' 2>/dev/null"
}
CMD=${1:-local}
case "$CMD" in
    stop)
        echo "→ 停止 MoeKoe（Root 清理）..."
        if ! api_up && ! vite_up; then echo "没有运行中的 MoeKoe 服务"; exit 0; fi
        cleanup; sleep 1
        if api_up || vite_up; then
            echo "⚠ 端口仍被占用："; S "ss -tlnp | grep -E '6521|8080'"
        else
            echo "→ MoeKoe 已停止"
        fi
        exit 0 ;;
    start)
        echo "→ 启动 MoeKoe Music（Root 版）..."
        cleanup; sleep 1
        if api_up || vite_up; then
            echo "⚠ 端口仍被占用，清理失败："; S "ss -tlnp | grep -E '6521|8080'"; exit 1
        fi
        cd "$MOEKOE_DIR/api" && nohup node app.js --platform=lite --port=6521 >/dev/null 2>&1 &
        i=0; while [ $i -lt 10 ]; do
            api_up && { echo "→ API 已启动 (6521)"; break; }
            sleep 1; i=$((i+1))
        done
        [ $i -ge 10 ] && { echo "⚠ API 启动失败"; exit 1; }
        HOTSPOT_IP=$(getip)
        if [ -n "$HOTSPOT_IP" ]; then
            export VITE_APP_API_URL="http://${HOTSPOT_IP}:6521"
            echo "→ 热点 IP: $HOTSPOT_IP"
            echo "→ 电脑访问 http://${HOTSPOT_IP}:8080"
        fi
        cd "$MOEKOE_DIR" && npx vite --host 0.0.0.0 ;;
    *)
        if api_up || vite_up; then echo "MoeKoe 已在运行，先执行 ~/moekoe_root stop"; exit 0; fi
        echo "→ 启动 MoeKoe Music（Root 版，手机 PWA）..."
        cleanup; sleep 1
        cd "$MOEKOE_DIR/api" && nohup node app.js --platform=lite --port=6521 >/dev/null 2>&1 &
        i=0; while [ $i -lt 10 ]; do
            api_up && { echo "→ API 已启动 (6521)"; break; }
            sleep 1; i=$((i+1))
        done
        echo "→ 手机 PWA http://localhost:8080"
        cd "$MOEKOE_DIR" && npx vite --host 0.0.0.0 ;;
esac
EOF
chmod +x ~/moekoe_root
```

### 用法

```bash
~/moekoe_root start   # 电脑模式（先 root 清理残留再启动）
~/moekoe_root stop    # root 强制清理
```

> **注意**：Root 版 `start` 会先强制清理所有 MoeKoe 相关进程，如与其他 root 进程冲突需谨慎。
> 日常推荐使用非 Root 版 `~/moekoe`。

---

> 最后更新：2026-08-06
> 设备：骁龙 8 Elite (SM8750) / ZeroTermux / Node.js v25  
