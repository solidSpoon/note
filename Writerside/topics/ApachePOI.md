# Apache POI

相关元素之间的关系如下：
```
XWPFDocument
├─ XWPFParagraph
│   └─ XWPFRun
│
└─ XWPFTable
    ├─ XWPFTableRow
    │   └─ XWPFTableCell
    │       ├─ XWPFParagraph
    │       │   └─ XWPFRun
    │       │
    │       └─ (其他内容，例如嵌套表格)
    │
    └─ (其他行)
```