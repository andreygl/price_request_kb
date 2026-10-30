---
name: sabydoc-convert
description: Конвертация документов SabyDoc в/из Markdown. Использовать при переносе .sabydoc в .md и обратно, а также при правке .sabydoc вне редактора БЗ Saby. Содержит инструменты sabydoc2md, Sabyverter и самодостаточный md_to_sabydoc.py.
---

# Конвертеры SabyDoc

Инструменты для конвертации документов SabyDoc (`.sabydoc`) в `md`/`doc` и обратно. Полезны при переносе технической документации между БЗ SabyDoc и заметками, а также при правке `.sabydoc` вне редактора.

## Инструменты

### sabydoc2md (GitLab)

Внутренняя утилита для конвертации `.sabydoc` в Markdown. Репозиторий: https://git.sbis.ru/ar.buranov/sabydoc2md. Доступна только авторизованным сотрудникам (LDAP).

### Sabyverter (веб)

https://tomtop.ru/sabydoc/ - конвертирует документы между `sabydoc`, `md`, `txt`, `docx`, `html`, `odt`, `pdf` (любой в любой). До 3 файлов, до 50 Мб каждый.

- Экспорт в SabyDoc нестабилен: файл может не открыться в SBIS viewer, потерять виджеты или требовать ручной правки. Надёжнее `md` или `docx`.
- LLM-фичи: Vision, Alt-text, Table → prose, Cleanup, QA. AI только для опциональных фич; конвертация алгоритмическая.
- MCP-инструмент `convert_document`: вход `sabydoc`/`docx`/`odt`/`html`/`md`/`txt`, выход `md`/`txt`.

### md_to_sabydoc.py (Markdown → .sabydoc)

Самодостаточный генератор `.sabydoc` из Markdown: собирает ZIP с `content.json`, `index.site`, `resources/` по структуре реальных документов БЗ Saby. Код ниже, внешний файл не нужен.

**Требования:** Python 3.8+, Pillow (`pip install pillow`).

**Запуск:**

```powershell
python md_to_sabydoc.py "doc.md"                  # рядом с .md создаст <имя>.sabydoc, картинки из <имя>.assets/
python md_to_sabydoc.py "doc.md" -o "out.sabydoc" --assets "doc.assets"
```

**Входной Markdown:**

```markdown
---
title: "Название документа"
last_edit: "Tue Oct 27 2026 17:00:00 GMT+0300 (Москва, стандартное время)"
---

# Заголовок документа (эта строка `# ...` в тексте игнорируется) - title берётся из frontmatter или имени файла и ставится в начало как documentHeader

## Раздел 1

Абзац с [ссылкой](https://example.com) и **жирным**.

## Схема

![](doc.assets/r1.png)

Подпись под картинкой - абзац сразу после изображения.

### Подраздел

| **Колонка 1** | **Колонка 2** |
| --- | --- |
| Значение | Значение |
```

**Поддерживается:** `##` -> Header1, `###` -> Header2 (жирный), абзацы, `**жирный**`, `[текст](url)`, `` `код` `` (маркеры снимаются, текст идёт как обычный), `![](...)` -> image:View, таблицы (первая строка - шапка), маркированные списки `-`/`*` -> list:View с маркером `bulletWhiteBulletSquare`, подпись - абзац сразу после картинки. Заголовок документа (`title` из frontmatter или имя файла) ставится в начало блоком `DocsEditor/documentHeader:View` и в оглавление не попадает. Оглавление строится автоматически по `##`/`###`.

**Ограничения:**
- только `##` и `###`; `####` и глубже попадут в абзац;
- code fence, цитаты, callout, горизонтальные линии не конвертируются в нативные блоки; строки code fence идут обычными абзацами;
- вложенные списки и списки с продолжением абзаца не поддерживаются: пункт - одна строка;
- инлайн: `**жирный**`, `[текст](url)` и обратные кавычки (снимаются, моноширинного шрифта нет);
- таблицы - с ведущим и замыкающим `|`, без colspan/rowspan;
- картинки берутся по basename из assets-папки; одноимённые схлопываются;
- картинка схемы (`r1.png`) - ручной экспорт из draw.io.

**Проверка:** обратная конвертация - `sabydoc_to_md.py` из Saby Skills (`saby-docs-import/sabydoc_to_md.py`) либо загрузка `.sabydoc` в SabyDoc.

## md_to_sabydoc.py (полный код)

```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
"""
md_to_sabydoc.py - генератор .sabydoc из Markdown + папки с картинками.

Собирает документ SabyDoc (.sabydoc - ZIP с content.json, index.site, resources/)
по шаблону, повторяющему структуру реальных документов базы знаний Saby.

Поддерживаемые блоки Markdown:
  - # заголовок документа -> DocsEditor/documentHeader:View (из frontmatter title или имени файла; в TOC не попадает)
  - ## ...        -> Header1
  - ### ...       -> Header2 (жирный)
  - абзацы        -> paragraph:View (ссылки [текст](url) и **жирный** - через formats)
  - ![](...r1.png) -> image:View (картинка берётся из <док>.assets/)
  - таблицы       -> table:View (первая строка - шапка, жирным)
  - - пункт / * пункт -> list:View с маркером bulletWhiteBulletSquare
  - code fence    -> абзацы (нативного codeBlock нет)
  - подпись под картинкой -> paragraph2:View (абзац сразу после ![](...))

Использование:
  python md_to_sabydoc.py "Описание информационной модели объекта Торг.md"
  python md_to_sabydoc.py doc.md -o out.sabydoc --assets "doc.assets"

Требования: Python 3.8+, Pillow (для определения размеров картинок).
"""

import argparse
import json
import os
import re
import uuid
import zipfile
import urllib.parse

from PIL import Image

# --- константы формата SabyDoc (повторяют эталон) ---------------------------

ELEMENT_META_TYPES = {
    "ModuleEditor/image:View": {"attributes": {"value": {"type": "resource"}}},
    "ModuleEditor/inlineLink:View": {"attributes": {"cover": {"type": "resource"}}},
    "ModuleEditor/table:View": {"relatedObjects": ["object"]},
    "ModuleEditor/textLayout:View": {
        "attachedAttributes": {"indent": {"type": "attachedAttribute"}},
        "attributes": {
            "background.image.resource": {"type": "resource"},
            "background.layers.image.original": {"type": "resource"},
        },
        "relatedObjects": ["object"],
    },
}

DATA_CONTEXT = '{"f":0,"d":[],"s":[],"_type":"record"}'
CONSTRUCTOR_ID = "SabyDocConstructor"
FENCE = "`" * 3


def uid():
    return str(uuid.uuid4())


# --- инлайновое форматирование: **жирный** и [текст](url) --------------------

def parse_inline(text):
    """Markdown-строка -> (plain_text, formats) с SabyDoc-атрибутами."""
    out = ""
    formats = []
    i = 0
    while i < len(text):
        if text.startswith("**", i):
            end = text.find("**", i + 2)
            if end != -1:
                inner = text[i + 2:end]
                start = len(out)
                out += inner
                formats.append([start, len(inner),
                                {"font": {"style": "normal", "weight": "bold"}}])
                i = end + 2
                continue
        if text[i] == "[":
            close = text.find("](", i)
            if close != -1:
                ep = text.find(")", close + 2)
                if ep != -1:
                    label = text[i + 1:close]
                    url = text[close + 2:ep]
                    start = len(out)
                    out += label
                    formats.append([start, len(label),
                                    {"font": {"style": "normal"}, "href": url,
                                     "visibleLinkHeading": "visible"}])
                    i = ep + 1
                    continue
        if text[i] == "`":
            end = text.find("`", i + 1)
            if end != -1:
                out += text[i + 1:end]
                i = end + 1
                continue
        out += text[i]
        i += 1
    out = re.sub(r"\\([\\`*_\[\]<>|])", r"\1", out)
    return out, formats


def strip_marks(text):
    return re.sub(r"\*+", "", text).strip()


# --- построители JSONML-узлов ----------------------------------------------

def ft(text, formats):
    return ["ModuleEditor/formattedText:View", {"formats": formats, "id": uid()}, text]


def header1(text, toc_id):
    return ["ModuleEditor/baseWidgets:Header1",
            {"id": toc_id, "size": 1, "textStyle": "default"},
            ft(text, [[0, len(text), {}]])]


def header2(text, toc_id):
    return ["ModuleEditor/baseWidgets:Header2",
            {"align": "left", "id": toc_id, "size": "2", "textStyle": "default"},
            ft(text, [[0, len(text), {"font": {"style": "normal", "weight": "bold"}}]])]


def paragraph(text):
    t, f = parse_inline(text)
    return ["ModuleEditor/paragraph:View", {"id": uid(), "textStyle": "default"}, ft(t, f)]


def caption(text):
    t, f = parse_inline(text)
    return ["ModuleEditor/paragraph2:View", {"id": uid(), "textStyle": "text2"}, ft(t, f)]


def bullet_list(items):
    item_nodes = []
    for item in items:
        t, f = parse_inline(item)
        item_nodes.append([
            "ModuleEditor/list:Item", {"id": uid()},
            ["ModuleEditor/list:ListItemContent", {"id": uid(), "textStyle": "default"},
             ft(t, f)],
        ])
    return ["ModuleEditor/list:View",
            {"id": uid(), "markerConfigId": "bulletWhiteBulletSquare"},
            *item_nodes]


def image_node(resource_key):
    return ["ModuleEditor/image:View", {
        ".style": {"aspectRatio": "auto", "width": "1100px"},
        "action": {"id": "openFullScreenRuntime"},
        "aspectRatioStyle": "auto",
        "id": uid(), "isAutoHeight": False, "saveProportions": None,
        "value": resource_key,
    }]


def table_node(rows):
    """rows: список строк; строка - список (text, is_header)."""
    ncols = max(len(r) for r in rows) or 1
    width = "%.15f%%" % (100.0 / ncols)
    row_nodes = []
    for cells in rows:
        cell_nodes = []
        for text, is_header in cells:
            if is_header:
                t = strip_marks(text)
                f = [[0, len(t), {"color": "color1",
                                  "font": {"style": "normal", "weight": "bold"},
                                  "font-size": "16px"}]]
                para = ["ModuleEditor/paragraph:View",
                        {"align": "center", "id": uid(), "textStyle": "default"}, ft(t, f)]
            else:
                t, f = parse_inline(text)
                para = ["ModuleEditor/paragraph:View",
                        {"id": uid(), "textStyle": "default"}, ft(t, f)]
            cell_nodes.append(["ModuleEditor/table:Cell",
                               {"colspan": 1, "id": uid(), "rowspan": 1}, para])
        row_nodes.append(["ModuleEditor/table:Row", {"id": uid()}] + cell_nodes)
    return ["ModuleEditor/table:View",
            {"columnsConfigs": [{"width": width} for _ in range(ncols)],
             "id": uid(), "stickyHeader": False}] + row_nodes


# --- разбор Markdown --------------------------------------------------------

def split_frontmatter(lines):
    """Отделяет YAML-frontmatter; возвращает (front_dict, body_lines)."""
    if not lines or lines[0].strip() != "---":
        return {}, lines
    front = {}
    for i in range(1, len(lines)):
        if lines[i].strip() == "---":
            return front, lines[i + 1:]
        if ":" in lines[i]:
            k, v = lines[i].split(":", 1)
            front[k.strip()] = v.strip().strip('"')
    return front, lines


def parse_blocks(body):
    blocks = []
    i = 0
    while i < len(body):
        ln = body[i]
        if not ln.strip():
            i += 1
            continue
        if ln.startswith("# "):
            i += 1
            continue  # заголовок документа задаётся именем файла
        if ln.startswith("## "):
            blocks.append(("h1", strip_marks(ln[3:]))); i += 1
        elif ln.startswith("### "):
            blocks.append(("h2", strip_marks(ln[4:]))); i += 1
        elif ln.startswith("!["):
            m = re.search(r"!\[[^\]]*\]\(([^)]+)\)", ln)
            blocks.append(("image", m.group(1) if m else "")); i += 1
        elif ln.startswith(FENCE):
            i += 1
            code = []
            while i < len(body) and not body[i].startswith(FENCE):
                code.append(body[i]); i += 1
            i += 1
            blocks.append(("code", "\n".join(code)))
        elif ln.startswith("|"):
            tbl = []
            while i < len(body) and body[i].startswith("|"):
                tbl.append(body[i]); i += 1
            rows = []
            for tln in tbl:
                if re.match(r"^\|(\s*:?-{3,}:?\s*\|)+$", tln.strip()):
                    continue
                rows.append([c.strip() for c in tln.strip().strip("|").split("|")])
            blocks.append(("table", rows))
        elif re.match(r"^[-*] ", ln):
            items = []
            while i < len(body) and re.match(r"^[-*] ", body[i]):
                items.append(body[i][2:].strip()); i += 1
            blocks.append(("list", items))
        else:
            para = []
            while (i < len(body) and body[i].strip()
                   and not body[i].startswith(("#", "|", "![", FENCE))
                   and not re.match(r"^[-*] ", body[i])):
                para.append(body[i]); i += 1
            blocks.append(("para", " ".join(para)))
    return blocks


# --- сборка .sabydoc ---------------------------------------------------------

def resolve_images(image_paths, assets_dir):
    """image_paths (url из ![](...)) -> {key: {resdict, src_abs}}."""
    resources = {}
    order = []
    for path in image_paths:
        decoded = urllib.parse.unquote(path)
        fname = os.path.basename(decoded)
        key = os.path.splitext(fname)[0] or fname
        if key in resources:
            continue
        src = os.path.join(assets_dir, fname)
        if not os.path.isfile(src):
            raise SystemExit("Не найден файл картинки: %s" % src)
        with Image.open(src) as im:
            w, h = im.size
        resources[key] = {
            "fileName": fname,
            "file_id": uid(),
            "naturalHeight": h,
            "naturalWidth": w,
            "type": "internal",
            "value": "./resources/" + fname,
        }
        order.append((key, fname, src))
    return resources, order


def build_sabydoc(md_path, out_path, assets_dir):
    md = open(md_path, encoding="utf-8").read()
    front, body = split_frontmatter(md.split("\n"))
    blocks = parse_blocks(body)

    toc = []
    content_blocks = []
    prev_was_image = False
    for kind, data in blocks:
        if kind == "h1":
            tid = uid()
            toc.append({"hasChild": False, "id": "toc_" + tid, "isHeader": True,
                        "isParent": False, "level": 1, "parentId": None, "text": data})
            content_blocks.append(header1(data, tid))
            prev_was_image = False
        elif kind == "h2":
            tid = uid()
            parent_toc_id = None
            for t in reversed(toc):
                if t["level"] == 1:
                    parent_toc_id = t["id"]; break
            if parent_toc_id:
                for t in toc:
                    if t["id"] == parent_toc_id:
                        t["hasChild"] = True
            toc.append({"hasChild": False, "id": "toc_" + tid, "isHeader": True,
                        "isParent": False, "level": 2, "parentId": parent_toc_id,
                        "text": data})
            content_blocks.append(header2(data, tid))
            prev_was_image = False
        elif kind == "image":
            content_blocks.append(image_node(
                os.path.splitext(os.path.basename(urllib.parse.unquote(data)))[0]))
            prev_was_image = True
        elif kind == "table":
            marked = [[(c, ri == 0) for c in row] for ri, row in enumerate(data)]
            content_blocks.append(table_node(marked))
            prev_was_image = False
        elif kind == "list":
            content_blocks.append(bullet_list(data))
            prev_was_image = False
        elif kind == "code":
            for line in data.split("\n"):
                content_blocks.append(paragraph(line))
            prev_was_image = False
        elif kind == "para":
            content_blocks.append(caption(data) if prev_was_image else paragraph(data))
            prev_was_image = False

    image_paths = [d for k, d in blocks if k == "image"]
    resources, order = resolve_images(image_paths, assets_dir)

    site_id = uid()
    origin_id = uid()

    title = front.get("title") or os.path.splitext(os.path.basename(md_path))[0]
    last_edit = front.get("last_edit", "")

    content = [
        "frame",
        {"elementMetaTypes": ELEMENT_META_TYPES, "resources": resources},
        ["DocsEditor/documentLayout:View", {"id": uid()},
         ["DocsEditor/documentHeader:View", {"id": uid()}, title],
         ["ModuleEditor/textLayout:View", {"id": uid()}, *content_blocks]],
    ]

    index_site = {
        "propertyValues": {
            "imagesLinks": [],
            "lastEditDate": last_edit,
            "lastRestoredDate": last_edit,
            "tableOfContents": toc,
            "version": 2,
        },
        "pages": [{
            "id": site_id, "originId": None, "name": None, "content": content,
            "order": 0, "visibility": None, "constructorId": CONSTRUCTOR_ID,
            "SEO": [{"content": "website", "name": "og:type"}],
            "anonymous": True, "url": None,
        }],
        "themeId": None, "fontThemeId": None, "themeData": None,
        "fontThemeData": None, "conditions": None,
    }

    content_json = {
        "view": {"siteId": site_id, "revisionId": None, "originId": origin_id,
                 "constructorId": CONSTRUCTOR_ID, "resourcesFolderId": None},
        "contextTypeProperties": None,
        "dataContext": DATA_CONTEXT,
        "version": 1,
    }

    with zipfile.ZipFile(out_path, "w", zipfile.ZIP_DEFLATED) as z:
        z.writestr("content.json", json.dumps(content_json, ensure_ascii=False))
        z.writestr("index.site", json.dumps(index_site, ensure_ascii=False))
        for key, fname, src in order:
            z.write(src, "resources/" + fname)

    return title, len(blocks), len(toc), len(resources)


def main(argv=None):
    parser = argparse.ArgumentParser(
        description="Генератор .sabydoc из Markdown + папки с картинками.")
    parser.add_argument("input", help="путь к .md-файлу")
    parser.add_argument("-o", "--output", help="путь к .sabydoc (по умолчанию рядом с .md)")
    parser.add_argument("--assets", help="папка с картинками (по умолчанию <input>.assets)")
    args = parser.parse_args(argv)

    md_path = os.path.abspath(args.input)
    if not os.path.isfile(md_path):
        raise SystemExit("Не найден .md-файл: %s" % md_path)

    assets_dir = os.path.abspath(args.assets) if args.assets \
        else os.path.splitext(md_path)[0] + ".assets"

    out_path = os.path.abspath(args.output) if args.output \
        else os.path.splitext(md_path)[0] + ".sabydoc"

    title, blocks, toc, images = build_sabydoc(md_path, out_path, assets_dir)
    print("OK: %s" % out_path)
    print("  title: %s" % title)
    print("  блоков: %d, разделов (TOC): %d, картинок: %d" % (blocks, toc, images))


if __name__ == "__main__":
    main()
```

## Источник

- `c:\Saby\sabydoc\md_to_sabydoc.py` (копия от 27.10.2026).
- Заметка WorkTensor: `Разработка/Утилиты/Конвертеры SabyDoc.md`.
