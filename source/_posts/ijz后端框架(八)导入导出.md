---
title: ijz后端框架(八)导入导出
date: 2019-02-16 16:51:00
categories: ijz后端框架
tags:
	- poi
	- jxls
---
导入导出可说是表单必备功能，目前系统中已有好几个工具类，但这些工具类考虑的都不够全面（尤其是在导出方面），它们或者不能满足于某些特定场景，或者不具备可扩展性，又或者实现得不够优雅。
结合我在实际开发中的经验，我认为导入应该满足以下需求：
* 值转换
* 必填校验
* 并行
* 批量

导出相对复杂一些，需要满足以下更多需求：
* 值转换
* 特殊值标识
* 批量
* 主子表
* 海量数据
* 并行
* 分页查询
* 打包
* 复杂表头
* 自定义样式
* 基于模板

为此我开发了一套类库：[ijz-export](https://git.yonyouccs.com/ijz-modules-jars/ijz-export)，它基本实现了以上各类需求。下面将依次介绍如何利用该库来做导入/导出。

## 导入
### 简单的
```java
List<People> dataList = ExcelImportUtils.imports(new FileInputStream("E:\\人员表.xlsx"),
        new ImportDefinition.Builder<People>()
                // .sheetName("人员表") // 未指定时默认读取第一个sheet
                // .startIndex(3) // 未指定时默认从第一行开始读取
                .modelClass(People.class) // 接收导入数据的类
                .simpleColumns(Arrays.asList("name", "birthday", "sex", "isSingle", "isMarried", "ratio")) // excel中每列依次对应的属性名
                .build());
```
最简单的导入只需要设置modelClass和simpleColumns。如果需要读取指定sheet，则调用sheetName；如果需要从指定行开始读取数据，则调用startIndex。
### 值转换
场景：People类中有int类型字段sex（1表示男，2表示女），但在excel中的sex对应列的值就是字符串“男”、“女”，那么如何自动给sex字段赋相应的值？
```java
List<People> dataList = ExcelImportUtils.imports(new FileInputStream("E:\\人员表.xlsx"),
        new ImportDefinition.Builder<People>()
                .modelClass(People.class)
                .columns(Arrays.asList(
                        new ColumnDefinition.Builder().name("name").build(), // 不需要转换
                        new ColumnDefinition.Builder().name("birthday").valueFunction(new DateImportFunction()).build(), // 调用预置日期导入转换类，将字符串转换成日期（默认yyyy-mm-dd格式）
                        new ColumnDefinition.Builder().name("sex").valueFunction((value) -> value.equals("男") ? 1 : 2).build(), // 自定义转换
                        new ColumnDefinition.Builder().name("isSingle").valueFunction(new BooleanIntegerImportFunction()).build(), // 调用预置布尔导入转换类，将“是”、“否”分别转换成1、2
                        new ColumnDefinition.Builder().name("isMarried").valueFunction(new BooleanImportFunction()).build(),  // 调用预置布尔导入转换类，将“是”、“否”分别转换成true、false
                        new ColumnDefinition.Builder().name("ratio").valueFunction(new PercentageImportFunction()).build() // 调用预置百分比导入转换类，将百分数转换成小数
                ))
                .build());
```
目前预置了DateImportFunction、BooleanImportFunction、BooleanIntegerImportFunction和PercentageImportFunction四个值导入转换类，对应的也预置了DateExportFunction、BooleanExportFunction、BooleanIntegerExportFunction和PercentageExportFunction这四个值导出转换类。
### 必填校验
```java
List<People> dataList = ExcelImportUtils.imports(new FileInputStream("E:\\人员表.xlsx"),
        new ImportDefinition.Builder<People>()
                .modelClass(People.class)
                .columns(Arrays.asList(
                        new ColumnDefinition.Builder().name("name").required(true).build(),
                        new ColumnDefinition.Builder().name("birthday").required(true).build(),
                        new ColumnDefinition.Builder().name("sex").required(true).build(),
                        new ColumnDefinition.Builder().name("isSingle").build(),
                        new ColumnDefinition.Builder().name("isMarried").required(false).build(),
                        new ColumnDefinition.Builder().name("ratio").required(false).build()
                ))
                .build());
```
默认是非必填，设置必填后如果excel中有值为空，就会抛出ImportException，并提示哪行哪列值设置了必填却为空。
### 并行
```java
List<People> dataList = ExcelImportUtils.imports(new FileInputStream("E:\\人员表.xlsx"),
       new ImportDefinition.Builder<People>()
                .modelClass(People.class)
                .simpleColumns(Arrays.asList("name", "birthday", "sex", "isSingle", "isMarried"))
               .parallel(true)
                // .threadNumber(50) // 未指定时线程数为可用处理器数量
               .build());
   ```
默认是串行导入，如果设置为并行导入，建议线程数根据实际情况设置大些（导入是io密集型任务）。
### 批量
```java
List<List<?>> dataList = ExcelImportUtils.batchImports(new FileInputStream("E:\\人员表.xlsx"),
        Arrays.asList(
                new ImportDefinition.Builder<People>()
                        // .sheetName("人员表1")
                        .modelClass(People.class)
                        .simpleColumns(Arrays.asList("name", "birthday", "sex", "isSingle", "isMarried"))
                        .build(),
                new ImportDefinition.Builder<People>()
                        // .sheetName("人员表2")
                        .modelClass(People.class)
                        .simpleColumns(Arrays.asList("name", "birthday", "sex", "isSingle", "isMarried"))
                        .build()
        ));
```
未指定sheetName时，将按顺序读取sheet数据。

看到这里，相信大家也发现了，要定制各种导入，重点在于如何构建ImportDefinition，而通过调用ImportDefinition.Builder类，可以很方面的构建ImportDefinition。不止是ImportDefinition，类库中各类Definition都有对应的Builder。（对，正是运用了建造者设计模式。）

## 导出
### 简单的
```java
ExcelExportUtils.export(new FileOutputStream("E:\\人员表.xlsx"),
        buildData(100), // 构建要导出数据列表
        new ExportDefinition.Builder()
                // .sheetName("人员表") // 未指定时默认为sheet1
                // .simpleTitle("人员表") // 未指定时没有标题
                // .title(new TitleDefinition.Builder().name("人员表").height(2).build()) // 自定义标题高度
                // .simpleLabels(Arrays.asList("姓名", "出生年月", "性别", "是否单身", "是否已婚", "百分比")) // 未指定时没有labels
                .simpleProperties(Arrays.asList("name", "birthday", "sex", "isSingle", "isMarried", "ratio"))
                .build());
```
最简单的导出只需设置simpleProperties。标题默认高度为一行，如果要指定标题高度，就调用title方法。如果要导出labels，注意labels要与properties一致。
### 值转换
```java
ExcelExportUtils.export(new FileOutputStream("E:\\人员表.xlsx"),
        buildData(100),
        new ExportDefinition.Builder()
                .properties(Arrays.asList(
                        new PropertyDefinition.Builder().name("name").build(), // 不需要转换
                        new PropertyDefinition.Builder().name("birthday").valueFunction(new DateExportFunction()).build(), // 调用预置日期导出转换类，将日期转换成字符串（默认yyyy-mm-dd格式）
                        new PropertyDefinition.Builder().name("sex").valueFunction((value) -> (int) value == 1 ? "男" : "女").build(), // 自定义转换
                        new PropertyDefinition.Builder().name("isSingle").valueFunction(new BooleanIntegerExportFunction()).build(), // 调用预置布尔导出转换类，将1、2分别转换成“是”、“否”
                        new PropertyDefinition.Builder().name("isMarried").valueFunction(new BooleanExportFunction()).build(), // 调用预置布尔导入转换类，将true、false分别转换成“是”、“否”
                        new PropertyDefinition.Builder().name("ratio").valueFunction(new PercentageExportFunction()).build()// 调用预置百分比导入转换类，将小数转换成百分数
                ))
                .build());
```
### 特殊值标识
场景：将人员数据中已经结婚的用红色字体突出显示。
```java
ExcelExportUtils.export(new FileOutputStream("E:\\人员表.xlsx"),
        buildData(100),
        new ExportDefinition.Builder()
                .properties(Arrays.asList(
                        new PropertyDefinition.Builder().name("name").build(),
                        new PropertyDefinition.Builder().name("birthday").build(),
                        new PropertyDefinition.Builder().name("sex").build(),
                        new PropertyDefinition.Builder().name("isSingle").build(),
                        new PropertyDefinition.Builder().name("isMarried").valueFunction(new BooleanExportFunction())
                                .styleFunction((wb, value) -> {
                                    if ((boolean) value) {
                                        CellStyle style = wb.createCellStyle();
                                        Font dataFont = wb.createFont();
                                        dataFont.setColor(HSSFColor.HSSFColorPredefined.RED.getIndex());
                                        style.setFont(dataFont);
                                        return style;
                                    }
                                    return new DefaultStyleFunctionFactory().getDataStyleFunction().apply(wb);
                                })
                                .build(),
                        new PropertyDefinition.Builder().name("ratio").valueFunction(new PercentageExportFunction()).build()
                ))
                .build());
```
### 批量
```java
ExcelExportUtils.batchExport(new FileOutputStream("E:\\人员表.xlsx"),
        Arrays.asList(
                buildData(100),
                buildData(100)
        ),
        Arrays.asList(
                new ExportDefinition.Builder()
                        .simpleProperties(Arrays.asList("name", "birthday", "sex", "isSingle", "isMarried", "ratio"))
                        .build(),
                new ExportDefinition.Builder()
                        .simpleProperties(Arrays.asList("name", "birthday", "sex", "isSingle", "isMarried", "ratio"))
                        .build()
        ));
```
### 主子表
首先要确定主子表导出到excel中的展现形式，我的想法是做成类似EasyUI中SubGrid这样的：
![主子表导出示例](/images/ijz/主子表导出示例.png)
由于实现起来较为复杂，且目前未遇到实际需求，所以暂时没做。后面如需要导出主子表，可以转变为批量导出。
### 海量数据
```java
ExcelExportUtils.export(new FileOutputStream("E:\\人员表.xlsx"),
        buildData(2000000),
        new ExportDefinition.Builder()
                .simpleProperties(Arrays.asList("name", "birthday", "sex", "isSingle", "isMarried", "ratio"))
                .windowSize(100) // 内存中数据量达到100，就刷新到磁盘，默认100
                .compress(false) // 临时文件非常大时，可以考虑压缩临时文件，但会引起性能损失，默认不压缩
                .build());
```
使用poi导出海量数据，需要注意两点：
* 使用SXSSFWorkbook
* 每张sheet中最多只能导出1048576行数据，数据量超出该范围时，将数据导出到多个sheet里

以上两点在内部实现中已经考虑到了,另外根据实际情况，可以设置内存中数据量达到多少时就刷新到磁盘、以及是否需要压缩临时文件。
### 并行
上面说过，导出使用了SXSSFWorkbook，因此当内存中数据达到一定数量时，这些数据就会刷新到磁盘，当后面的数据刷新到磁盘，就不能回头去写前面的数据了，这样就不能使用多线程来同时往多个行写数据。当然可以设置不刷新数据到磁盘，但这样又会大大降低导出性能。因此目前未实现并行导出。
### 分页查询
场景：要导出1000w条数据，如果一次性都查出来放内存中，很可能会导出内存溢出，此时就需要考虑分页查询，数据一部分一部分的写到磁盘。
```java
ExcelExportUtils.export(new FileOutputStream("E:\\人员表.xlsx"),
        new QueryModel(
                new PageQuery(
                        0, // 起始id值
                        "id", // id字段名
                        1000, // 每次查询数量
                        null // 查询条件
                ), // 分页查询对象
                pageQuery -> {
                    /*
                        方法体内代码是模拟数据库查询
                    */
                    List<People> dataList = buildData(1000);
                    dataList.get(dataList.size() - 1).setId(pageQuery.getId() + pageQuery.getPageSize());

                    if (pageQuery.getId() == 2000000) {
                        dataList.remove(dataList.size() - 1);
                        dataList.get(dataList.size() - 1).setId(pageQuery.getId() + pageQuery.getPageSize() - 1);
                    }
                    return dataList;
                } // 分页查询函数
        ), // 查询模型对象
        new ExportDefinition.Builder()
                .simpleProperties(Arrays.asList("name", "birthday", "sex", "isSingle", "isMarried", "ratio"))
                .build());
```
此时就不是传递dataList而是改为传递QueryModel了。PageQuery对象里之所以是id而不是pageIndex，是因为海量数据分页查询时，主键应设置为long且保持自增，不然查询会有性能问题，且可能会重复查询数据。
### 打包
场景：当sheet中数据量超过100w时，打开excel是很慢的，可以考虑把数据分批导出到多个excel，最后压缩成zip导出。
```java
ExcelExportUtils.packageExport(new FileOutputStream("E:\\人员表.zip"),
        buildData(1000000),
        new ExportDefinition.Builder()
                .simpleProperties(Arrays.asList("name", "birthday", "sex", "isSingle", "isMarried", "ratio"))
                .packageSize(200000) // 默认每个excel的最大存储数据量为200000
                .build());
```
### 复杂表头
场景：很多时候我们导出的表头是这种简单的：
![简单表头示例](/images/ijz/简单表头示例.png)
但也会遇到需要合并单元格等复杂需求，比如这样：
![复杂表头示例](/images/ijz/复杂表头示例.png)
```java
ExcelExportUtils.export(new FileOutputStream("E:\\人员表.xlsx"), 
        buildData(100),
        new ExportDefinition.Builder()
                .labels(Arrays.asList(
                        new LabelDefinition.Builder().name("a").children(
                                Arrays.asList(
                                        new LabelDefinition.Builder().name("b").children(
                                                Arrays.asList(
                                                        new LabelDefinition.Builder().name("姓名").build(),
                                                        new LabelDefinition.Builder().name("出生年月").build()
                                                )
                                        ).build(),
                                        new LabelDefinition.Builder().name("性别").build()
                                )
                        ).build(),
                        new LabelDefinition.Builder().name("c").children(
                                Arrays.asList(
                                        new LabelDefinition.Builder().name("是否单身").build(),
                                        new LabelDefinition.Builder().name("是否已婚").build(),
                                        new LabelDefinition.Builder().name("百分比").build()
                                )
                        ).build()
                ))
                .simpleProperties(Arrays.asList("name", "birthday", "sex", "isSingle", "isMarried", "ratio"))
                .build());
```
唯一需要注意的是，最底层的labels应与properties一一对应。
### 自定义样式
```java
ExcelExportUtils.export(new FileOutputStream("E:\\人员表.xlsx"),
        buildData(100),
        new ExportDefinition.Builder()
                .simpleProperties(Arrays.asList("name", "birthday", "sex", "isSingle", "isMarried", "ratio"))
                .styleFunctionFactory(new DefaultStyleFunctionFactory())
                .build());
```
DefaultStyleFunctionFactory实现了StyleFunctionFactory接口，类结构如下：
![主题样式类结构](/images/ijz/export主题样式类结构.png)
利用工厂模式，可以很方面扩展主题样式。
### 基于模板
场景：实际业务中，可能需要导出各种复杂的excel，当封装的库不能实现时，就可以考虑使用[jxls](http://jxls.sourceforge.net/getting_started.html)基于模板来导出excel。
```java
ExcelExportUtils.exportByTemplate(
        new FileOutputStream("E:\\template.xlsx"), 
        "template.xlsx",  // 模板地址
        null // 数据
);
```
jxls的缺点在于，当数据量很大时性能会有问题，且每处导出都需要创建模板略显麻烦。