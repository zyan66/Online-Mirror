# 📝 更新日志

所有重要的项目更改都将记录在此文件中。

## [9.2.1] - 2026-04-27

### 🐛 Bug 修复

#### 代理链路与上下文崩塌修复 (Context Collapse)

- **问题**：
  1. 目标页面包含相对路径的图片、CSS、JS 资源加载经常报 `404`。
  2. 被代理的目标站如果有 `/api/xxx` 请求会导致 `500 Internal Server Error` 甚至整体请求瘫痪。
  3. 跳转至下游带有复杂参数的目标站（如百度搜索）时，返回 `net::ERR_HTTP_RESPONSE_CODE_FAILURE 400` 错误。
- **原因**：
  1. 硬编码注入了 `<base href="/">`，强行把上下文路径拍扁到根目录，造成子目录的资源的相对寻址发生逃逸。
  2. Web worker 的请求路由霸道抢占了所有前缀为 `/api/` 的流量，且在错误校验时调用未闭包的 `corsHeaders` 引发 ReferenceError 崩溃。
  3. `worker.js` 在处理目标网址时执行了无差别的 `decodeURIComponent(targetUrl)`，直接摧毁并还原了如 `%2F`（斜杠）、`%3D` 等安全转移字符的 URL 参数结构，致使百度的防护网关直接将破坏数据结构的请求判为 `400 Bad Request`。
  4. 页面重定向器 `wrapNav` 中的冗余 `btoa` 组合操作反复引发乱码污染。
- **解决**：
  1. 更换为精准的子目录寻址机制：根据代理实时路径动态插入正确的 `<base href="${targetPathDir}">` + `document.baseURI` 对齐。
  2. 解耦 API 路由拦截器，精准释放业务系统的 API 调用。
  3. 全卷清查并铲除了污染隧道透传的全局 `decodeURIComponent` 解码，实现字节级无损转发。
  4. 重构成干净规范的 `v?url=${URL}&id=${ID}` 参数方式去实现跨域重绘和 Location 二次重定向。

### ✨ 新特性

#### 边缘驱动版本注入 (Edge-Driven Version Inject)

- **内容**：主界面引擎版本号升级为动态化自动下发机制。
- **机制**：
  1. 采用 `wrangler.toml` 的 `[vars]` (系统级环境变量) 作为 `SYSTEM_VERSION` 的单一真实来源 (Source of truth)。
  2. 复用无服务器隧道的 `/api/quota` 响应通道，在下发配额状态时直接并推当前版本的上下文数据至渲染层。
  3. `home.html` 前端增加非侵入式插桩节点，完成渲染。
- **收益**：避免了版本号被零碎地硬编码于前端视图代码内，彻底实现一处命令推送、整站组件同频感知，免除了 HTML 结构被反复改动所导致的开发损耗。

## [1.2.0] - 2025-10-29

### 🚀 性能优化

#### 速度提升

- **DNS 预解析** - 所有页面添加 `dns-prefetch` 和 `preconnect`，提速 200-500ms
- **10ms 极速拍照** - 优化拍照延迟从 100ms 降至 10ms（提升 90%）
- **JPEG 压缩** - 改用 JPEG 格式（质量 0.7），文件大小减小 60%+
- **先跳转后上传** - 完全非阻塞，用户无需等待上传完成
- **HTTP 缓存策略** - 通过 `_headers` 配置合理的缓存

#### 网络优化

- 添加 `_headers` 文件配置 HTTP 头部
- 优化资源加载顺序（preload config.js）
- 减少首字节时间（TTFB）

### 🔐 安全 & 隐蔽性

- **Base64 参数加密** - URL 参数加密处理，长度缩短 36%
- **伪装文件名** - 使用 `/v` 替代 `/camera`，降低警觉性
- **动态页面标题** - 根据目标网站动态设置标题
- **URL 参数隐藏** - 不再在地址栏显示明文参数

### 🛠️ 功能增强

- **网络诊断页面** - 新增 `test.html` 用于诊断网络问题
  - 浏览器支持检测
  - API 连接测试
  - HTTPS 安全检查
  - 网络延迟测试
  - DNS 解析检测

- **健康检查端点** - Worker 添加 `/api/ping` 端点
- **兼容性改进** - 保留 `camera.html` 兼容旧链接

### 🐛 Bug 修复

#### 重定向循环修复

- **问题**：访问 `/v` 出现 `ERR_TOO_MANY_REDIRECTS` 错误
- **原因**：Cloudflare Pages 的自动 `.html` 去除功能与自定义重定向冲突
- **解决**：使用 `200!` 强制重写规则覆盖默认行为
- **影响**：所有使用伪装路径的链接

#### 配置加载失败

- 修复 `config.js` 缓存导致的 API 地址未定义问题
- 添加版本控制和调试日志

### 📦 配置变更

#### 新增文件

- `.gitignore` - Git 忽略文件配置
- `config.example.js` - API 配置模板
- `_headers` - HTTP 头部配置
- `test.html` - 网络诊断页面
- `SECURITY.md` - 安全说明文档
- `CHANGELOG.md` - 本文件

#### 修改文件

- `_redirects` - 使用 `200!` 强制重写
- `home.html` - 添加 DNS 预解析，优化链接生成
- `v.html` - 新的拍照页面，添加性能优化
- `view.html` - 添加 DNS 预解析
- `camera.html` - 添加 DNS 预解析（保留兼容）
- `worker.js` - 添加健康检查端点
- `README.md` - 完善文档，添加安全和部署说明

### 📋 部署脚本

- `scripts/deploy-main.bat` - Windows 生产环境部署
- `scripts/deploy-main.sh` - Linux/Mac 生产环境部署
- `scripts/test-dns.bat` - DNS 诊断工具

### 🔄 API 变更

- 新增 `GET /api/ping` - 健康检查端点
- 返回格式：`{ "status": "ok", "timestamp": "...", "message": "..." }`

---

## [1.1.0] - 2025-10-28

### ✨ 新特性

- **去除短链接依赖** - 直接生成原始链接，提升可靠性
- **Base64 编码优化** - 缩短 URL 长度 36%
- **即时链接生成** - 0ms 延迟，无需等待

### 🛠️ 改进

- 优化 UI 交互，使用自定义 Toast 提示
- 改进照片查看页面的图片展示
- 添加自定义确认对话框

### 📝 文档

- 创建 Cursor Rules 规范项目开发
- 合并多个 README 文档
- 添加架构图和流程图

---

## [1.0.0] - 2025-10-27

### 🎉 初始版本

- 基于 Cloudflare Workers + R2 + Pages 架构
- 自动拍照功能
- 照片云端存储
- 分页查看和删除功能
- 完全免费部署

### 核心功能

- **自动拍照** - 访问链接自动调用摄像头
- **云端存储** - R2 对象存储
- **照片管理** - 查看、分页、删除
- **无服务器** - Serverless 架构

---

## 版本说明

格式基于 [Keep a Changelog](https://keepachangelog.com/zh-CN/1.0.0/)

版本号遵循 [语义化版本](https://semver.org/lang/zh-CN/)

### 类型说明

- `🚀 性能优化` - 性能改进
- `🔐 安全` - 安全相关更新
- `✨ 新特性` - 新增功能
- `🛠️ 改进` - 功能改进
- `🐛 Bug 修复` - 问题修复
- `📝 文档` - 文档更新
- `🔄 API 变更` - API 接口变更
- `⚠️ 破坏性变更` - 不兼容的更改
