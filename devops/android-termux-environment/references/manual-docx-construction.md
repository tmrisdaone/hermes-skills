# Manual DOCX Construction on Termux

When python-docx is unavailable (lxml build fails on ARM64), construct .docx files
using Python's built-in `zipfile` + `xml.sax.saxutils`.

## Minimum File Structure

```
my_document.docx
├── [Content_Types].xml
├── _rels/
│   └── .rels
└── word/
    ├── document.xml          # ← main content
    ├── _rels/
    │   └── document.xml.rels
    ├── styles.xml
    ├── fontTable.xml
    └── settings.xml
```

## Boilerplate Templates

### [Content_Types].xml
```xml
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<Types xmlns="http://schemas.openxmlformats.org/package/2006/content-types">
  <Default Extension="rels" ContentType="application/vnd.openxmlformats-package.relationships+xml"/>
  <Default Extension="xml" ContentType="application/xml"/>
  <Override PartName="/word/document.xml"
    ContentType="application/vnd.openxmlformats-officedocument.wordprocessingml.document.main+xml"/>
  <Override PartName="/word/styles.xml"
    ContentType="application/vnd.openxmlformats-officedocument.wordprocessingml.styles+xml"/>
  <Override PartName="/word/fontTable.xml"
    ContentType="application/vnd.openxmlformats-officedocument.wordprocessingml.fontTable+xml"/>
  <Override PartName="/word/settings.xml"
    ContentType="application/vnd.openxmlformats-officedocument.wordprocessingml.settings+xml"/>
</Types>
```

### _rels/.rels
```xml
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<Relationships xmlns="http://schemas.openxmlformats.org/package/2006/relationships">
  <Relationship Id="rId1"
    Type="http://schemas.openxmlformats.org/officeDocument/2006/relationships/officeDocument"
    Target="word/document.xml"/>
</Relationships>
```

### word/_rels/document.xml.rels
```xml
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<Relationships xmlns="http://schemas.openxmlformats.org/package/2006/relationships">
  <Relationship Id="rId1"
    Type="http://schemas.openxmlformats.org/officeDocument/2006/relationships/styles"
    Target="styles.xml"/>
  <Relationship Id="rId2"
    Type="http://schemas.openxmlformats.org/officeDocument/2006/relationships/fontTable"
    Target="fontTable.xml"/>
  <Relationship Id="rId3"
    Type="http://schemas.openxmlformats.org/officeDocument/2006/relationships/settings"
    Target="settings.xml"/>
</Relationships>
```

### word/styles.xml
```xml
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<w:styles xmlns:w="http://schemas.openxmlformats.org/wordprocessingml/2006/main">
  <w:style w:type="paragraph" w:styleId="Normal" w:default="1">
    <w:name w:val="Normal"/>
    <w:pPr>
      <w:spacing w:after="120" w:line="240" w:lineRule="auto"/>
    </w:pPr>
    <w:rPr>
      <w:rFonts w:ascii="Times New Roman" w:hAnsi="Times New Roman"/>
      <w:sz w:val="22"/>
      <w:szCs w:val="22"/>
    </w:rPr>
  </w:style>
</w:styles>
```

### word/fontTable.xml
```xml
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<w:fonts xmlns:w="http://schemas.openxmlformats.org/wordprocessingml/2006/main">
  <w:font w:name="Times New Roman">
    <w:panose1 w:val="02020603050405020304"/>
    <w:charset w:val="00"/>
    <w:family w:val="roman"/>
    <w:pitch w:val="variable"/>
  </w:font>
</w:fonts>
```

### word/settings.xml
```xml
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<w:settings xmlns:w="http://schemas.openxmlformats.org/wordprocessingml/2006/main">
  <w:defaultTabStop w:val="720"/>
</w:settings>
```

## Content XML Reference (word/document.xml)

**Namespace:** `w:http://schemas.openxmlformats.org/wordprocessingml/2006/main`

| Element | Meaning | Key Attributes |
|---------|---------|----------------|
| `<w:p>` | Paragraph | — |
| `<w:pPr>` | Paragraph properties | — |
| `<w:jc w:val="x"/>` | Justification | `val` = `left`, `center`, `right`, `both` (justify) |
| `<w:spacing w:after="N" w:line="240"/>` | Paragraph spacing | `after` = twips after para, `line` = 240 = single spacing |
| `<w:ind w:firstLine="360"/>` | First-line indent | 360 twips ≈ 0.25 inch |
| `<w:r>` | Run (formatted span) | — |
| `<w:rPr>` | Run properties | — |
| `<w:b/>` | Bold | Include tag for bold, omit otherwise |
| `<w:i/>` | Italic | Include tag for italic |
| `<w:u w:val="single"/>` | Underline | `val` = `single`, `double`, `none` |
| `<w:rFonts w:ascii="X" w:hAnsi="X"/>` | Font name | Set both ascii + hAnsi |
| `<w:sz w:val="N"/>` | Font size | In half-points (22 = 11pt, 24 = 12pt) |
| `<w:szCs w:val="N"/>` | Font size (complex script) | Same value as `w:sz` |
| `<w:t>` | Text content | Use `xml:space="preserve"` attribute |
| `<w:sectPr>` | Section properties | End of document — page size/margins |
| `<w:pgSz w:w="12240" w:h="15840"/>` | Page size | 12240×15840 = US Letter (8.5×11 inches in twips) |
| `<w:pgMar w:top="1440" .../>` | Margins | 1440 twips = 1 inch |

## Paragraph Helper (Python)

```python
from xml.sax.saxutils import escape

def run(text, bold=False, italic=False, underline=False,
        font="Times New Roman", size=22):
    """Build a run element. size in half-points (22 = 11pt)."""
    rpr_parts = []
    if bold:     rpr_parts.append('<w:b/>')
    if italic:   rpr_parts.append('<w:i/>')
    if underline: rpr_parts.append('<w:u w:val="single"/>')
    rpr_parts.append(f'<w:rFonts w:ascii="{font}" w:hAnsi="{font}"/>')
    rpr_parts.append(f'<w:sz w:val="{size}"/><w:szCs w:val="{size}"/>')
    rpr = f'<w:rPr>{"".join(rpr_parts)}</w:rPr>'
    return f'<w:r>{rpr}<w:t xml:space="preserve">{escape(text)}</w:t></w:r>'

def paragraph(text, bold=False, italic=False, alignment="justify",
              font="Times New Roman", size=22, spacing_after=120,
              first_line_indent=0):
    jc = {"left":"left","center":"center","right":"right","justify":"both"}[alignment]
    ppr = f'<w:pPr><w:jc w:val="{jc}"/><w:spacing w:after="{spacing_after}" w:line="240" w:lineRule="auto"/>'
    if first_line_indent:
        ppr += f'<w:ind w:firstLine="{first_line_indent}"/>'
    ppr += '</w:pPr>'
    return f'<w:p>{ppr}{run(text, bold, italic, font=font, size=size)}</w:p>'

def mixed_paragraph(parts, alignment="justify", spacing_after=120):
    """parts is list of (text, bold, italic, underline) tuples."""
    jc = {"left":"left","center":"center","right":"right","justify":"both"}[alignment]
    ppr = f'<w:pPr><w:jc w:val="{jc}"/><w:spacing w:after="{spacing_after}" w:line="240" w:lineRule="auto"/></w:pPr>'
    runs = ''.join(run(t, b, i, u) for t, b, i, u in parts)
    return f'<w:p>{ppr}{runs}</w:p>'
```

## Assembly

```python
import zipfile

def build_docx(output_path, body_xml_str):
    """Zip up a complete .docx from a document.xml body string."""
    def ct(): return open("[Content_Types].xml").read()
    def rr(): return open("_rels/.rels").read()
    def dr(): return open("word/_rels/document.xml.rels").read()
    # ... use the boilerplate templates above

    xml = ('<?xml version="1.0" encoding="UTF-8" standalone="yes"?>'
           '<w:document xmlns:w="http://schemas.openxmlformats.org/wordprocessingml/2006/main" '
           'xmlns:r="http://schemas.openxmlformats.org/officeDocument/2006/relationships">'
           '<w:body>' + body_xml_str + '</w:body></w:document>')

    with zipfile.ZipFile(output_path, 'w', zipfile.ZIP_DEFLATED) as z:
        z.writestr('[Content_Types].xml', ct())
        z.writestr('_rels/.rels', rr())
        z.writestr('word/_rels/document.xml.rels', dr())
        z.writestr('word/document.xml', xml)
        z.writestr('word/styles.xml', make_styles())
        z.writestr('word/fontTable.xml', make_font_table())
        z.writestr('word/settings.xml', make_settings())
```

## Twip Reference

- **1 inch** = 1440 twips
- **1 cm** ≈ 567 twips
- **11pt font** = 22 half-points
- **US Letter page** = 12240 × 15840 twips
- **1 inch margin** = 1440 twips
