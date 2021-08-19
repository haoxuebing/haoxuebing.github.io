---
title: go操作excel
date: 2021-08-18 18:45:14
categories: go
tags: [go, excelize]
description: 本篇文章介绍如何使用go语言操作excel
---
# go操作excel
> go 语言下操作excel的包很多，这里主要介绍 excelize 的使用，使用本类库要求使用的 Go 语言为 <font color=red> 1.15 </font>或更高版本 

## 安装 
- 使用 go Modules 管理软件包
> go get -u github.com/xuri/excelize/v2



## 创建 Excel 文档
```golang
package main

import (
    "fmt"
    "github.com/xuri/excelize/v2"
)

func main() {
    f := excelize.NewFile()
    // 创建一个工作表
    index := f.NewSheet("Sheet2")
    // 设置单元格的值
    f.SetCellValue("Sheet2", "A2", "Hello world.")
    f.SetCellValue("Sheet1", "B2", 100)
    // 设置工作簿的默认工作表
    f.SetActiveSheet(index)
    // 根据指定路径保存文件
    if err := f.SaveAs("Book1.xlsx"); err != nil {
        fmt.Println(err)
    }
}
```

## 读取 Excel 文档
```golang
package main

import (
    "fmt"
    "github.com/xuri/excelize/v2"
)

func main() {
    f, err := excelize.OpenFile("Book1.xlsx")
    if err != nil {
        fmt.Println(err)
        return
    }
    // 获取工作表中指定单元格的值
    cell, err := f.GetCellValue("Sheet1", "B2")
    if err != nil {
        fmt.Println(err)
        return
    }
    fmt.Println(cell)
    // 获取 Sheet1 上所有单元格
    rows, err := f.GetRows("Sheet1")
    for _, row := range rows {
        for _, colCell := range row {
            fmt.Print(colCell, "\t")
        }
        fmt.Println()
    }
}
```

## 使用文件流并返回前端下载
- 以gin框架为例
```golang
func downloadExcel(content *gin.Context) {
        
    file := excelize.NewFile()
    streamWriter, _ := file.NewStreamWriter("Sheet1")
    streamWriter.SetColWidth(1, 2, 5)  // 设置第一列的宽度
    streamWriter.SetColWidth(2, 3, 10) // 设置第二列的宽度
    // 设置第一行单元格的值
    firstRow := []interface{}{
        excelize.Cell{Value: "id"},
        excelize.Cell{Value: "name"},
    }
    if err := streamWriter.SetRow("A1", firstRow); err != nil {
        fmt.Println(err) //错误处理
    }

    // 给其他行单元格赋值
    for rowID := 1; rowID < 10; rowID++ {
        row := make([]interface{}, 2)
        row[0] = rowID
        row[1] = "xxx"
        cell, _ := excelize.CoordinatesToCellName(1, rowID+1) //决定写入的位置
        if err := streamWriter.SetRow(cell, row); err != nil {
            fmt.Println(err) //错误处理
        }
    }
    //结束流式写入过程
    if err := streamWriter.Flush(); err != nil {
        fmt.Println(err) //错误处理
    }

    fileName := fmt.Sprintf("%s.xlsx", time.Now().Format("20060102150405")) //设置文件名
    // 设置返回值的header
    context.Header("Content-Disposition", fmt.Sprintf(`attachment; filename="%s"`, fileName))
    context.Header("Content-Type", "application/vnd.openxmlformats-officedocument.spreadsheetml.sheet")

    var buffer bytes.Buffer
    _ = file.Write(&buffer)
    content := bytes.NewReader(buffer.Bytes())
    http.ServeContent(context.Writer, context.Request, fileName, time.Now(), content)
}    
```











