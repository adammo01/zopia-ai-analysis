# Zopia.ai 接口分析报告

> 抓取时间: 2026-04-15
> 目标: https://zopia.ai
> 技术栈: Next.js + Amplitude + 私有API

---

## 一、API 端点汇总

### 1. 认证相关
| 端点 | 方法 | 状态码 | 说明 |
|------|------|--------|------|
| `/api/auth/session` | GET | 200 | 获取会话，返回 `{}` (未登录) |

### 2. 核心业务API (需要认证)
| 端点 | 方法 | 状态码 | 说明 |
|------|------|--------|------|
| `/api/base/` | GET | 308 | 基础端点 |
| `/api/base/list` | GET | 401 | 列表接口 |
| `/api/base/create` | POST | 401 | 创建接口 |
| `/api/chat/start` | POST | 400 | 开始聊天 |
| `/api/chat/cancel/` | POST | 308 | 取消聊天 |
| `/api/chat_session/` | GET/POST | 308 | 聊天会话 |
| `/api/session/` | GET/POST | 308 | 会话管理 |
| `/api/storyboard/parse-import` | POST | 401 | 故事板导入解析 |
| `/api/upload/attachment` | POST | 401 | 上传附件 |

### 3. Showcase API (需要认证)
| 端点 | 方法 | 状态码 | 说明 |
|------|------|--------|------|
| `/api/showcase/` | GET | 308 | 展示内容 |
| `/api/showcase/review` | POST | 401 | 审核 |
| `/api/showcase/migrate` | POST | 401 | 迁移 |

---

## 二、第三方服务

### Amplitude (分析/埋点)
```
API Key: 77fc9c424eb838f0e210c6e1b26a1247
Config: https://sr-client-cfg.amplitude.com/config?api_key=77fc9c...
Events: https://api2.amplitude.com/2/httpapi
```

### CDN/资源
- 视频: `https://zopia.ai/assets/videos/*.mp4`
- 图片: `https://cdn.zopia.ai/upload/*`
- 字体: Google Fonts (Roboto, Outfit)

---

## 三、技术架构

### 前端
- 框架: Next.js (SSR + SSG)
- 国际化: next-i18next (`/zh`, `/en`, `/ja`)
- 样式: Tailwind CSS
- 分析: Amplitude Session Replay + Analytics SDK

### SSG预渲染路由
```
/, /blog, /blog/[slug], /openclaw, /payment-callback, /seedance
```

### 静态资源
- JS Chunks: `/_next/static/chunks/`
- Next.js Image: `/_next/image?url=...&w=...&q=...`

---

## 四、反爬/反调试分析

### 自动化检测
- `webdriver` 检测
- `navigator.webdriver === true` 时可能被拦截
- 按钮点击被 `div` 层遮挡，需要滚动或强制点击

### 绕过方案
```python
# Playwright 反检测参数
browser = await p.chromium.launch(
    args=[
        "--disable-blink-features=AutomationControlled",
        "--no-sandbox",
        "--disable-dev-shm-usage",
    ]
)
```

---

## 五、接口请求示例

### 获取Session
```bash
curl -s https://zopia.ai/api/auth/session
# Response: {}
```

### 创建项目 (需认证)
```bash
curl -X POST https://zopia.ai/api/base/create \
  -H "Content-Type: application/json" \
  -d '{"type":"video","prompt":"test"}'
# Response: {"error":"Authentication required"}
```

---

## 六、JS源码中的API定义

```javascript
// from 9084-86642685d559db0f.js
async function getBase(e) {
  // GET /api/base/{endpoint}
}
async function listBase({page, pageSize}) {
  // GET /api/base/list?page=...&pageSize=...
}
```

---

## 七、视频资源

| 文件 | 类型 | 说明 |
|------|------|------|
| fashion-{1-4}.mp4 | 竖版视频 | 时尚类 |
| vertical-{1-4}.mp4 | 竖版视频 | 通用类 |
| dancer-{1-3}.mp4 | 横版视频 | 舞蹈类 |

---

*报告由 Hermes Agent 自动生成*
