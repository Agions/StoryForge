---
title: 架构概览
description: CutDeck 系统架构和技术设计的核心要点速查。
---

# 架构概览

本文档帮助你快速理解 CutDeck 的核心系统架构。

---

## 系统架构

CutDeck 采用 **分层模块化架构**：

```
┌──────────────────────────────────────────────────────────────┐
│                        UI 层 (React 18 + TypeScript)          │
│   素材面板 · 时间线面板 · 预览区域 · AI 控制台 · 导出面板       │
├──────────────────────────────────────────────────────────────┤
│                     服务层 (Services — 类）                   │
│   AI 服务 · 视频处理 · 视觉分析 · ASR · 配乐 · 导出服务       │
├──────────────────────────────────────────────────────────────┤
│                    核心层 (Core — 域驱动）                   │
│   工作流引擎 · 状态管理 · 类型定义 · Hooks · 模板系统         │
├──────────────────────────────────────────────────────────────┤
│                     外部依赖层                              │
│       FFmpeg · Tauri v2 IPC · Whisper · AI APIs           │
└──────────────────────────────────────────────────────────────┘
```

**设计原则**：UI 层与业务逻辑完全解耦；核心逻辑通过 Tauri IPC 与 Rust 后端通信。

---

## 技术栈

| 层级 | 技术选型 |
|------|---------|
| UI 框架 | **React 18** + **TypeScript 5** |
| 状态管理 | **Zustand v5**（持久化） |
| 样式 | **Tailwind CSS** + **CSS Variables** |
| 构建工具 | **Vite 6** |
| 桌面框架 | **Tauri v2** |
| AI 服务 | OpenAI / Anthropic / Google / DeepSeek / 阿里 / 智谱 / Kimi |
| ASR | Faster-Whisper（Tauri 后端） |
| 文档 | **VitePress** |

---

## 目录结构

```
CutDeck/
├── src/
│   ├── core/                    # 核心业务（域驱动）
│   │   ├── services/             # 服务层
│   │   │   ├── ai.service.ts       # AI 模型调用
│   │   │   ├── vision.service.ts    # 场景/情绪/关键帧检测
│   │   │   ├── asr.service.ts       # 语音识别
│   │   │   ├── auto-music.service.ts # 配乐推荐
│   │   │   ├── smart-cut.service.ts  # 智能剪辑
│   │   │   ├── clipRepurposing/     # 内容复用管道（v1.6.0）
│   │   │   │   ├── clipScorer.ts    # 多维评分引擎
│   │   │   │   ├── seoGenerator.ts   # SEO 元数据生成
│   │   │   │   ├── multiFormatExport.ts # 多格式导出
│   │   │   │   └── pipeline.ts      # 完整拆条管道
│   │   │   └── workflow/            # 工作流引擎
│   │   │       ├── WorkflowEngine.ts  # 状态机 + 订阅者模式
│   │   │       ├── types.ts         # 类型定义（WorkflowStep 等）
│   │   │       ├── steps/           # 步骤执行器
│   │   │       │   ├── adapters.ts   # IStepExecutor 适配器
│   │   │       │   ├── aiClipStep.ts
│   │   │       │   ├── repurposingStep.ts # 内容复用步骤
│   │   │       │   ├── musicStep.ts
│   │   │       │   ├── timelineStep.ts
│   │   │       │   └── exportStep.ts
│   │   │       └── featureBlueprint.ts # 工作流模式定义
│   │   ├── types.ts             # 全局类型（唯一类型出口）
│   │   ├── hooks/               # React Hooks
│   │   │   ├── useWorkflowEngine.ts  # 工作流 Hook
│   │   │   └── use-editor-state.ts  # 编辑器状态 Hook
│   │   └── templates/           # Prompt 模板
│   │       ├── script.templates.ts
│   │       └── dedup.templates.ts
│   ├── components/              # UI 组件
│   │   ├── AIClipPanel/        # AI 剪辑面板
│   │   ├── AIPanel/            # AI 控制台
│   │   ├── editor/              # 时间线编辑器
│   │   └── Dashboard/           # 仪表板
│   ├── pages/                  # 页面
│   ├── store/                  # Zustand Stores
│   │   ├── appStore.ts         # App 级（主题/通知）
│   │   ├── projectStore.ts     # 项目列表
│   │   └── editorStore.ts      # 编辑器状态
│   └── hooks/                  # 通用 Hooks
│
├── src-tauri/                  # Rust 后端（Tauri v2）
├── docs/                       # VitePress 文档
├── SPEC.md                     # 架构详细文档（代码层面）
└── ROADMAP.md                  # 版本路线图
```

---

## 核心模块

### WorkflowEngine — 工作流引擎

状态机 + 订阅者模式，管理 AI 剪辑完整流程：

```
WorkflowEngine
  ├── _state: EngineState { status, currentStep, completedSteps, progress }
  ├── _data: WorkflowData  ← 所有步骤产出数据
  ├── registerStepExecutor(step, executor)
  ├── run(projectId, mode, config)
  ├── subscribe(fn) → unsubscribe
  └── getData(): WorkflowData
```

**已注册步骤**（v1.6.0）：
`upload → analyze → template-select → script-generate → script-dedup → script-edit → subtitle → ai-clip → repurposing → music → timeline-edit → preview → export`

**工作流模式**：
- `ai-commentary` — AI 解说（严格同步）
- `ai-mixclip` — AI 混剪（自动）
- `ai-first-person` — AI 第一人称
- `ai-repurposing` — AI 内容复用（全自动，10 分钟）

### ClipRepurposingPipeline — 内容复用管道

长视频 → 多短片段自动拆条（v1.6.0）：

```
Stage 1: 场景边界构建候选片段
Stage 2: ClipScorer 6 维度评分
Stage 3: SEO 元数据批量生成（YouTube/TikTok/Instagram）
Stage 4: 多格式导出任务（9:16 / 1:1 / 16:9）
```

### VideoProcessor — 视频处理

```
IVideoProcessor（接口）
    ↑
BaseVideoProcessor（基类）— FFmpeg 缓存、错误归一化、参数校验
    ↑
TauriVideoProcessor（实现）— Tauri invoke 调用 Rust 后端
```

---

## Tauri IPC 通信

```typescript
// 前端调用 Rust 后端
import { invoke } from '@tauri-apps/api/tauri';

const result = await invoke<VideoMetadata>('get_video_metadata', {
  path: '/path/to/video.mp4'
});
```

---

## 相关文档

- 📋 [SPEC.md](../SPEC.md) — 详细架构文档（代码层面）
- 🗺️ [ROADMAP.md](../ROADMAP.md) — 版本路线图
- 🤖 [AI 模型配置](./ai-config.md) — AI 服务集成
- 🔧 [安装配置](./installation.md) — 环境搭建
- ⚙️ [项目结构](./project-structure.md) — 详细目录说明
