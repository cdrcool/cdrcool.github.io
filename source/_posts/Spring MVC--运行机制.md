---
title: Spring MVC--运行机制
date: 2019-09-16 10:51:42
categories: Spring MVC
---
## 整体流程
官方请求流程示意图：
![Spring MVC请求处理流程示意图](/images/spring-framework/Spring MVC请求处理流程示意图.png)

上图描述比较简单，详细描述如下：

1. 用户发送请求，请求到达SpringMVC的前端控制器（DispatcherServlet）
2. 根据请求的url，从请求处理器映射器(HandlerMapping)列表中查找与之匹配的handler，并将其与拦截器（HandlerInterceptor）列表一起包装为HandlerExecutionChain返回
3. 根据前面的handler，从请求处理器映射适配器（HandlerAdapter）列表中查找与之适配的handler adapter
4. 遍历HandlerExecutionChain中的拦截器（HandlerInterceptor）列表，依次调用拦截器的前置处理方法（preHandle）
5. 执行请求处理器映射适配器（HandlerAdapter）中的处理方法（handle），返回ModelAndView
6. 遍历HandlerExecutionChain中的拦截器（HandlerInterceptor）列表，依次调用拦截器的后置处理方法（postHandle）
7. 从视图解析器（ViewResolver）列表中找到合适的view resolver，解析前面的ModelAndView
8. 最后将返回的视图进行渲染并把数据装入到request域，返回给用户

流程图可表示为：
![Spring MVC请求处理流程](/images/spring-framework/Spring MVC请求处理流程.png)

源码如下：
```java
/**
 * Process the actual dispatching to the handler.
 * <p>The handler will be obtained by applying the servlet's HandlerMappings in order.
 * The HandlerAdapter will be obtained by querying the servlet's installed HandlerAdapters
 * to find the first that supports the handler class.
 * <p>All HTTP methods are handled by this method. It's up to HandlerAdapters or handlers
 * themselves to decide which methods are acceptable.
 * @param request current HTTP request
 * @param response current HTTP response
 * @throws Exception in case of any kind of processing failure
 */
protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
    HttpServletRequest processedRequest = request;
    HandlerExecutionChain mappedHandler = null;
    boolean multipartRequestParsed = false;

    WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);

    try {
        ModelAndView mv = null;
        Exception dispatchException = null;

        try {
            // 处理附件：将请求转换为多部分请求，并使多部分解析器可用。如果没有设置多部分解析器，只需使用现有的请求
            processedRequest = checkMultipart(request);
            multipartRequestParsed = (processedRequest != request);

            // Determine handler for the current request.
            // 根据请求的url，获取与之匹配的handler，并将其与拦截器（HandlerInterceptor）列表一起包装为HandlerExecutionChain返回
            // 这里的handler可能是HandlerMethod，可能是Controller，也可能是HttpRequestHandler或Servlet 对象
            mappedHandler = getHandler(processedRequest);
            if (mappedHandler == null) {
                noHandlerFound(processedRequest, response);
                return;
            }

            // Determine handler adapter for the current request.
            // 将不同类型的handler统一转换成HandlerAdapter，以便统一调用
            HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());

            // Process last-modified header, if supported by the handler.
            String method = request.getMethod();
            boolean isGet = "GET".equals(method);
            if (isGet || "HEAD".equals(method)) {
                long lastModified = ha.getLastModified(request, mappedHandler.getHandler());
                if (new ServletWebRequest(request, response).checkNotModified(lastModified) && isGet) {
                    return;
                }
            }

            // 遍历拦截器（HandlerInterceptor）列表，依次调用拦截器的前置处理方法（preHandle）
            // 如果其中有拦截器返回false，表示该拦截器已经处理了响应本身，这里就直接返回
            if (!mappedHandler.applyPreHandle(processedRequest, response)) {
                return;
            }

            // Actually invoke the handler.
            // 处理请求，返回ModelAndView
            mv = ha.handle(processedRequest, response, mappedHandler.getHandler());

            if (asyncManager.isConcurrentHandlingStarted()) {
                return;
            }

            // 视情况设置view name
            applyDefaultViewName(processedRequest, mv);
            // 遍历拦截器（HandlerInterceptor）列表，依次调用拦截器的后置处理方法（postHandle）
            // 如果使用了@ResponseBody或ResponseEntity，则不建议使用postHandle，而是使用ResponseBodyAdvice，并将其声明为控制器通知bean，或者直接在RequestMappingHandlerAdapter上配置它
            mappedHandler.applyPostHandle(processedRequest, response, mv);
        }
        catch (Exception ex) {
            dispatchException = ex;
        }
        catch (Throwable err) {
            // As of 4.3, we're processing Errors thrown from handler methods as well,
            // making them available for @ExceptionHandler methods and other scenarios.
            dispatchException = new NestedServletException("Handler dispatch failed", err);
        }
        // 处理调用结果，处理异常、国际化、视图解析等
        processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException);
    }
    catch (Exception ex) {
        triggerAfterCompletion(processedRequest, response, mappedHandler, ex);
    }
    catch (Throwable err) {
        // 请求处理完成后回调，即呈现视图后回调
        // 遍历拦截器（HandlerInterceptor）列表，依次调用拦截器的完成后处理方法（afterCompletion）
        triggerAfterCompletion(processedRequest, response, mappedHandler,
                new NestedServletException("Handler processing failed", err));
    }
    finally {
        if (asyncManager.isConcurrentHandlingStarted()) {
            // Instead of postHandle and afterCompletion
            if (mappedHandler != null) {
                mappedHandler.applyAfterConcurrentHandlingStarted(processedRequest, response);
            }
        }
        else {
            // Clean up any resources used by a multipart request.
            // 清理多部分请求相关资源
            if (multipartRequestParsed) {
                cleanupMultipart(processedRequest);
            }
        }
    }
}
```

## DispatcherServlet
UML类图如下：
![DispatcherServlet UML](/images/spring-framework/DispatcherServlet UML.png)

由类图可知，`DispatcherServlet`继承自类`FrameworkServlet`，而`FrameworkServlet`又实现了接口`ApplicationContextAware`，这样`FrameworkServlet`就能拿到`ApplicationContext`来完成一些初始化工作。

```java
public abstract class FrameworkServlet extends HttpServletBean implements ApplicationContextAware {
    ...
    
    /**
     * Called by Spring via {@link ApplicationContextAware} to inject the current
     * application context. This method allows FrameworkServlets to be registered as
     * Spring beans inside an existing {@link WebApplicationContext} rather than
     * {@link #findWebApplicationContext() finding} a
     * {@link org.springframework.web.context.ContextLoaderListener bootstrapped} context.
     * <p>Primarily added to support use in embedded servlet containers.
     * @since 4.0
     */
    @Override
    public void setApplicationContext(ApplicationContext applicationContext) {
        if (this.webApplicationContext == null && applicationContext instanceof WebApplicationContext) {
            this.webApplicationContext = (WebApplicationContext) applicationContext;
            this.webApplicationContextInjected = true;
        }
    }
    
    
    /**
     * Overridden method of {@link HttpServletBean}, invoked after any bean properties
     * have been set. Creates this servlet's WebApplicationContext.
     */
    @Override
    protected final void initServletBean() throws ServletException {
        getServletContext().log("Initializing Spring " + getClass().getSimpleName() + " '" + getServletName() + "'");
        if (logger.isInfoEnabled()) {
            logger.info("Initializing Servlet '" + getServletName() + "'");
        }
        long startTime = System.currentTimeMillis();
    
        try {
            this.webApplicationContext = initWebApplicationContext();
            initFrameworkServlet();
        }
        catch (ServletException | RuntimeException ex) {
            logger.error("Context initialization failed", ex);
            throw ex;
        }
    
        if (logger.isDebugEnabled()) {
            String value = this.enableLoggingRequestDetails ?
                    "shown which may lead to unsafe logging of potentially sensitive data" :
                    "masked to prevent unsafe logging of potentially sensitive data";
            logger.debug("enableLoggingRequestDetails='" + this.enableLoggingRequestDetails +
                    "': request parameters and headers will be " + value);
        }
    
        if (logger.isInfoEnabled()) {
            logger.info("Completed initialization in " + (System.currentTimeMillis() - startTime) + " ms");
        }
    }
    
    /**
     * Initialize and publish the WebApplicationContext for this servlet.
     * <p>Delegates to {@link #createWebApplicationContext} for actual creation
     * of the context. Can be overridden in subclasses.
     * @return the WebApplicationContext instance
     * @see #FrameworkServlet(WebApplicationContext)
     * @see #setContextClass
     * @see #setContextConfigLocation
     */
    protected WebApplicationContext initWebApplicationContext() {
        WebApplicationContext rootContext =
                WebApplicationContextUtils.getWebApplicationContext(getServletContext());
        WebApplicationContext wac = null;
    
        if (this.webApplicationContext != null) {
            // A context instance was injected at construction time -> use it
            wac = this.webApplicationContext;
            if (wac instanceof ConfigurableWebApplicationContext) {
                ConfigurableWebApplicationContext cwac = (ConfigurableWebApplicationContext) wac;
                if (!cwac.isActive()) {
                    // The context has not yet been refreshed -> provide services such as
                    // setting the parent context, setting the application context id, etc
                    if (cwac.getParent() == null) {
                        // The context instance was injected without an explicit parent -> set
                        // the root application context (if any; may be null) as the parent
                        cwac.setParent(rootContext);
                    }
                    configureAndRefreshWebApplicationContext(cwac);
                }
            }
        }
        if (wac == null) {
            // No context instance was injected at construction time -> see if one
            // has been registered in the servlet context. If one exists, it is assumed
            // that the parent context (if any) has already been set and that the
            // user has performed any initialization such as setting the context id
            wac = findWebApplicationContext();
        }
        if (wac == null) {
            // No context instance is defined for this servlet -> create a local one
            wac = createWebApplicationContext(rootContext);
        }
    
        if (!this.refreshEventReceived) {
            // Either the context is not a ConfigurableApplicationContext with refresh
            // support or the context injected at construction time had already been
            // refreshed -> trigger initial onRefresh manually here.
            synchronized (this.onRefreshMonitor) {
                // 调用子类的onRefresh
                onRefresh(wac);
            }
        }
    
        if (this.publishContext) {
            // Publish the context as a servlet context attribute.
            String attrName = getServletContextAttributeName();
            getServletContext().setAttribute(attrName, wac);
        }
    
        return wac;
    }
    
    ...
}

public class DispatcherServlet extends FrameworkServlet {
    ...

    /**
	 * This implementation calls {@link #initStrategies}.
	 */
	@Override
	protected void onRefresh(ApplicationContext context) {
		initStrategies(context);
	}

	/**
	 * Initialize the strategy objects that this servlet uses.
	 * <p>May be overridden in subclasses in order to initialize further strategy objects.
	 */
	protected void initStrategies(ApplicationContext context) {
		initMultipartResolver(context);
		initLocaleResolver(context);
		initThemeResolver(context);
		initHandlerMappings(context);
		initHandlerAdapters(context);
		initHandlerExceptionResolvers(context);
		initRequestToViewNameTranslator(context);
		initViewResolvers(context);
		initFlashMapManager(context);
	}
    
    ...
}
```

另外在`FrameworkServlet`类中还定义了内部类`ContextRefreshListener`，该内部类实现了`ApplicationListener<ContextRefreshedEvent>`接口，这样当上下文发生刷新时，`FrameworkServlet`能同步刷新内部的属性。
```java
public abstract class FrameworkServlet extends HttpServletBean implements ApplicationContextAware {
    ...

    /**
     * Callback that receives refresh events from this servlet's WebApplicationContext.
     * <p>The default implementation calls {@link #onRefresh},
     * triggering a refresh of this servlet's context-dependent state.
     * @param event the incoming ApplicationContext event
     */
    public void onApplicationEvent(ContextRefreshedEvent event) {
        this.refreshEventReceived = true;
        synchronized (this.onRefreshMonitor) {
            onRefresh(event.getApplicationContext());
        }
    }

    ...

    /**
	 * ApplicationListener endpoint that receives events from this servlet's WebApplicationContext
	 * only, delegating to {@code onApplicationEvent} on the FrameworkServlet instance.
	 */
	private class ContextRefreshListener implements ApplicationListener<ContextRefreshedEvent> {

		@Override
		public void onApplicationEvent(ContextRefreshedEvent event) {
			FrameworkServlet.this.onApplicationEvent(event);
		}
	}

    ...
}
```

## HandlerMapping
![HandlerMapping](/images/spring-framework/HandlerMapping.png)

该接口只有一个方法`getHandler`，作用是根据当前请求的找到对应的Handler，并将Handler（执行程序）与一堆HandlerInterceptor（拦截器）封装到HandlerExecutionChain对象中。
这里的**Handler**可能是HandlerMethod（封装了Controller中的方法），可能是Controller，也可能是HttpRequestHandler或Servlet 对象，而这个Handler具体是什么对象，跟HandlerMapping的实现类有关。
如下图所示，可以看到`HandlerMapping`实现类有两个分支，分别继承自`AbstractHandlerMethodMapping`（得到 HandlerMethod）和`AbstractUrlHandlerMapping`（得到Controller、HttpRequestHandler或Servlet），它们又统一继承自`AbstractHandlerMapping`。

## HandlerExecutionChain
HandlerExecutionChain用于将前面的Handler与拦截器（HandlerInterceptor）列表包装起来，并提供获取handler、添加拦截器、调用拦截器前后处理方法等操作。
![HandlerExecutionChain](/images/spring-framework/HandlerExecutionChain.png)

## HandlerAdapter
由于Handler的类型有多种，因此调用方式就是不确定的，为此Spring创建了一个适配器接口（HandlerAdapter），使得每一种handler有一种对应的适配器实现类，让适配器代替handler执行相应的方法，这样在后面需要扩展Handler的类型时，只需要增加一个适配器类即可。

接口定义如下：
![HandlerAdapter](/images/spring-framework/HandlerAdapter.png)

类继承结构如下：
![HandlerAdapter UML](/images/spring-framework/HandlerAdapter UML.png)

通过观察源码可知：
* `RequestMappingHandlerAdapter`用于适配`HandlerMethod`类型的handler
* `SimpleControllerHandlerAdapter`用于适配`Controller`类型的handler
* `HttpRequestHandlerAdapter`用于适配`HttpRequestHandler`类型的handler
* `SimpleServletHandlerAdapter`用于适配`Servlet`类型的handler

## HandlerInterceptor
应用程序可以为某些处理器注册任意数量的现有或自定义拦截器，以添加公共预处理行为，而无需修改每个处理器实现。
在适当的HandlerAdapter触发处理器本身的执行之前，将调用HandlerInterceptor。这种机制可以用于预处理方面的大量领域，例如授权检查，或者常见的处理程序行为，如区域设置或主题更改。它的主要目的是允许分解出重复的处理程序代码。
HandlerInterceptor基本上类似于Servlet过滤器，但与Servlet过滤器不同的是，它只允许自定义预处理（可选则禁止执行处理器本身）和自定义后处理。过滤器更强大，例如，它们允许交换传递给链的请求和响应对象。

接口定义如下：
![HandlerInterceptor](/images/spring-framework/HandlerInterceptor.png)
* preHandle
在业务处理器处理请求之前被调用。预处理，可以进行编码、安全控制、权限校验等处理。可以选择是否终止后续处理而直接返回。
* postHandle
在业务处理器处理请求执行完成后，生成视图之前执行。后处理（调用了Service并返回ModelAndView，但未进行页面渲染），有机会修改ModelAndView。
注意，postHandle对于@ResponseBody和ResponseEntity方法不太有用，这些方法在HandlerAdapter中以及postHandle之前编写和提交响应。这意味着对响应进行任何更改都太晚了，比如添加额外的头部。对于此类场景，您可以实现ResponseBodyAdvice，并将其声明为控制器通知bean，或者直接在RequestMappingHandlerAdapter上配置它。
* afterCompletion
在DispatcherServlet完全处理完请求后被调用，可用于清理资源等。返回处理（已经渲染了页面）。

类继承结构如下：
![HandlerInterceptor UML](/images/spring-framework/HandlerInterceptor UML.png)

这里也有个适配类`HandlerInterceptorAdapter`，其作用是实现`HandlerInterceptor`并提供接口的默认实现，这样子类可以选择性覆盖相关方法。

