---
layout: mypost
title: java生成excel,可生成复合表头及结构化行数据
categories: [JAVA]
---

## 前言
上次实现了[excel的自定义复合表头](https://yuyu888.github.io/posts/2024/03/07/java%E7%94%9F%E6%88%90excel-%E5%8F%AF%E8%87%AA%E5%AE%9A%E4%B9%89%E5%A4%8D%E5%90%88%E8%A1%A8%E5%A4%B4.html){:target="_blank"}

最近产品上又需要实现单行数据上能加明细行，于是乎，就加强了一下

##  效果展示

![示例](1754548102407.jpg)

## 完整代码

ExcelContentVo

````java

import lombok.Data;

import java.util.List;

@Data
public class ExcelContentVo {

    private Object finalItem; // 最终的值（实际列数据）

    private List<List<ExcelContentVo>> listItem; // 结构化的list（抽象列数据，需要分拆为多行的数据）

    private Integer  totalRows;  // 总行数
    private Integer beginRowNumber; // 初始行
    private Integer endRowNumber; // 结束行

    private Integer beginColumnNumber; // 初始列
    private Integer endColumnNumber; // 结束列



    public ExcelContentVo(Object finalItem) {
        if(finalItem!=null){
            this.finalItem = finalItem;
        }else {
            this.finalItem = "";
        }
    }

    public ExcelContentVo(List<List<ExcelContentVo>> listItem) {
        this.listItem = listItem;
    }

    public Integer getEndRowNumber() {
        if(endRowNumber!=null){
            return endRowNumber;
        }
        if(totalRows!=null && beginRowNumber!=null){
            return beginRowNumber+totalRows-1;
        }
        return null;
    }
}
````

主程序

````java
import lombok.extern.slf4j.Slf4j;
import org.apache.poi.ss.usermodel.*;
import org.apache.poi.ss.util.CellRangeAddress;
import org.apache.poi.xssf.streaming.SXSSFRow;
import org.apache.poi.xssf.streaming.SXSSFSheet;
import org.apache.poi.xssf.streaming.SXSSFWorkbook;
import org.springframework.stereotype.Service;

import javax.servlet.http.HttpServletResponse;
import java.io.FileNotFoundException;
import java.io.FileOutputStream;
import java.io.IOException;
import java.io.OutputStream;
import java.math.BigDecimal;
import java.net.URLEncoder;
import java.nio.charset.StandardCharsets;
import java.util.ArrayList;
import java.util.Collections;
import java.util.List;

@Slf4j
@Service("excelCreator")
public class ExcelCreator {

	/**
	 * 一个sheet页中的最大行数
	 */
	public static final int MaxLineTotalInOneSheet = 1048576;
	
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

    public void createExcleFile(String filePath, String sheetname, List<ExcelTitleVo> title, ExcelContentVo contentVo){
        Workbook workbook = getWorkbook(sheetname, title, contentVo);
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

    public Workbook getWorkbook(String sheetname, List<ExcelTitleVo> title, ExcelContentVo contentVo) {
        //新建文档实例
        SXSSFWorkbook workbook = new SXSSFWorkbook(100);
        //在文档中添加表单
        SXSSFSheet sheet = workbook.createSheet(sheetname);
        Integer titleRows = writeCompositeTitle(workbook, sheet, title);
        writeCompositeContent(workbook, sheet, contentVo, titleRows);
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

    public void writeCompositeContent(SXSSFWorkbook workbook, SXSSFSheet sheet, ExcelContentVo contentVo, int beginRowNumber){
        List<List<Integer>> mergeReginList = new ArrayList<>();
        List<List<Object>> content = buildoContentArrray(contentVo, beginRowNumber, mergeReginList);
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
            SXSSFRow row = sheet.createRow(i+beginRowNumber);
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
        // 合并单元格
        for(List<Integer> mergeIndex: mergeReginList){
            sheet.addMergedRegion(new CellRangeAddress(mergeIndex.get(0), mergeIndex.get(1), mergeIndex.get(2), mergeIndex.get(3)));
        }
        try {
            sheet.flushRows();
        } catch (IOException e) {
            log.error(e.getMessage(), e);
        }
    }


    public void writeContent(SXSSFWorkbook workbook, SXSSFSheet sheet, List<List<Object>> content, int beginRowNumber){
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
            SXSSFRow row = sheet.createRow(i+beginRowNumber);
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
        try {
			sheet.flushRows();
		} catch (IOException e) {
			log.error(e.getMessage(), e);
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

        return  writeCompositeTitle(sheet, title, titleStyle);
    }

    public int writeCompositeTitle(SXSSFSheet sheet, List<ExcelTitleVo> title, CellStyle titleStyle){
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

    public List<List<Object>> buildoContentArrray(ExcelContentVo content, int beginRowNumber, List<List<Integer>> mergeReginList){
        rebuildContentVo(content, beginRowNumber, 0);
        List<List<Object>> contentArrray = buildContentArrray(content, mergeReginList);
        return contentArrray;
    }

    private List<List<Object>> buildContentArrray(ExcelContentVo contentVo, List<List<Integer>> mergeReginList) {
        List<List<Object>> contentArray = new ArrayList<>();
        if(contentVo.getListItem()!=null && contentVo.getListItem().size()>0){
            for(List<ExcelContentVo> listVo:contentVo.getListItem()){
                List<List<List<Object>>> fieldContentList = new ArrayList<>();
                for(ExcelContentVo itemVo: listVo){
                    List<List<Object>> itemVoContentArray =  new ArrayList<>();
                    if(itemVo.getListItem()!=null) {
                        if(itemVo.getListItem().size()>0){
                            itemVoContentArray = buildContentArrray(itemVo, mergeReginList);
                            fieldContentList.add(itemVoContentArray);
                        }else {
                            log.info("=======================itemVo:{}", itemVo);
                            log.info("=======================listVo:{}", listVo);
                            throw new RRException("数据源格式错误");
                        }
                    }else {
                        itemVoContentArray.add(List.of(itemVo.getFinalItem()));
                        for(int num=itemVo.getBeginRowNumber(); num<itemVo.getEndRowNumber(); num++){
                            itemVoContentArray.add(List.of(""));
                        }
                        fieldContentList.add(itemVoContentArray);
                        if(itemVo.getBeginRowNumber()!=itemVo.getEndRowNumber()){
                            List<Integer> mergeData = List.of(itemVo.getBeginRowNumber(), itemVo.getEndRowNumber(), itemVo.getBeginColumnNumber(), itemVo.getEndColumnNumber());
                            mergeReginList.add(mergeData);
                        }
                    }
                }
                Integer rows = listVo.get(0).getTotalRows();
                for(int i=0; i<rows;i++){
                    List<Object> rowList = new ArrayList<>();
                    for(List<List<Object>> item: fieldContentList){
                        if(item.size()>i){
                            rowList.addAll(item.get(i));
                        }else {
                            List<String> list = new ArrayList<>(Collections.nCopies(item.get(0).size(), ""));
                        }
                    }
                    contentArray.add(rowList);
                }
            }
            return contentArray;
        }
        return contentArray;
    }

    public void rebuildContentVo(ExcelContentVo contentVo, Integer beginRowNumber, Integer beginColumnNumber) {
        contentVo.setBeginRowNumber(beginRowNumber);
        contentVo.setBeginColumnNumber(beginColumnNumber);

        if(contentVo.getListItem()!=null && contentVo.getListItem().size()>0){
            Integer totalRows = 0;
            Integer itemVoBeginRowNumber = beginRowNumber;
            Integer itemVoEndColumnNumber = beginColumnNumber;
            for(List<ExcelContentVo> listVo:contentVo.getListItem()){
                Integer itemVoBeginColumnNumber = beginColumnNumber;
                Integer itemVoTotalRows = 1;
                for(ExcelContentVo itemVo: listVo){
                    if(itemVo.getListItem()!=null && itemVo.getListItem().size()>0) {
                        rebuildContentVo(itemVo, itemVoBeginRowNumber, itemVoBeginColumnNumber);
                        itemVoBeginColumnNumber = itemVo.getEndColumnNumber()+1;
                        if(itemVoTotalRows<itemVo.getTotalRows()){
                            itemVoTotalRows = itemVo.getTotalRows();
                        }
                    }else {
                        itemVo.setBeginRowNumber(itemVoBeginRowNumber);
                        itemVo.setBeginColumnNumber(itemVoBeginColumnNumber);
                        itemVo.setEndColumnNumber(itemVoBeginColumnNumber);
                        itemVo.setTotalRows(1);
                        itemVoBeginColumnNumber++;
                    }
                }
                if(itemVoEndColumnNumber<itemVoBeginColumnNumber){
                    itemVoEndColumnNumber = itemVoBeginColumnNumber;
                }
                itemVoBeginRowNumber += itemVoTotalRows;
                for(ExcelContentVo itemVo: listVo){
                    itemVo.setTotalRows(itemVoTotalRows);
                }
                totalRows+=itemVoTotalRows;
            }
            contentVo.setEndColumnNumber(itemVoEndColumnNumber-1);
            contentVo.setTotalRows(totalRows);
        } else {
            if(contentVo.getListItem()!=null && contentVo.getListItem().size()==0){
                contentVo.setFinalItem("");
            }
            if(contentVo.getFinalItem()==null){
                contentVo.setFinalItem("");
            }
//            contentVo.setBeginRowNumber(beginRowNumber);
//            contentVo.setBeginColumnNumber(beginColumnNumber);
            contentVo.setEndColumnNumber(beginColumnNumber);
            contentVo.setTotalRows(1);
        }
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

## demo

执行 contentVotest 方法

注意content的数据结构

````java

    public List<ExcelContentVo> createMainTestVo(Integer number){
        List<ExcelContentVo> list = new ArrayList<>();
        ExcelContentVo field1 = new ExcelContentVo("需求单号"+number);
        ExcelContentVo field2 = new ExcelContentVo("需求名称"+number);
        ExcelContentVo field3 = new ExcelContentVo("报价单号"+number);
        ExcelContentVo field4 = new ExcelContentVo("供应商"+number);
        ExcelContentVo field5 = new ExcelContentVo("报价金额"+number);
        ExcelContentVo field6 = new ExcelContentVo("报价时间"+number);
        list.add(field1);
        list.add(field2);
        list.add(field3);
        list.add(field4);
        list.add(field5);
        list.add(createDetailTestVo(number));
        list.add(field6);
        list.add(createDetailTestVo(3));
        return list;
    }

    public ExcelContentVo createDetailTestVo(Integer number){
        List<List<ExcelContentVo>> list = new ArrayList<>();
        List<ExcelContentVo> list1 = new ArrayList<>();
        ExcelContentVo field1 = new ExcelContentVo("项目1名称"+number);
        ExcelContentVo field2 = new ExcelContentVo("项目1进度"+number);
        ExcelContentVo field3 = new ExcelContentVo("项目1报价"+number);
        ExcelContentVo field4 = new ExcelContentVo("项目1资源（人/天）"+number);
        list1.add(field1);
        list1.add(field2);
        list1.add(field3);
        list1.add(field4);
        List<ExcelContentVo> list2 = new ArrayList<>();
        ExcelContentVo field21 = new ExcelContentVo("项目2名称"+number);
        ExcelContentVo field22 = new ExcelContentVo("项目2进度"+number);
        ExcelContentVo field23 = new ExcelContentVo("项目2报价"+number);
        ExcelContentVo field24 = new ExcelContentVo("项目2资源（人/天）"+number);
        list2.add(field21);
        list2.add(field22);
        list2.add(field23);
        list2.add(field24);
        list.add(list1);
        list.add(list2);
        List<ExcelContentVo> list3 = new ArrayList<>();
        ExcelContentVo field31 = new ExcelContentVo("项目3名称"+number);
        ExcelContentVo field32 = new ExcelContentVo("项目3进度"+number);
        ExcelContentVo field33 = new ExcelContentVo("项目3报价"+number);
        ExcelContentVo field34 = new ExcelContentVo("项目3资源（人/天）"+number);
        list3.add(field31);
        list3.add(field32);
        list3.add(field33);
        list3.add(field34);
        list.add(list3);
        List<ExcelContentVo> list4 = new ArrayList<>();
        ExcelContentVo field41 = new ExcelContentVo("项目4名称"+number);
        ExcelContentVo field42 = new ExcelContentVo("项目4进度"+number);
        ExcelContentVo field43 = new ExcelContentVo("项目4报价"+number);
        ExcelContentVo field44 = new ExcelContentVo("项目4资源（人/天）"+number);
        list4.add(field41);
        list4.add(field42);
        list4.add(field43);
        list4.add(field44);
        if(number<3){
            list.add(list4);
        }
        return new ExcelContentVo(list);
    }

    public ExcelContentVo contentVotest(){
        String filePath = "/var/tmp/pdf_dir/复合表头+内容合并excel测试.xlsx";
        String sheetname = "复合表头";
        ExcelTitleVo titleVo1 = new ExcelTitleVo("需求单号", new ArrayList<>());
        ExcelTitleVo titleVo2 = new ExcelTitleVo("需求名称", new ArrayList<>());
        ExcelTitleVo titleVo3 = new ExcelTitleVo("报价单号", new ArrayList<>());
        ExcelTitleVo titleVo4 = new ExcelTitleVo("供应商", new ArrayList<>());
        ExcelTitleVo titleVo5 = new ExcelTitleVo("报价金额", new ArrayList<>());
        ExcelTitleVo titleVo7 = new ExcelTitleVo("报价时间", new ArrayList<>());



        List<ExcelTitleVo> title6Children = new ArrayList<>();
        ExcelTitleVo title6Child1 = new ExcelTitleVo("项目名称", new ArrayList<>());
        ExcelTitleVo title6Child2 = new ExcelTitleVo("项目进度", new ArrayList<>());
        ExcelTitleVo title6Child3 = new ExcelTitleVo("项目报价", new ArrayList<>());
        ExcelTitleVo title6Child4 = new ExcelTitleVo("项目资源（人/天)", new ArrayList<>());

        title6Children.add(title6Child1);
        title6Children.add(title6Child2);
        title6Children.add(title6Child3);
        title6Children.add(title6Child4);


        ExcelTitleVo titleVo6 = new ExcelTitleVo("报价明细", title6Children);
        ExcelTitleVo titleVo8 = new ExcelTitleVo("验收明细", title6Children);


        List<ExcelTitleVo> title = new ArrayList<>();
        title.add(titleVo1);
        title.add(titleVo2);
        title.add(titleVo3);
        title.add(titleVo4);
        title.add(titleVo5);
        title.add(titleVo6);
        title.add(titleVo7);
        title.add(titleVo8);

        List<List<ExcelContentVo>> list = new ArrayList<>();
        List<ExcelContentVo> contentVo1 = createMainTestVo(1);
        List<ExcelContentVo> contentVo2 = createMainTestVo(2);
        list.add(contentVo1);
        list.add(contentVo2);

        ExcelContentVo contentVo = new ExcelContentVo(list);
        ExcelCreator excelCreator = new ExcelCreator();
//        excelCreator.rebuildContentVo(contentVo,0,0);
        excelCreator.createExcleFile(filePath, sheetname, title, contentVo);
        return contentVo;
    }
````

按分页获取数据并写入的例子：

````java
        List<ExcelTitleVo> titleVoList = getTitleVoList();
        String fileName = "报价工单列表_" + String.valueOf(System.currentTimeMillis()) + ".xlsx";
        String filePath = HecFbiArtBussinessConstant.TEMP_DOWNLOAD_FILE_DIR + "/" + fileName;
        String sheetname = "报价工单列表";

        //新建文档实例
        SXSSFWorkbook workbook = new SXSSFWorkbook(100);
        //在文档中添加表单
        SXSSFSheet sheet = workbook.createSheet(sheetname);

        ExcelCreator excelCreator = new ExcelCreator();
        Integer beginRowNumber = excelCreator.writeCompositeTitle(workbook, sheet, titleVoList);
        Integer totalPage = 1;
        Long pageIndex = 1L;
        formData.setLimit(500L);
        while (pageIndex <= totalPage) {
            formData.setPage(pageIndex);
            PageUtils page = allPage(formData);
            List<PriceQuotationWorkOrderVO> voList = (List<PriceQuotationWorkOrderVO>) page.getList();
            if (voList != null && voList.size() > 0) {
                ExcelContentVo contentVo = getContentVo(voList);
                excelCreator.writeCompositeContent(workbook, sheet, contentVo, beginRowNumber);
                beginRowNumber = contentVo.getEndRowNumber() + 1;
            }
            totalPage = page.getTotalPage();
            pageIndex++;
        }

        try (FileOutputStream fileOut = new FileOutputStream(filePath)) {
            workbook.write(fileOut);
        } catch (FileNotFoundException e) {
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
````