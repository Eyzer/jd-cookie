# jd-cookie-magisk

Magisk模块 —— 自动读取京东 Cookie 并同步至青龙面板。

搭配 [jdpro](https://github.com/6dylan6/jdpro) 使用，显著减少 Cookie 过期后的手动维护。

## 为什么需要这个模块

jdpro 等京东脚本依赖 `JD_COOKIE` 环境变量��但 Cookie 有效期越来越短（部分用户每天过期）。手动抓包费时费力。

本模块无需抓包——直接读取手机京东 App 的 WebView Cookie 数据库（`/data/data/com.jingdong.app.mall/app_webview/Default/Cookies`），提取 `pt_key` / `pt_pin` 后上传青龙面板。只要京东 App 登录态有效，Cookie 就能自动获取，省去反复抓包的麻烦。

## 功能

- 从京东 App WebView Cookie 数据库读取 `pt_key` / `pt_pin`
- Cookie 去重后自动上传至青龙面板 `JD_COOKIE` 环境变量
- Go 守护进程，内存占用约 7.5 MB
- WebUI 配置管理 + SSE 实时日志流
- Token 鉴权，外部请求一律拒绝

## 对原项目的修改
本 PR 在保留原有 KernelSU 支持的基础上：

 - 新增对 Magisk 与 APatch 的支持
 - 新增 kernelsu/sepolicy.rule，放行 Magisk 域读取京东 WebView Cookie 数据库、监听本地端口所需的 SELinux 权限；
 - 新增 kernelsu/uninstall.sh 卸载清理脚本（终止守护进程、清理 token/日志）；
 - customize.sh、service.sh、module.prop 适配多 root 方案（Magisk/KernelSU/APatch），service.sh 增加 boot_completed 等待与崩溃自动重启；
 - 守护进程（Go）直接托管 WebUI 静态页面，解决 Magisk 下无界面入口、无法访问 WebUI 的问题，浏览器访问 http://127.0.0.1:17320 即可。
 - 修复 Android 下 DNS 解析失败导致青龙面板无法连接

## 使用中发现的问题
 - 问题：Android 的 /etc/resolv.conf 常缺失或指向未监听的环回地址（如 [::1]:53），Go 默认解析器报 dial tcp: lookup xxx: connection refused / i/o timeout；
 - 方案：新增 backend/dns.go，完全自建 DNS 客户端，读取系统实际 DNS（getprop net.dns*）并回退公共 DNS（含 IPv6），逐服务器发送标准 DNS 查询（AAAA 优先），一个超时自动换下一个；
 - 通过自定义 DialContext 接入 HTTP 客户端，解析出 IP 后直连，TLS 仍按原域名校验。
## Changes
 -  backend/dns.go：自建 DNS 解析器（新增）
 -  backend/ql.go：HTTP 客户端接入自定义 DNS 拨号
 -  backend/server.go：Go 守护进程直接托管 WebUI 静态资源
 -  kernelsu/sepolicy.rule：Magisk SELinux 策略（新增）
-   kernelsu/uninstall.sh：卸载清理脚本（新增）
 -  kernelsu/customize.sh / service.sh / module.prop：多 root 适配
 -  README.md：补充 Magisk 安装/使用说明

## 编译产物
  jd-cookie-magisk-v1.2.0.zip 可于 [Releases](https://github.com/Eyzer/jd-cookie/releases/tag/v1.2.0) 使用，或按 README「构建」自行编译。

###  安装本模块

  下载 [Releases](https://github.com/Eyzer/jd-cookie/releases/tag/v1.2.0) 中最新 `jd_assistant-magisk.zip`，使用 ** Magisk ** 刷入。

### 配置

1. 手机打开**京东 App** 登录账号（Cookie 写入 WebView 数据库）
2. Magisk → 模块 → 京东助手 → 打开
3. 用手机自带浏览器打开 **http://127.0.0.1:17320**
4. 页面里填青龙面板地址、用户名、密码，点保存
5. 点击**读取**按钮，或等待自动上传

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


## License

MIT
