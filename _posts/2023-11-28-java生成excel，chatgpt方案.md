---
layout: mypost
title: java生成excel，chatgpt方案
categories: [JAVA]
---


### 问：用java 实现一个生成excel的程序，excel的头包含姓名，年龄，成绩，其中成绩包含子标题语文， 数学，其中成绩需要合并列，姓名，年龄 需要合并行；填充一些测试数据后， 并给出展示效果 


我们可以使用 Apache POI 库来创建和操作 Excel 文件。以下是一个使用 Apache POI 创建 Excel 文件的示例，根据您的要求，我们在表头中添加了“姓名”，“年龄”和“成绩”，其中“成绩”下有子标题“语文”和“数学”。并且合并了姓名和年龄的行，以及成绩的列。

首先，确保您的项目已经添加了 Apache POI 的依赖。如果您正在使用 Maven，可以在 pom.xml 文件中添加以下依赖：

````xml
<‌dependencies>
    <‌!-- https://mvnrepository.com/artifact/org.apache.poi/poi -->
    <‌dependency>
        <‌groupId>org.apache.poi<‌/groupId>
        <‌artifactId>poi<‌/artifactId>
        <‌version>5.0.0<‌/version>
    <‌/dependency>
    <‌!-- https://mvnrepository.com/artifact/org.apache.poi/poi-ooxml -->
    <‌dependency>
        <‌groupId>org.apache.poi<‌/groupId>
        <‌artifactId>poi-ooxml<‌/artifactId>
        <‌version>5.0.0<‌/version>
    <‌/dependency>
<‌/dependencies>
````

然后，我们来创建一个 Java 类来生成 Excel 文件：

````java
import org.apache.poi.ss.usermodel.*;
import org.apache.poi.ss.util.CellRangeAddress;
import org.apache.poi.xssf.usermodel.XSSFWorkbook;

import java.io.FileOutputStream;
import java.io.IOException;

public class ExcelCreator {

    public static void main(String[] args) throws IOException {
        // 创建 Workbook 和 Sheet
        Workbook workbook = new XSSFWorkbook();
        Sheet sheet = workbook.createSheet("学生成绩表");

        // 创建标题行
        Row titleRow = sheet.createRow(0);
        titleRow.createCell(0).setCellValue("姓名");
        titleRow.createCell(1).setCellValue("年龄");
        titleRow.createCell(2).setCellValue("成绩");

        // 创建子标题行
        Row subTitleRow = sheet.createRow(1);
        subTitleRow.createCell(2).setCellValue("语文");
        subTitleRow.createCell(3).setCellValue("数学");

        // 合并“姓名”和“年龄”的行
        sheet.addMergedRegion(new CellRangeAddress(0, 1, 0, 0)); // 合并单元格，参数分别为：first row, last row, first column, last column
        sheet.addMergedRegion(new CellRangeAddress(0, 1, 1, 1));

        // 合并“成绩”的列
        sheet.addMergedRegion(new CellRangeAddress(0, 0, 2, 3));

        // 填充一些测试数据
        for (int i = 2; i <‌ 10; i++) {
            Row row = sheet.createRow(i);
            row.createCell(0).setCellValue("学生" + i);
            row.createCell(1).setCellValue(10 + i);
            row.createCell(2).setCellValue(85 + i);
            row.createCell(3).setCellValue(90 + i);
        }

        // 保存到文件
        try (FileOutputStream fileOut = new FileOutputStream("学生成绩表.xlsx")) {
            workbook.write(fileOut);
        }
        workbook.close();
    }
}
````

这个程序将生成一个名为 "学生成绩表.xlsx" 的 Excel 文件，文件中将包含一个表格，表头是“姓名”，“年龄”和“成绩”，其中“成绩”下有子标题“语文”和“数学”。并且合并了姓名和年龄的行，以及成绩的列。

请注意，这只是一个基础示例，您可能需要根据你的实际情况对其进行修改和扩展。
