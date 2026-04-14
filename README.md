# Zopia.ai 完整逆向分析报告
> 生成时间: 2026-04-15
> 分析工具: Playwright + curl + 正则提取 + curl_cffi
> Cookie来源: 已登录账户 130069

---

## 一、认证机制分析

### 1.1 NextAuth Session 认证
Zopia 使用 NextAuth.js 进行身份认证，**无任何签名/API Key机制**。

```
认证流程:
1. 用户登录 → NextAuth 创建 Session
2. Session Token 存储在 Cookie: __Secure-next-auth.session-token
3. 所有 API 请求自动携带此 Cookie
4. 服务端验证 Session Token 有效性
```

### 1.2 Cookie 清单（实测）
| Cookie名 | 用途 |
|---------|------|
| NEXT_LOCALE | 语言偏好 (zh/en) |
| sessionId | NextAuth Session ID |
| userId | 用户ID（内部） |
| amplitude_session_id | Analytics 会话ID |
| __Secure-next-auth.session-token | **核心认证Token** |
| __Host-test.rt | 测试Cookie |
| RefreshTokens | Refresh Token |

### 1.3 无签名证据
- 所有 API 请求只需要 Cookie 头
- 没有 `X-Sign`、`X-Timestamp`、`Authorization: Bearer` 等常见签名头
- `POST /api/base/task` 的 `generation_params` 直接明文传递，无额外签名
- API 响应中未发现任何签名验证字段

---

## 二、JS 源码架构分析

### 2.1 JS 文件清单（8个）
```
_app-5c3c38d95c4b13a7.js     # 主包，71个fetch调用，AWS SDK
9084-86642685d559db0f.js     # Base相关功能，7个API调用
3613-b2c24d705a6d83f7.js      # 基础工具库
9131-dc3134ed5e999287.js     # 工具库
3495-f5c01cf413d27a23.js      # UI组件
home-2836f25a35b3a400.js      # 首页/Showcase，8个fetch
7-f1001aca161db422.js         # Base管理（rename/delete/export），5个API
index-4d219976891f7455.js     # 入口文件
```

### 2.2 核心发现
#### AWS SDK (`_app-*.js`)
- 导入了完整 AWS SDK (`@aws-sdk/*`)
- 发现 `AWS4-HMAC-SHA256` 签名逻辑
- 用于 S3 存储相关操作（presigned-url 上传）
- **与视频生成 API 无关** — 仅用于文件上传

#### NextAuth (`_app-*.js`)
```javascript
// NextAuth 登录流程
fetch("/api/auth/signin", {...})
// 登录后自动创建 Session
// Session 信息存储在 Cookie 中
```

#### Canvas (`9084-*.js`)
- 使用 Canvas API 进行图形编辑
- `canvas/` 相关 API 操作画布数据

### 2.3 加密算法使用情况
| 算法 | 用途 | 位置 |
|------|------|------|
| MD5 | 仅用于内部数据校验（可能） | _app-*.js |
| SHA1 | AWS SDK 内部 | _app-*.js |
| SHA256 | AWS S3 签名 + 密码学 | _app-*.js |
| HMAC-SHA256 | AWS 请求签名（仅S3） | _app-*.js |
| AES | 未知（未发现实际使用） | _app-*.js |
| btoa/atob | Cookie 序列化 | _app-*.js |

**结论**: 视频生成 API **不涉及任何签名/加密**。

---

## 三、完整 API 清单（41个已验证）

### 3.1 核心视频工作流
| # | Method | Path | 关键发现 |
|---|--------|------|---------|
| 1 | POST | `/api/base/create` | **一键创建 base+episode**，返回 baseId + episodeId |
| 2 | POST | `/api/base/task` | 发送视频任务，需 baseId + episodeId(table_id) |
| 3 | GET | `/api/task/{task_id}` | **正确轮询接口**，不是 billing/listTransactions |
| 4 | GET | `/api/task/queue-status` | baseId query param，队列状态 |

### 3.2 Base CRUD
| # | Method | Path | 关键发现 |
|---|--------|------|---------|
| 5 | GET | `/api/base/list` | 分页返回 Base 列表 |
| 6 | GET | `/api/base/{base_id}` | 需 episode_id query param |
| 7 | POST | `/api/base/{base_id}/rename` | body: {name} |
| 8 | DELETE | `/api/base/{base_id}/delete` | **DELETE 方法，不是 POST** |
| 9 | GET | `/api/base/{base_id}/export` | **GET 方法**，返回完整数据结构 |
| 10 | POST | `/api/base/{base_id}/remove-from-team` | |
| 11 | POST | `/api/base/{base_id}/transfer-to-team` | |

### 3.3 余额/账单
| # | Method | Path | 关键发现 |
|---|--------|------|---------|
| 12 | GET | `/api/billing/getBalance` | **summary.totalAvailable = 真实可用额度** |
| 13 | GET | `/api/billing/listTransactions` | 账单记录 |

### 3.4 内容数据
| # | Method | Path | 关键发现 |
|---|--------|------|---------|
| 14 | GET | `/api/showcase/list` | 作品展示列表 |
| 15 | GET | `/api/showcase/{id}` | 单个 showcase |
| 16 | POST | `/api/showcase/migrate` | |
| 17 | POST | `/api/showcase/review` | action: approve/reject |
| 18 | GET | `/api/seedance-modes` | 模式列表 |
| 19 | GET | `/api/base/canvas/{base_id}` | Canvas 快照 |
| 20 | GET | `/api/base/document/{doc_id}` | 文档详情 |

### 3.5 用户/账户
| # | Method | Path | 关键发现 |
|---|--------|------|---------|
| 21 | GET | `/api/auth/session` | NextAuth Session |
| 22 | GET | `/api/user/me` | userLevel: free/pro |
| 23 | GET | `/api/subscription/get` | |
| 24 | POST | `/api/user/update_my_profile` | |
| 25 | POST | `/api/user/redeem-coupon` | |

### 3.6 团队/邀请
| # | Method | Path | 关键发现 |
|---|--------|------|---------|
| 26 | GET | `/api/team/list` | |
| 27 | POST | `/api/team/create` | free 账号无法创建 |
| 28 | GET | `/api/invite/list` | 邀请码+链接 |

### 3.7 文件/存储
| # | Method | Path | 关键发现 |
|---|--------|------|---------|
| 29 | POST | `/api/s3/presigned-url` | AWS S3 预签名 URL |
| 30 | POST | `/api/upload/attachment` | |
| 31 | POST | `/api/storyboard/parse-import` | 分镜板 CSV 导入 |
| 32 | GET | `/api/storage/listMyStorage` | |
| 33 | GET | `/api/machine/listMyMachines` | |

### 3.8 AI 聊天
| # | Method | Path | 关键发现 |
|---|--------|------|---------|
| 34 | POST | `/api/chat/start` | body: {baseId} 或 {session_id} |
| 35 | POST | `/api/chat/cancel/{session_id}` | |

### 3.9 图片处理
| # | Method | Path | 关键发现 |
|---|--------|------|---------|
| 36 | POST | `/api/image/volces-moderation` | body: {imageUrl} |

### 3.10 充值/优惠
| # | Method | Path | 关键发现 |
|---|--------|------|---------|
| 37 | POST | `/api/subscription/recharge-discount-codes/preview` | |
| 38 | GET | `/api/billing/recharge-discount-codes/summary` | |

### 3.11 工作流
| # | Method | Path | 关键发现 |
|---|--------|------|---------|
| 39 | GET | `/api/workflow/listMyWorkflows` | |
| 40 | GET | `/api/workflow/listMyRunJob` | |

### 3.12 模型
| # | Method | Path | 关键发现 |
|---|--------|------|---------|
| 41 | GET | `/api/model/listModelByFolderName` | folder: checkpoints/loras/controlnet |

---

## 四、模型 ID 映射（7个）
```python
MODEL_IDS = {
    "kling_o3":         "generate_video_by_kling_o3",
    "kling_v3_0":       "generate_video_by_kling_v3_0",
    "seedance_1_5_pro": "generate_video_by_seedance_1_5_pro",
    "seedance_2_0_pro": "generate_video_by_seedance_2_0_pro",
    "hailuo_2_3":       "generate_video_by_hailuo_2_3",
    "wan_2_6_i2v":      "generate_video_by_wan_2_6_i2v",
    "vidu_q3_pro":      "generate_video_by_vidu_q3_pro",
}
```

---

## 五、关键 Bug 与陷阱

### ❌ episode 创建陷阱
- `POST /api/episode/create` → 404 Not Found
- `POST /api/base/{id}/episode` → 404 Not Found
- **正确方式**: `POST /api/base/create` 自动创建第一个 episode

### ❌ 轮询陷阱
- `GET /api/billing/listTransactions` → 不是轮询接口
- **正确方式**: `GET /api/task/{task_id}`

### ❌ base_id 格式陷阱
- 旧格式: `tekjQbieDk8otMYM_QTVS`（无前缀）→ 返回 `episode_id_required`
- **正确格式**: `base_xxx`（有 `base_` 前缀）

### ❌ Base 详情陷阱
- `GET /api/base/{id}` → 需 `?episode_id=1` query 参数

### ❌ 余额陷阱
- 响应: `{"summary": {"totalAvailable": 0.20, ...}}`
- 免费账号只有 $0.20 可用，大部分模型最低消费 > $0.20

### ❌ 状态码陷阱
- `POST /api/base/create` → **201** Created，不是 200

### ❌ delete/export 方法陷阱
- delete: DELETE 不是 POST
- export: GET 不是 POST

---

## 六、curl_cffi TLS 指纹测试

| Browser | Result |
|---------|--------|
| chrome120 | ✅ 200 OK |
| chrome110 | ✅ 200 OK |
| chrome107 | ✅ 200 OK |
| safari15_5 | ✅ 200 OK |
| firefox120 | ❌ 不支持 |

---

## 七、请求示例

### 发送视频任务（完整版）
```bash
curl -X POST https://zopia.ai/api/base/task \
  -H "Host: zopia.ai" \
  -H "Content-Type: application/json" \
  -H "Origin: https://zopia.ai" \
  -H "User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7)..." \
  --cookie "NEXT_LOCALE=zh; sessionId=xxx; userId=xxx; ..." \
  -d '{
    "field_id": "videos",
    "record_id": "auto_rec_1234567890",
    "base_id": "base_xxx",
    "table_id": "episode_yyy",
    "generation_params": {
      "model_id": "generate_video_by_kling_v3_0",
      "prompt": "一只可爱的小猫在草地上奔跑",
      "size": "720",
      "duration": "5",
      "video_mode": "entity_reference",
      "resolution": "720p",
      "aspect_ratio": "16:9",
      "ref_assets": [],
      "base_id": "base_xxx",
      "table_id": "episode_yyy"
    }
  }'
```

### 轮询任务
```bash
curl -s https://zopia.ai/api/task/{task_id} \
  --cookie "NEXT_LOCALE=zh; sessionId=xxx; ..."
```

---

## 八、架构总结

```
Zopia.ai 架构
├── 前端: Next.js (App Router) + TypeScript
├── 认证: NextAuth.js (Cookie Session)
├── 状态管理: React (useState/useCallback)
├── 存储: AWS S3 (文件上传)
├── AI模型: 第三方API (Kling, Seedance, Hailuo, Wan, Vidu)
├── 数据库: PostgreSQL (推断)
├── 实时通信: WebSocket (未深入分析)
└── API: REST JSON (无签名，纯Cookie认证)
```
