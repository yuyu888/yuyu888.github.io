---
layout: mypost
title: java生成excel，可以自定义复合表头
categories: [JAVA]
---

话不多说， 直接上代码

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