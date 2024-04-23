---
layout: mypost
title: java生成excel，可以自定义复合表头
categories: [JAVA]
---

## 功能概述

1、节省资源， 减少内存消耗  
2、提供基础的样式处理  
3、实现生成自定义无限层复合表头  

## 相关知识点

### XSSFWorkbook 和 SXSSFWorkbook 的区别

Apache POI 提供了两种不同的类来处理 XLSX 文件：XSSFWorkbook 和 SXSSFWorkbook。这两种工作簿都可以用来创建、读取和编辑 XLSX 文件，但它们在内存使用和性能方面有一些重要的区别。

以下是 XSSFWorkbook 和 SXSSFWorkbook 之间的主要区别：
- 内存使用：XSSFWorkbook 是一个完全内存驻留的模型，这意味着当你处理一个大的 XLSX 文件时，它会消耗相当多的内存。在处理大量数据或大型工作簿时，可能会导致 OutOfMemoryError。相比之下，SXSSFWorkbook 是一个内存友好型的模型，它使用了一种叫做 "sliding window" 的机制，这使得它可以在处理大型工作簿时显著节省内存。
- 性能：由于 XSSFWorkbook 在内存中保存了整个工作簿，所以从性能的角度来看，对于小型或中型的工作簿，XSSFWorkbook 的读写性能可能会比 SXSSFWorkbook 更好。然而，对于大型工作簿，由于 SXSSFWorkbook 的内存优化，它的性能通常会优于 XSSFWorkbook。
- 功能：XSSFWorkbook 提供了对 XLSX 文件的全面支持，包括读取、写入、编辑等操作。相比之下，SXSSFWorkbook 仅支持写入操作，不支持读取或修改现有的工作簿。
总的来说，如果你需要处理的 XLSX 文件较小，并且需要对其进行复杂的操作（如读取、修改等），那么 XSSFWorkbook 可能是更好的选择。然而，如果你需要处理大型工作簿，并且只需要写入操作，那么 SXSSFWorkbook 就是一个更好的选择，因为它可以显著减少内存使用并提高性能。

OutOfMemoryError

![OutOfMemoryError](image.png)

### SXSSFWorkbook 介绍

SXSSFWorkbook 是 Apache POI 库中的一个类，该类对于处理大型 Excel 文件（例如 .xlsx 文件）非常有用。它实现了一种名为流式写入的技术，可以显著减少在处理大型数据集时的内存使用。

以下是 SXSSFWorkbook 节省内存的原理：
1. 数据流式写入：在处理大型数据集时，常规的工作簿（如 XSSFWorkbook）会将所有数据保留在内存中，这可能导致大量内存使用。而 SXSSFWorkbook 采用流式写入技术，只保持一部分数据在内存中。当数据写入磁盘后，这部分内存就可以释放，以便存储新的数据。
2. 临时文件：为了支持流式写入，SXSSFWorkbook 创建临时文件存储那些已经写入但尚未持久化的数据。这些临时文件在调用 dispose() 方法后会被删除。
3. 可配置的滑动窗口：SXSSFWorkbook 允许你配置在内存中保持的行数。这个行数称为 windowSize，可以根据可用内存进行设置。一旦写入的行数超过这个滑动窗口，最早写入的行将被推出内存并写入临时文件。这个特性使得 SXSSFWorkbook 能够处理大于可用内存的数据集。
总的来说，SXSSFWorkbook 通过流式写入和临时文件存储实现了内存的有效管理，并通过可配置的滑动窗口提供了灵活的内存使用控制，从而在处理大型数据集时能够显著节省内存。

## 代码实现

````java
import lombok.extern.slf4j.Slf4j;
import org.apache.poi.ss.usermodel.*;
import org.apache.poi.ss.util.CellRangeAddress;
import org.apache.poi.xssf.streaming.SXSSFRow;
import org.apache.poi.xssf.streaming.SXSSFSheet;
import org.apache.poi.xssf.streaming.SXSSFWorkbook;

import javax.servlet.http.HttpServletResponse;
import java.io.FileNotFoundException;
import java.io.FileOutputStream;
import java.io.IOException;
import java.io.OutputStream;
import java.math.BigDecimal;
import java.net.URLEncoder;
import java.util.ArrayList;
import java.util.List;

@Slf4j
public class ExcelCreator {

    public void createExcleFile(String filePath, String sheetname, List<ExcelTitleVo> title, List<List<Object>> content){
        Workbook workbook = getWorkbook(sheetname, title, content);
        try (FileOutputStream fileOut = new FileOutputStream(filePath)) {
            workbook.write(fileOut);
        }catch (FileNotFoundException e) {
            e.printStackTrace();
            throw new RRException("文件不存在：" + e.getMessage());
        } catch (IOException e) {
            e.printStackTrace();
            throw new RRException("IO错误, 文件生成失败：" + e.getMessage());
        } finally {
            try {
                workbook.close();
            } catch (IOException e) {
                e.printStackTrace();
                throw new RRException("workbook不能成功close：" + e.getMessage());
            }
        }
    }

    public void createExcleFile(String filePath, String sheetname, String[] title, List<List<Object>> content){
        Workbook workbook = getWorkbook(sheetname, title, content);
        try (FileOutputStream fileOut = new FileOutputStream(filePath)) {
            workbook.write(fileOut);
        }catch (FileNotFoundException e) {
            e.printStackTrace();
            throw new RRException("文件不存在：" + e.getMessage());
        } catch (IOException e) {
            e.printStackTrace();
            throw new RRException("IO错误, 文件生成失败：" + e.getMessage());
        } finally {
            try {
                workbook.close();
            } catch (IOException e) {
                e.printStackTrace();
                throw new RRException("workbook不能成功close：" + e.getMessage());
            }
        }
    }

    public void exportToPage(String fileName, String sheetname, List<ExcelTitleVo> title, List<List<Object>> content, HttpServletResponse response){
        Workbook workbook = getWorkbook(sheetname, title, content);
        outputToPage(fileName, workbook, response);
    }

    public void exportToPage(String fileName, String sheetname, String[] title, List<List<Object>> content, HttpServletResponse response){
        Workbook workbook = getWorkbook(sheetname, title, content);
        outputToPage(fileName, workbook, response);
    }

    public Workbook getWorkbook(String sheetname, String[] title, List<List<Object>> content){
        //新建文档实例
        SXSSFWorkbook workbook = new SXSSFWorkbook(100);
        //在文档中添加表单
        SXSSFSheet sheet = workbook.createSheet(sheetname);
        writeTitle(workbook, sheet, title);
        writeContent(workbook, sheet, content, 1);
        return workbook;
    }

    public Workbook getWorkbook(String sheetname, List<ExcelTitleVo> title, List<List<Object>> content){
        //新建文档实例
        SXSSFWorkbook workbook = new SXSSFWorkbook(100);
        //在文档中添加表单
        SXSSFSheet sheet = workbook.createSheet(sheetname);
        Integer titleRows = writeCompositeTitle(workbook, sheet, title);
        writeContent(workbook, sheet, content, titleRows);
        return workbook;
    }

    public void writeTitle(SXSSFWorkbook workbook, SXSSFSheet sheet, String[] title){
        // 设置标题样式
        Font font = workbook.createFont();
        font.setBold(true);
        font.setFontHeightInPoints((short) 12);
        CellStyle titleStyle = workbook.createCellStyle();
        titleStyle.setAlignment(HorizontalAlignment.CENTER);
        titleStyle.setVerticalAlignment(VerticalAlignment.CENTER);
        titleStyle.setFont(font);

        //创建第一行，用于填充标题
        SXSSFRow titleRow = sheet.createRow(0);
        //填充标题
        for (int i=0 ; i<title.length ; i++) {
            //创建单元格
            Cell cell = titleRow.createCell(i);
            //设置单元格内容
            cell.setCellValue(title[i]);
            //设置单元格样式
            cell.setCellStyle(titleStyle);
        }
    }

    public void writeContent(SXSSFWorkbook workbook, SXSSFSheet sheet, List<List<Object>> content, int titleRows){
        // 字符串样式
        CellStyle style = workbook.createCellStyle();
        style.setAlignment(HorizontalAlignment.CENTER);
        style.setVerticalAlignment(VerticalAlignment.CENTER);

        // 设置内容格式
        DataFormat df = workbook.createDataFormat();
        CellStyle cellStyleAmount = workbook.createCellStyle(); // 创建 BigDecimal 单元格样式
        cellStyleAmount.setDataFormat(df.getFormat("0.00"));
        cellStyleAmount.setAlignment(HorizontalAlignment.CENTER);
        cellStyleAmount.setVerticalAlignment(VerticalAlignment.CENTER);
        CellStyle cellStyleInt = workbook.createCellStyle(); // 创建Int单元格样式
        cellStyleInt.setDataFormat(df.getFormat("0"));
        cellStyleInt.setAlignment(HorizontalAlignment.CENTER);
        cellStyleInt.setVerticalAlignment(VerticalAlignment.CENTER);

        //填充内容
        for (int i=0 ; i<content.size() ; i++) {
            //创建行
            SXSSFRow row = sheet.createRow(i+titleRows);
            //遍历某一行
            List<Object> rowContent = content.get(i);
            for (int j=0 ; j<rowContent.size() ; j++) {
                //创建单元格
                Cell cell = row.createCell(j);
                //设置单元格内容
                if (rowContent.get(j) == null) {
                    cell.setCellValue("");
                    cell.setCellStyle(style);
                }else if(rowContent.get(j)  instanceof BigDecimal){
                    cell.setCellValue(((BigDecimal) rowContent.get(j)).doubleValue());
                    cell.setCellStyle(cellStyleAmount);
                } else if (rowContent.get(j) instanceof Double) {
                    cell.setCellValue(Double.parseDouble(rowContent.get(j).toString()));
                    cell.setCellStyle(style);
                } else if (rowContent.get(j) instanceof Integer) {
                    cell.setCellValue(Integer.valueOf(rowContent.get(j).toString()));
                    cell.setCellStyle(cellStyleInt);
                }else {
                    cell.setCellValue(rowContent.get(j).toString());
                    cell.setCellStyle(style);
                }
                //设置单元格样式
            }
        }
    }

    public int writeCompositeTitle(SXSSFWorkbook workbook, SXSSFSheet sheet, List<ExcelTitleVo> title){
        // 设置标题样式
        Font font = workbook.createFont();
        font.setBold(true);
        font.setFontHeightInPoints((short) 12);
        CellStyle titleStyle = workbook.createCellStyle();
        titleStyle.setAlignment(HorizontalAlignment.CENTER);
        titleStyle.setVerticalAlignment(VerticalAlignment.CENTER);
        titleStyle.setFont(font);

        List<List<Integer>> mergeReginList = new ArrayList<>();
        List<List<String>> titleArrray = buildTitleArrray(title, mergeReginList);
        for(int j=0; j<titleArrray.size();j++){
            //创建第一行，用于填充标题
            SXSSFRow titleRow = sheet.createRow(j);
            List<String> rowData =  titleArrray.get(j);
            //填充标题
            for (int i=0 ; i<rowData.size() ; i++) {
                //创建单元格
                Cell cell = titleRow.createCell(i);
                //设置单元格内容
                cell.setCellValue(rowData.get(i));
                //设置单元格样式
                cell.setCellStyle(titleStyle);
            }
        }
        // 合并单元格
        for(List<Integer> mergeIndex: mergeReginList){
            sheet.addMergedRegion(new CellRangeAddress(mergeIndex.get(0)-1, mergeIndex.get(1)-1, mergeIndex.get(2), mergeIndex.get(3)));
        }
        return titleArrray.size();
    }

    public List<List<String>> buildTitleArrray(List<ExcelTitleVo> title, List<List<Integer>> mergeReginList){
        ExcelTitleVo titleVo = new ExcelTitleVo("第一行补充结构用", title);
        ExcelTitleVo newTitleVo = rebuildTitleVo(titleVo, 0);
        List<List<String>> titleArrray = buildTitleArrray(newTitleVo, newTitleVo.getLayers(), 0, 0, mergeReginList);
        titleArrray.remove(0);
        mergeReginList.remove(0);
        return titleArrray;
    }
    private List<List<String>> buildTitleArrray(ExcelTitleVo titleVo, Integer fatherLayerNum, Integer beginRow, Integer beginColumn, List<List<Integer>> mergeReginList) {
        List<List<String>> titleArray = new ArrayList<>();
        List<String> titleList = new ArrayList<>();
        titleList.add(titleVo.getTitle());
        for(int i=1; i<titleVo.getBottomChildrenCount();i++){
            titleList.add("");
        }
        int mergeRowNum = mergeTitleWitchChild(titleVo,0);
        if(mergeRowNum>0){
            if(titleVo.getBottomChildrenCount()>1){
                mergeReginList.add(getMergeIndexList(beginRow, beginRow+mergeRowNum, beginColumn, beginColumn+titleVo.getBottomChildrenCount()-1));
            }
        }else{
            if(beginRow==0 && beginColumn==0){
                mergeReginList.add(getMergeIndexList(beginRow, beginRow, beginColumn, beginColumn+titleVo.getBottomChildrenCount()-1));
            }else{
                if(titleVo.getBottomChildrenCount()>1 && titleVo.getNumberOfBrothers()>0){
//            if(titleVo.getBottomChildrenCount()>1 && !titleVo.getTitle().isEmpty()){
                    mergeReginList.add(getMergeIndexList(beginRow, beginRow, beginColumn, beginColumn+titleVo.getBottomChildrenCount()-1));
                }
            }
        }

        titleArray.add(titleList);
        if(titleVo.getChildren()==null || titleVo.getChildren().size()==0){
            if(fatherLayerNum-1>0){
                for(int j=0; j<fatherLayerNum-1;j++){
                    List<String> titleFillList = new ArrayList<>();
                    titleFillList.add("");
                    titleArray.add(titleFillList);
                }
                mergeReginList.add(getMergeIndexList(beginRow, beginRow+fatherLayerNum-1, beginColumn, beginColumn));
            }
        }else {
            List<List<List<String>>> childTitleArrayList = new ArrayList<>();
            Integer columnOffset = 0;
            for(ExcelTitleVo childVo :titleVo.getChildren()){
                List<List<String>> childTitleArray = buildTitleArrray(childVo, fatherLayerNum-1, beginRow+1, beginColumn+columnOffset, mergeReginList);
                columnOffset += childVo.getBottomChildrenCount();
                childTitleArrayList.add(childTitleArray);
            }
            for(int i = 0; i<fatherLayerNum-1; i++){
                List<String> childTitleList= new ArrayList<>();
                for(List<List<String>> childTitleArrayItem: childTitleArrayList){
                        childTitleList.addAll(childTitleArrayItem.get(i));
                }
                titleArray.add(childTitleList);
            }
        }
        return titleArray;
    }

    private List<Integer> getMergeIndexList(Integer firstRow, Integer lastRow, Integer firstColumn, Integer lastColumn){
        List<Integer> mergeIndexList = new ArrayList<>();
        mergeIndexList.add(firstRow);
        mergeIndexList.add(lastRow);
        mergeIndexList.add(firstColumn);
        mergeIndexList.add(lastColumn);
        return mergeIndexList;
    }

    private int  mergeTitleWitchChild(ExcelTitleVo childVo, int num){
        if(childVo.getChildren()!=null&&childVo.getChildren().size()==1){
            if(childVo.getChildren().get(0).getTitle().isEmpty()){
                num++;
                num = mergeTitleWitchChild(childVo.getChildren().get(0), num);
            }
        }
        return num;
    }

    public ExcelTitleVo rebuildTitleVo(ExcelTitleVo titleVo, int numberOfBrothers){
        ExcelTitleVo newTitleVo = new ExcelTitleVo(titleVo.getTitle(), new ArrayList<>());

        if(titleVo.getChildren()==null||titleVo.getChildren().size()==0){
            newTitleVo.setLayers(1);
            newTitleVo.setBottomChildrenCount(1);
        }else {
            Integer m = 0;
            Integer n = 2;
            List<ExcelTitleVo> children = new ArrayList<>();
            for(ExcelTitleVo childVo :titleVo.getChildren()){
                ExcelTitleVo child =  rebuildTitleVo(childVo, titleVo.getChildren().size()-1);
                children.add(child);
                if(child.getChildren()==null||child.getChildren().size()==0){
                    m += 1;
                }else{
                    m += child.getBottomChildrenCount();
                   if(n < 1+child.getLayers()){
                       n= 1+child.getLayers();
                   }
                }
            }
            newTitleVo.setChildren(children);
            newTitleVo.setLayers(n);
            newTitleVo.setBottomChildrenCount(m);
        }
        newTitleVo.setNumberOfBrothers(numberOfBrothers);
        return  newTitleVo;
    }

    private void outputToPage(String fileName, Workbook workbook, HttpServletResponse response){
        OutputStream outputStream = null;
        try {
            //设置响应头
            response.setCharacterEncoding("UTF-8");
            response.setHeader("filename", URLEncoder.encode(fileName, "UTF-8") + ".xlsx");
            response.addHeader("Access-Control-Expose-Headers", "filename");
            response.setContentType("application/vnd.ms-excel;charset=utf-8");
            response.setHeader("Content-Disposition", "attachment;filename=" + URLEncoder.encode(fileName, "UTF-8") + ".xlsx");
            response.addHeader("Pargam", "no-cache");
            response.addHeader("Cache-Control", "no-cache");
            //获取输出流
            outputStream = response.getOutputStream();
            //用文档写输出流
            workbook.write(outputStream);
            //刷新输出流
            outputStream.flush();
        } catch (Exception e) {
            e.printStackTrace();
            throw new RRException("outputStream 写入失败：" + e.getMessage());
        } finally {
            if (workbook != null) {
                try {
                    workbook.close();
                } catch (IOException e) {
                    e.printStackTrace();
                    throw new RRException("workbook不能成功close：" + e.getMessage());
                }
            }
            //关闭输出流
            if (outputStream != null) {
                try {
                    outputStream.close();
                } catch (IOException e) {
                    e.printStackTrace();
                    throw new RRException("outputStream不能成功close：" + e.getMessage());
                }
            }
        }
    }

}

````

ExcelTitleVo  
````java
@Data
public class ExcelTitleVo {
    private String title;
    private List<ExcelTitleVo> children;
    private Integer  bottomChildrenCount;  // 子类别宽度
    private Integer layers; // 子类别层数
    private Integer numberOfBrothers; // 兄弟节点数量

    public ExcelTitleVo(String title, List<ExcelTitleVo> children) {
        this.title = title;
        this.children = children;
    }
}
````