这需要给你的应用加一个"**文档转文字**"的处理层——因为 Gemma 4 E2B 是纯文本模型,它没法直接读 PDF/docx/excel 的二进制格式,得先用 Python 库把里面的**文字提取出来**,再喂给模型。

思路和你现在的文本文件上传一样,只是多了"提取"这一步。

**需要装 3 个库:**

```bash
pip install pypdf python-docx openpyxl
```

| 库 | 处理 |
|----|------|
| `pypdf` | PDF → 文字 |
| `python-docx` | Word (.docx) → 文字 |
| `openpyxl` | Excel (.xlsx) → 文字 |

---

**改动:扩展你的 `_read_text_file` 逻辑**

在 `Gemma4` 里,顶部 import 区加上:

```python
import os
```
(已经有了就跳过)

然后新增几个提取函数,放在 `_read_text_file` **上方**:

```python
def _extract_pdf(fpath: str) -> str:
    from pypdf import PdfReader
    reader = PdfReader(fpath)
    texts = []
    for i, page in enumerate(reader.pages, 1):
        t = page.extract_text() or ""
        if t.strip():
            texts.append(f"[第{i}页]\n{t}")
    return "\n\n".join(texts) if texts else "（PDF 未提取到文字，可能是扫描件/图片型 PDF）"


def _extract_docx(fpath: str) -> str:
    from docx import Document
    doc = Document(fpath)
    parts = [p.text for p in doc.paragraphs if p.text.strip()]
    # 顺便提取表格
    for table in doc.tables:
        for row in table.rows:
            cells = [c.text.strip() for c in row.cells]
            if any(cells):
                parts.append(" | ".join(cells))
    return "\n".join(parts) if parts else "（Word 文档为空或无文字内容）"


def _extract_xlsx(fpath: str) -> str:
    from openpyxl import load_workbook
    wb = load_workbook(fpath, read_only=True, data_only=True)
    parts = []
    for ws in wb.worksheets:
        parts.append(f"【工作表：{ws.title}】")
        for row in ws.iter_rows(values_only=True):
            cells = [str(c) if c is not None else "" for c in row]
            if any(cells):
                parts.append(" | ".join(cells))
    wb.close()
    return "\n".join(parts) if parts else "（Excel 为空）"
```

---

**替换 `_read_text_file` 函数**,让它按扩展名分派到对应提取器:

```python
TEXT_EXTS = {".txt", ".md", ".csv", ".tsv", ".fasta", ".fa", ".fna", ".json"}
MAX_CHARS_PER_FILE = 50000

def _read_text_file(fpath: str) -> str:
    name = os.path.basename(fpath)
    ext = os.path.splitext(fpath)[1].lower()

    try:
        if ext in TEXT_EXTS:
            with open(fpath, "r", encoding="utf-8", errors="replace") as f:
                content = f.read(MAX_CHARS_PER_FILE + 1)
        elif ext == ".pdf":
            content = _extract_pdf(fpath)
        elif ext == ".docx":
            content = _extract_docx(fpath)
        elif ext in (".xlsx", ".xlsm"):
            content = _extract_xlsx(fpath)
        else:
            return (f"\n\n【文件 {name}】暂不支持该类型"
                    f"（支持：文本类 + pdf/docx/xlsx）")
    except Exception as e:
        return f"\n\n【文件 {name}】读取失败：{e}"

    truncated = len(content) > MAX_CHARS_PER_FILE
    if truncated:
        content = content[:MAX_CHARS_PER_FILE]
    note = f"（已截断，仅取前 {MAX_CHARS_PER_FILE} 字符）" if truncated else ""
    return f"\n\n【上传文件 {name} 内容{note}】\n```\n{content}\n```"
```

---

**重启:**

```bash
ps aux | grep Gemma4 | grep -v grep
kill <PID>
python3 Gemma4
```

---

**几个重要提醒:**

1. **扫描件/图片型 PDF 提取不到文字** — `pypdf` 只能提取"文字层"。如果 PDF 是扫描的图片,会提不到内容,那需要 OCR(`pytesseract`+`pdf2image`),要另装,复杂一些。你需要的话再说。

2. **E2B 小模型 + 50000 字符上限** — 大文档会被截断。一份长 PDF 可能只读到前几页。这是小模型的硬限制,不是 bug。如果要处理长文档,得做**分块**或换更大的模型。

3. **Excel 复杂格式(合并单元格、公式、图表)** — `openpyxl` 只提取单元格的值,格式和图表会丢失。数据表类的 xlsx 没问题,复杂报表会失真。

4. **docx 里的图片** — 提取不了,只拿文字和表格。

先把这版跑起来,传个普通 PDF/Word/Excel 测测。遇到扫描件需要 OCR,或长文档需要分块,再告诉我。

## 处理前的Gemma4文件
```


```
## 处理后的Gemma4文件
```

```
