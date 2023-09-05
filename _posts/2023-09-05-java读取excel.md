---
layout: mypost
title: java 读取excel
categories: [JAVA]
---

## 引入工作类

````java
import org.apache.poi.ss.usermodel.Sheet;
import org.apache.poi.ss.usermodel.Workbook;
import org.apache.poi.ss.usermodel.WorkbookFactory;
````

## 实现方法

````java
    public void readExcel(String path) throws Exception {
        Map<String, List<String>> map = new HashMap<>();
        //根据文件位置找到需要读取的excel文件
        File Inputfile = new File(path);

        //根据路径生成 FileInputStream字节流
        FileInputStream fileInputStream = new FileInputStream(Inputfile);

        //将FileInputStream转换为Workbook
        Workbook workbook = WorkbookFactory.create(fileInputStream);

        // 默认获取第一张工作表，可以自定义
        Sheet sheet = workbook.getSheet("Sheet1");

        // 循环获取每一行数据 因为默认第一行为标题行，我们可以从 1 开始循环，如果没有标题行，i从 0 开始
        // sheet.getLastRowNum() 获取总行数
        for (int i = 0; i < sheet.getLastRowNum()+1; i++) {
            // 获取第 i 行的第0列信息
            String content1 = sheet.getRow(i).getCell(0).toString();
            log.info("==============================");
            log.info(content1);
        }
    }
````