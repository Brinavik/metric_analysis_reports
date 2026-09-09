---
name: pdf-literature-reading
description: 阅读、检查和结构化处理学术 PDF；根据文档类型选择文本抽取、页面渲染、OCR、表格提取或版面解析工具。适用于论文、报告、扫描文献和包含图表/公式的 PDF，不用于直接编辑 PDF 内容。
metadata:
  short-description: 学术 PDF 阅读与工具选择
---

# 学术 PDF 阅读 Skill

## 目标

当用户要求阅读、理解、检查或整理 PDF 文献时，建立一条可验证的本地工作流：先判断 PDF 是否包含原生文本，再按需使用页面渲染、图像抽取、OCR、结构化解析或表格提取。输出应服务于用户的具体任务；除非用户要求，不要把整篇 PDF 原文全部复制到回答中。

## 适用场景

- **原生文本论文**：可以直接搜索、复制文字，优先使用 `pdftotext`。
- **扫描件或图片型 PDF**：文本抽取为空或每页几乎没有文字时，使用 `ocrmypdf` 和 Tesseract。
- **复杂双栏排版、公式、代码、图表**：同时保留文本抽取和页面渲染结果；需要核对视觉布局时使用 `pdftoppm` 或 `mutool`。
- **表格密集型文献**：先判断表格是否是文本对象，再考虑 `camelot-py`；必要时用渲染图像人工核验。
- **损坏、加密或结构异常的 PDF**：先使用 `pdfinfo` 和 `qpdf --check`，不要直接假设抽取失败是内容缺失。
- **需要章节、表格和公式结构的长论文**：在基础抽取结果不足时使用 `marker-pdf` 等版面解析工具。

不应因为普通 PDF 阅读就自动安装或运行重量级工具。先做低成本检查，再按缺失能力补工具。

## 当前环境基线

在 Ubuntu 22.04 类环境中，优先检查以下命令是否存在：

```bash
command -v pdfinfo pdftotext pdftoppm pdfimages pdftohtml
```

本工作流假定 Poppler 可用。`poppler-utils` 通常提供：

- `pdfinfo`：页数、加密、尺寸、元数据和版本检查。
- `pdftotext`：抽取可搜索文本。
- `pdftoppm`：将指定页渲染为 PNG/JPEG，便于阅读图、公式和复杂版式。
- `pdfimages`：列出或导出 PDF 中的内嵌图像。
- `pdftohtml`：在需要保留部分版式时进行辅助转换。

如果这些命令缺失，可安装：

```bash
sudo apt update
sudo apt install -y poppler-utils
```

## 工具选择与作用

| 工具 | 主要作用 | 何时使用 | Ubuntu 安装方式 |
|---|---|---|---|
| `poppler-utils` | 基础 PDF 信息、文本、渲染和图像操作 | 所有 PDF 阅读任务的第一层 | `sudo apt update && sudo apt install -y poppler-utils` |
| `qpdf` | 检查 PDF 结构、处理部分损坏/加密、线性化和按页操作 | `pdfinfo` 异常、文件疑似损坏或需要结构诊断时 | `sudo apt update && sudo apt install -y qpdf` |
| `mupdf-tools` / `mutool` | 另一套轻量 PDF 渲染和对象处理工具 | Poppler 结果异常、复杂矢量内容或需要第二实现交叉核验时 | `sudo apt update && sudo apt install -y mupdf-tools` |
| `ocrmypdf` | 对扫描件进行纠偏、旋转和 OCR，并把文本层嵌回 PDF | 原生文本抽取为空或严重不完整时 | `sudo apt update && sudo apt install -y ocrmypdf` |
| `tesseract-ocr` | OCR 引擎 | 与 `ocrmypdf` 一起处理图片型 PDF | `sudo apt install -y tesseract-ocr` |
| `tesseract-ocr-eng` | 英文 OCR 语言包 | 英文论文、代码周边文字 | `sudo apt install -y tesseract-ocr-eng` |
| `tesseract-ocr-chi-sim` | 简体中文 OCR 语言包 | 中文或中英混排文献 | `sudo apt install -y tesseract-ocr-chi-sim` |
| `marker-pdf` | 将论文解析为较有结构的 Markdown/JSON，通常比裸文本更适合长文献分析 | 双栏结构、标题层级、表格或公式需要保留时 | `python3 -m pip install --user marker-pdf` |
| `camelot-py` | 从文本型 PDF 提取表格 | 表格是规则文本对象，而不是扫描图时 | 先 `sudo apt install -y ghostscript`，再 `python3 -m pip install --user 'camelot-py[cv]'` |
| `pandoc` | 在抽取后的文本/Markdown之间转换 | 需要整理笔记、导出 Markdown 或其他文档格式时 | `sudo apt update && sudo apt install -y pandoc` |

推荐的最小增强安装集是 `qpdf`、`ocrmypdf`、英文和简体中文 Tesseract 语言包以及 `mupdf-tools`。只有确实需要结构化长文献或表格抽取时，再安装 `marker-pdf` 和 `camelot-py`。不要为了原生文本 PDF 预先运行 OCR。

## 标准阅读流程

### 1. 检查文件和结构

```bash
pdfinfo paper.pdf
qpdf --check paper.pdf
```

记录页数、是否加密、页面尺寸、文件版本和元数据。若文件有密码或访问控制，先向用户说明，不要尝试绕过权限。

### 2. 优先抽取原生文本

```bash
pdftotext -layout -enc UTF-8 paper.pdf paper.txt
wc -l -w -c paper.txt
```

逐页检查是否有空页或异常短页。例如：

```bash
for page in $(seq 1 "$(pdfinfo paper.pdf | awk '/^Pages:/ {print $2}')"); do
  printf '%s ' "$page"
  pdftotext -f "$page" -l "$page" paper.pdf - | wc -w
done
```

如果所有页面都有合理文字、没有明显乱码或大量缺失，直接在文本结果上阅读，并仅对关键页面做视觉核验。

### 3. 对图、公式、代码和版式做视觉核验

```bash
mkdir -p rendered-pages
pdftoppm -f 1 -l 3 -png -r 180 paper.pdf rendered-pages/page
```

只渲染需要查看的页，避免无必要地生成大量图像。需要检查内嵌图像时：

```bash
pdfimages -list paper.pdf
```

若 Poppler 渲染或对象解释异常，可用 `mutool` 作为第二实现进行对照；不要求所有 PDF 都同时经过两套工具。

### 4. 仅在需要时运行 OCR

先用页面词数和文本质量判断是否真的需要 OCR。对扫描件使用：

```bash
ocrmypdf --skip-text --deskew --rotate-pages \
  -l eng+chi_sim paper.pdf paper-ocr.pdf
pdftotext -layout -enc UTF-8 paper-ocr.pdf paper-ocr.txt
```

`--skip-text` 可避免覆盖已有的原生文本层。OCR 后仍需抽查页码、公式、代码、专有名词和中英文混排；OCR 结果不能默认视为精确转录。

### 5. 需要更高层结构时再解析

当用户需要章节层级、公式、表格或长文献的结构化分析时，使用 `marker-pdf` 生成 Markdown/JSON，并以页面渲染结果核对关键表格和公式。对规则表格可尝试 `camelot-py`，但应明确区分“抽取成功”与“表格语义正确”。

## 质量检查

完成阅读前至少确认：

1. 页数与 `pdfinfo` 一致，没有意外跳页。
2. 没有整页为空或大量页面异常短的情况；若有，说明文本抽取可能不完整。
3. UTF-8 文本没有大量替换字符、乱码或字符顺序错乱。
4. 首页、末页以及包含关键图表/公式的页面可以被渲染并看清。
5. OCR 只用于确实没有文本层的页面；原生文本应优先于 OCR。
6. 表格、公式和代码块不能只依据裸文本判断，必要时回看渲染页面。

## 对用户的输出约定

- 先报告是否成功读取、页数和文本完整性，再回答用户要求的内容。
- 用户若只要求工具建议，说明当前已安装能力、缺失能力、工具用途和安装命令；不要未经授权安装软件。
- 用户若要求阅读内容，可按章节、问题或关键结论摘要，不必默认输出全文。
- 明确区分“PDF 中直接读到的内容”“由 OCR 得到的内容”和“基于图表/上下文的推断”。
- PDF 可能包含链接、脚本或恶意内容；阅读工作流只做解析和渲染，不执行 PDF 内嵌程序或未知附件。

## 推荐的一次性安装命令

在用户明确允许修改环境时，可按需执行以下命令；网络受限时应说明失败原因并保留已完成的本地检查：

```bash
sudo apt update && sudo apt install -y \
  poppler-utils qpdf mupdf-tools \
  ocrmypdf tesseract-ocr tesseract-ocr-eng tesseract-ocr-chi-sim \
  ghostscript pandoc

python3 -m pip install --user marker-pdf 'camelot-py[cv]'
```

安装后验证：

```bash
command -v pdfinfo pdftotext pdftoppm pdfimages pdftohtml
command -v qpdf mutool ocrmypdf tesseract pandoc
python3 -m pip show marker-pdf camelot-py
```

