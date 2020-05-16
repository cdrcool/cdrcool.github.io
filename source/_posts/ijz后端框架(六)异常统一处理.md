---
title: ijz后端框架(六)异常统一处理
date: 2019-02-17 20:37:16
categories: ijz后端框架
tags:
	- RestControllerAdvice
	- ExceptionHandler
---
在文章[返回值类型统一处理中](https://cdrcool.github.io/2019/02/22/ijz后端框架%28五%29返回值类型统一处理/)，我介绍了怎样利用aop统一包装正常返回的值，其实异常情况也可以利用aop来实现。
不过Spring为异常统一处理提供了好几种其它的方式，我是使用注解@RestControllerAdvice和@ExceptionHandler来实现的，代码如下：
```java
/**
 * 异常统一处理类
 * 异常也统一包装成UnifyResult对象
 */
@RestControllerAdvice
public class ExceptionAdvice {
    private static final Logger LOGGER = LoggerFactory.getLogger(ExceptionAdvice.class);

    /**
     * 处理spring校验异常
     */
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<UnifyResult> handleException(MethodArgumentNotValidException e) {
        return this.handle(new ValidateException(e));
    }

    /**
     * 处理其他异常
     */
    @ExceptionHandler(Exception.class)
    public ResponseEntity<UnifyResult> handleException(Exception e) {
        return this.handle(e);
    }

    private ResponseEntity<UnifyResult> handle(Exception e) {
        String message = StringUtils.isEmpty(e.getMessage()) ? "系统异常，请联系管理员！" : e.getMessage();
        LOGGER.error(message, e);

        UnifyResult backData = new UnifyResult.Builder().success(false).backMsg(message).build();

        HttpHeaders headers = new HttpHeaders();
        headers.set("Content-Type", "application/json;charset=UTF-8");

        return new ResponseEntity<>(backData, headers, HttpStatus.OK);
    }
}
```
嗯，就这么简单，一个类就完成了异常统一处理。上面对Spring校验异常做了额外处理，是为了返回给前端更友好的提示。

另外关于异常处理，我认为项目中这地方可以调整下：
* service层的好多接口都有显示抛出BusinessException（继承自RuntimeException），这个是没必要的。就像我们知道系统随处可能抛出RuntimeException一样，service层中也随处可能抛出BusinessException，但系统可没哪处有显示抛出RuntimeException，因为这是默认的。
* 很多dubbo service接口也显示抛出了BusinessException（继承Exception），比如CompanyService，但这里的BusinessException是checked exception。我的建议是一般不要使用这类异常，如果一定要使用它，请明确告知开发者当捕获到该类异常时要如何处理（具体的处理方式，不是捕获完又抛出去）。另外要注意当遇到checked exception时，spring事务默认是不会回滚的。
* 目前我们抛出异常，有些地方是直接抛出，有些地方是调用ExceptionUtils类，直接抛出即可。