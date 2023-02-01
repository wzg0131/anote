# 制作开关注解@EnableXXX

[toc]

@Enable** 注解，一般用于开启某一类功能。类似于一种开关，只有加了这个注解，才能使用某些功能。例如Spring Boot中的`@EnableScheduling`+`@Scheduled`，`@EnableCaching`+`@Cache`，`@EnableAsync`+`@Async` 。



## 自定义处理http请求日志的开关注解

通过过滤器来记录http请求日志

### 定义日志过滤器

```java
public abstract class CltHttpContentLogFilter extends OncePerRequestFilter {

    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain) throws ServletException, IOException {
        ContentCachingRequestWrapper requestWrapper = new ContentCachingRequestWrapper(request);
        ContentCachingResponseWrapper responseWrapper = new ContentCachingResponseWrapper(response);
        HttpUtil.markRequestWithUUID();
        try {
            filterChain.doFilter(requestWrapper, responseWrapper);
        } finally {
            dealContent(requestWrapper, responseWrapper);
            // Do not forget this line after reading response content or actual response will be empty!
            responseWrapper.copyBodyToResponse();
        }
    }

    /**
     * 自定义处理request和response
     * @param requestWrapper 请求
     * @param responseWrapper 响应
     */
    protected abstract void dealContent(ContentCachingRequestWrapper requestWrapper, ContentCachingResponseWrapper responseWrapper);

}

@Slf4j
public class DefaultHttpContentLogFilter extends CltHttpContentLogFilter {
    @Override
    protected void dealContent(ContentCachingRequestWrapper requestWrapper, ContentCachingResponseWrapper responseWrapper) {
        String requestBody = new String(requestWrapper.getContentAsByteArray(), StandardCharsets.UTF_8);
        log.info("Request body: {}", requestBody);
        String responseBody = new String(responseWrapper.getContentAsByteArray(), StandardCharsets.UTF_8);
        log.info("Response body: {}", responseBody);
    }
}
```



### 开关注解

定义一个开启http请求日志的开关注解：

```java
import org.springframework.context.annotation.Import;

@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.TYPE})
@Documented
@Import({HttpRequestLogFilterImportSelector.class})
public @interface EnableHttpRequestLog {
    Class<? extends CltHttpContentLogFilter> logFilterClass() default CltHttpContentLogFilter.class;
}
```

关键是使用了Spring的`@Import`注解，这个注解的作用是**导入一些特定的配置类，由配置类注入Bean到容器**，配置类有三种：

- `@Configuration`注解的配置类
- 实现`ImportSelector`接口的类
- 实现`ImportBeanDefinitionRegistrar`接口的类

这里采用第二种方式，`@Import({HttpRequestLogFilterImportSelector.class})`

### 实现`ImportSelector`，注入过滤器

```java
@Slf4j
public class HttpRequestLogFilterImportSelector implements ImportSelector {

    private final static String LOG_FILTER_CLASS = "logFilterClass";
    @Override
    public String[] selectImports(AnnotationMetadata annotationMetadata) {

        //获取注解参数
        AnnotationAttributes attributes = AnnotationAttributes.fromMap(annotationMetadata.getAnnotationAttributes(EnableHttpRequestLog.class.getName(), true));

        assert attributes != null;
        String filterName = attributes.getString(LOG_FILTER_CLASS);

        if (StringUtils.isEmpty(filterName)) {
            return new String[] {DefaultHttpContentLogFilter.class.getName()};
        }

        String typeName = null;
        try {
            typeName = Class.forName(filterName).getGenericSuperclass().getTypeName();
        } catch (ClassNotFoundException e) {
            return new String[] {DefaultHttpContentLogFilter.class.getName()};
        }
        log.info("CLT: Filter [{}] imported!", filterName);
        return new String[] {filterName};
    }
}
```

### 使用注解

在Spring Boot启动类使用：

```java
@SpringBootApplication
@EnableHttpRequestLog(logFilterClass = MyHttpContentLogFilter.class)
public class CcloudLogProducerDemoApplication {

    public static void main(String[] args) {
        SpringApplication.run(CcloudLogProducerDemoApplication.class, args);
    }

}
```

### 参考

https://www.liaoxuefeng.com/wiki/1252599548343744/1265102803921888

https://juejin.cn/post/6844903839506628621



## @ConditionalOn条件注入

以**Feign调用令牌中继**作为例子：

### 定义web请求拦截器

当服务被调用时处理：

```java
/**
 * 用于构建调用上下文的拦截器，提取上游服务传递的信息
 */
@Slf4j
public class InvocationContextExtractInterceptor implements WebRequestInterceptor {

    @Override
    public void preHandle(@NonNull WebRequest request) {
        final ServletWebRequest servletRequest = (ServletWebRequest) request;
        GenericInvocationContext context = new GenericInvocationContext();

        String requestUserEncoded = servletRequest.getHeader(CustomHttpHeaders.REQUEST_USER);
        if (StringUtils.hasText(requestUserEncoded)) {
            String json = new String(Base64Utils.decodeFromString(requestUserEncoded));
            context.setRequestUser(json);
        }

        context.setBatchProcessId(servletRequest.getHeader(CustomHttpHeaders.PROCESS_ID));
        context.setTransactionId(servletRequest.getHeader(CustomHttpHeaders.TRANSACTION_ID));
        context.setTraceId(servletRequest.getHeader(CustomHttpHeaders.TRACE_ID));

        InvocationContextHolder.setContext(context);
    }

    @Override
    public void postHandle(@NonNull WebRequest request, ModelMap model) {

    }

    @Override
    public void afterCompletion(@NonNull WebRequest request, Exception ex) {

    }
}

```

### 定义feign拦截器

发起feign调用时拦截：

```java
/**
 * 发起Feign请求拦截器，将调用信息传递给下游
 * @author Nan
 */
@Slf4j
public class FeignInvocationInterceptor implements RequestInterceptor {

    private final static String KEY_TRACE_ID = "traceId";

    @Override
    public void apply(RequestTemplate requestTemplate) {
        final ServletRequestAttributes attributes = (ServletRequestAttributes) RequestContextHolder.getRequestAttributes();
        if (Objects.isNull(attributes)) {
            return;
        }
        final GenericInvocationContext invocationContext = InvocationContextHolder.getContext();

        final String traceId = Optional.ofNullable(invocationContext.getTraceId())
                .orElseGet(this::extractTraceIdByMDC);
        if (StringUtils.hasText(traceId)) {
            requestTemplate.header(TRACE_ID, traceId);
        }

        if (invocationContext.isUserRequest()) {
            requestTemplate.header(REQUEST_USER, Base64Utils.encodeToString(
                    invocationContext.getRequestUserAsJson().getBytes(StandardCharsets.UTF_8)));
        }

        if (invocationContext.isBatchProcessInvocation()) {
            requestTemplate.header(PROCESS_ID, invocationContext.getBatchProcessId());
        }

        final String transactionId = invocationContext.getTransactionId();
        if (StringUtils.hasText(transactionId)) {
            requestTemplate.header(TRANSACTION_ID, invocationContext.getTransactionId());
        }
    }

    /**
     * 获取sleuth的TraceId
     *
     * @return traceID
     */
    private String extractTraceIdByMDC() {
        MDCAdapter mdc = MDC.getMDCAdapter();
        Map<String, String> map = ((LogbackMDCAdapter) mdc).getPropertyMap();
        return map.get(KEY_TRACE_ID);
    }
}
```

### 配置类

只有在Servlet的Web应用环境下才启用这个配置类：

```java
@Configuration
@ConditionalOnWebApplication(type = ConditionalOnWebApplication.Type.SERVLET)
@ConditionalOnDiscoveryEnabled
public class WebMvcConfiguration implements WebMvcConfigurer {
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addWebRequestInterceptor(new InvocationContextExtractInterceptor());
    }
    @Bean
    public FeignInvocationInterceptor feignInterceptor() {
        return new FeignInvocationInterceptor();
    }
}

```

## 排除注入Bean

通过启动类`SpringBootApplication`注解的`exclude`属性：

```java
@SpringBootApplication(exclude = {RedisAutoConfiguration.class, RedisRepositoriesAutoConfiguration.class})
public class Application {
    public static void main(String[] args) {
       //..
    }
}
```

