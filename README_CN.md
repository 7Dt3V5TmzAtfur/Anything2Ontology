# Anything2Ontology

**将任何文档、URL 或代码仓库转化为结构化本体，让 AI 编程代理能立即基于其进行构建。**

Anything2Ontology 是一个由 4 个模块组成的管道，它能摄取原始媒体（PDF、幻灯片、电子表格、YouTube 视频、GitHub 仓库、网站），提取结构化知识，并组装成一个自包含的本体，附带产品规格说明——可供 AI 编程代理直接使用并开始构建。

```
 ┌──────────┐    ┌───────────────┐    ┌──────────────┐    ┌────────────┐    ┌──────────────┐
 │  输入     │───>│  模块 1       │───>│  模块 2      │───>│  模块 3    │───>│  模块 4      │
 │  文件 &   │    │  Anything2   │    │  Markdown2  │    │  Chunks2  │    │  SKUs2      │
 │  URL      │    │  Markdown    │    │  Chunks     │    │  SKUs     │    │  Ontology   │
 └──────────┘    └───────────────┘    └──────────────┘    └────────────┘    └──────────────┘
  PDF, PPTX,       统一的              按 Token 分割       事实型、关系型、    ontology/
  XLSX, YouTube,   Markdown/JSON       的块                程序型、元 SKU     ├── spec.md
  GitHub 仓库,                          (100K tokens)                         ├── mapping.md
  网站                                                                └── skus/
```

## 为什么需要这个工具？

你有一份 2000 页的法规 PDF。或者一个包含 500 个文件的 GitHub 仓库。或者一堆幻灯片、电子表格和 YouTube 教程。你希望 AI 代理能基于这些知识进行构建。

**问题**：将原始文件直接塞进 LLM 上下文是行不通的——它们太大、太杂乱、充满噪音。

**Anything2Ontology** 通过以下方式解决这一问题：
1. **解析**所有内容为干净的 Markdown
2. **分割**过大的文档为 LLM 友好的小块
3. **提取**结构化的知识单元（事实、技能、关系、创意洞察）
4. **组装**一个包含可导航知识 + 产品规格的本体

输出的 `ontology/` 文件夹设计为：编程代理读取 `spec.md` 即可开始构建，`mapping.md` 作为路由器用于查找相关知识。

## 快速开始

### 本地安装

```bash
git clone https://github.com/kitchen-engineer42/Anything2Ontology.git
cd Anything2Ontology

python -m venv .venv
source .venv/bin/activate

pip install -e .
npm install -g repomix    # 用于 GitHub 仓库解析

cp .env.example .env      # 添加你的 API 密钥
```

### Docker

```bash
docker compose build
docker compose run anything2ontology bash
```

### 运行管道

```bash
# 将文件放入 input/ 或将 URL 添加到 input/urls.txt

anything2md run              # 步骤 1：解析为 Markdown
md2chunks run                # 步骤 2：分割成块
chunks2skus run              # 步骤 3：提取知识
skus2ontology run            # 步骤 4：组装本体（含交互式聊天机器人）
```

或在全自动运行中跳过聊天机器人：

```bash
skus2ontology run --skip-chatbot
```

### Web UI（Streamlit）

如需可视化界面，你也可以通过 Streamlit 运行管道：

```bash
pip install streamlit
streamlit run streamlit_app.py
```

然后在浏览器中打开 http://localhost:8501。

Web UI 提供以下功能：
- 文件上传和 URL 输入
- 带进度显示的逐步管道执行
- 用于查看结果的输出浏览器
- 本体导出和下载
- 通过 Web 聊天机器人直接生成 spec.md

## 模块

### 模块 1：Anything2Markdown

将多种文件类型和 URL 转换为 Markdown 或 JSON。

| 输入 | 解析器 |
|-------|--------|
| PDF | MarkItDown（普通）/ PaddleOCR-VL（扫描件/低质量备用） |
| PPTX、DOCX、媒体文件 | MarkItDown |
| XLSX、CSV | TabularParser（JSON 输出） |
| YouTube URL | YouTubeParser（字幕提取） |
| Bilibili URL | BilibiliParser（CC 字幕或 faster-whisper 转录） |
| GitHub 仓库 | RepomixParser（完整仓库 → 单个 Markdown） |
| 其他 URL | FireCrawlParser（网页爬取） |

```bash
anything2md run                               # 处理所有输入
anything2md parse-file ./input/document.pdf   # 单个文件
anything2md parse-url "https://example.com"   # 单个 URL
```

### 模块 2：Markdown2Chunks

将长 Markdown 分割为受 Token 限制的块（默认 100K Tokens），使用两种策略：

- **标题分割器** — 按 Markdown 标题层级分割（H1 > H2 > H3）
- **LLM 分割器** — 针对非结构化文本的备用方案，使用 LLM 查找语义切分点

```bash
md2chunks run                     # 处理来自模块 1 的所有 Markdown
md2chunks chunk-file <file>       # 分割单个文件
md2chunks estimate-tokens <file>  # 显示 Token 数量
```

### 模块 3：Chunks2SKUs

从块中提取 4 种类型的标准知识单元（SKU）：

| 类型 | 描述 | 输出 |
|------|------|------|
| **事实型** | 事实、定义、数据点 | `sku_NNN/content.md` |
| **关系型** | 类别层级、术语表 | `label_tree.json` + `glossary.json` |
| **程序型** | 工作流程、分步技能 | `SKILL.md`（Claude Code 格式） |
| **元** | 知识地图 + 创意洞察 | `mapping.md` + `eureka.md` |

包含后处理：基于相似度的桶归类、两层去重、基于网络检索的可信度评分。

```bash
chunks2skus run                    # 从所有块中提取
chunks2skus show-index             # 显示 SKU 摘要
chunks2skus postprocess all        # 运行桶归类 + 去重 + 校对
```

### 模块 4：SKUs2Ontology

将 SKU 组装为自包含的本体：

1. **组装** — 复制 SKU、重写内部路径、将关键文件提升到根目录
2. **聊天机器人** — 交互式 LLM 对话，从知识库生成 `spec.md`
3. **README** — 为 AI 代理自动生成的入口文件

```bash
skus2ontology run                    # 完整管道
skus2ontology run --skip-chatbot     # 自动化（无交互式聊天机器人）
skus2ontology assemble               # 仅复制/整理
skus2ontology chatbot -w ontology/   # 仅聊天机器人
```

**输出本体：**
```
ontology/
├── spec.md              # 产品规格说明（来自聊天机器人）
├── mapping.md           # SKU 路由器——按主题查找知识
├── eureka.md            # 创意洞察和功能想法
├── README.md            # AI 编程代理的入口文件
└── skus/
    ├── factual/         # 基于事实的知识单元
    ├── procedural/      # 分步技能
    ├── relational/      # 分类法和术语表
    └── skus_index.json  # 主索引
```

## 配置

复制 `.env.example` 到 `.env` 并设置你的 API 密钥：

```bash
# 必需
SILICONFLOW_API_KEY=       # LLM 功能（模块 2-4）+ 通过 API 的 PaddleOCR-VL

# 可选（启用特定解析器）
FIRECRAWL_API_KEY=         # 网站爬取
JINA_API_KEY=              # 基于网络的可信度评分
```

### 本地 OCR 部署（可选）

对于扫描版 PDF，PaddleOCR-VL 可以在 Apple Silicon 上通过 mlx-vlm 本地运行，无需使用 SiliconFlow API：

```bash
pip install mlx-vlm
cd /tmp && python -m mlx_vlm.server --port 8080 --trust-remote-code
```

然后在 `.env` 中：
```bash
OCR_BASE_URL=http://localhost:8080
PADDLEOCR_MODEL=mlx-community/PaddleOCR-VL-1.5-8bit
```

详见 `.env.example` 了解完整的可配置选项列表（模型、Token 限制、温度参数等）。

## 项目结构

```
src/
├── anything2markdown/     # 模块 1：通用解析器
├── markdown2chunks/       # 模块 2：智能分割
├── chunks2skus/           # 模块 3：知识提取 + 后处理
└── skus2ontology/         # 模块 4：本体组装 + 聊天机器人
```

```
input/          # 在此放置文件和 urls.txt
output/         # 中间输出（Markdown、块、SKU）
ontology/       # 最终输出——交给你的 AI 编程代理
logs/           # 双格式日志（JSON + 纯文本）
```

## 系统要求

- Python 3.10+
- Node.js 20+（用于 `repomix`）
- ffmpeg（用于 Bilibili 音频提取）
- SiliconFlow API 密钥（用于 LLM 功能）

## 许可证

Apache License 2.0 —— 详见 [LICENSE](LICENSE)。