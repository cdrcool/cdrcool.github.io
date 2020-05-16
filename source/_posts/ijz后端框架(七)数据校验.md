---
title: ijz后端框架(七)数据校验
date: 2019-02-17 22:19:16
categories: ijz后端框架
tags:
	- ConstraintValidator
---
像必填、邮箱、最大值、不能重复、正则表达式等表单校验，我们前端都有做，其实后端也应该做的。利用@Valid等注解很容易实现，比如下面这样：
```java
@PostMapping("save")
public OtherCostVO save(@Valid @RequestBody OtherCostVO vo) {
    return StringUtils.isEmpty(vo.getId()) ? service.create(vo) : service.update(vo);
}

public class OtherCostVO extends BillVO {
    @NotNull(message = "项目不能为空")
    private String projectId; // 项目id
    private String projectName; // 项目名称
    @NotNull(message = "登记月份不能为空")
    private String periodId; // 登记月份
    private String period; // 登记月份
    @NotNull(message = "合计金额不能为空")
    private BigDecimal totalAmount; // 合计金额
    @SublistNotEmpty(message = "费用明细不能为空")
    // @NotEmpty
    @Valid
    private List<OtherCostDetailVO> costDetails; // 费用明细
    
    ...
}
```
对于基本校验，spring都提供了相应注解，但有些特殊情况则需要我们自定义注解。比如我们系统中都是逻辑删除，这时我们如果要做子表非空校验，就不能再使用注解@NotEmpty了。那么如何创建自定义注解呢？
首先，创建注解：
```java
/**
 * 子表非空校验注解
 */
@Constraint(validatedBy = SublistNotEmptyValidator.class)
@Target({METHOD, FIELD, ANNOTATION_TYPE, CONSTRUCTOR, PARAMETER})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface SublistNotEmpty {

    String message() default "{org.hibernate.validator.constraints.NotEmpty.message}";

    Class<?>[] groups() default {};

    Class<? extends Payload>[] payload() default {};
}
```
然后，创建注解解析器：
```java
/**
 * 子表非空校验器
 * 业务中采用逻辑删除，所以非空校验需要判断是否全部逻辑删除
 */
public class SublistNotEmptyValidator implements ConstraintValidator<SublistNotEmpty, List<? extends LogicDeleteAware>> {

    @Override
    public void initialize(SublistNotEmpty constraintAnnotation) {

    }

    @Override
    public boolean isValid(List<? extends LogicDeleteAware> value, ConstraintValidatorContext context) {
        return !CollectionUtils.isEmpty(value) && !value.stream().allMatch(LogicDeleteAware::isDeleted);
    }
}
```
最后，只需要在要校验的字段上添加该注解就可以了，Spring会帮我们搞定的！（最上面的字段costDetails正是用到了该注解）
目前，除了子表非空@SublistNotEmpty注解，我还提供了子表不能重复注解@SublistNotRepeated。