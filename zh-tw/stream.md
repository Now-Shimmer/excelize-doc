# 流式寫入

`StreamWriter` 用於定義流式寫入器的資料類別。

```go
type StreamWriter struct {
    File    *File
    Sheet   string
    SheetID int
    // 還包含其他已過濾或未導出的欄位
}
```

`Cell` 在 `StreamWriter.SetRow` 中使用，用於指定存儲格的值、公式和樣式。

```go
type Cell struct {
    StyleID int
    Formula string
    Value   interface{}
}
```

`RowOpts` 在 `StreamWriter.SetRow` 中使用，用於設定列樣式。

```go
type RowOpts struct {
    Height       float64
    Hidden       bool
    StyleID      int
    OutlineLevel int
}
```

## 獲取流式寫入器 {#NewStreamWriter}

```go
func (f *File) NewStreamWriter(sheet string) (*StreamWriter, error)
```

NewStreamWriter 通過給定的工作表名稱傳回流式寫入器，用於生成包含大規模資料的工作表。請注意通過此方法按列向工作表寫入資料後，必須調用 [`Flush`](stream.md#Flush) 函數來結束流式寫入過程，並需要確所保寫入的行號是遞增的，普通函数不能與流式函数混合使用在工作表中寫入資料。例如，向工作表流式按行寫入 `102400` 行 x `50` 列帶有樣式的數據：

```go
file := excelize.NewFile()
defer func() {
    if err := file.Close(); err != nil {
        fmt.Println(err)
    }
}()
streamWriter, err := file.NewStreamWriter("Sheet1")
if err != nil {
    fmt.Println(err)
}
styleID, err := file.NewStyle(&excelize.Style{Font: &excelize.Font{Color: "#777777"}})
if err != nil {
    fmt.Println(err)
}
if err := streamWriter.SetRow("A1",
    []interface{}{
        excelize.Cell{StyleID: styleID, Value: "Data"},
        []excelize.RichTextRun{
            {Text: "Rich ", Font: &excelize.Font{Color: "2354e8"}},
            {Text: "Text", Font: &excelize.Font{Color: "e83723"}},
        },
    },
    excelize.RowOpts{Height: 45, Hidden: false}); err != nil {
    fmt.Println(err)
}
for rowID := 2; rowID <= 102400; rowID++ {
    row := make([]interface{}, 50)
    for colID := 0; colID < 50; colID++ {
        row[colID] = rand.Intn(640000)
    }
    cell, _ := excelize.CoordinatesToCellName(1, rowID)
    if err := streamWriter.SetRow(cell, row); err != nil {
        fmt.Println(err)
    }
}
if err := streamWriter.Flush(); err != nil {
    fmt.Println(err)
}
if err := file.SaveAs("Book1.xlsx"); err != nil {
    fmt.Println(err)
}
```

流式設定存儲格的公式和值：

```go
err := streamWriter.SetRow("A1", []interface{}{
    excelize.Cell{Value: 1},
    excelize.Cell{Value: 2},
    excelize.Cell{Formula: "SUM(A1,B1)"}})
```

流式設定存儲格的值和列樣式：

```go
err := streamWriter.SetRow("A1", []interface{}{
    excelize.Cell{Value: 1}},
    excelize.RowOpts{StyleID: styleID, Height: 20, Hidden: false})
```

流式設定存儲格的值和列的分級顯示：

```go
err := streamWriter.SetRow("A1", []interface{}{
    excelize.Cell{Value: 1}}, excelize.RowOpts{OutlineLevel: 1})
```

## 按行流式寫入工作表 {#SetRow}

```go
func (sw *StreamWriter) SetRow(cell string, values []interface{}, opts ...RowOpts) error
```

SetRow 通過給定的起始坐標和指向數組類別「切片」的指針將資料按行流式寫入工作表中。請注意，在設定行之後，必須調用 [`Flush`](stream.md#Flush) 函數來結束流式寫入過程，並需要確所保寫入的列號是遞增的。

## 流式創建表格 {#AddTable}

```go
func (sw *StreamWriter) AddTable(hCell, vCell, opts string) error
```

根據給定的存儲格坐標區域和條件格式流式創建表格。

例1，在 `A1:D5` 區域流式創建表格：

```go
err := streamWriter.AddTable("A1", "D5", "")
```

例2，在工作表 `F2:H6` 區域創建帶有條件格式的表格：

```go
err := streamWriter.AddTable("F2", "H6", `{
    "table_name": "table",
    "table_style": "TableStyleMedium2",
    "show_first_column": true,
    "show_last_column": true,
    "show_row_stripes": false,
    "show_column_stripes": true
}`)
```

注意，表格坐標區域至少需要包含兩列：字符型的標題列和內容列。每欄標題列的字符需保證是唯一的，當前僅支持在每個工作表中流式創建一張表格，並且必須在調用該函數前通過 [`SetRow`](stream.md#SetRow) 流式設定表格的標題列數據。支持的表格樣式與非流式創建表格 [`AddTable`](utils.md#AddTable) 相同。

## 流式插入分頁符 {#InsertPageBreak}

```go
func (sw *StreamWriter) InsertPageBreak(cell string) error
```

根據給定的儲存格坐標插入分頁符。分頁符是將工作表分成單獨的頁面以便打印的分隔線。

## 流式设定窗格 {#SetPanes}

```go
func (sw *StreamWriter) SetPanes(panes string) error
```

通過給定的窗格樣式參數流式設定凍結窗格，必須在調用 [`SetRow`](stream.md#SetRow) 之前調用該函數設定窗格。

## 流式合併存儲格 {#MergeCell}

```go
func (sw *StreamWriter) MergeCell(hCell, vCell string) error
```

通過給定的存儲格坐標區域流式合併存儲格，當前僅支持合併非交疊區域存儲格。

## 流式設定欄寬度 {#SetColWidth}

```go
func (sw *StreamWriter) SetColWidth(min, max int, width float64) error
```

根據給定的欄範圍和寬度值設定單個或多個欄的寬度，必須在調用 [`SetRow`](stream.md#SetRow) 之前調用該函數設定欄寬度。例如設定工作表上 `B` 到 `C` 欄的寬度為 `20`：

```go
err := streamWriter.SetColWidth(2, 3, 20)
```

## 結束流式寫入 {#Flush}

```go
func (sw *StreamWriter) Flush() error
```

Flush 用於結束流式寫入過程。
