# OpenMAIC 定制开发指南（给后续接手的 AI）

> 本文件是项目级指令。**做任何改动前先读完本文件**，尤其是「已知坑」和「Git 同步」两节。
> 它会进 git、活在 `my-custom` 分支上，不与上游冲突（上游无此文件）。

## 1. 这是什么项目

- **OpenMAIC**（Open Multi-Agent Interactive Classroom）：清华开源的多智能体互动课堂平台。技术栈 Next.js 16 + React 19 + TypeScript + LangGraph。把任意主题/文档变成一节带 AI 老师讲课、白板、测验、3D 仿真、PBL 的互动课。
- 本仓库是 **fork 后的定制部署版**，不是上游原生：
  - `upstream`（原仓库）：`https://github.com/THU-MAIC/OpenMAIC.git`
  - `origin`（我们的 fork）：`git@github.com:modaxian-nj/OpenMAIC.git`
- 部署在本地 Mac：`/Users/menglj/work/codes/OpenMAIC`

## 2. Git 结构（最重要，先记牢）

- **`main`**：上游镜像，**永远保持 = `upstream/main`，绝不在上面直接改**
- **`my-custom`**：我们的定制分支，所有本地改动只在这里
- remote：`origin` = 我们的 fork，`upstream` = 原仓库

### 同步上游更新（上游有新提交时执行）
```bash
cd /Users/menglj/work/codes/OpenMAIC
git fetch upstream
git checkout main && git merge upstream/main && git push origin main
git checkout my-custom && git rebase main
# 若冲突：改文件 → git add → git rebase --continue
git push -f origin my-custom     # rebase 改写历史，只对自己的定制分支 force push
```

### 做新功能/改动
- 从 `my-custom` 切 feature 分支改，改完合回 `my-custom`
- **`main` 永远干净**（= 上游）
- 查看我们相对上游改了什么：`git log my-custom --not main --oneline`

## 3. 配置与定制（一处源码补丁）

**原则：尽量和上游一致。仅一处源码补丁（`llm.ts` 的 max_tokens），其余差异全在 `.env.local`（不进 git）+ 页面配置。** `my-custom` 比上游多：`CLAUDE.md`（本文件）+ 一行 max_tokens 补丁。

### 源码补丁：`lib/ai/llm.ts` → `injectProviderOptions`
- 默认 `maxOutputTokens` 从 SDK 默认 **4096 → 16384**
- 原因：OpenMAIC 所有 LLM 调用都没传 max_tokens（`AICallFn` 签名不支持 per-call 传），用 SDK 默认 4096，**游戏/互动 HTML 等长输出被截断** → `extractHtml` 提取不到闭合 `</html>` → 场景生成失败（症状：`Could not extract HTML from game response`）
- 这是 OpenMAIC 的设计缺陷，本地补丁绕过；可上报上游
- rebase 冲突面：极小（`injectProviderOptions` 函数开头几行）

### 源码补丁：`lib/agent/runtime/build-agent.ts` → `buildSystemPrompt`
- 给「专业模式 / Edit with AI」agent 的系统指令加了**编辑互动页的行为规范**（5 条硬约束）
- 规范内容（针对 `edit_interactive_html`）：① 改前必须 `read_scene_content` 读全 HTML，理解音频/时序全局；② 时序用事件回调（onend/onerror/用户输入）驱动，禁止用固定 `setTimeout` 猜时长；③ 不在 TTS ↔ Web Audio 等技术间反复横跳，不确定环境能力就探测或问用户；④ 不能运行代码，禁止谎报"已修复"，要说明"未验证、请刷新确认"；⑤ 做不到的（如逼真动物叫声）直说，建议简化或手改，不要绕圈子
- 原因：默认 agent 改互动页时反复绕圈子（看不全就改、赌延迟、来回换方案、未验证就说修好），把那次失败教训固化成硬约束
- rebase 冲突面：小（`buildSystemPrompt` 数组里加一条规则字符串）

### A. server 端配置（`.env.local`，被 gitignore，不进 git）
只有这 4 行（server 端异步任务必需，`DEFAULT_MODEL` 必须指向 server 配置的 provider）：
```
ANTHROPIC_API_KEY=<GLM key>
ANTHROPIC_BASE_URL=https://open.bigmodel.cn/api/anthropic/v1   # 必须 带 /v1
ANTHROPIC_MODELS=glm-5.2,glm-4.6
DEFAULT_MODEL=anthropic:glm-5.2
NEXT_PUBLIC_MAIC_EDITOR_ENABLED=true   # 启用「专业模式」开关（不设则开关不渲染）；NEXT_PUBLIC_* 构建时注入，改后必须重启 dev
LLM_THINKING_DISABLED=true             # 可选：禁 thinking 把 token 让给长输出，配合 §3 源码补丁防游戏 HTML 截断；想要更高质量删掉
```
**为什么用 Anthropic provider 接 GLM（不是 GLM provider）**：用户的 GLM Code Plan 额度在 Anthropic 兼容端点（glm-5.2、glm-4.6 可用）；paas/OpenAI 兼容端点只有 `glm-4-flash` 免费，glm-5.2 等报 `1113 余额不足`。

### B. 页面配置（浏览器 localStorage，一次性手动开）
- **TTS**：设置 → 文字转语音 → 启用「浏览器原生」+ 开声音总开关。OpenMAIC 默认 TTS 关、browser-native opt-in，**手动开一次就持久记住**。GLM Code Plan 无语音额度，只能用浏览器原生。
- **模型**：生成时选 `anthropic:glm-5.2`
- **thinking**：默认禁用（`.env.local` 的 `LLM_THINKING_DISABLED=true`），配合 §3 源码补丁避免长输出截断；想要更高质量可删掉让它推理

### C. 已放弃的方案（别走回头路）
- ❌ 不改 `settings.ts` 默认开 TTS —— 破坏上游"native 不该静默默认"的设计；页面开一次即可，源码零改动优先

## 4. ⚠️ 已知坑（必读，避免重踩）

### 坑 1：ANTHROPIC_* 环境变量污染 dev server（最难查的一个）
- Claude Code 进程自带 `ANTHROPIC_BASE_URL`（**无 /v1**）、`ANTHROPIC_AUTH_TOKEN` 等
- 这些被 `pnpm dev` 继承；Next.js 加载 `.env.local` 时**系统环境变量优先、不被覆盖**
- 结果：SDK 请求路径缺 `/v1`（`/api/anthropic/messages` 而非 `/api/anthropic/v1/messages`）→ GLM 404 → LLM 返回空响应
- **由 Claude Code 启动 dev 时，必须剥离这些变量**：
  ```bash
  env -u ANTHROPIC_BASE_URL -u ANTHROPIC_AUTH_TOKEN -u ANTHROPIC_API_KEY -u ANTHROPIC_MODEL pnpm dev
  ```
- 用户的普通终端没有这些变量，直接 `pnpm dev` 即可
- 诊断手法：在 `lib/ai/providers.ts` 的 anthropic case 临时打印 `config.baseUrl` 和 `process.env.ANTHROPIC_BASE_URL` 对比

### 坑 2：GLM Code Plan 的能力边界
- paas 端点（`open.bigmodel.cn/api/paas/v4`）：只有 `glm-4-flash` 免费，glm-5.2/4.6/4.5 等全部 `1113`
- anthropic 端点（`/api/anthropic/v1`）：`glm-5.2`、`glm-4.6` 可用 ← **LLM 走这里**
- **TTS（glm-tts）完全不可用**：Code Plan 无语音额度，调 `/audio/speech` 返回 `1113`。所以 TTS 用浏览器原生

### 坑 3：pnpm verify-deps-before-run
- 本机 pnpm 开了「跑前依赖校验」，`pnpm dev`/`pnpm start` 前会自动 `pnpm install`
- **必须在项目根目录执行**，否则在上一层目录报 `ERR_PNPM_NO_PKG_MANIFEST`
- 依赖已装好（`pnpm install` 跑过）；如果 native 包脚本被跳过影响功能，跑 `pnpm approve-builds`

### 坑 4：TTS 出声依赖系统层
- 引擎用「浏览器原生 (Web Speech API)」，靠 Mac/浏览器自带语音包发音
- 没声音先查：① 浏览器是否拦截自动播放 ② Mac 是否装了中文语音包（系统设置 → 辅助功能 → 语音内容）

## 5. 数据存储说明（改存储相关功能前必读）

**OpenMAIC 的所有用户数据都在浏览器本地，server 端无持久化、无数据库。**

| 数据 | 存哪 | 代码 |
|---|---|---|
| 设置 / provider key / 模型选择 / TTS / 布局偏好 | **localStorage**（key=`settings-storage`） | `lib/store/settings.ts` |
| 用户资料（头像/昵称/简介） | **localStorage** | `lib/store/user-profile.ts` |
| 课堂内容（大纲/场景/编辑快照历史） | **IndexedDB**（via Dexie） | `lib/store/stage.ts`、`snapshot.ts` |
| 语音克隆参考片段（VoxCPM） | **IndexedDB** | — |
| server 端 | **无持久化**（API routes 无状态，调 LLM 即返） | — |

开发时的注意点：
1. **server 端无状态、无数据库** —— 不要假设 server 存了用户数据；API routes 只调 LLM 返回。要持久化必须走客户端存储或外接存储。
2. **数据浏览器本地绑定** —— 换浏览器/换设备/清缓存就丢，**不跨设备同步、无账号系统**。做"多设备/多人协作"类功能要先意识到这点（当前架构不支持）。
3. **改存储格式要做迁移** —— 参考 `lib/store/settings.ts` 的 `version` + `migrate`（我们改 TTS 默认值就用了 version 4→5 migrate）。改 IndexedDB schema 同样要考虑老用户数据。
4. **备份/迁移靠导出** —— 课堂页导出 `.maic.zip`（完整课堂）/ `.pptx` / `.html`。
5. **key 是明文存 localStorage** —— `settings-storage` 里含明文 API key（含你的 GLM key），F12 → Application → Local Storage 可见；注意别让截图/日志泄露。

## 6. 开发约定

- **配置外置优先**：API key、base URL、provider 选择放 `.env.local` 或 `server-providers.yml`（都被 gitignore），**尽量不改源码**。源码改动是 rebase 冲突的主要来源。
- **源码改动**：集中在少数文件、独立 commit、只进 `my-custom`。能用 env/配置开关实现的，就别硬改源码。
- **不主动 commit/push**（用户全局约定）：改完只展示 diff/改动摘要，等用户明确说「提交 / commit / push」再执行。同样适用于建分支、tag、force push。
- **文档依赖单向**：设计文档（TDD/PRD）不引用审查报告；审查→设计可以，反向不行（用户偏好）。
- 用户偏好产物统一放 `/Users/menglj/work/{任务名}/` 下（本项目的开发指南除外，它必须在仓库内供 AI 读取）。

## 7. 常用命令速查

```bash
cd /Users/menglj/work/codes/OpenMAIC

# 启动 dev（普通终端）
pnpm dev
# → http://localhost:3000   局域网: http://192.168.56.207:3000

# Claude Code 会话里启动 dev（必须剥离 ANTHROPIC 污染，见坑 1）
env -u ANTHROPIC_BASE_URL -u ANTHROPIC_AUTH_TOKEN -u ANTHROPIC_API_KEY -u ANTHROPIC_MODEL pnpm dev

# 同步上游更新
git fetch upstream
git checkout main && git merge upstream/main && git push origin main
git checkout my-custom && git rebase main && git push -f origin my-custom

# 查看本仓库相对上游的定制改动
git log my-custom --not main --oneline

# 装/更新依赖
pnpm install
```

## 8. 环境要求
- Node >= 20（本机 v26）
- pnpm >= 10（本机 11.9，全局通过 npm 装的）
- LLM：需要至少一个 provider key；当前用 GLM Code Plan（走 anthropic 端点）

---

**一句话给后续 AI**：这是 OpenMAIC 的 fork，源码仅**一处补丁**（`llm.ts` 的 max_tokens 4096→16384，修游戏 HTML 截断）+ `CLAUDE.md`，其余定制在 `.env.local`（智谱 GLM 走 Anthropic 端点）+ 页面（TTS/模型）；`main` 跟上游；启动 dev 用普通终端（避免 `ANTHROPIC_*` 环境变量污染）；不要主动 commit。
