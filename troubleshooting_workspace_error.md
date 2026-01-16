# Refly 工作区报错修复完整记录

本文档汇总了解决工作区页面（`/workspace`）出现 "Please try again later" 报错的完整过程及其它相关问题的排查记录。

## 1. 核心问题修复：后端配置与依赖

### 环境变量补全 (`apps/api/.env`)
后端 NestJS 服务因缺少核心依赖配置导致启动失败或运行异常。
- **Redis 设置**: 将 `REDIS_HOST` 设置为 `localhost`，解决了后端无法连接缓存和任务队列的问题。
- **安全密钥**: 生成并添加了 `JWT_SECRET` 和 `ENCRYPTION_KEY`，恢复了身份验证和数据加解密功能。
- **CORS 允许源**: 更新了 `ORIGIN` 以包含前端运行端口（如 `5173`, `5174`）。

### 进程与端口冲突解决
- 解决了 `EADDRINUSE: address already in use :::5800` 和 `:::9229` 错误。
- 清理了内存中残留的 `nodemon` 和 `ts-node` 进程，确保服务能干净地重新启动。

---

## 2. 深入排查：前端 "Please try again later"

### 前端日志注入调试
在 `packages/web-core/src/components/form-onboarding-modal/index.tsx` 注入了调试日志：
```typescript
console.log('[DEBUG] FormOnboardingModal - Query State:', { ... });
```
**发现**: 接口 `v1/form/definition` 虽然返回成功 (`success: true`)，但其数据内容 `data` 为 `null`。

### 数据库数据修复 (根本原因)
排查发现 `refly.form_definitions` 表中没有任何数据，导致入站引导表单（Onboarding Form）无法加载。
- **解决方案**: 向数据库手动注入了一条基础表单定义。
- **执行指令**:
```sql
INSERT INTO refly.form_definitions
(pk, form_id, uid, title, description, "schema", ui_schema, status, created_at, updated_at, deleted_at)
VALUES(2, 'onboarding', 'u-d1x6dfyvug9561rmhvpd5xn2', 'Welcome to Refly', 'Please tell us a bit about yourself to get started.', '{"type":"object","required":["role"],"properties":{"role":{"type":"string","title":"What is your primary role?","enum":["Developer","Product Manager","Designer","Data Scientist","Other"]}}}', '{
    "ui:options": {
        "progressSteps": [
            {"key": "role", "title": "Role Selection"}
        ],
        "submitText": "Get Started",
        "variant": "white",
        "emoji": "🚀"
    }
}', 'published', '2026-01-14 15:46:39.888', '2026-01-14 15:46:39.888', NULL);
```

但是加上后，也只是显示了选项卡出来，还不能进入主界面，需要修改theme.tsx中的 const isPaginated = totalPages > 1; 变为 const isPaginated = totalPages >= 1; 才可以，这个修改的意思是把步骤变为一步。

---

## 3. 其它辅助配置

- **Debug 日志开启**: 
    - 建议用户使用 `PINO_LEVEL=debug`。
    - 开启 `LOG_JSON=true` 以获取更细粒度的结构化日志输出。
- **Sentry 兼容性**: 
    - 暂时禁用了 `Sentry.nodeProfilingIntegration`，解决了本地开发环境中由于原生模块缺失导致的崩溃。

---

## 验证结果
- **后端**: 正常监听 `5800` 端口，显示 `Nest application successfully started`。
- **前端**: Onboarding 表单成功渲染，用户已确认能够进入引导流程。

> [!IMPORTANT]
> 以后建议在执行初始化安装后，务必检查数据库是否有基础 Seed 数据，并确保 `.env` 中的密钥非空。

## 4. 修复 Onboarding 流程

### 4.1 导航按钮逻辑修复
- **问题描述**：Onboarding 模态框中的“下一步”/“完成”按钮未显示，导致用户无法正常提交表单。
- **原因分析**：RJSF 表单模板 ([theme.tsx](file:///Users/yuheng.huang/workspace/github/refly/packages/ai-workspace-common/src/components/rjsf/theme.tsx)) 原逻辑仅在 `totalPages > 1` 时显示导航。当前的单步 Onboarding 表单被判定为无需导航。
- **修复方案**：将判断条件修改为 `totalPages >= 1`，确保即使单步表单也能触发导航区域渲染。

### 4.2 表格定义回滚
- **修复操作**：在数据库中清理了错误的 `ui_schema` 配置，将表格定义还原为标准格式。

## 5. 修复 AI 模型配置错误 (Chat Model Not Configured)

### 5.1 数据库种子注入
- **问题描述**：后台执行工作流生成时报错 `chat model not configured`。
- **原因分析**：数据库 `providers` 和 `provider_items` 表为空，系统无法找到可用的对话模型。
- **修复方案**：
    - 向 `providers` 表注入了 **OpenAI** 提供商预设。
    - 向 `provider_items` 表注入了 **GPT-4o** 模型项，并绑定到当前用户。
    - 更新用户偏好设置 (`preferences`)，将默认 Chat/Agent 模型指向 GPT-4o。

### 5.2 Token 限制优化
- **问题描述**：生成复杂工作流时报错 `generation exceeded max tokens limit`。
- **修复方案**：将 `provider_items` 中的 `maxOutput` 从 **4096** 提升至 **16384** (16k)，确保长内容生成的稳定性。

## 6. 恢复前端模型管理界面

- **操作描述**：在设置页面 ([settings/index.tsx](file:///Users/yuheng.huang/workspace/github/refly/packages/ai-workspace-common/src/components/settings/index.tsx)) 中，取消了“模型配置”和“模型供应商”选项卡的源代码注释。
- **效果**：用户现在可以直接通过 **Settings -> Model Providers** 界面自定义 OpenAI 协议（支持自定义 Base URL 和 API Key），无需再手动修改数据库。

---

## 验证结果

1. **Onboarding 完成**：用户现在可以通过显示出的按钮正常完成引导。
2. **AI 生成可用**：工作流生成逻辑已能正确识别配置的 GPT-4o 模型并开始流化输出。
3. **配置入口可见**：设置菜单中已出现完整的 AI 配置管理选项。

> [!NOTE]
> 如果您使用的是第三方 OpenAI 代理，请在 **Model Providers** 设置中确认您的 **Base URL** 是否包含后缀（如 `/v1`），这通常取决于代理商的协议要求。
