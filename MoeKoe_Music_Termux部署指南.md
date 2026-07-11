# MoeKoe Music 在 Termux 中部署完整指南
## ARM64设备都可以 / ZeroTermux（Termux） / Node.js 直跑

---

## 零、前置条件

- Android 设备（ARM64架构）
- ZeroTermux（或标准 Termux）
- 网络连接(Watt Toolkit，小猫咪，国内代理源如清华大学代理源等）
- Ai软件（可选建议deepseek或者付费AI可选卡住复制代码问题直接问）

---

## 一、安装基础依赖

Termux / ZeroTermux 新装后没有任何编程工具，全部从头装：

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
git clone --recurse-submodules https://github.com/MakcRe/MoeKoeMusic.git
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

## 四、移动端适配（关键步骤）（此操作需要根据手机调整不建议使用，建议用方案八-方案B）

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

## 五、创建一键启动脚本

```bash
cat > ~/moekoe << 'EOF'
#!/data/data/com.termux/files/usr/bin/bash
cd ~/MoeKoeMusic/api && node app.js --platform=lite --port=6521 &
cd ~/MoeKoeMusic && npx vite --host 0.0.0.0
EOF
chmod +x ~/moekoe
```

---

## 六、启动服务

```bash
~/moekoe
```

看到以下输出表示成功：

```
server running @ http://localhost:6521
VITE v7.x.x ready in xxx ms
➜  Local:   http://localhost:8080/
➜  Network: http://<你的IP>:8080/
```

---

## 七、浏览器访问

手机浏览器打开：

```
http://localhost:8080
```

---

## 八、添加到主屏幕（PWA，类原生 App）

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

> **不支持**：Firefox / 狐猴浏览器（Android 版 Firefox 不支持 PWA 安装）。

---
注：后续使用便捷方案建议给ZeroTermux（Termux）锁定后台+电池省电策略改为无限制
此保活机制依旧存在冻结后台的情况可以添加入游戏助手为利用游戏白名单保活，极客用户直接用软件固化为系统应用保活，例如：爱玩机工具箱
## 九、架构说明

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

## 十、后续更新

```bash
cd ~/MoeKoeMusic
git stash                      # 暂存移动端 CSS 修改(使用步骤八方案可省略过）
git pull
git submodule update --remote --merge
npm install --ignore-scripts   # 如有新依赖
cd api && npm install && cd ..
git stash pop                  # 恢复移动端 CSS(使用步骤八方案可省略过）
```

> 如 stash pop 有冲突，手动将第四步的 `<style>` 块重新插入 `index.html` 即可。

重启：

```bash
# 先停掉旧进程（停用zerotermux/termux），然后重启启动即可
~/moekoe
```

---

## 十一、常见问题

| 问题 | 原因 | 解决 |
|------|------|------|
| `./bin/app_linux: cannot execute` | pkg 二进制依赖 glibc | 改用 `node app.js`，不跑二进制 |
| `npm run serve` 无输出 | 命令不对 | 用 `npx vite --host 0.0.0.0` |
| localhost 拒绝连接 | 前端未启动 | 确保 API 和前端两个进程都在跑 |
| API 返回错误 | 未加参数 | 确认 `--platform=lite --port=6521` |
| 侧边栏太宽挡住内容 | 未注入移动端 CSS | 检查 `index.html` 是否含第四步的 `<style>` |

---

## 十二、端口与进程管理

```bash
# 查看端口占用
netstat -tlnp 2>/dev/null | grep -E '6521|8080'

# 手动杀进程
kill $(lsof -t -i:6521)
kill $(lsof -t -i:8080)

# 或直接
pkill -f "node app.js"
pkill -f "npx vite"
```

---

> 最后更新：2026-07-06
> 设备：骁龙 8 Elite (SM8750) / ZeroTermux / Node.js v25  
> 项目：MoeKoe Music — 高颜值酷狗第三方播放器
