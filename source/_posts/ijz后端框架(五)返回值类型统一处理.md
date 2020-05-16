---
title: ijz后端框架(五)返回值类型统一处理
date: 2019-02-17 21:50:17
categories: ijz后端框架
tags:
	- aop
---
由于后端要求接口返回值类型是JsonBackData，因此目前在每个controller中的方法里，我们都是按下面这种方式手动包装返回值：
```java
@RequestMapping(value = "xx")
@ResponseBody
public JsonBackData xx(@RequestBody XX vo) {
    JsonBackData back = new JsonBackData();
    try {
        back.setBackData(service.xx(vo));
        back.setBackMsg("xx成功");
    } catch (BusinessException e) {
        back.setSuccess(false);
        back.setBackMsg("xx失败:"+e.getMessage());
    }
    return back;
}
```
这样有两个弊端，一是重复编码，二是从自动生成的API文档中，我们不知道每个方法的返回值类型具体是什么。

方法执行分两种情况：正常返回和抛出异常。异常的统一处理我会在文章[异常统一处理](https://cdrcool.github.io/2019/02/22/ijz后端框架%28六%29异常统一处理/)中再做介绍，下面只讲述如何利用aop来实现返回值类型统一。
先来看看添加aop之后，我们再怎么写controller：
```java
@UnifyReturn
@RequestMapping("otherCost")
@RestController
public class OtherCostController {
    @Autowired
    private OtherCostService service;

    @PostMapping("save")
    public OtherCostVO save(@Valid @RequestBody OtherCostVO vo) {
        return StringUtils.isEmpty(vo.getId()) ? service.create(vo) : service.update(vo);
    }

    @PostMapping("delete")
    public void delete(@RequestBody List<OtherCostVO> voList) {
        service.delete(voList);
    }

    @GetMapping("queryDetail")
    public OtherCostVO findOne(String id) {
        return service.findById(id);
    }
    
    @UnifyReturn(unify = false)
    @PostMapping("print")
    public JSONObject print(String id) {
        return PrintUtil.convertPrintData(service.findById(id), new PrintAttributeConvert() {
            @Override
            public void convert(JSONObject jsonObject) {
            }
        });
    }
}
```
如上，如果想要对某个controller中所有方法的返回值类型都做统一处理，只需在该controller类上添加注解@UnifyReturn；如果其中某个方法就要返回原始类型（比如上面的print方法），那么就在该方法上也添加注解@UnifyReturn，并覆盖注解的unify属性为false即可。

切面代码如下（UnifyResult就是JsonBackData，只是改了下类名）：
```java
/**
 * 返回值类型统一切面
 */
@Component
@Aspect
public class UnifyResultAspect {
    private MappingJackson2HttpMessageConverter converter;

    public UnifyResultAspect(MappingJackson2HttpMessageConverter converter) {
        this.converter = converter;
    }

    /**
     * 切入点定义
     *
     * @param unifyReturn {@link UnifyReturn}
     */
    @Pointcut(value = "@within(unifyReturn)", argNames = "unifyReturn")
    public void pointcut(UnifyReturn unifyReturn) {
    }

    /**
     * 后置通知
     *
     * @param joinPoint   链接点
     * @param unifyReturn {@link UnifyReturn}
     * @param result      返回值
     * @throws IOException io异常
     */
    @AfterReturning(pointcut = "pointcut(unifyReturn)", returning = "result", argNames = "joinPoint,unifyReturn,result")
    public void unifyResult(JoinPoint joinPoint, UnifyReturn unifyReturn, Object result) throws IOException {
        if (unifyReturn.unify()) {
            // 如果方法上也添加了该注解，则覆盖类上的定义
            Method method = ((MethodSignature) (joinPoint.getSignature())).getMethod();
            UnifyReturn methodUnifyReturn = method.getAnnotation(UnifyReturn.class);
            if (methodUnifyReturn == null || methodUnifyReturn.unify()) {
                String message = this.getMessage(method);

                UnifyResult backData = new UnifyResult.Builder()
                        .backData(result)
                        .backMsg(message)
                        .build();

                // aop不能改变返回值类型
                HttpServletResponse response = ((ServletRequestAttributes) RequestContextHolder.getRequestAttributes()).getResponse();
                HttpOutputMessage outputMessage = new ServletServerHttpResponse(response);
                converter.write(backData, MediaType.APPLICATION_JSON, outputMessage);
                response.getOutputStream().close();
            }
        }
    }

    /**
     * 返回提示消息
     *
     * @param method 执行的方法
     * @return 提示消息
     */
    private String getMessage(Method method) {
        String message = null;
        if (method.isAnnotationPresent(RequestMapping.class)) {
            message = method.getAnnotation(RequestMapping.class).name();
        } else if (method.isAnnotationPresent(GetMapping.class)) {
            message = method.getAnnotation(GetMapping.class).name();
        } else if (method.isAnnotationPresent(PostMapping.class)) {
            message = method.getAnnotation(PostMapping.class).name();
        } else if (method.isAnnotationPresent(PutMapping.class)) {
            message = method.getAnnotation(PutMapping.class).name();
        } else if (method.isAnnotationPresent(DeleteMapping.class)) {
            message = method.getAnnotation(DeleteMapping.class).name();
        } else if (method.isAnnotationPresent(PatchMapping.class)) {
            message = method.getAnnotation(PatchMapping.class).name();
        }
        return StringUtils.isEmpty(message) ? "操作成功" : message + "成功";
    }
}
```