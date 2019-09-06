---
layout:     post
title:      springboot静态资源映射
subtitle:   springboot
date:       2019-01-25
author:     skaleto
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - springboot


---



我们都知道，在springboot中，可以通过配置静态资源映射，达到自定义资源映射路径，示例如下：

```java
@Configuration
public class GoWebMvcConfigurerAdapter extends WebMvcConfigurerAdapter {

    @Override
    public void addResourceHandlers(ResourceHandlerRegistry registry) {
        //配置静态资源处理
        registry.addResourceHandler("/**")
                .addResourceLocations("classpath:/resources2/", "classpath:/static2/", 
                "classpath:/public2/", "file:///tmp/webapps/");
    }
}
```

或者也可以在springboot的配置文件中配置

```properties
#配置资源映射源路径
spring.mvc.static-path-pattern=/**
#配置资源映射目标路径
spring.resources.static-locations=classpath:/META-INF/resources/,classpath:/resources/,\classpath:/static/,classpath:/public/,file:D:
```



但是，上面这些属性配置之后，springboot到底是怎么实现当你请求某个url的时候，把对应真实文件返回给你的呢？

首先找到我们上面配置路径时继承的WebMvcConfigurerAdapter，我们跟进去看，它是一个抽象类，继承它的除了我们手写的配置类，还有一个有名的WebMvcAutoConfigurationAdapter，这个后面详细分析。



静态资源在请求的时候，肯定是通过某个接口进来的，于是我们去看DispatcherServlet，在初始化的时候做了些什么。DispatcherServlet继承自FrameworkServlet，FrameworkServlet继承自HttpServletBean，笔者一开始以为DispatcherServlet是在服务启动的时候就初始化好的，然而在服务启动的时候在其中打的断点并没有经过，在尝试发起请求后，才发现是在第一次请求的时候初始化的，看下面的调用栈

```java
//initStrategies中初始化了一堆servlet映射策略
initStrategies:488, DispatcherServlet (org.springframework.web.servlet)
onRefresh:480, DispatcherServlet (org.springframework.web.servlet)
//初始化WebApplicationContext
initWebApplicationContext:560, FrameworkServlet (org.springframework.web.servlet)
initServletBean:494, FrameworkServlet (org.springframework.web.servlet)
init:171, HttpServletBean (org.springframework.web.servlet)
init:158, GenericServlet (javax.servlet)
//此处代码见下方
initServlet:1183, StandardWrapper (org.apache.catalina.core)
//tomcat的servlet分配策略，在这个地方需要为请求分配一个servlet去执行
allocate:795, StandardWrapper (org.apache.catalina.core)
invoke:133, StandardWrapperValve (org.apache.catalina.core)
invoke:96, StandardContextValve (org.apache.catalina.core)
invoke:478, AuthenticatorBase (org.apache.catalina.authenticator)
invoke:140, StandardHostValve (org.apache.catalina.core)
invoke:81, ErrorReportValve (org.apache.catalina.valves)
invoke:87, StandardEngineValve (org.apache.catalina.core)
service:342, CoyoteAdapter (org.apache.catalina.connector)
service:803, Http11Processor (org.apache.coyote.http11)
process:66, AbstractProcessorLight (org.apache.coyote)
process:868, AbstractProtocol$ConnectionHandler (org.apache.coyote)
doRun:1459, NioEndpoint$SocketProcessor (org.apache.tomcat.util.net)
run:49, SocketProcessorBase (org.apache.tomcat.util.net)
runWorker:1142, ThreadPoolExecutor (java.util.concurrent)
run:617, ThreadPoolExecutor$Worker (java.util.concurrent)
run:61, TaskThread$WrappingRunnable (org.apache.tomcat.util.threads)
run:745, Thread (java.lang)
```

```java
@Override
public Servlet allocate() throws ServletException {
    ...
            if (instance == null || !instanceInitialized) {
                synchronized (this) {
                    if (instance == null) {
                        try {
                            if (log.isDebugEnabled()) {
                                log.debug("Allocating non-STM instance");
                            }
                            instance = loadServlet();
                            newInstance = true;
                            if (!singleThreadModel) {
                                countAllocated.incrementAndGet();
                            }
                        } catch (ServletException e) {
                        } catch (Throwable e) {
                        }
                    }
                    if (!instanceInitialized) {
                  //第一次请求进来的时候，instance是没有被实例化的，所以这里会在第一次请求的时候被调用
                        initServlet(instance);
                    }
                }
            }
    ...
}
```

上面是题外话，解释了ServletDispatcher是在什么时候被初始化的，我们再回到ServletDispatcher#initStrategies这个方法

```java
protected void initStrategies(ApplicationContext context) {
    //
    initMultipartResolver(context);
    //
	initLocaleResolver(context);   
    initThemeResolver(context);   
    initHandlerMappings(context);   
    initHandlerAdapters(context);   
    initHandlerExceptionResolvers(context);   
    initRequestToViewNameTranslator(context);   
    initViewResolvers(context);   
    initFlashMapManager(context);
}
```

#### MultipartResolver 

MultipartResolver 用于处理文件上传，当收到请求时 DispatcherServlet 的 checkMultipart() 方法会调用 MultipartResolver 的 isMultipart() 方法判断请求中是否包含文件。如果请求数据中包含文件，则调用 MultipartResolver 的 resolveMultipart() 方法对请求的数据进行解析，然后将文件数据解析成 MultipartFile 并封装在 MultipartHttpServletRequest (继承了 HttpServletRequest) 对象中，最后传递给 Controller

#### HandlerMappings

我们来看initHandlerMappings的实现

```java
private void initHandlerMappings(ApplicationContext context) {   
    this.handlerMappings = null;   
    if (this.detectAllHandlerMappings) {      // Find all HandlerMappings in the ApplicationContext, including ancestor contexts.      
        Map<String, HandlerMapping> matchingBeans =            BeanFactoryUtils.beansOfTypeIncludingAncestors(context, HandlerMapping.class, true, false);      
        if (!matchingBeans.isEmpty()) {         
            this.handlerMappings = new ArrayList<HandlerMapping>(matchingBeans.values()); 
            //将handler以@Order和@Priority注解进行排序
            AnnotationAwareOrderComparator.sort(this.handlerMappings);      
        }   
    }else {      
        try {         
            HandlerMapping hm = context.getBean(HANDLER_MAPPING_BEAN_NAME, HandlerMapping.class);         
            this.handlerMappings = Collections.singletonList(hm);      
        }catch (NoSuchBeanDefinitionException ex) {   
        }   
    }
    if (this.handlerMappings == null) {      
        this.handlerMappings = getDefaultStrategies(context, HandlerMapping.class);      
        ...
    }
}
```

上面的代码中，调试的过程中可以看到matchingBeans中有requestMappingHandlerMapping，viewControllerHandlerMapping，beanNameHandlerMapping，resourceHandlerMapping等等，其中resourceHandlerMapping就是针对静态资源映射的处理类，在WebMvcConfigurationSupport中作为一个bean被注入

```java
@Bean
public HandlerMapping resourceHandlerMapping() {   
    Assert.state(this.applicationContext != null, "No ApplicationContext set");   
    Assert.state(this.servletContext != null, "No ServletContext set");   
    ResourceHandlerRegistry registry = new ResourceHandlerRegistry(this.applicationContext,this.servletContext, mvcContentNegotiationManager(), mvcUrlPathHelper());   
    addResourceHandlers(registry);   
    AbstractHandlerMapping handlerMapping = registry.getHandlerMapping();   
    if (handlerMapping != null) {      
        handlerMapping.setPathMatcher(mvcPathMatcher());      
        handlerMapping.setUrlPathHelper(mvcUrlPathHelper());      
        handlerMapping.setInterceptors(new ResourceUrlProviderExposingInterceptor(mvcResourceUrlProvider()));      
        handlerMapping.setCorsConfigurations(getCorsConfigurations());   
    }else {      
        handlerMapping = new EmptyHandlerMapping();   
    }   
    return handlerMapping;
}
```

```java
private final WebMvcConfigurerComposite configurers = new WebMvcConfigurerComposite();

@Autowired(required = false)
public void setConfigurers(List<WebMvcConfigurer> configurers) {
	if (!CollectionUtils.isEmpty(configurers)) {
		this.configurers.addWebMvcConfigurers(configurers);
	}
}

@Override
protected void addResourceHandlers(ResourceHandlerRegistry registry) {
	this.configurers.addResourceHandlers(registry);
}

public void addResourceHandlers(ResourceHandlerRegistry registry) {   
    for (WebMvcConfigurer delegate : this.delegates) {      
        delegate.addResourceHandlers(registry);   
    }
}
```

上面的delegate是对实现了WebMvcConfigurer的类的代理，有两个，一个就是我们自己实现的WebMvcConfig，另一个是

```java
@Override
public void addResourceHandlers(ResourceHandlerRegistry registry) {
	if (!this.resourceProperties.isAddMappings()) {
		logger.debug("Default resource handling disabled");
		return;
	}
	Integer cachePeriod = this.resourceProperties.getCachePeriod();
    //这个地方会把/webjars/下面的资源请求映射到classpath下
	if (!registry.hasMappingForPattern("/webjars/**")) {
		customizeResourceHandlerRegistration(
				registry.addResourceHandler("/webjars/**")
						.addResourceLocations(
								"classpath:/META-INF/resources/webjars/")
				.setCachePeriod(cachePeriod));
	}
	String staticPathPattern = this.mvcProperties.getStaticPathPattern();
    //这个地方会把/下面的资源请求映射为如下几个地方
    //(1)/
    //(2)classpath:/META-INF/resources/
    //(3)classpath:/resources/
    //(4)classpath:/static/
    //(5)classpath:/public/
	if (!registry.hasMappingForPattern(staticPathPattern)) {
        //staticPathPattern为/**
		customizeResourceHandlerRegistration(
				registry.addResourceHandler(staticPathPattern)
						.addResourceLocations(
								this.resourceProperties.getStaticLocations())
						.setCachePeriod(cachePeriod));
	}
}
```

再来看我们自定义的WebMvcConfig，正是重写了addResourceHandlers，将我们需要的静态资源映射添加到了handlermapping中

```java
@Configuration
public class WebMvcConfig extends WebMvcConfigurerAdapter {

    @Autowired
    private ServiceConfig serviceConfig;

    @Bean
    public BaseInterceptor baseInterceptor() {
        return new BaseInterceptor();
    }

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(baseInterceptor()).addPathPatterns("/**");
    }

    /**
     * 配置静态资源映射路径
     *
     * @param registry
     */
    @Override
    public void addResourceHandlers(ResourceHandlerRegistry registry) {
        registry.addResourceHandler(serviceConfig.getStaticResourceUrl())
                .addResourceLocations(serviceConfig.getResourceMapping());
    }

}
```



以上我们了解了静态资源映射配置的基本步骤，那么，当一个请求过来的时候，究竟内部做了什么，能够让请求获得对应的静态资源呢？

我们再回到上面的DispatchServlet，经过调试，我们再请求一个文件的时候，会进入DispatchServlet#doDispatch方法，

```java
protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
	HttpServletRequest processedRequest = request;
	HandlerExecutionChain mappedHandler = null;
	boolean multipartRequestParsed = false;

	WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);

	try {
		ModelAndView mv = null;
		Exception dispatchException = null;

		try {
			processedRequest = checkMultipart(request);
			multipartRequestParsed = (processedRequest != request);
			// 在这一步里面，经过调试可以看到，针对静态资源文件的请求，被映射为了ResourceHttpRequestHandler
			mappedHandler = getHandler(processedRequest);
			if (mappedHandler == null || mappedHandler.getHandler() == null) {
				noHandlerFound(processedRequest, response);
				return;
			}

			// 这里获得的adapter为HttpRequestHandlerAdapter
			HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());

			// 普通资源文件请求一般都是GET
			String method = request.getMethod();
			boolean isGet = "GET".equals(method);
			if (isGet || "HEAD".equals(method)) {
                //这个地方会去检查请求资源的修改时间
				long lastModified = ha.getLastModified(request, mappedHandler.getHandler());
                //假如请求的资源文件修改时间没有变化，那么这次请求返回304，告诉浏览器不需要重复请求，使用它的缓存就可以了
				if (new ServletWebRequest(request, response).checkNotModified(lastModified) && isGet) {
					return;
				}
			}

			if (!mappedHandler.applyPreHandle(processedRequest, response)) {
				return;
			}

			// 这个地方触发了真正的资源文件请求的处理类
			mv = ha.handle(processedRequest, response, mappedHandler.getHandler());

			if (asyncManager.isConcurrentHandlingStarted()) {
				return;
			}
			applyDefaultViewName(processedRequest, mv);
			mappedHandler.applyPostHandle(processedRequest, response, mv);
		}
		...
	}
}
```

来看看实际的handler里面做了什么

```java
@Override
public ModelAndView handle(HttpServletRequest request, HttpServletResponse response, Object handler)throws Exception {
		((HttpRequestHandler) handler).handleRequest(request, response);
	return null;
}

@Override
public void handleRequest(HttpServletRequest request, HttpServletResponse response)throws ServletException, IOException {

		// For very general mappings (e.g. "/") we need to check 404 first
    	// 首先第一步，获得resource对象，将我们请求中的路径拼接处理为实际映射的路径
    	// 例如我们配置了请求路径为/abc/**,映射路径为file:D:/test
    	// 假设请求的uri为/abc/1.txt,那么最终映射路径为file:D:/test/1.txt
		Resource resource = getResource(request);
		if (resource == null) {
			logger.trace("No matching resource found - returning 404");
			response.sendError(HttpServletResponse.SC_NOT_FOUND);
			return;
		}

		if (HttpMethod.OPTIONS.matches(request.getMethod())) {
			response.setHeader("Allow", getAllowHeader());
			return;
		}

		// 检查请求方式和session要求
		checkRequest(request);

		// 为啥这里又来一次文件是否修改的校验呢？不是特别理解
		if (new ServletWebRequest(request, response).checkNotModified(resource.lastModified())) {
			logger.trace("Resource not modified - returning 304");
			return;
		}

		// Apply cache settings, if any
		prepareResponse(response);

		// 检查请求资源的文件类型，以这里的txt为例，会被解析成text/plain，浏览器会直接展示
    	// 如果这里是一个zip文件，则会被解析成application/zip，浏览器无法直接展示，会通过触发下载的方式得到文件
		MediaType mediaType = getMediaType(request, resource);
    	...
            
		// Content phase
		if (METHOD_HEAD.equals(request.getMethod())) {
			setHeaders(response, resource, mediaType);
			logger.trace("HEAD request - skipping content");
			return;
		}

		ServletServerHttpResponse outputMessage = new ServletServerHttpResponse(response);
		if (request.getHeader(HttpHeaders.RANGE) == null) {
			setHeaders(response, resource, mediaType);
            //这个地方就将文件写入了httpresponse中
			this.resourceHttpMessageConverter.write(resource, mediaType, outputMessage);
		}
		else {
			response.setHeader(HttpHeaders.ACCEPT_RANGES, "bytes");
			ServletServerHttpRequest inputMessage = new ServletServerHttpRequest(request);
			try {
				List<HttpRange> httpRanges = inputMessage.getHeaders().getRange();
				response.setStatus(HttpServletResponse.SC_PARTIAL_CONTENT);
				if (httpRanges.size() == 1) {
					ResourceRegion resourceRegion = httpRanges.get(0).toResourceRegion(resource);
					this.resourceRegionHttpMessageConverter.write(resourceRegion, mediaType, outputMessage);
				}
				else {
					this.resourceRegionHttpMessageConverter.write(
							HttpRange.toResourceRegions(httpRanges, resource), mediaType, outputMessage);
				}
			}
			catch (IllegalArgumentException ex) {}
		}
	}
```

至此，通过请求方式获得静态资源的大致流程基本就差不多了

