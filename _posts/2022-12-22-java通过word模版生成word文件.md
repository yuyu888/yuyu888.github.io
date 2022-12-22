---
layout: mypost
title: java 通过word模版生成word文件
categories: [JAVA]
---

## 前言

主要是通过试事先准备好的模板，然后动态的把数据渲染进去的；

这里主要介绍了普通数据填充，图片渲染， 表格循环等方式， 足够满足大多数需求

主要使用了 poi-tl 官网：http://deepoove.com/poi-tl/

## Maven

````
		<dependency>
			<groupId>com.deepoove</groupId>
			<artifactId>poi-tl</artifactId>
			<version>1.10.0</version>
		</dependency>

````

## word 模版

![图例](wordtestTpl.jpg)



## 渲染结果

![图例](guangtouqiangjianli.jpg)



## java 代码


````JAVA
   public void createByTpl(){
        try {
            // 模板文件地址
            String filePath = "/Users/xxx/Documents/doc/wordtest.docx"; // src/main/resources/wordtpl/wordtest.docx
            // 读取模板后保存生成word的地址
            String outPath = "/Users/xxx/Documents/doc/guangtouqiangjianli.docx";

            LoopRowTableRenderPolicy policy = new LoopRowTableRenderPolicy();
            LoopRowTableRenderPolicy policy2 = new LoopRowTableRenderPolicy();

            Configure configure = Configure.builder().bind("children", policy).bind("eduData", policy2).build();

            // 读取模板、数据并渲染
            XWPFTemplate template = XWPFTemplate.compile(new File(filePath), configure).render(
                    // 创建一个map并设置为只读map
                    Collections.unmodifiableMap(new HashMap<String, Object>() {
                        {
                            // 设置普通文本的值
                            put("name", "光头强");
                            put("gender", "男");
                            // 设置照片的值
                            FileInputStream inputStream = new FileInputStream(new File("/Users/xxx/Documents/yu.jpg"));
                            PictureRenderData pictureRenderData = new PictureRenderData(250, 200, PictureType.suggestFileType(".png"), inputStream);
                            put("photo", pictureRenderData);
                            inputStream.close();

                            List<ChildDto> children = new ArrayList<>();
                            ChildDto childVo1 = new ChildDto();
                            childVo1.setName("大毛");
                            childVo1.setGender("男");
                            childVo1.setBirthday("2020-12-21");
                            childVo1.setAge("2");
                            children.add(childVo1);

                            ChildDto childVo2 = new ChildDto();
                            childVo2.setName("二毛");
                            childVo2.setGender("男");
                            childVo2.setBirthday("2021-12-21");
                            childVo2.setAge("1");
                            children.add(childVo2);

                            ChildDto childVo3 = new ChildDto();
                            childVo3.setName("三毛");
                            childVo3.setGender("男");
                            childVo3.setBirthday("2022-12-21");
                            childVo3.setAge("一个月");
                            children.add(childVo3);

                            put("children", children);

                            List<EduDto> eduData = new ArrayList<>();
                            EduDto eduVo1 = new EduDto();
                            eduVo1.setPeriod("2002.9-2006.6");
                            eduVo1.setSchool("家里蹲林业大学");
                            eduVo1.setDegree("本科");
                            eduVo1.setMajor("伐木专业");
                            eduData.add(eduVo1);

                            EduDto eduVo2 = new EduDto();
                            eduVo2.setPeriod("2006.9-2008.6");
                            eduVo2.setSchool("家里蹲林业大学");
                            eduVo2.setDegree("硕士");
                            eduVo2.setMajor("猎熊专业");
                            eduData.add(eduVo2);

                            put("eduData", eduData);

                            List<InterestDto> interestDtos = new ArrayList<>();
                            InterestDto interestDto = new InterestDto();
                            interestDto.setItem("足球");
                            interestDtos.add(interestDto);
                            InterestDto interestDto2 = new InterestDto();
                            interestDto2.setItem("唱歌");
                            interestDtos.add(interestDto2);
                            InterestDto interestDto3 = new InterestDto();
                            interestDto3.setItem("旅游");
                            interestDtos.add(interestDto3);

                            put("interest", interestDtos);

                        }
                    })
            );
            // 文件是否已存在，则删除
            File file = new File(outPath);
            if (file.exists()){
                file.delete();
            }
            // 生成word保存在指定目录
            template.writeToFile(outPath);
            template.close();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
````


