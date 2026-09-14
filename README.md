# CourseHub · AI 课程复习网站生成器（Agent Skill）

> 把任意课程的 **PPT + Syllabus** 丢给 AI，自动生成一个**单文件、免费部署、跨设备同步**的课程复习网站。
> 这是一个标准 **Agent Skill**：任何支持 Skill 机制的 AI（Claude Code / Cursor / 豆包等）装入后，
> 只要收到「生成课程复习网站 / 更新到 Week X」的请求，就会自动按本 Skill 的完整工作流执行。

---

## ✨ 它能产出什么

一个功能完整的单文件课程复习网站：

- 📑 多周分页：Syllabus / Week 1..N / Assignment / 必背 / 复习清单 / 课堂笔记
- 🔍 三级搜索：全局搜索、每页搜索、单词表独立搜索
- ✅ 复习清单：勾选 → 一键批量加入 → 内嵌详情 → Know 移出
- 📝 课堂笔记：随时记录，与复习清单 / 单词一样**跨设备同步**
- 📚 词汇速查：内置本周单词表，可手动添加生词
- 📱 移动端完整适配（桌面与移动布局隔离，互不影响）
- ☁️ 可选云端同步：任何设备、任何浏览器打开同一链接，看到同一份数据

**示例成品**：`assets/example.html`（一份真实课程复习网站，可直接打开体验）。

---

## 📦 安装（把本 Skill 装进你的 AI）

1. 把本仓库 `download / clone` 到本地，或直接下载 zip 解压；
2. 将仓库内容放进你的 AI 工具认可的 skills 目录（例如 Claude Code 的
   `~/.claude/skills/` 或项目 `.claude/skills/` 下的 `coursehub/` 文件夹）；
3. 完成。之后对你的 AI 说「根据这周的 PPT 生成课程复习网站」即可。

> 如果你用的 AI 不支持 Skill 机制：直接打开 `references/main-prompt.md`，
> 把里面的提示词整段复制给它，效果一样。

---

## 🚀 使用流程

| 步骤 | 你要做的事 |
|---|---|
| **① 首次建站** | 给 AI 课程名 + Syllabus + 前几周 PPT，说「生成课程复习网站」 |
| **② 每周更新** | 只把本周 PPT 发给 AI，说「更新到 Week X」 |
| **③ 发布分享** | 让 AI 按 `references/deploy-cloudflare.md` 部署，得到链接，分享给同学 |

## 📁 文件结构

```
coursehub/
├── SKILL.md                            # Skill 入口：何时用、工作流程、输出规范
├── references/
│   ├── main-prompt.md                  # 首次建站主提示词（结构/样式/内容/功能/校验清单）
│   ├── weekly-update-prompt.md         # 每周增量更新提示词
│   └── deploy-cloudflare.md            # 免费部署 + 跨设备同步（Worker + KV）完整方案
└── assets/
    └── example.html                    # 示例成品：一份真实课程复习网站
```

---

## 🛠 技术栈

- 产物：**单个 HTML 文件**（CSS / JS 全部内联，零外部依赖，离线可用）
- 前端：原生 HTML / CSS / JavaScript（三级搜索、复习清单、课堂笔记、响应式）
- 部署与同步：Cloudflare Workers + Workers KV（免费额度：10 万读 / 1 千写每天）
- 同步策略：版本号方向判断（云端较新覆盖本地 → 删除可传播；本地较新推回云端 → 刚删/刚加立即同步）

## 📄 License

MIT License —— 自由使用、修改、分享。
