---
title: ijz后端框架(十一)API文档
date: 2019-02-18 21:44:52
categories: ijz后端框架
tags:
	- Swagger
---
[Swagger](https://swagger.io/)是一个规范和完整的框架，用于生成、描述、调用和可视化RESTful风格的Web服务。
利用[SpringFox](https://springfox.github.io/springfox/)我们可以很快的将Swagger集成到Spring Boot项目中：
1. 添加依赖
```xml
<!-- api doc -->
<dependency>
    <groupId>io.springfox</groupId>
    <artifactId>springfox-swagger2</artifactId>
</dependency>
<dependency>
    <groupId>io.springfox</groupId>
    <artifactId>springfox-swagger-ui</artifactId>
</dependency>
```
2. 在Spring Boot应用启动类上添加注解@@EnableSwagger2
```java
@EnableSwagger2
@SpringBootApplication
public class CostApplication extends SpringBootServletInitializer {
    ...
}
```
3. Controller类上添加相关文档注解
```java
@Api(description = "其他费用")
@UnifyReturn
@RequestMapping("otherCost")
@RestController
public class OtherCostController {
    @Autowired
    private OtherCostService service;

    @ApiOperation(httpMethod = "POST", value = "保存其他费用")
    @PostMapping("save")
    public OtherCostVO save(@Valid @RequestBody OtherCostVO vo) {
        return StringUtils.isEmpty(vo.getId()) ? service.create(vo) : service.update(vo);
    }

    @ApiOperation(httpMethod = "POST", value = "删除其他费用")
    @PostMapping("delete")
    public void delete(@RequestBody List<OtherCostVO> voList) {
        service.delete(voList);
    }

    @ApiOperation(httpMethod = "GET", value = "查找其他费用")
    @GetMapping("queryDetail")
    public OtherCostVO findOne(String id) {
        return service.findById(id);
    }

    @ApiOperation(httpMethod = "POST", value = "分页查询其他费用")
    @PostMapping("queryList")
    public Page<OtherCostVO> queryList(@RequestBody QueryCondition condition) {
        return service.queryForPage(condition.getSearchText(), condition.getFilter(), condition.getPageable());
    }

    @ApiOperation(httpMethod = "GET", value = "生成单据编号")
    @GetMapping("generateCode")
    public String generateBillCode() {
        return service.generateBillCode();
    }

    @ApiOperation(httpMethod = "GET", value = "获取成本费用类别档案")
    @RequestMapping(value = "getCostTypes")
    public List<DefdocAPIBO> getCostTypes() {
        return service.getCostTypes();
    }

    @UnifyReturn(unify = false)
    @ApiOperation(httpMethod = "GET", value = "导出其他费用")
    @PostMapping("/export")
    public void exportExcel(HttpServletResponse response, @RequestBody ExportParams<OtherCostVO> params) {
        List<OtherCostVO> dataList;
        if (ExportDataScope.ALL.name().equalsIgnoreCase(params.getScope())) {
            QueryCondition condition = params.getCondition();
            Page<OtherCostVO> page = service.queryForPage(condition.getSearchText(), condition.getFilter(),
                    PageRequest.of(0, 10000, condition.getSort()));
            dataList = page.getContent();
        } else {
            dataList = params.getData();
        }
        ExportUtils.exports(response, params.getFileName(), dataList, new ExportDefinitionBuilder<OtherCostVO>()
                .model(OtherCostVO.class)
                .simpleLabels(Arrays.asList("单据编号", "项目名称", "登记月份", "创建人", "合计金额"))
                .simpleProperties(Arrays.asList("billCode", "projectName", "period", "creator", "totalAmount"))
                .build());
    }

    @UnifyReturn(unify = false)
    @ApiOperation(httpMethod = "GET", value = "打印其他费用")
    @PostMapping("print")
    public JSONObject queryList(String id) {
        return PrintUtil.convertPrintData(service.findById(id), new PrintAttributeConvert() {
            @Override
            public void convert(JSONObject jsonObject) {
            }
        });
    }
}
```
4. 在vo类上添加相关注解
```javas
@ApiModel(description = "其他费用")
public class OtherCostVO extends BillVO {
    @ApiModelProperty("项目id")
    @NotNull(message = "项目不能为空")
    private String projectId; // 项目id
    @ApiModelProperty("项目id")
    private String projectName; // 项目名称
    @ApiModelProperty("登记月份id")
    @NotNull(message = "登记月份不能为空")
    private String periodId; // 登记月份
    @ApiModelProperty("登记月份")
    private String period; // 登记月份
    @NotNull(message = "合计金额不能为空")
    @ApiModelProperty("合计金额")
    private BigDecimal totalAmount; // 合计金额
    @ApiModelProperty("费用明细")
    @SublistNotEmpty(message = "费用明细不能为空")
    @Valid
    private List<OtherCostDetailVO> costDetails; // 费用明细
    ...
}
```
大功告成！启动项目后，访问地址http://{ip}:{port}/{projectname}/swagger-ui.html即可看到自动生成的API文档。
下面以公有云成本)示例：
![API文档示例-公有云成本](/images/API文档示例-公有云成本.png)
大家可以访问地址：[https://cc.yonyouccs.com/ijz-cost-web/swagger-ui.html](https://cc.yonyouccs.com/ijz-cost-web/swagger-ui.html)查看详细。