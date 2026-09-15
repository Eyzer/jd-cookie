# jd-cookie

Magisk / KernelSU / APatch 模块 —— 自动读取京东 Cookie 并同步至青龙面板。

搭配 [jdpro](https://github.com/6dylan6/jdpro) 使用，显著减少 Cookie 过期后的手动维护。

> **Magisk 支持（本分支新增）**：原模块仅面向 KernelSU，本分支在保留 KernelSU 兼容的同时，补充了 Magisk 适配 —— 新增 `sepolicy.rule` 放行 SELinux 策略、`uninstall.sh` 卸载清理脚本，并让内置 Go 守护进程直接提供 WebUI 静态页面，解决 Magisk 下无界面入口的问题。
>
> **修复 Android DNS 解析失败**：新增 `backend/dns.go` 自建 DNS 解析器，读取糺统实际 DNS（`getprop net.dns*`）并回退公共 DNS，规避 Android 下 `/etc/resolv.conf` 损坏（`dial tcp: lookup ... connection refused / i/o timeout`）导致的青龙面板连接失败。

## 为什么需要这个模块

jdpro 等京东脚本依赖 `JD_COOKIE` 环境变量��但 Cookie 有效期越来越短（部分用户每天过期）。手动抓包费时费力。

本模块无需抓包——直接读取手机京东 App 的 WebView Cookie 数据库（`/data/data/com.jingdong.app.mall/app_webview/Default/Cookies`），提取 `pt_key` / `pt_pin` 后上传青龙面板。只要京东 App 登录态有效，Cookie 就能自动获取，省去反复抓包的麻烦。

## 功能

- 从京东 App WebView Cookie 数据库读取 `pt_key` / `pt_pin`
- Cookie 去重后自动上传至青龙面板 `JD_COOKIE` 环境变量
- Go 守护进程，内存占用约 7.5 MB
- WebUI 配置管理 + SSE 实时日志流
- Token 鉴权，外部请求一律拒绝

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

下载 [Releases](https://github.com/Gesoy/jd-cookie/releases) 中最新 `jd_assistant.zip`，通过 **Magisk Manager / KernelSU Manager** 刷入并重启。

Magisk 用户：刷入后模块会注入 `sepolicy.rule` 放行 SELinux 权限、注册 `uninstall.sh` 清理脚本，无需额外配置。

### 4. 配置

1. 手机打开**京东 App** 登录账号（Cookie 写入 WebView 数据库）
2. **Magisk Manager / KernelSU Manager** → 模块 → 京东助手 → 打开（守护进程自动挂载在 `127.0.0.1:17320`）
3. 浏览器访问 `http://127.0.0.1:17320` 打开内置 WebUI，填写青龙面板**地址、用户名、密码**，保存
4. 点击**读取**按钮，或等待自动上传

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
├── backend/        Go 守护进程（含 dns.go 内置 DNS 解析）
├── frontend/       Vue 3 WebUI
├── kernelsu/       模块打包目录（兼容 Magisk / KernelSU / APatch）
│   ├── customize.sh  安装/权限适配脚本
│   ├── service.sh   启动脚本（崩溃自动重启）
│   ├── sepolicy.rule Magisk SELinux 策略
│   ├── uninstall.sh  卸载清理脚本
│   └── module.prop
└── .github/workflows/  CI/CD
```

## 构建

```bash
# 后端
cd backend
CGO_ENABLED=0 GOOS=android GOARCH=arm64 go build -ldflags="-s -w" -o ../kernelsu/bin/jd-cookie .

# 前端
cd frontend && npm install && npm run build

# 打包
cd kernelsu && zip -r ../jd_assistant.zip .
```

## License

MIT
