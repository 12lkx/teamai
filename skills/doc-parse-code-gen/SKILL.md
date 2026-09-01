---
name: doc-parse-code-gen
description: |
  文档解析代码生成助手。当用户需要从 Word(.docx)/PDF/Excel(.xlsx) 文档中提取内容，并根据指定片段生成 Java/Kotlin 代码时使用。
  触发场景包括：
  1. 用户要求"根据文档生成代码"、"把这段文档转成代码"、"按文档实现"
  2. 用户要求"读取/解析 PDF/Word/Excel"并基于其中内容写 Java/Kotlin
  3. 用户给出文档路径和页码/段落/关键词/Sheet/行号，要求提取后生成代码
  4. 将文档中的接口定义、数据结构、状态机、业务流程、Excel 配置表转为代码实现
  若用户未指定语言，默认生成 Java 或 Kotlin（根据当前项目上下文判断）。
---

# 文档解析代码生成助手

## 工作流程

1. **确认需求**：明确用户要解析的文档路径、目标片段（页码/章节/关键词/段落），以及生成的代码类型（Java/Kotlin）。
2. **环境准备**：若未安装依赖，先执行 `pip install python-docx pdfplumber openpyxl`（Windows 用 `python -m pip install ...`）。
3. **提取文本**：根据文档类型选择下方方法读取全文或指定片段。
4. **分析与映射**：将文档中的业务描述映射为代码结构（类、接口、方法、枚举、数据类）。
5. **生成代码**：遵循当前项目的 `AGENTS.md` 与代码风格，直接输出到指定文件。
6. **验证**：若生成了可运行脚本，执行一次确保无语法错误。

## 读取 Word (.docx)

```python
from docx import Document

def read_docx(path: str) -> str:
    doc = Document(path)
    paragraphs = [p.text for p in doc.paragraphs if p.text.strip()]
    return "\n".join(paragraphs)

def read_docx_tables(path: str) -> list:
    doc = Document(path)
    tables = []
    for table in doc.tables:
        rows = []
        for row in table.rows:
            rows.append([cell.text for cell in row.cells])
        tables.append(rows)
    return tables
```

- `doc.paragraphs`：按段落读取，保留文档顺序。
- `doc.tables`：提取表格，常用于接口字段、配置项、枚举映射。

## 读取 PDF

```python
import pdfplumber

def read_pdf(path: str, pages: list[int] = None) -> str:
    text = []
    with pdfplumber.open(path) as pdf:
        target = pages if pages else range(len(pdf.pages))
        for i in target:
            page = pdf.pages[i]
            t = page.extract_text()
            if t:
                text.append(f"--- Page {i+1} ---\n{t}")
    return "\n".join(text)
```

- `pdfplumber` 对中文排版支持较好，优先使用。
- 若用户指定了页码，仅提取目标页；否则提取全部。

## 读取 Excel (.xlsx)

```python
from openpyxl import load_workbook

def read_excel(path: str, sheet_name: str = None, min_row: int = 1, max_row: int = None) -> list[dict]:
    wb = load_workbook(path, data_only=True)
    ws = wb[sheet_name] if sheet_name else wb.active
    rows = list(ws.iter_rows(min_row=min_row, max_row=max_row, values_only=True))
    if not rows:
        return []
    headers = rows[0]
    data = []
    for row in rows[1:]:
        data.append({h: v for h, v in zip(headers, row) if h is not None})
    return data

def read_excel_sheets(path: str) -> list[str]:
    wb = load_workbook(path, read_only=True)
    return wb.sheetnames
```

- `data_only=True`：读取公式计算后的值，而非公式文本。
- `sheet_name`：未指定时默认读取第一个工作表；不确定时可先用 `read_excel_sheets` 列出所有表名供用户选择。
- 返回 `list[dict]` 便于直接映射为 Java/Kotlin 的字段列表或枚举条目。

## 定位目标片段

常见策略（按用户描述选择）：

| 用户输入 | 策略 |
|---|---|
| "第3页到第5页" | 提取 `pages=[2,3,4]`（0-based） |
| "关于XXX的段落" | 全文提取后，用 `if keyword in paragraph` 过滤 |
| "表格里的字段定义" | 提取所有 `doc.tables` 或 `page.extract_tables()`，匹配表头关键词 |
| "Excel 的 SheetX" | 指定 `sheet_name`，未指定时默认第一个工作表 |
| "Excel 第3行到第10行" | 指定 `min_row=3, max_row=10`，结合表头解析为字典列表 |
| "Excel A列到D列" | 用 `iter_cols(min_col=1, max_col=4)` 提取，再按行重组 |
| "第X章" | 搜索章节标题（如 `"第X章"` / `"Chapter X"`），截取到下一同层级标题 |

若用户描述模糊，先提取目录/大纲，列出可选项请用户确认。

## 生成 Java/Kotlin 代码规范

- **语言判断**：如果当前工作目录存在 `CLAUDE.md` 且指定了语言，优先遵循；否则根据项目文件（`.kt`/`.java` 比例）判断。
- **类结构**：
  - Java：按文档中的名词提取为类/接口，动词提取为方法，状态提取为枚举。
  - Kotlin：优先使用 `data class`、`sealed class`、`enum class`，配合可空类型与默认参数。
- **命名**：文档中的中文术语转英文驼峰命名；若文档已提供英文，直接使用。
- **注释**：保留文档原文作为 KDoc/JavaDoc，尤其是业务规则、阈值、异常分支。
- **常量提取**：文档中出现的魔法数字、固定字符串、配置值，提取为 `const val` 或 `static final`。
- **空安全**：Kotlin 侧优先使用非空类型，必要处用 `?.let`；Java 侧添加 `@NonNull`/`@Nullable` 或 Objects.requireNonNull。
- **输出方式**：使用 `WriteFile` 或 `StrReplaceFile` 直接写入文件，不要仅在回复中展示代码。

## 边界情况处理

- **扫描版 PDF**（图片）：`pdfplumber.extract_text()` 可能返回空。告知用户该 PDF 为扫描件，建议使用 OCR（如 `pytesseract` + `pdf2image`），或请用户提供可编辑版本。
- **超大文档**（>100页）：不要一次性全部读入上下文。先提取目录/每页首行，定位目标页后再精细读取。
- **代码片段与文档混合**：若文档本身已包含代码示例，优先参考其结构，但修正其中明显的过时写法或空安全问题。

## 示例调用模式

```
用户：把需求文档第10页的接口定义转成Kotlin数据类
助手：
1. pip install python-docx pdfplumber openpyxl
2. 读取文档第10页（index=9）
3. 解析接口字段 -> 生成 data class
4. WriteFile 到指定路径

用户：把 Excel 的"错误码配置表"转成 Java 枚举
助手：
1. pip install openpyxl
2. `read_excel_sheets` 列出表名，定位"错误码配置表"
3. `read_excel` 提取表头行（code/name/desc）
4. 每行映射为枚举常量，生成 Java enum
5. WriteFile 到指定路径
```
