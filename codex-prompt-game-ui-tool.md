# Codex Prompt: 游戏 UI/UX 智能设计工具

> 将此提示词交给 Codex，配合 subagents 并行开发。建议使用 GPT-5.3-codex 或更新模型。
> 配置要求：`approval_policy = "never"`, `sandbox_mode = "danger-full-access"`, `max_threads = 4`

---

## 项目概述

构建一个面向游戏 UI/UX 设计师的 Web 应用，核心能力：

1. **设计画布** — 标准 UI/UX 设计器，支持拖拽、图层、属性编辑
2. **截图生 UI** — 上传游戏截图，AI 自动生成匹配风格的 UI 布局
3. **AI 辅助 UX** — 自然语言描述需求，AI 生成 UX 方案和交互流程
4. **元素智能拆分** — 上传 UI 图片，AI 分割出每个元素（透明通道 PNG），可拖拽复用

---

## 技术栈约束

| 层级 | 选型 |
|------|------|
| 前端框架 | **Next.js 14 + TypeScript**（App Router） |
| UI 组件 | **shadcn/ui + Tailwind CSS** |
| 设计画布 | **Fabric.js 6**（SVG/Canvas 混合渲染） |
| 后端 API | **Next.js API Routes**（同仓库，无需额外部署） |
| AI 调用层 | **通用 API 网关**（用户自配模型，支持 OpenAI 兼容格式） |
| 元素分割 | **SAM 2 + RMBG-2**（可通过用户自己的 Replicate / 本地部署接入） |
| 存储 | **Supabase**（PostgreSQL + Storage） |
| 部署 | **Vercel** |

### AI 通用网关设计（核心设计决策）

**不绑定任何厂商。** 所有 AI 调用走统一抽象层，用户在设置页自行配置。

工具内置 4 个 AI 能力槽位，每个槽位独立配置：

| 槽位 | 用途 | 需要的能力 | 推荐模型（可选） |
|------|------|-----------|-----------------|
| **Chat** | UX 对话分析 | 文本理解 + 生成 | DeepSeek-V3 / GPT-5.5 / Qwen3 / Claude |
| **Vision** | 截图→布局JSON | 图像理解 → 结构化输出 | GPT-5.5 / GPT-4o / Qwen-VL / Gemini Vision |
| **Image** | UI 素材生成 | 文生图 / 图生图 | GPT-image-2 / 豆包 Seedream / Flux / SD3 |
| **Segment** | 元素分割+抠图 | 目标检测 + 背景移除 | SAM 2 + RMBG-2（Replicate / 本地） |

**统一 API 调用格式**（OpenAI 兼容）：

```typescript
// src/lib/ai-client.ts — 通用 AI 客户端
interface AIProvider {
  id: string;                    // 用户自定义名称
  baseURL: string;               // https://api.deepseek.com/v1 | https://ark.cn-beijing.volces.com/api/v3
  apiKey: string;
  model: string;                 // deepseek-chat | doubao-seedream-4.0 | gpt-5.5
  slot: 'chat' | 'vision' | 'image' | 'segment';
}

// 所有 API Route 通过此客户端调用，不直接 import OpenAI SDK
async function aiCall(providers: AIProvider[], slot: string, input: AIInput): Promise<AIOutput>
```

**设置页 UI**：独立页面 `/settings`，每个槽位一个配置卡片：
- 提供商下拉（OpenAI / DeepSeek / 豆包 / 硅基流动 / 自定义）
- Base URL 输入框
- API Key 输入框（密码掩码）
- 模型名输入框
- 连接测试按钮

**内置预设**（一键填入 Base URL + 推荐模型）：

| 服务商 | Chat 模型 | Vision 模型 | Image 模型 |
|--------|----------|------------|------------|
| OpenAI | gpt-5.5 | gpt-5.5 | gpt-image-2 |
| DeepSeek | deepseek-chat | —（不支持Vision） | — |
| 豆包（火山引擎） | doubao-pro-256k | doubao-vision-pro | doubao-seedream-4.0 |
| 硅基流动 | Qwen/Qwen3-235B | Qwen/Qwen2.5-VL-72B | stabilityai/stable-diffusion-3.5 |
| 自定义 | 用户填 | 用户填 | 用户填 |

---

## 设计系统（Design System）

**双模式设计**：UI 面板支持暗色/亮色切换，画布背景独立控制（暗/亮/透明棋盘格）。
默认启动暗色模式（对标 Figma），用户在顶部工具栏一键切换。

### 色彩 Tokens（双模式）

所有颜色通过 CSS 变量驱动，shadcn/ui 的 `dark` / `light` 类名自动切换。

| Token | Dark 模式 | Light 模式 | 用途 |
|-------|-----------|------------|------|
| `--bg-root` | `#0f0f1a` | `#f8f8fa` | 页面根背景 |
| `--bg-surface` | `#161625` | `#ffffff` | 面板/侧边栏背景 |
| `--bg-card` | `#1c1c2e` | `#ffffff` | 卡片/弹窗背景 |
| `--bg-hover` | `rgba(255,255,255,0.04)` | `rgba(0,0,0,0.04)` | 悬停高亮 |
| `--bg-selected` | `rgba(124,58,237,0.08)` | `rgba(124,58,237,0.06)` | 选中态背景 |
| `--bg-canvas-container` | `#161625` | `#e8e8ec` | 画布外层容器底（与面板区分） |
| `--border-default` | `rgba(255,255,255,0.06)` | `rgba(0,0,0,0.08)` | 默认边框 |
| `--border-hover` | `rgba(255,255,255,0.12)` | `rgba(0,0,0,0.15)` | 悬停边框 |
| `--border-active` | `rgba(255,255,255,0.18)` | `rgba(0,0,0,0.22)` | 激活边框 |
| `--text-primary` | `rgba(255,255,255,0.92)` | `rgba(0,0,0,0.88)` | 主要文字 |
| `--text-secondary` | `rgba(255,255,255,0.60)` | `rgba(0,0,0,0.55)` | 次要文字 |
| `--text-tertiary` | `rgba(255,255,255,0.38)` | `rgba(0,0,0,0.35)` | 辅助/禁用文字 |
| `--accent-primary` | `#7c3aed` | `#7c3aed` | 品牌主色（紫） |
| `--accent-hover` | `#a78bfa` | `#8b5cf6` | 悬停色 |
| `--accent-active` | `#5b21b6` | `#6d28d9` | 点击色 |
| `--success` / `--danger` / `--warning` | `#22c55e` / `#ef4444` / `#f59e0b` | 同 dark | 语义色 |

**Accent 色跨模式保持一致**，保证品牌识别度。

### 画布背景独立控制

画布（Fabric.js 内部）的底色与 UI 面板模式解耦，提供 3 种预设：

| 预设 | 背景色 | 适用场景 |
|------|--------|----------|
| **深色画布** | `#1a1a2e` | 暗色游戏 UI 设计（RPG/FPS） |
| **浅色画布** | `#ffffff` | 休闲/二次元/明亮风格游戏 UI |
| **透明棋盘格** | 16px 灰白格子 | 需要看透明通道时（图标/元素拆分后预览） |

**实现方式**：
- 工具栏上放 3 个画布背景切换按钮（🌙 深色 / ☀️ 浅色 / ▦ 棋盘格）
- 切换时只改 Fabric.js 画布的 `backgroundColor` 属性，不影响 UI 面板
- 棋盘格：CSS `background-image` 用 `repeating-conic-gradient` 或 Canvas 内绘制

### 排版（双模式通用）

| 层级 | 字号 | 字重 | 用途 |
|------|------|------|------|
| Heading | 18px | 500 | 面板标题、区块标题 |
| Body | 14px | 400 | 正文、列表项 |
| Caption | 12px | 400 | 辅助说明、标签 |
| Micro | 11px | 400 | 快捷键提示、极小标注 |

**字体栈**：`"Inter", -apple-system, sans-serif`（UI） + `"JetBrains Mono", monospace`（数值/坐标输入）。

### 间距（双模式通用）

| 层级 | 值 | 用途 |
|------|-----|------|
| Tight | 4–8px | 图标与文字、行内元素 |
| Base | 12px | 面板 padding、组件内间距 |
| Comfortable | 16px | 卡片间距 |
| Spacious | 20–32px | 大区块分隔 |
| Nav height | 48px | 顶部导航栏 |
| Sidebar | 260px | 左侧图层面板 |
| Inspector | 300px | 右侧属性面板 |

### 组件规范（双模式适配）

- **按钮**：默认 `outline` 风格（透明底 + `--border-default` 描边），主操作按钮使用品牌紫微渐变底（`linear-gradient(135deg, #7c3aed, #6d28d9)`）
- **输入框**：`--bg-card` 底 + `--border-default` 边框，focus 时边框变 `--accent-primary`
- **卡片/面板**：`--bg-surface` 底 + `--border-default` 边框，圆角 `8px`
- **选中态**：`--bg-selected` 底 + 左侧 3px `--accent-primary` 竖线指示
- **图标**：使用 `lucide-react` 图标库，尺寸 16px（组件内）/ 20px（工具栏）
- **滚动条**：宽度 6px，轨道透明，滑块 `rgba(128,128,128,0.3)`（双模式通用），悬停 `rgba(128,128,128,0.5)`
- **主题切换按钮**：放在顶部导航栏右侧，一个 Sun/Moon 图标 toggle，点击切换 `document.documentElement` 的 `dark` class

### 动画（双模式通用）

- 过渡统一使用 `transition-all duration-150`（hover/focus）或 `duration-200`（展开/折叠）
- 主题切换使用 `transition-colors duration-300` 让颜色平滑过渡
- 面板展开使用 CSS `@starting-style` + `transition` 实现高度动画
- 元素拆分完成的缩略图出现使用 `opacity` + `transform: scale(0.95) → scale(1)` 微动效

### Tailwind / shadcn 配置要点

```typescript
// tailwind.config.ts 关键覆盖
theme: {
  extend: {
    borderRadius: {
      sm: '4px',
      md: '6px',
      lg: '8px',  // 主要使用
      xl: '12px',
    },
    fontSize: {
      micro: '11px',
      caption: '12px',
      body: '14px',
      heading: '18px',
    }
  }
}
// globals.css 中定义 :root (light) 和 .dark 两套 CSS 变量
// shadcn/ui 的 darkMode: 'class' 配置自动处理组件主题
```

### globals.css 关键结构

```css
@tailwind base;
@tailwind components;
@tailwind utilities;

/* Light 模式（默认） */
:root {
  --bg-root: #f8f8fa;
  --bg-surface: #ffffff;
  --bg-card: #ffffff;
  --bg-hover: rgba(0,0,0,0.04);
  --bg-selected: rgba(124,58,237,0.06);
  --bg-canvas-container: #e8e8ec;
  --border-default: rgba(0,0,0,0.08);
  --border-hover: rgba(0,0,0,0.15);
  --border-active: rgba(0,0,0,0.22);
  --text-primary: rgba(0,0,0,0.88);
  --text-secondary: rgba(0,0,0,0.55);
  --text-tertiary: rgba(0,0,0,0.35);
  --accent-primary: #7c3aed;
  --accent-hover: #8b5cf6;
  --accent-active: #6d28d9;
}

/* Dark 模式（应用默认使用此模式） */
.dark {
  --bg-root: #0f0f1a;
  --bg-surface: #161625;
  --bg-card: #1c1c2e;
  --bg-hover: rgba(255,255,255,0.04);
  --bg-selected: rgba(124,58,237,0.08);
  --bg-canvas-container: #161625;
  --border-default: rgba(255,255,255,0.06);
  --border-hover: rgba(255,255,255,0.12);
  --border-active: rgba(255,255,255,0.18);
  --text-primary: rgba(255,255,255,0.92);
  --text-secondary: rgba(255,255,255,0.60);
  --text-tertiary: rgba(255,255,255,0.38);
  /* accent 保持不变 */
}
```

---

## Phase 0：项目脚手架 + 设置页（API 配置优先）

### 任务
搭建 Next.js 项目骨架，**优先实现设置页面**，因为后续所有 AI 功能都依赖用户配置的 API。

**设置页 `src/app/settings/page.tsx`：**
- 4 个 AI 槽位配置卡片：Chat / Vision / Image / Segment
- 每张卡片：服务商下拉（内置预设）→ 自动填 Base URL + 推荐模型 → 用户填 API Key → 测试连接按钮
- API Key 存储到 localStorage（不经过后端，隐私优先）
- 设置保存后写入全局状态（Zustand persist）

**通用 AI 客户端 `src/lib/ai-client.ts`：**
- 统一 `aiCall(slot, input)` 接口，根据用户配置路由到不同厂商
- 处理 OpenAI / DeepSeek / 豆包 / 硅基流动 的细微差异（如豆包需要 `Authorization: Bearer` 头）
- Vision 槽位：自动将 base64 图片转为对应格式
- Image 槽位：适配各厂商的生图 API 差异

**内置服务商预设：**

```typescript
// src/lib/providers.ts
export const PROVIDER_PRESETS = {
  openai: {
    name: 'OpenAI',
    chat: { baseURL: 'https://api.openai.com/v1', model: 'gpt-5.5' },
    vision: { baseURL: 'https://api.openai.com/v1', model: 'gpt-5.5' },
    image: { baseURL: 'https://api.openai.com/v1', model: 'gpt-image-2' },
  },
  deepseek: {
    name: 'DeepSeek',
    chat: { baseURL: 'https://api.deepseek.com/v1', model: 'deepseek-chat' },
    vision: null, // DeepSeek 不支持 Vision
    image: null,
  },
  doubao: {
    name: '豆包（火山引擎）',
    chat: { baseURL: 'https://ark.cn-beijing.volces.com/api/v3', model: 'doubao-pro-256k' },
    vision: { baseURL: 'https://ark.cn-beijing.volces.com/api/v3', model: 'doubao-vision-pro' },
    image: { baseURL: 'https://ark.cn-beijing.volces.com/api/v3', model: 'doubao-seedream-4.0' },
  },
  siliconflow: {
    name: '硅基流动',
    chat: { baseURL: 'https://api.siliconflow.cn/v1', model: 'Qwen/Qwen3-235B' },
    vision: { baseURL: 'https://api.siliconflow.cn/v1', model: 'Qwen/Qwen2.5-VL-72B' },
    image: { baseURL: 'https://api.siliconflow.cn/v1', model: 'stabilityai/stable-diffusion-3.5' },
  },
  custom: {
    name: '自定义',
    chat: null, vision: null, image: null, // 用户自行填写
  },
};
```

### Subagents
```
Subagent A（设置页 UI + 服务商预设）:
  实现 /settings 页面，4 个槽位配置卡片，服务商下拉 + 自动填充 + 连接测试。
  创建 providers.ts 预设数据。
  创建文件：src/app/settings/page.tsx, src/lib/providers.ts

Subagent B（通用 AI 客户端）:
  实现 ai-client.ts，统一 4 个槽位的调用接口。
  处理各厂商差异（Auth 头格式、图片 base64 格式、错误码映射）。
  创建文件：src/lib/ai-client.ts

Subagent C（全局状态 + localStorage）:
  创建 Zustand store，管理 AI 配置（4 个槽位），支持 localStorage 持久化。
  创建文件：src/store/settingsStore.ts
```

---

## Phase 1：项目脚手架 + 设计画布

### 任务
搭建 Next.js 项目，实现基础设计画布（Fabric.js），支持：
- 画布无限缩放、平移
- 矩形、圆形、文本、图片元素的创建/选中/拖拽/缩放/旋转
- 图层面板（排序、显隐、锁定、重命名）
- 属性面板（位置、尺寸、颜色、透明度、圆角）
- 导出为 PNG/SVG
- 撤销/重做（Ctrl+Z / Ctrl+Shift+Z）

### Subagents
请并行启动 3 个 subagents：

```
Subagent D（画布核心）:
  实现 Fabric.js 画布初始化、元素创建/选中/变换、撤销重做栈。
  创建文件：src/components/canvas/GameCanvas.tsx, src/hooks/useCanvasHistory.ts

Subagent E（图层面板 + 属性面板）:
  实现图层面板（排序/显隐/锁定）和属性面板（位置/尺寸/颜色/透明度）。
  创建文件：src/components/panels/LayersPanel.tsx, src/components/panels/PropertiesPanel.tsx

Subagent F（工具栏 + 导出）:
  实现顶部工具栏（创建元素按钮、缩放控制）和导出功能（PNG/SVG）。
  创建文件：src/components/toolbar/Toolbar.tsx, src/utils/export.ts
```

---

## Phase 2：截图生 UI 布局

### 任务
用户上传游戏截图 → AI 分析布局结构 → 自动在设计画布中生成匹配的 UI 框架。

**处理流程：**
1. 用户上传/粘贴截图到上传区域
2. 调用 **Vision 槽位** 的 AI 模型分析截图：
   - 识别各 UI 区域类型（标题栏、按钮组、列表、HUD、对话框等）
   - 输出 JSON 布局描述（每个区域的 bounds、类型、层级关系、颜色/字体风格）
3. 根据 JSON 在 Fabric.js 画布中生成占位框架（半透明区域 + 标注）
4. 用户可在此基础上替换素材、调整布局

### Subagents
```
Subagent G（上传组件 + API 路由）:
  实现拖拽/粘贴上传区域，调用 **Vision 槽位** 的 AI 模型，解析返回的布局 JSON。
  创建文件：src/components/upload/ScreenshotUploader.tsx, src/app/api/analyze-screenshot/route.ts

Subagent H（布局生成器）:
  根据 AI 返回的 JSON 在 Fabric.js 画布中生成占位框架（带标注的半透明矩形）。
  创建文件：src/utils/layoutGenerator.ts, src/components/canvas/LayoutOverlay.tsx
```

**Vision 槽位 System Prompt（截图分析）：**
```
你是一个游戏 UI 布局分析专家。分析上传的游戏截图，输出严格的 JSON 格式：

{
  "canvasSize": { "width": 1920, "height": 1080 },
  "style": { "theme": "fantasy|sci-fi|modern|...", "colorPalette": ["#hex", ...], "fontStyle": "serif|sans|..." },
  "elements": [
    {
      "id": "elem_1",
      "type": "panel|button|text|list|hud|dialog|icon",
      "bounds": { "x": 0, "y": 0, "width": 300, "height": 100 },
      "label": "顶部标题栏",
      "zIndex": 1,
      "style": { "bgColor": "#...", "borderRadius": 8, "opacity": 0.9 }
    }
  ]
}

规则：
- bounds 使用相对于截图左上角的像素坐标，只需粗略估算
- type 从枚举中选择最匹配的
- 每个可见 UI 元素都要列出，不要遗漏
- 只输出 JSON，不要添加任何解释文字
```

---

## Phase 3：AI 辅助 UX 生成

### 任务
用户在对话框中用自然语言描述需求 → AI 生成 UX 流程图和线框图方案。

**交互方式：**
- 侧边栏聊天面板，用户输入如"设计一个 MOBA 游戏的英雄选择界面，包含英雄列表、技能预览、确认按钮"
- AI 返回：
  1. 文字版 UX 分析（信息架构、交互流程、关键决策点）
  2. ASCII 线框图或 Mermaid 流程图
  3. 可选：自动在设计画布中生成对应的占位布局

### Subagents
```
Subagent I（UX 聊天面板 + API）:
  实现聊天 UI 组件和 API 路由，调用 **Chat 槽位** 的 AI 模型做游戏 UX 分析。
  创建文件：src/components/chat/UXChatPanel.tsx, src/app/api/generate-ux/route.ts

Subagent J（UX 可视化）:
  将 AI 返回的 UX 方案渲染为 Mermaid 流程图，并提供"应用到画布"按钮。
  创建文件：src/components/chat/UXFlowChart.tsx, src/components/chat/UXWireframe.tsx
```

**Chat 槽位 System Prompt（UX 分析）：**
```
你是资深游戏 UX 设计师，专精手游/端游交互设计。用户会描述一个游戏界面需求，请输出：

1. **信息架构** — 该界面需要展示哪些信息，优先级如何排序
2. **交互流程** — 用户操作的完整路径（用 Mermaid flowchart 语法）
3. **布局建议** — 每个区域的建议位置和尺寸（用 JSON 格式，同 Phase2 的 elements 结构）
4. **游戏化考虑** — 如何增强反馈感、成就感、引导性

输出格式：Markdown，结构清晰，结论先行。
```

---

## Phase 3.5：AI UI 素材生成（Image 槽位）

### 任务（新增差异化功能）
利用用户配置的 Image 槽位 AI 模型（如豆包 Seedream / GPT-image-2 / Flux），在工具内直接生成游戏 UI 素材。

**素材类型（纯文字提示词生成）：**
- 按钮（各种状态：normal/hover/pressed/disabled）
- 面板/卡片（不同风格：木质/金属/魔法/科幻）
- 图标（技能图标、道具图标、货币图标）
- 边框/装饰元素（花纹边框、能量条、进度条）
- HUD 元素（血条、蓝条、小地图、准星）

**工作流：**
1. 用户选择素材类型 + 输入风格描述 + 可选上传风格参考图
2. 调用 Image 槽位的 AI 模型生成（支持多轮迭代调整）
3. 生成的素材直接拖入 Fabric.js 画布使用

### Subagents
```
Subagent K（素材生成面板 + API）:
  实现素材生成面板 UI（类型选择、风格描述输入、参考图上传、结果预览）。
  调用 Image 槽位的 AI 模型（OpenAI 兼容格式，支持多轮编辑）。
  创建文件：src/components/generate/AssetGenerator.tsx, src/app/api/generate-asset/route.ts

Subagent L（素材管理 + 画布集成）:
  管理生成的素材历史（缩略图网格），支持拖入画布、删除、重新生成。
  创建文件：src/components/generate/AssetLibrary.tsx
```

**Image 槽位 API 调用示例（OpenAI 兼容格式）：**
```typescript
// src/app/api/generate-asset/route.ts
// 通过 ai-client.ts 的 Image 槽位调用，不绑定具体厂商
import { getProviderForSlot, aiCall } from '@/lib/ai-client';

export async function POST(req: Request) {
  const { prompt, referenceImage, assetType, style } = await req.json();
  const provider = await getProviderForSlot('image');

  const result = await aiCall(provider, {
    prompt: `Generate a game UI ${assetType} in ${style} style. ${prompt}. Clean design, game-ready quality.`,
    referenceImage: referenceImage || undefined,
  });

  return Response.json({ imageUrl: result.imageUrl });
}
```

**注意**：多数生图模型不支持透明通道输出。如需透明素材，生成后走 Phase 4 的 Segment 槽位去背景流程。

---

## Phase 4：图片元素智能拆分

### 任务（核心差异化功能）
用户上传 UI 图片 → AI 自动分割出每个独立元素 → 输出透明通道 PNG → 在设计画布中作为独立可拖拽图层。

**处理流程：**
1. 用户上传/粘贴 UI 图片到拆分面板
2. 调用分割 API（SAM 2 / 专用模型）识别图中各元素边界
3. 对每个检测到的元素：裁剪 + 背景移除 → 生成透明 PNG
4. 所有元素以缩略图列表展示，用户可拖入画布
5. 支持手动调整分割结果（合并/拆分选区）

### Subagents
```
Subagent M（拆分面板 UI + API 路由）:
  实现拆分功能面板（上传区、结果列表、拖入画布），对接分割 API。
  创建文件：src/components/decompose/DecomposePanel.tsx, src/app/api/decompose-image/route.ts

Subagent N（分割模型集成 + 后处理）:
  封装 Segment 槽位的分割/背景移除 API 调用，实现元素裁剪和透明 PNG 生成。
  处理合并/拆分选区的手动调整逻辑。
  创建文件：src/utils/segmentation.ts, src/components/decompose/SegmentEditor.tsx
```

**Segment 槽位推荐方案（用户自行选择接入方式）：**

| 方案 | 接入方式 | 优点 | 缺点 |
|------|---------|------|------|
| Replicate SAM 2 | API Key | 精度最高、无需 GPU | COCO 类物体为主，UI 元素需微调 |
| HuggingFace 专用 UI 检测模型 | API Key | 针对 UI 优化 | 模型较少 |
| 本地 SAM 2 部署 | 用户自己的服务器 | 免费、隐私 | 需 GPU |
| RMBG-2 背景移除 | Replicate / 本地 | 抠图质量高 | 单独一步 |

**Segment 槽位调用示例（以 Replicate 为例，用户可替换为自己的接口）：**
```typescript
// src/utils/segmentation.ts
// 通过 Segment 槽位配置调用，不绑定厂商
import { getProviderForSlot } from '@/lib/ai-client';

// 1. 先用 SAM 2 自动分割（或用户配置的其他分割模型）
const masks = await segmentImage(base64Image, provider);

// 2. 对每个 mask 区域裁剪后用背景移除模型去背景
const transparentElements = await Promise.all(
  masks.map(mask => removeBackground(cropImage(base64Image, mask), provider))
);
// 返回透明 PNG 数组
```

---

## Phase 5：整合 + 打磨

### 任务
- 所有模块整合到统一界面，侧边栏切换功能
- 响应式布局适配
- Loading 状态、错误处理、空状态引导
- 键盘快捷键（Space+拖拽平移画布、Delete 删除元素、Ctrl+D 复制等）
- 性能优化（虚拟化图层面板、canvas 脏矩形重绘）

### Subagents
```
Subagent O（整合 + 布局）:
  将所有模块组合到主页面，实现侧边栏导航切换。
  创建文件：src/app/page.tsx, src/components/layout/AppShell.tsx

Subagent P（状态管理 + 性能）:
  统一状态管理（Zustand），实现快捷键系统，优化 Canvas 渲染性能。
```

---

## 文件结构期望

```
src/
├── app/
│   ├── page.tsx                    # 主页面
│   ├── layout.tsx                  # 根布局
│   ├── globals.css                 # Tailwind + 自定义样式
│   └── api/
│       ├── analyze-screenshot/     # Phase 2: 截图分析
│       ├── generate-ux/            # Phase 3: UX 生成
│       └── decompose-image/        # Phase 4: 元素拆分
├── components/
│   ├── layout/
│   │   └── AppShell.tsx            # 主壳布局
│   ├── canvas/
│   │   ├── GameCanvas.tsx          # Fabric.js 画布
│   │   ├── LayoutOverlay.tsx       # AI 布局叠加层
│   │   └── ExportDialog.tsx        # 导出对话框
│   ├── toolbar/
│   │   └── Toolbar.tsx             # 顶部工具栏
│   ├── panels/
│   │   ├── LayersPanel.tsx         # 图层面板
│   │   ├── PropertiesPanel.tsx     # 属性面板
│   │   ├── DecomposePanel.tsx      # 拆分面板
│   │   └── ScreenshotUploader.tsx  # 截图上传
│   ├── chat/
│   │   ├── UXChatPanel.tsx         # UX 对话面板
│   │   ├── UXFlowChart.tsx         # 流程图渲染
│   │   └── UXWireframe.tsx         # 线框图渲染
│   └── decompose/
│       ├── SegmentList.tsx         # 分割结果列表
│       └── SegmentEditor.tsx       # 分割编辑器
├── hooks/
│   ├── useCanvasHistory.ts         # 撤销重做
│   └── useKeyboardShortcuts.ts     # 快捷键
├── utils/
│   ├── layoutGenerator.ts          # 布局生成器
│   ├── segmentation.ts             # 图像分割
│   └── export.ts                   # 导出工具
├── store/
│   └── designStore.ts              # Zustand 全局状态
└── types/
    └── index.ts                    # TypeScript 类型定义
```

---

## 关键配置

在项目根目录创建 `.codex/agents/`，将上述 subagents 定义写入。每个 subagent 文件示例：

```toml
# .codex/agents/canvas-core.toml
name = "canvas_core"
description = "实现 Fabric.js 画布核心：初始化、元素变换、撤销重做"
sandbox_mode = "danger-full-access"

developer_instructions = """
你是前端工程师，专注于 Canvas 交互。任务：
1. 使用 Fabric.js 6 创建画布组件 GameCanvas.tsx
2. 实现 useCanvasHistory hook（命令模式的撤销重做）
3. 支持矩形、文本、图片元素的创建/选中/拖拽/缩放/旋转
4. 画布无限缩放（鼠标滚轮）、平移（空格+拖拽）

技术细节：
- Canvas 尺寸跟随容器，支持 ResizeObserver
- 撤销栈用 JSON 序列化 Fabric.js 状态，限制 50 步
- 导出 createElement、deleteElement、updateElement 三个方法供外部调用
- 所有回调通过 props.onChange(canvasState) 通知父组件

不要引入 Fabric.js 之外的重型库。代码风格：TypeScript 严格模式，函数式组件 + hooks。
"""
```

---

## 质量标准

- ✅ 每个 Phase 可独立验证运行
- ✅ 所有 API 路由有 Loading/Error/Empty 三种状态处理
- ✅ 画布操作流畅（60fps，元素 < 200 时）
- ✅ 导出 PNG 分辨率不低于 2x
- ✅ 移动端至少保证画布可查看（不要求编辑）
- ✅ 遵循 shadcn/ui 设计语言，不做花哨的自定义样式

---

## 如何向 Codex 提交

### 方式一：一次性提交全部（推荐先做 Phase 1）

```
请阅读 codex-prompt-game-ui-tool.md，先实现 Phase 0（设置页 + AI 客户端）。
并行启动 settings_ui、ai_client、settings_store 三个 subagents。
```

### 方式二：逐 Phase 推进

```
Phase 0 完成，设置页和 AI 客户端就绪。现在继续 Phase 1（设计画布）。
并行启动 canvas_core、layers_panel、toolbar 三个 subagents。
```

### 方式三：全量并行（项目骨架已确认后）

```
请实现全部 7 个 Phase，并行启动 16 个 subagents。
每个 Phase 完成后汇总进度。
```

---

## 环境变量模板 (.env.local)

> 注意：所有 AI API Key 由用户在设置页自行填入（存 localStorage），不写入 env.local。
> 以下仅存放非用户密钥的服务配置。

```env
# Supabase（数据库 + 文件存储）
NEXT_PUBLIC_SUPABASE_URL=https://xxx.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=eyJxxx
SUPABASE_SERVICE_ROLE_KEY=eyJxxx
```
