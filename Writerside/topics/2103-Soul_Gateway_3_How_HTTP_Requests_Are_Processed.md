---
title: Soul 网关 3：Http 请求是怎么被处理的？
date: 2021-03-06 13:43:37
updated: 
tags: 
 - Soul
 - Http
categories: Soul 学习笔记
mathjax: false
---

前文体验了使用 divide 插件转发 Http 请求，本文来分析一个 Http 网络包在 Soul 网关中都经历了什么。

## 粗略观察


用 PostMan 发送网络包，观察日志，摘要如下
```bash
2021-03-05 19:34:12.367  INFO 21748 --- [work-threads-20] o.d.soul.plugin.base.AbstractSoulPlugin  : divide selector success match , selector name :/http
2021-03-05 19:34:12.367  INFO 21748 --- [work-threads-20] o.d.soul.plugin.base.AbstractSoulPlugin  : divide rule success match , rule name :/http/order/findById
```
这些日志与 divide 插件有关，是 `AbstractSoulPlugin` 打印出来的
双击 Shift 打开 Search Everywhere 窗口搜索 `AbstractSoulPlugin` ，跳转到对应代码，点击左侧向下箭头，查看实现类，可以看到我们要找的 `DividePlugin`
![](https://ced-md-picture.oss-cn-beijing.aliyuncs.com/img/20210306134447.png)
`DividePlugin` 中有一个很大的 `doExecute` 方法，在这打个断点，使用 PostMan 发起请求，通过调用栈定位到了 `SoulWebHandler` 的 `execute()` 方法
![](https://ced-md-picture.oss-cn-beijing.aliyuncs.com/img/20210328105206.png)
大概就是遍历所有的 `plugin` 来处理 `ServerWebExchange`

```java
        /**
         * Delegate to the next {@code WebFilter} in the chain.
         *
         * @param exchange the current server exchange
         * @return {@code Mono<Void>} to indicate when request handling is complete
         */
        @Override
        public Mono<Void> execute(final ServerWebExchange exchange) {
            return Mono.defer(() -> {
                if (this.index < plugins.size()) {
                    SoulPlugin plugin = plugins.get(this.index++);
                    Boolean skip = plugin.skip(exchange);
                    if (skip) {
                        return this.execute(exchange);
                    }
                    return plugin.execute(exchange, this);
                }
                return Mono.empty();
            });
        }
    }
```
接着往上翻阅代码，发现是 `SoulWebHandler` 的 `handle()` 方法调用的上面这个 `execute()`
```java
    /**
     * Handle the web server exchange.
     *
     * @param exchange the current server exchange
     * @return {@code Mono<Void>} to indicate when request handling is complete
     */
    @Override
    public Mono<Void> handle(@NonNull final ServerWebExchange exchange) {
        return new DefaultSoulPluginChain(plugins).execute(exchange).subscribeOn(scheduler);
    }
```
在上面的函数上打断点，再次调试，进入了 `DefaultWebFilterChain` 类的 `filter()` 方法
```java
  @Override
  public Mono<Void> filter(ServerWebExchange exchange) {
    return Mono.defer(() ->
        this.currentFilter != null && this.chain != null ?
            invokeFilter(this.currentFilter, this.chain, exchange) :
            this.handler.handle(exchange));
  }
```
再次在上面方法上打断点
![](https://ced-md-picture.oss-cn-beijing.aliyuncs.com/img/20210306134621.png)
来到 `FilteringWebHandler` 类

```java
  @Override
  public Mono<Void> handle(ServerWebExchange exchange) {
    return this.chain.filter(exchange);
  }
```
来到 `WebHandlerDecorator` 类
```java
  @Override
  public Mono<Void> handle(ServerWebExchange exchange) {
    return this.delegate.handle(exchange);
  }
```
来到 `ExceptionHandlingWebHandler` 类
```java
  @Override
  public Mono<Void> handle(ServerWebExchange exchange) {
    Mono<Void> completion;
    try {
    // 当前位置
      completion = super.handle(exchange);
    }
    catch (Throwable ex) {
      completion = Mono.error(ex);
    }

    for (WebExceptionHandler handler : this.exceptionHandlers) {
      completion = completion.onErrorResume(ex -> handler.handle(exchange, ex));
    }
    return completion;
  }
```
来到 `HttpWebHandlerAdapter` 类，之前一直在传递的 `ServerWebExchange` 就是在这生成的
```java
  @Override
  public Mono<Void> handle(ServerHttpRequest request, ServerHttpResponse response) {
    if (this.forwardedHeaderTransformer != null) {
      request = this.forwardedHeaderTransformer.apply(request);
    }
    
    // 生成 exchange
    ServerWebExchange exchange = createExchange(request, response);

    LogFormatUtils.traceDebug(logger, traceOn ->
        exchange.getLogPrefix() + formatRequest(exchange.getRequest()) +
            (traceOn ? ", headers=" + formatHeaders(exchange.getRequest().getHeaders()) : ""));

    // 在这调用
    return getDelegate().handle(exchange)
        .doOnSuccess(aVoid -> logResponse(exchange))
        .onErrorResume(ex -> handleUnresolvedError(exchange, ex))
        .then(Mono.defer(response::setComplete));
  }

```
来到 `ReactiveWebServerApplicationContext`
```java
    @Override
    public Mono<Void> handle(ServerHttpRequest request, ServerHttpResponse response) {
      return this.handler.handle(request, response);
    }
```
来到 `ReactorHttpHandlerAdapter`
```java
  @Override
  public Mono<Void> apply(HttpServerRequest reactorRequest, HttpServerResponse reactorResponse) {
    NettyDataBufferFactory bufferFactory = new NettyDataBufferFactory(reactorResponse.alloc());
    try {
    // 生成了 exchange 需要的 request 和 response
      ReactorServerHttpRequest request = new ReactorServerHttpRequest(reactorRequest, bufferFactory);
      ServerHttpResponse response = new ReactorServerHttpResponse(reactorResponse, bufferFactory);

      if (request.getMethod() == HttpMethod.HEAD) {
        response = new HttpHeadResponseDecorator(response);
      }

      return this.httpHandler.handle(request, response)
          .doOnError(ex -> logger.trace(request.getLogPrefix() + "Failed to complete: " + ex.getMessage()))
          .doOnSuccess(aVoid -> logger.trace(request.getLogPrefix() + "Handling completed"));
    }
    catch (URISyntaxException ex) {
      if (logger.isDebugEnabled()) {
        logger.debug("Failed to get request URI: " + ex.getMessage());
      }
      reactorResponse.status(HttpResponseStatus.BAD_REQUEST);
      return Mono.empty();
    }
  }
```
来到 `HttpServerHandle`
```java
  @Override
  @SuppressWarnings("unchecked")
  public void onStateChange(Connection connection, State newState) {
    if (newState == HttpServerState.REQUEST_RECEIVED) {
      try {
        if (log.isDebugEnabled()) {
          log.debug(format(connection.channel(), "Handler is being applied: {}"), handler);
        }
        HttpServerOperations ops = (HttpServerOperations) connection;
        Mono.fromDirect(handler.apply(ops, ops))
            .subscribe(ops.disposeSubscriber());
      }
      catch (Throwable t) {
        log.error(format(connection.channel(), ""), t);
        connection.channel()
                  .close();
      }
    }
  }
```
来到 `TcpServerBind`
```java
    @Override
    public void onStateChange(Connection connection, State newState) {
      if (newState == State.DISCONNECTING) {
        if (connection.channel()
                      .isActive() && !connection.isPersistent()) {
          connection.dispose();
        }
      }

      childObs.onStateChange(connection, newState);

    }
  }
```
来到 `HttpServerOperations` 再然后就是 Netty 的东西了
```java
  @Override
  protected void onInboundNext(ChannelHandlerContext ctx, Object msg) {
    if (msg instanceof HttpRequest) {
      try {
        listener().onStateChange(this, HttpServerState.REQUEST_RECEIVED);
      }
      catch (Exception e) {
        onInboundError(e);
        ReferenceCountUtil.release(msg);
        return;
      }
      if (msg instanceof FullHttpRequest) {
        super.onInboundNext(ctx, msg);
        if (isHttp2()) {
          onInboundComplete();
        }
      }
      return;
    }
    if (msg instanceof HttpContent) {
      if (msg != LastHttpContent.EMPTY_LAST_CONTENT) {
        super.onInboundNext(ctx, msg);
      }
      if (msg instanceof LastHttpContent) {
        onInboundComplete();
      }
    }
    else {
      super.onInboundNext(ctx, msg);
    }
  }
```
### 小结

- HttpServerOperations : 接受 Netty 请求的地方
- TcpServerBind
- HttpServerHandle
- ReactorHttpHandlerAdapter ：生成 response 和 request
- ReactiveWebServerApplicationContext
- HttpWebHandlerAdapter ：exchange 的生成
- ExceptionHandlingWebHandler
- WebHandlerDecorator
- FilteringWebHandler
- DefaultWebFilterChain
- SoulWebHandler ：plugins 调用链
- DividePlugin ：plugin 具体处理
## 调用链
接下来看看
`SoulWebHandler` 的 plugins 调用链，在下面的函数上打断点分析 plugin 的执行情况
![](https://ced-md-picture.oss-cn-beijing.aliyuncs.com/img/20210306134700.png)
下图红色的插件被跳过了
![](https://ced-md-picture.oss-cn-beijing.aliyuncs.com/img/20210306134715.png)
下面是 `AbstractSoulPlugin` 的 `execute()`，注释位置要判断是否 enable，我们只启用了  `DividePlugin`，`WafPlugin` ，所以只有这两个插件会运行这个方法的 `if` 部分，在 `if` 中进行了一些规则的匹配。

![](https://ced-md-picture.oss-cn-beijing.aliyuncs.com/img/20210306160806.png)

![](https://ced-md-picture.oss-cn-beijing.aliyuncs.com/img/20210306134730.png)

看一下 SoulAdmin 的界面

![](https://ced-md-picture.oss-cn-beijing.aliyuncs.com/img/20210306160829.png)

每个插件界面都有「选择器列表」和「选择器规则列表」，就是在这个地方匹配的

```java
    @Override
    public Mono<Void> execute(final ServerWebExchange exchange, final SoulPluginChain chain) {
        String pluginName = named();
        final PluginData pluginData = BaseDataCache.getInstance().obtainPluginData(pluginName);
        // 此处判断插件是否启用
        if (pluginData != null && pluginData.getEnabled()) {
            final Collection<SelectorData> selectors = BaseDataCache.getInstance().obtainSelectorData(pluginName);
            if (CollectionUtils.isEmpty(selectors)) {
                return handleSelectorIsNull(pluginName, exchange, chain);
            }
            // 匹配选择器
            final SelectorData selectorData = matchSelector(exchange, selectors);
            if (Objects.isNull(selectorData)) {
                return handleSelectorIsNull(pluginName, exchange, chain);
            }
            // 打印日志
            selectorLog(selectorData, pluginName);
            final List<RuleData> rules = BaseDataCache.getInstance().obtainRuleData(selectorData.getId());
            if (CollectionUtils.isEmpty(rules)) {
                return handleRuleIsNull(pluginName, exchange, chain);
            }
            // 匹配选择器规则
            RuleData rule;
            if (selectorData.getType() == SelectorTypeEnum.FULL_FLOW.getCode()) {
                //get last
                rule = rules.get(rules.size() - 1);
            } else {
                // 匹配规则
                rule = matchRule(exchange, rules);
            }
            if (Objects.isNull(rule)) {
                return handleRuleIsNull(pluginName, exchange, chain);
            }
            // 日志
            ruleLog(rule, pluginName);
            // 进入插件的处理方法
            return doExecute(exchange, chain, selectorData, rule);
        }
        return chain.execute(exchange);
    }
```
以 divide 插件为例，上面方法会打印日志

```bash
2021-03-06 15:06:51.105  INFO 22368 --- [-work-threads-1] o.d.soul.plugin.base.AbstractSoulPlugin  : divide selector success match , selector name :/http
2021-03-06 15:07:03.683  INFO 22368 --- [-work-threads-1] o.d.soul.plugin.base.AbstractSoulPlugin  : divide rule success match , rule name :/http/order/findById
```

然后再 `WebClientPlugin` 中看到了似乎发送了网络请求

```java
    @Override
    public Mono<Void> execute(final ServerWebExchange exchange, final SoulPluginChain chain) {
        final SoulContext soulContext = exchange.getAttribute(Constants.CONTEXT);
        assert soulContext != null;
        String urlPath = exchange.getAttribute(Constants.HTTP_URL);
        if (StringUtils.isEmpty(urlPath)) {
            Object error = SoulResultWrap.error(SoulResultEnum.CANNOT_FIND_URL.getCode(), SoulResultEnum.CANNOT_FIND_URL.getMsg(), null);
            return WebFluxResultUtils.result(exchange, error);
        }
        long timeout = (long) Optional.ofNullable(exchange.getAttribute(Constants.HTTP_TIME_OUT)).orElse(3000L);
        int retryTimes = (int) Optional.ofNullable(exchange.getAttribute(Constants.HTTP_RETRY)).orElse(0);
        log.info("The request urlPath is {}, retryTimes is {}", urlPath, retryTimes);
        HttpMethod method = HttpMethod.valueOf(exchange.getRequest().getMethodValue());
        WebClient.RequestBodySpec requestBodySpec = webClient.method(method).uri(urlPath);
        return handleRequestBody(requestBodySpec, exchange, timeout, retryTimes, chain);
    }
    
    private Mono<Void> handleRequestBody(final WebClient.RequestBodySpec requestBodySpec,
                                         final ServerWebExchange exchange,
                                         final long timeout,
                                         final int retryTimes,
                                         final SoulPluginChain chain) {
        return requestBodySpec.headers(httpHeaders -> {
            httpHeaders.addAll(exchange.getRequest().getHeaders());
            httpHeaders.remove(HttpHeaders.HOST);
        })
                .contentType(buildMediaType(exchange))
                .body(BodyInserters.fromDataBuffers(exchange.getRequest().getBody()))
                .exchange()
                .doOnError(e -> log.error(e.getMessage()))
                .timeout(Duration.ofMillis(timeout))
                .retryWhen(Retry.onlyIf(x -> x.exception() instanceof ConnectTimeoutException)
                    .retryMax(retryTimes)
                    .backoff(Backoff.exponential(Duration.ofMillis(200), Duration.ofSeconds(20), 2, true)))
                .flatMap(e -> doNext(e, exchange, chain));

    }

    private Mono<Void> doNext(final ClientResponse res, final ServerWebExchange exchange, final SoulPluginChain chain) {
        if (res.statusCode().is2xxSuccessful()) {
            exchange.getAttributes().put(Constants.CLIENT_RESPONSE_RESULT_TYPE, ResultEnum.SUCCESS.getName());
        } else {
            exchange.getAttributes().put(Constants.CLIENT_RESPONSE_RESULT_TYPE, ResultEnum.ERROR.getName());
        }
        exchange.getAttributes().put(Constants.CLIENT_RESPONSE_ATTR, res);
        return chain.execute(exchange);
    }
```
来到 `WebClientResponsePlugin` ，好像返回了一个 response
```java
    /**
     * Process the Web request and (optionally) delegate to the next
     * {@code WebFilter} through the given {@link SoulPluginChain}.
     *
     * @param exchange the current server exchange
     * @param chain    provides a way to delegate to the next filter
     * @return {@code Mono<Void>} to indicate when request processing is complete
     */
    @Override
    public Mono<Void> execute(final ServerWebExchange exchange, final SoulPluginChain chain) {
        return chain.execute(exchange).then(Mono.defer(() -> {
            ServerHttpResponse response = exchange.getResponse();
            ClientResponse clientResponse = exchange.getAttribute(Constants.CLIENT_RESPONSE_ATTR);
            if (Objects.isNull(clientResponse)
                    || response.getStatusCode() == HttpStatus.BAD_GATEWAY
                    || response.getStatusCode() == HttpStatus.INTERNAL_SERVER_ERROR) {
                Object error = SoulResultWrap.error(SoulResultEnum.SERVICE_RESULT_ERROR.getCode(), SoulResultEnum.SERVICE_RESULT_ERROR.getMsg(), null);
                return WebFluxResultUtils.result(exchange, error);
            }
            if (response.getStatusCode() == HttpStatus.GATEWAY_TIMEOUT) {
                Object error = SoulResultWrap.error(SoulResultEnum.SERVICE_TIMEOUT.getCode(), SoulResultEnum.SERVICE_TIMEOUT.getMsg(), null);
                return WebFluxResultUtils.result(exchange, error);
            }
            response.setStatusCode(clientResponse.statusCode());
            response.getCookies().putAll(clientResponse.cookies());
            response.getHeaders().putAll(clientResponse.headers().asHttpHeaders());
            return response.writeWith(clientResponse.body(BodyExtractors.toDataBuffers()));
        }));
    }
```
## 复查
详细看一下 WafPlugin、DividePlugin 、WebClientPlugin 、 WebClientResponsePlugin 等
### DividePlugin 
发送一个错误请求试试，这个请求多打了一个字母 `d`

- [http://localhost:9195/http/order/findByIdd?id=1](http://localhost:9195/http/order/findByIdd?id=1)



发现下面这俩东西


`FallbackUtils` 类
```java
    /**
     * get no rule result.
     *
     * @param pluginName the plugin name
     * @param exchange   the exchange
     * @return the mono
     */
    public static Mono<Void> getNoRuleResult(final String pluginName, final ServerWebExchange exchange) {
        log.error("can not match rule data: {}", pluginName);
        Object error = SoulResultWrap.error(SoulResultEnum.RULE_NOT_FIND.getCode(), SoulResultEnum.RULE_NOT_FIND.getMsg(), null);
        return WebFluxResultUtils.result(exchange, error);
    }
```
`WebFluxResultUtils`，这里构建错误响应
```java
    /**
     * Error mono.
     *
     * @param exchange the exchange
     * @param result    the result
     * @return the mono
     */
    public static Mono<Void> result(final ServerWebExchange exchange, final Object result) {
        exchange.getResponse().getHeaders().setContentType(MediaType.APPLICATION_JSON);
        return exchange.getResponse().writeWith(Mono.just(exchange.getResponse()
                .bufferFactory().wrap(Objects.requireNonNull(JsonUtils.toJson(result)).getBytes())));
    }
```
![](https://ced-md-picture.oss-cn-beijing.aliyuncs.com/img/20210328105232.png)
这是 PostMan 得到的响应，与上图匹配
![image.png](https://ced-md-picture.oss-cn-beijing.aliyuncs.com/img/20210306134834.png)

### WafPlugin

发送一条可以被 WafPlugin 拒绝的请求。同样会到 `WebFluxResultUtils` 来构建错误响应。

![](https://ced-md-picture.oss-cn-beijing.aliyuncs.com/img/20210306161025.png)

### WebClientPlugin 

前面猜测 `WebClientPlugin` 的 `handleRequestBody` 会给真正的后台服务发送请求，`doNext()`，会做接下来的处理，为了验证这个猜测，首先定位到 `soul-examples-http` - `controller` - 的 `OrderController` 类，里面的 `findById()` 就是我们之前请求的方法，给他加上个日志功能。
![](https://ced-md-picture.oss-cn-beijing.aliyuncs.com/img/20210306134850.png)
然后在 `WebClientPlugin` 的 `handleRequestBody` 和 `doNext()` 上打上断点，再发送一个正确的请求，发现日志的确是在这个时候打印出来的。

### WebClientResponsePlugin
之前猜测 `WebClientResponsePlugin` 的 `execute()` 方法会返回请求的数据给 `postman()`，打断点，发现`execute()` 执行完后会返回到 `SoulWebHandler` 也就是前文调用链的地方。 
![image.png](https://ced-md-picture.oss-cn-beijing.aliyuncs.com/img/20210306134902.png)
之分析调用链的时候已经知道 `WebClientResponsePlugin` 是最后一个插件，它后边两个都是跳过的状态，在之后就是一些 reactor 和 Netty 的东西了，相信应该是刚刚那个地方给 PostMan 返回的结果。
![](https://ced-md-picture.oss-cn-beijing.aliyuncs.com/img/20210306134917.png)

### DefaultWebFilterChain 
还记得 DefaultWebFilterChain 的这个方法吗？在这打个断点看看
```bash
  @Override
  public Mono<Void> filter(ServerWebExchange exchange) {
    return Mono.defer(() ->
        this.currentFilter != null && this.chain != null ?
            invokeFilter(this.currentFilter, this.chain, exchange) :
            this.handler.handle(exchange));
  }
```
发现了下面几个相关类，也就跟据名字猜测一下意思吧。

- MetricsWebFilter
- HealthFilter
- FileSizeFilter
## 总结

- HttpServerOperations : 接收 Netty 请求的地方
- ReactorHttpHandlerAdapter ：生成 response 和 request
- HttpWebHandlerAdapter ：exchange 的生成
- DefaultWebFilterChain 
- SoulWebHandler：plugins 调用链

请求由 Netty 收到后，来到 Filter，这里进行一些处理：健康检查，文件大小检查等待，然后来到核心的 plugins，这里实现了 Soul 的核心功能：过滤，转发，返回等等。

![Soul plugins 调用链](https://ced-md-picture.oss-cn-beijing.aliyuncs.com/img/20210306161215.png)