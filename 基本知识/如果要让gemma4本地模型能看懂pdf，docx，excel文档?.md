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

## 以上处理还不够，问问题的第二次就出现错误

这就是之前那个 **400 `unsupported content[].type`** 错误——说明你**还没改 `run_agent`,或者 `_to_text` 那步没加上**。

原因回顾:第一次正常、第二次出错,是因为第二轮时第一轮的**文件消息进了 history**,它的 content 是复杂结构(list/dict),被原样传给 llama-server,而 llama.cpp 不认。

**确认一下:你前面加过 `_to_text` 函数、改过 `run_agent` 吗?**

如果没有,现在补上。打开 `Gemma4`:

**第一步:在 `run_agent` 上方加 `_to_text` 函数**

```python
def _to_text(content) -> str:
    """把历史里可能出现的复杂 content(str/dict/list)统一压成纯文本。"""
    if content is None:
        return ""
    if isinstance(content, str):
        return content
    if isinstance(content, (list, tuple)):
        return "\n".join(_to_text(item) for item in content if _to_text(item))
    if isinstance(content, dict):
        if "text" in content:
            return str(content["text"])
        if "content" in content:
            return _to_text(content["content"])
        if "path" in content or "file" in content or content.get("type") == "file":
            return "（此前上传过一个文件）"
        return ""
    return str(content)
```

**第二步:替换 `run_agent` 里解析 history 的部分**

找到 `run_agent` 里 `for turn in history` 那段循环,把处理 content 的地方都包上 `_to_text()`:

```python
def run_agent(message: str, history: list):
    messages = []
    for turn in history or []:
        if isinstance(turn, dict):
            role = turn.get("role")
            content = _to_text(turn.get("content", ""))      # ← 关键:包 _to_text
            if role in ("user", "assistant") and content:
                messages.append((role, content))
        elif isinstance(turn, (list, tuple)) and len(turn) == 2:
            user_msg = _to_text(turn[0])                      # ← 包 _to_text
            bot_msg = _to_text(turn[1])                       # ← 包 _to_text
            if user_msg:
                messages.append(("user", user_msg))
            if bot_msg:
                messages.append(("assistant", bot_msg))
    messages.append(("user", message))

    result = agent.invoke({"messages": messages})
    used_search = any(getattr(m, "type", None) == "tool" for m in result["messages"])
    answer = result["messages"][-1].content
    return answer, used_search
```

**第三步:重启**

```bash
ps aux | grep Gemma4 | grep -v grep
kill <PID>
python3 Gemma4
```

---

**验证:** 传文件问一次 → 再问一次 → 不再报 400 就对了。

如果改了还报错,把你**当前 `run_agent` 函数的完整内容**贴给我(`sed -n '95,130p' Gemma4` 大概能截到),我直接帮你对着改。
