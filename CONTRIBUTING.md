# 贡献指南

欢迎参与 CutDeck 开发！请阅读完整指南：[docs/contributing.md](docs/contributing.md)

## 快速规范

```bash
# 开发
pnpm install
pnpm dev

# 检查
pnpm run lint     # ESLint
pnpm run type-check  # TypeScript
pnpm run build    # 生产构建
pnpm vitest run   # 测试（单线程：--no-file-parallelism）
```

## 分支规范

| 前缀 | 用途 |
|------|------|
| `main` | 主分支，稳定可发布 |
| `feature/*` | 新功能 |
| `fix/*` | Bug 修复 |
| `docs/*` | 文档更新 |
| `refactor/*` | 重构（不影响功能） |

## Commit 规范

使用 `--no-verify` 跳过 commit-msg hook（已内置 pre-commit lint）：

```bash
git commit -m "feat(clipRepurposing): add ClipScorer multi-dimension scoring engine"
git commit -m "fix(asr): implement tryWebSpeechASR fallback"
```

## 发布版本

```bash
# 1. 更新 CHANGELOG.md（添加新版本条目）
# 2. 更新 package.json 版本号
# 3. git tag v{x.y.z}
# 4. gh release create v{x.y.z}
```

## 许可

MIT License
