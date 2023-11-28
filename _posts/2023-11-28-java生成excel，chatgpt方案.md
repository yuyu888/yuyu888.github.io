---
layout: mypost
title: java生成excel，chatgpt方案
categories: [JAVA]
---


问：用java 实现一个生成excel的程序，excel的头包含姓名，年龄，成绩，其中成绩包含子标题语文， 数学  


要使用Java来生成Excel文件，你可以使用Apache POI库，它是一个用于读取和写入微软Office文件的强大工具，包括Excel。

以下是一个简单的示例，使用Apache POI创建一个Excel文件，包含姓名、年龄和成绩（其中成绩又包含语文和数学）：

首先，需要在你的项目中引入Apache POI库。如果你的项目是Maven项目，你可以在`pom.xml`文件中添加以下依赖：

````xml
<‌dependencies>
    <‌dependency>
        <‌groupId>org.apache.poi<‌/groupId>
        <‌artifactId>poi<‌/artifactId>
        <‌version>5.0.0<‌/version>
    <‌/dependency>
    <‌dependency>
        <‌groupId>org.apache.poi<‌/groupId>
        <‌artifactId>poi-ooxml<‌/artifactId>
        <‌version>5.0.0<‌/version>
    <‌/dependency>
<‌/dependencies>
````

然后，你可以使用以下代码来创建一个Excel文件：

````java
import org.apache.poi.ss.usermodel.*;
import org.apache.poi.xssf.usermodel.XSSFWorkbook;

import java.io.FileOutputStream;
import java.io.IOException;

public class ExcelCreator {
    public static void main(String[] args) {
        Workbook workbook = new XSSFWorkbook();
        Sheet sheet = workbook.createSheet("Student Data");

        // 创建头部
        Row headerRow = sheet.createRow(0);
        Cell cell = headerRow.createCell(0);
        cell.setCellValue("姓名");
        cell = headerRow.createCell(1);
        cell.setCellValue("年龄");
        cell = headerRow.createCell(2);
        cell.setCellValue("成绩");

        // 创建子标题
        Row subHeaderRow = sheet.createRow(1);
        cell = subHeaderRow.createCell(2);
        cell.setCellValue("语文");
        cell = subHeaderRow.createCell(3);
        cell.setCellValue("数学");

        // 在这里添加你的数据
        // 例如：
        Row dataRow = sheet.createRow(2);
        cell = dataRow.createCell(0);
        cell.setCellValue("张三");
        cell = dataRow.createCell(1);
        cell.setCellValue(18);
        cell = dataRow.createCell(2);
        cell.setCellValue(80);
        cell = dataRow.createCell(3);
        cell.setCellValue(90);

        // 保存到文件
        try (FileOutputStream out = new FileOutputStream("student_data.xlsx")) {
            workbook.write(out);
        } catch (IOException e) {
            e.printStackTrace();
        }

        // 记得关闭workbook
        try {
            workbook.close();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
````

这个程序首先创建了一个工作簿和一个工作表，然后在工作表中创建了两行，分别用于存储头部和子标题。然后，它在工作表中创建了一行用于存储数据。你可以根据你的需要添加更多的数据行。最后，它将工作簿保存到一个文件中，并关闭了工作簿。

注意，这只是一个基础示例，你可能需要根据你的具体需求来修改和扩展这个程序。例如，你可能需要设置单元格的样式，或者需要从数据库中读取数据来填充工作表。