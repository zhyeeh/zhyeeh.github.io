# 项目结构整理

这个仓库是一个基于 Astro/Fuwari 的静态博客站点。下面按用途整理主要目录和文件，后续维护时可以按这个分类查找。

## 内容目录

| 路径 | 用途 |
| --- | --- |
| `src/content/posts/` | 博客文章，Markdown/MDX 内容放在这里 |
| `src/content/spec/about.md` | 关于页面内容 |
| `public/` | 直接复制到站点根目录的静态资源 |
| `src/assets/images/` | 由 Astro 管理和优化的图片资源 |
| `docs/` | 项目说明、内容索引和维护文档 |

## 站点源码

| 路径 | 用途 |
| --- | --- |
| `src/pages/` | 路由页面，例如首页、文章页、归档页、RSS |
| `src/components/` | Astro/Svelte 组件 |
| `src/layouts/` | 页面布局 |
| `src/styles/` | 全局样式、Markdown 样式和主题样式 |
| `src/utils/` | URL、日期、内容处理等工具函数 |
| `src/constants/` | 常量和链接预设 |
| `src/i18n/` | 多语言文案 |
| `src/plugins/` | Markdown/Rehype/Expressive Code 插件 |
| `src/types/` | TypeScript 类型定义 |

## 配置与脚本

| 路径 | 用途 |
| --- | --- |
| `package.json` | 项目依赖和运行脚本 |
| `pnpm-lock.yaml` | pnpm 依赖锁定文件 |
| `astro.config.mjs` | Astro 主配置 |
| `src/content/config.ts` | 内容集合 schema |
| `src/config.ts` | 站点主题与个人信息配置 |
| `tailwind.config.cjs` | Tailwind CSS 配置 |
| `biome.json` | 代码格式化和检查配置 |
| `scripts/new-post.js` | 新建文章脚本 |

## 本地生成目录

这些目录不需要手动维护，已经在 `.gitignore` 中忽略。

| 路径 | 用途 |
| --- | --- |
| `.astro/` | Astro 生成的内容缓存和类型文件 |
| `dist/` | `pnpm run build` 生成的静态站点产物 |
| `node_modules/` | 本地依赖安装目录 |

## Git 与编辑器配置

| 路径 | 用途 |
| --- | --- |
| `.git/` | Git 仓库数据 |
| `.github/` | GitHub Actions、Dependabot 等配置 |
| `.vscode/` | VS Code 工作区建议配置 |
| `.agents/` | Codex/代理相关本地配置 |

## 维护建议

- 新博客文章继续放在 `src/content/posts/`。
- 带图片或附件的文章可以使用子目录，例如 `src/content/posts/my-post/index.md`。
- 已发布文章尽量不要随意移动路径，否则旧链接会变化。
- 模板演示文章统一使用 `模板示例` 分类。
- 个人记录使用 `随笔` 分类。
- 汽车诊断类文章使用 `汽车诊断` 分类。
- 系统架构和 Agent 工作流设计使用 `架构设计` 分类。
- 站点配置和使用说明使用 `站点指南` 分类。
