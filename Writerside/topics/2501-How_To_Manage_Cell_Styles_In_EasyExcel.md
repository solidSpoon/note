# 如何在 EasyExcel 的 Handler 中安全设置单元格样式

## 简要说明

本指南演示了如何在 **EasyExcel** 的自定义 `Handler` 中安全地修改单元格样式，避免因为滥用或直接修改导致的样式冲突，或者超出 Excel 对样式数量的限制。

## 1. 简介

在使用 [EasyExcel](https://github.com/alibaba/easyexcel) 生成 Excel 文件时，常常需要在自定义 `Handler` 中调整单元格样式，比如字体颜色、背景色、对齐方式等。然而，若不注意以下问题，容易踩坑：

- **直接修改**同一个 `CellStyle` 对象会影响到所有共享该样式的单元格。
- **过量**创建新的 `CellStyle`（例如在循环中反复 `cloneStyleFrom`）会导致样式数轻松突破 **64,000** 的 Excel 上限并抛出异常。

本教程将带你**避开**这些踩坑点，并展示 **最佳实践** 来在 EasyExcel 中安全地自定义单元格样式。

## 2. 先决条件

- 熟悉 **EasyExcel** 的基本用法（如生成 Excel、编写自定义 Handler 等）。
- 已在项目中引入 **EasyExcel** 依赖并具备 Java 开发环境。
- 对 [Apache POI](https://poi.apache.org/) 有一定了解，因为 EasyExcel 底层使用 POI 来管理单元格样式。

## 3. 两个常见的错误用法

### 3.1 错误用法 1：直接修改现有的 CellStyle

下面的示例中，作者从单元格中直接获取现有的 `CellStyle` 并设置字体为红色。由于同一个 `CellStyle` 可能被多个单元格引用，这样的修改会无意中影响到其他单元格。

```java
class ProblematicFontColorHandler extends AbstractCellWriteHandler {
    @Override
    public void afterCellDispose(WriteSheetHolder writeSheetHolder, 
                                 WriteTableHolder writeTableHolder,
                                 List<CellData> cellDataList, 
                                 Cell cell, 
                                 Head head, 
                                 Integer rowIndex, 
                                 Boolean isHead) {
        if (isHead) {
            return;
        }

        // 仅作示例：在第二列设置样式
        if (cell.getColumnIndex() == 1) {
            Workbook workbook = writeSheetHolder.getSheet().getWorkbook();

            // ⚠️ 不正确：复用已有的 CellStyle，可能会影响所有引用它的单元格
            CellStyle existingStyle = cell.getCellStyle();
            Font redFont = workbook.createFont();
            redFont.setColor(IndexedColors.RED.getIndex());
            existingStyle.setFont(redFont);  // 其他共享此样式的单元格也会变红
        }
    }
}
```

#### 这样做的问题

- **CellStyle 复用**：Excel（或 POI）会将同一个 `CellStyle` 对象用于多个单元格。修改一次就会影响到所有使用该样式的单元格。

### 3.2 错误用法 2：过度使用 `cloneStyleFrom` 并为每个单元格都创建新样式

如果每个单元格都通过 `cloneStyleFrom` 的方式创建一个新样式，样式数量很快就会超过 **64,000**（Excel 限制），然后抛出异常。

```java
class ExcessiveStyleHandler extends AbstractCellWriteHandler {
    @Override
    public void afterCellDispose(WriteSheetHolder writeSheetHolder, 
                                 WriteTableHolder writeTableHolder,
                                 List<CellData> cellDataList, 
                                 Cell cell, 
                                 Head head, 
                                 Integer rowIndex, 
                                 Boolean isHead) {
        if (isHead) {
            return;
        }

        // ⚠️ 不正确：为每一个单元格都创建新的样式
        Workbook workbook = writeSheetHolder.getSheet().getWorkbook();
        CellStyle newStyle = workbook.createCellStyle();
        CellStyle originalStyle = cell.getCellStyle();
        newStyle.cloneStyleFrom(originalStyle);
        newStyle.setFillForegroundColor(IndexedColors.LIGHT_BLUE.getIndex());
        newStyle.setFillPattern(FillPatternType.SOLID_FOREGROUND);
        cell.setCellStyle(newStyle); 
        // 大规模数据时，这种操作很快会触发超过 64,000 样式的限制
    }
}
```

#### 这样做的问题

- **突破样式上限**：Excel 一张表格最多允许使用 **64,000** 个样式。如果为每个单元格都创建一次，就会轻易越过此阈值。

## 4. 正确的修改单元格样式方式

推荐使用 Apache POI 提供的 `CellUtil.setCellStyleProperty(cell, key, value)` 或类似 API 来设置样式。该方法会**智能复用**已有的样式对象，而不是每次都新建一个。同时，它也避免了直接修改共享 `CellStyle` 带来的风险。

**要点**：

1. **一般属性**：对于大多数单元格样式属性（如对齐、背景色、边框等），直接使用
   `CellUtil.setCellStyleProperty(cell, key, value)`
   或
   `CellUtil.setCellStyleProperties(cell, propertiesMap)`
   即可完成修改。
2. **字体（Font）特殊处理**：字体作为独立对象需要特殊处理。优先使用 `Workbook.findFont(...)` 查找匹配的现有字体，找不到再用 `Workbook.createFont()` 创建。然后通过 `CellUtil.setFont(cell, font)` 应用。尽量复用现有字体，这样可以避免 EasyExcel 内部去重算法失效，并让 POI 更有效地匹配和复用已存在的样式。

## 5. 完整示例

### 5.1 ExcelData 类

首先定义数据模型，并使用 EasyExcel 的注解：

```java
@Data
class ExcelData {
    @ExcelProperty("字符串标题")
    private String text;

    @ExcelProperty("日期标题")
    private Date date;

    @ExcelProperty("数字标题")
    private Double number;
}
```

### 5.2 使用正确的 Handler

下面是一个**正确**的做法，只在第二列修改字体颜色：

```java
class FontColorHandler extends AbstractCellWriteHandler {
    @Override
    public void afterCellDispose(WriteSheetHolder writeSheetHolder, 
                                 WriteTableHolder writeTableHolder,
                                 List<CellData> cellDataList, 
                                 Cell cell, 
                                 Head head, 
                                 Integer rowIndex, 
                                 Boolean isHead) {
        if (isHead) {
            return;
        }

        // 仅处理第 2 列（下标为 1）
        if (cell.getColumnIndex() == 1) {
            Workbook workbook = writeSheetHolder.getSheet().getWorkbook();

            // 获取当前单元格所使用的字体
            Font existingFont = workbook.getFontAt(cell.getCellStyle().getFontIndexAsInt());

            // 检查是否已存在匹配的红色字体
            Font newFont = workbook.findFont(
                    existingFont.getBold(),
                    IndexedColors.RED.getIndex(),
                    existingFont.getFontHeight(),
                    existingFont.getFontName(),
                    existingFont.getItalic(),
                    existingFont.getStrikeout(),
                    existingFont.getTypeOffset(),
                    existingFont.getUnderline()
            );

            // 如果没有找到匹配的字体，则创建一个新的
            if (newFont == null) {
                newFont = workbook.createFont();
                newFont.setBold(existingFont.getBold());
                newFont.setColor(IndexedColors.RED.getIndex());
                newFont.setFontHeight(existingFont.getFontHeight());
                newFont.setFontName(existingFont.getFontName());
                newFont.setItalic(existingFont.getItalic());
                newFont.setStrikeout(existingFont.getStrikeout());
                newFont.setTypeOffset(existingFont.getTypeOffset());
                newFont.setUnderline(existingFont.getUnderline());
            }

            // 使用 CellUtil 来保证对样式的复用处理
            CellUtil.setFont(cell, newFont);

            // 可选：设置对齐方式
            CellUtil.setCellStyleProperty(cell, CellUtil.ALIGNMENT, HorizontalAlignment.CENTER);

            // 或通过 Map 一次性传入多个属性
            Map<String, Object> properties = new HashMap<>();
            properties.put(CellUtil.ALIGNMENT, HorizontalAlignment.CENTER);
            CellUtil.setCellStyleProperties(cell, properties);
        }
    }
}
```

### 5.3 校验代码有效性

下面代码可以打印当前 Excel 中已经创建的 CellStyle 数量，在 Handler 前后打印一下可以用来检验 Handler 中设置的样式是否被复用。

```java
log.info(workbook.getNumCellStyles());
```

#### 如何避免问题

- **不直接**调用 `cell.getCellStyle().setFont(...)` 修改现有样式。
- **尽量重用**匹配的字体或样式，减少无谓的新建。

## 6. 总结

如果想在 EasyExcel 中安全地自定义单元格样式，可以遵循以下要点：

1. **不要**直接修改已有 `CellStyle`，否则可能影响所有共享该样式的单元格。
2. **不要**为每个单元格都创建新的样式，否则极易触发 Excel 64,000 样式上限。
3. 合理使用 **`CellUtil`** 和 **`findFont`**，让 Apache POI 来对已有的样式进行复用。

采用这些方法即可在避免样式冲突的同时，得到所需的单元格格式。

---

> **更多阅读**
>
> - [EasyExcel GitHub 仓库](https://github.com/alibaba/easyexcel)
> - [Apache POI `CellUtil` 源码文档](https://poi.apache.org/apidocs/dev/org/apache/poi/ss/util/CellUtil.html)
> - [The Good Docs Project](https://thegooddocsproject.dev/) 了解更多文档写作模板和最佳实践。