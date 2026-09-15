# jd-cookie

京东 Cookie 自动同步模块 —— 支持 **Magisk / KernelSU / APatch**。自动读取京东 Cookie 并同步至青龙面板。

搭配 [jdpro](https://github.com/6dylan6/jdpro) 使用，显著减少 Cookie 过期后的手动维护。

> 本分支（`codex/magisk-support`）：在原生 KernelSU 支持的基础上，新增 **Magisk / APatch** 兼容，并让 WebUI 摆脱对内核管理器的依赖。

## 为什么需要这个模块

jdpro 等京东脚本依赖 `JD_COOKIE` 环境变量，但 Cookie 有效期越来越短（部分用户每天过期）。手动抓包费时费力。

本模块无需抓包——直接读取手机京东 App 的 WebView Cookie 数据库（`/data/data/com.jingdong.app.mall/app_webview/Default/Cookies`），提取 `pt_key` / `pt_pin` 后上传青龙面板。只要京东 App 登录态有效，Cookie 就能自动获取，省去反复抓包的麻烦。

## 功能

- 从京东 App WebView Cookie 数据库读取 `pt_key` / `pt_pin`
- Cookie 去重后自动上传至青龙面板 `JD_COOKIE` 环境变量
- Go 守护进程，内存占用约 7.5 MB
- WebUI 配置管理 + SSE 实时日志流
- Token 鉴权，外部请求一律拒绝
- 兼容 KernelSU / Magisk / APatch

## 兼容性与 Magisk 支持（本分支新增）

| 框架 | 支持 | 说明 |
| --- | --- | --- |
| KernelSU | ✅ | 原生支持，Manager 自动渲染 WebUI |
| Magisk | ✅ | 本分支新增，见下文 |
| APatch | ✅ | 支持 `sepolicy.rule` 放行 |

针对 Magisk / APatch 的主要改动：

1. **WebUI 免内核管理器**：KernelSU Manager 会自动渲染模块的 `webroot` 并注入 `token.txt`；Magisk 没有此能力。因此守护进程改为同时提供静态页面与 API，浏览器直接访问 **`http://127.0.0.1:17320`** 即可打开配置页（token 同源加载，无需手动填写）。
2. **SELinux 放行**：新增 `sepolicy.rule`，放行 Magisk 域读取京东 App WebView Cookie 数据库、监听本地端口等权限。Magisk 与 APatch 均会读取该文件。
3. **IPv6 DNS 解析修复**：部分 Android 设备 `/etc/resolv.conf` 损坏，导致青龙面板域名解析失败（`lookup ... connection refused` / `i/o timeout`）。新增自建 DNS 客户端，读取系统实际 DNS（`getprop net.dns*`）并回退公共 DNS，优先解析 IPv6、逐个服务器重试。
4. **卸载清理**：新增 `uninstall.sh`，卸载时自动终止守护进程并清理运行时文件。

## 完整教程：jdpro + 本模块

### 1. 部署青龙面板

推荐内网部署，青龙 2.15+ 即可。

### 2. 订阅 jdpro

青龙面板 → 订阅管理 → 创建订阅：

```
名称: jdpro
类型: 公开仓库
链接: https://github.com/6dylan6/jdpro.git
分支: main
白名单: jd_|jx_|jddj_
黑名单: backUp
依赖文件: ^jd[^_]|USER|JD|function|sendNotify|utils
```

运行订阅 → 依赖安装任务 → 配置通知。

### 3. 安装本模块

下载本仓库 **Releases** 或构建产物中的 `jd-cookie-magisk-*.zip`：

- **KernelSU**：模块 → 安装 → 选择 zip 刷入
- **Magisk**：模块 → 从本地安装 → 选择 zip
- **APatch**：模块 → 刷入该 zip

### 4. 配置

1. 手机打开**京东 App** 登录账号（Cookie 写入 WebView 数据库）
2. 在 KernelSU / Magisk Manager 中开启模块，或直接在浏览器访问 **`http://127.0.0.1:17320`**
3. WebUI 填写青龙面板**地址、用户名、密码**，保存
4. 点击**读取**按钮，或等待自动上传

> 提示：Magisk 环境下若浏览器不便访问，可先开启系统代理或用 KernelSU Manager 的 WebUI 入口打开上述地址。

### 5. 验证

青龙面板 → 环境变量 → `JD_COOKIE` 应出现最新值。jdpro 脚本会自动使用。

## 流程示意

```
京东 App 登录 → WebView Cookie 数据库
                    ↓ 本模块自动读取
            pt_key=xxx;pt_pin=xxx
                    ↓ 上传青龙面板
            JD_COOKIE 环境变量
                    ↓ jdpro 脚本调用
              自动签到、领豆...
```

## 结构

```
├── backend/        Go 守护进程（含自建 DNS 解析器）
├── frontend/       Vue 3 WebUI
├── kernelsu/       模块打包目录
│   ├── customize.sh  安装脚本（Magisk/KernelSU/APatch 通用）
│   ├── service.sh    启动脚本（含崩溃自重启）
│   ├── sepolicy.rule  SELinux 放行（Magisk/APatch 读取）
│   ├── uninstall.sh   卸载清理脚本
│   └── module.prop
└── .github/workflows/  CI/CD
```

## 构建

```bash
# 后端（交叉编译 Android/arm64 静态二进制）
cd backend
CGO_ENABLED=0 GOOS=android GOARCH=arm64 go build -ldflags="-s -w" -o ../kernelsu/bin/jd-cookie .

# 前端
cd frontend && npm install && npm run build

# 打包
cd kernelsu && zip -r ../jd-cookie-magisk-v1.2.0.zip .
```

## License

MIT
