# LANShare Web 端

LANShare Web 端是 LANShare 的**跨平台公共网页界面**，由 LANShare 各端内置的 HTTP 文件服务器统一承载 —— 无论是 **Android 移动端**还是 **PC 桌面端（Win / Mac / Linux）**，对外提供的都是同一套 Web 界面。目标设备启动 LANShare 并开启文件服务后，同一局域网内的其他设备只需在浏览器中打开对应地址，即可在**不安装任何客户端**的情况下浏览、下载、上传文件，收发聊天消息，并实时双向同步绘图。

作为各端共用的前端，本模块与各端保持**一致的协议与交互体验**：页面采用单页应用（SPA）结构，通过 URL hash 进行前端路由，基于 WebSocket 与宿主端保持长连接，实现消息推送、设备列表同步、主题切换和绘图事件的实时下发。宿主端不同（移动端 / PC 端）时，界面与操作方式完全一致，用户无需重新适应。

> 说明：由于本界面被移动端与 PC 端共享，文中出现的「本机 / 宿主端」泛指承载该 Web 界面的 LANShare 客户端，可能是 Android 手机，也可能是桌面程序。

---

## 目录结构

```
web/
├── lanshare.html          # 单页入口，包含侧边栏/头部/五大内容面板及各类弹窗
├── css/
│   ├── bootstrap.min.css  # Bootstrap 基础样式（第三方）
│   ├── style.min.css      # lightyear 后台模板样式（第三方）
│   ├── animate.min.css    # 动画库（第三方）
│   ├── lanshare.css       # 业务布局样式，含响应式断点（639px / 640~1000px / 1000px+）
│   └── theme.css          # 多主题系统，基于 CSS 变量定义 light / dark / emerald 配色
├── js/
│   ├── jquery.min.js      # jQuery（第三方）
│   ├── bootstrap.min.js   # Bootstrap 交互（第三方）
│   ├── bootstrap-notify.min.js  # 通知组件（第三方）
│   ├── perfect-scrollbar.min.js # 滚动条美化（第三方）
│   ├── lightyear.min.js   # 后台模板交互与 notify/loading 封装（第三方）
│   ├── main.min.js        # 模板初始化脚本（第三方）
│   ├── request.min.js     # 统一 POST 请求封装（第三方）
│   ├── WSCmd.js           # WebSocket 命令码常量定义
│   ├── lanshareChat.js    # WebSocket 连接、聊天消息、远程绘图逻辑
│   └── lanshare.js        # 主逻辑：初始化、主题、路由、文件/媒体/应用列表、上传下载
└── images/                # 页面图标资源（Logo、文件夹、文档、播放器等）
```

---

## 功能面板

入口页 `lanshare.html` 的侧边栏菜单由服务端下发的 `menus` 动态构建，共五个内容面板，通过 hash 切换显示：

| 面板 | Hash 前缀 | 说明 |
|------|-----------|------|
| 应用（apps） | `nav-item-apps` | 展示已安装应用列表，可下载对应 APK |
| 文件（files） | `nav-item-files/<path>` | 文件浏览器，逐级进入目录、勾选打包下载、上传文件 |
| 媒体（media） | `nav-item-media?index=<n>` | 图片/视频瀑布流，支持缩略图懒加载、目录进入、批量下载 |
| 聊天（chat） | `nav-item-chat` | 与局域网设备互发消息、图片、文件，支持多选删除、本地缓存、不保存模式 |
| 绘图（draw） | `nav-item-draw` | Canvas 画布，与宿主端（移动端 / PC 端）实时双向同步绘图 |

菜单项对应的 Material Design 线性图标以内联 SVG 形式定义在 `lanshare.js` 的 `NAV_ICONS` 中。

---

## 与宿主端的通信

### HTTP 接口

页面所有数据交互均通过 `request.min.js` 封装的 `post()` 发起，请求头携带 `token`（首次 `/initConfig` 时下发并存于 `localStorage`）。主要接口：

| 接口 | 用途 |
|------|------|
| `/initConfig` | 拉取根路径、设备名、token、主题、菜单配置；`pass=false` 时提示在宿主端授权 |
| `/checkPass` | 未授权时轮询，用户同意后自动刷新页面 |
| `/files` | 获取指定目录的文件列表 |
| `/media` | 获取媒体列表（支持按目录索引进入） |
| `/apps` | 获取已安装应用列表 |
| `/file/<name>` | 下载/打开单个文件 |
| `/apkfile/<name>` | 下载指定包的 APK |
| `/imageload/<name>` | 加载媒体缩略图 |
| `/appicon` | 加载应用图标 |
| `/uploadFile` | 上传文件到本机 |
| `/chatUploadFile` | 聊天场景下上传并发送给指定设备 |
| `/compressFiles` / `/compressMedias` | 将勾选项打包，返回临时文件 |
| `/downloadZipFile` | 下载打包生成的 zip |
| `/updateWebName` | 更新顶部显示的设备名称 |

### WebSocket 协议

`lanshareChat.js` 中的 `initWS()` 建立 `ws://<host>/wss?token=...` 长连接，命令码定义于 `WSCmd.js`：

| 命令 | 值 | 方向 | 说明 |
|------|----|------|------|
| `SEND_MSSAGE` | 1 | 双向 | 收发文本 / 图片 / 文件消息 |
| `SYNC_DEVICE_LIST` | 2 | 服务端→前端 | 同步在线设备列表，刷新聊天目标下拉框 |
| `CHANGE_THEME` | 3 | 服务端→前端 | 宿主端切换主题时实时推送，前端立即应用 |
| `DRAW_EVENT` | 4 | 双向 | 绘图笔迹事件（start/move/end/clear），坐标归一化以适配不同屏幕 |
| `PING` / `PONG` | 5 / 6 | 心跳 | 每 30 秒发送心跳维持连接 |

连接断开后每 5 秒自动重连，并以通知条和断连弹窗提示用户。绘图事件通过 `from` 字段区分来源，前端忽略 `from === 'web'` 的事件以避免回声。

---

## 主题系统

`theme.css` 基于 CSS 变量实现多主题，切换 `body` 的 `theme-*` 类即可整体换肤：

- **light**（默认蓝色系）
- **dark**（暗色）
- **emerald**（翠绿 / 祖母绿系）
- **follow_system**（跟随系统 `prefers-color-scheme`，并监听系统主题变化动态切换）

主题优先级：宿主端通过 `/initConfig` 或 `CHANGE_THEME` 推送的主题 > `localStorage` 缓存的主题。所有颜色（背景、卡片、边框、文字、聊天气泡、焦点环等）均通过 `--t-*` 变量引用，避免硬编码。

---

## 页面初始化流程

1. 加载 `lanshare.html`，隐藏主体布局，等待鉴权结果
2. `lanshare.js` 调用 `initConfig()` 请求 `/initConfig`
3. 应用主题、缓存 token、根据 `menus` 构建侧边栏
4. `pass=true`：显示布局 → `initWS()` 建立长连接 → `initView()` 按 hash 渲染对应面板
5. `pass=false`：弹出授权提示，`checkPass()` 轮询直到用户在宿主端同意后刷新

---

## 访问方式

1. 在目标设备（Android 移动端或 PC 桌面端）打开 LANShare 并开启文件服务
2. 确保访问设备与目标设备处于同一局域网
3. 在目标端查看访问地址（形如 `http://<设备IP>:<端口>`），在浏览器打开
4. 首次访问时在目标端确认授权，即可进入页面

---

## 备注

- 页面为纯静态资源，由各端内置的 HTTP 服务器托管，移动端与 PC 端共用同一份前端，无需独立构建流程。
- 第三方库（jQuery、Bootstrap、lightyear 等）以 `.min.js` / `.min.css` 形式内联引入，不依赖外网 CDN，确保局域网离线可用。
- `lanshare.css` 通过媒体查询做了移动端 / 平板 / 桌面三档响应式适配。

---

## 相关项目

- **LANShare（Android 移动端）**：本项目 — [fgsq-team/LANShare](https://github.com/fgsq-team/LANShare)
- **LANShare-PC**：桌面端（Win / Mac / Linux）— [fgsq-team/LANShare-PC](https://github.com/fgsq-team/LANShare-PC)
- **LANShare-Harmony**：鸿蒙版（开发中）— [fgsq-team/LANShare-Harmony](https://github.com/fgsq-team/LANShare-Harmony)
- **HttpServer**：内置 HTTP 文件服务器 — [fgsq-team/HttpServer](https://github.com/fgsq-team/HttpServer)

---

## License

```
Copyright 2021 fgsqme

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
```
