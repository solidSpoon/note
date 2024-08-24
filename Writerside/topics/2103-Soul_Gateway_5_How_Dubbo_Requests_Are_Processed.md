---
title: Soul 网关 5：Dubbo 请求是怎么被处理的
date: 2021-03-10 21:02:54
updated: 
tags: 
 - Soul
 - Dubbo
categories: Soul 学习笔记
mathjax: false
---

## 前言

前文我们已经搭建了 Dubbo 和 Http 的示例，并且大致了解了 Soul 对 Http 的处理流程，本节我们来看看 Soul 网关对于 Dubbo 和 Http 的处理有什么异同。


前文我们知道，Soul 网关的 Plugin 链式处理核心是 SoulWebHandler，并且知道了调用链中的插件的用途。
> [https://solidspoon.xyz/2021/03/06/Soul-%E7%BD%91%E5%85%B3-3%EF%BC%9AHttp-%E8%AF%B7%E6%B1%82%E6%98%AF%E6%80%8E%E4%B9%88%E8%A2%AB%E5%A4%84%E7%90%86%E7%9A%84%EF%BC%9F/#%E6%80%BB%E7%BB%93](https://solidspoon.xyz/2021/03/06/Soul-%E7%BD%91%E5%85%B3-3%EF%BC%9AHttp-%E8%AF%B7%E6%B1%82%E6%98%AF%E6%80%8E%E4%B9%88%E8%A2%AB%E5%A4%84%E7%90%86%E7%9A%84%EF%BC%9F/#%E6%80%BB%E7%BB%93)



这是本次示例中 `SoulWebHandler` 持有的插件链，与上文同理，同样红色的插件被 Soul 跳过了。
![本文插件链](https://ced-md-picture.oss-cn-beijing.aliyuncs.com/img/20210312161204.png)
下面的图片是前文分析 Http 的 Divide 插件时的插件链，对比二者还是有区别的，例如 Divide 插件被我们禁用了，所以没了。原本的好多 `AlibabaDubbo` 的 Plugin  变成了 `ApacheDubbo` 的 Plugin（注：红色部分指的是前文分析时那些插件被 Soul 跳过了）
![前文插件链](https://ced-md-picture.oss-cn-beijing.aliyuncs.com/img/20210310210344.png)


这么对比一看发现似乎本文重点应该放在 `ApacheDubboBodyPlugin`，`ApacheDubboPlugin`，`DubboResponsePlugin` 上面。
## ApacheDubboPlugin
`ApacheDubboPlugin`，该插件在调用链的第 9 位，因为前面进行了 `index++`，所以条件断点应该设置为 `index == 10`
![](https://ced-md-picture.oss-cn-beijing.aliyuncs.com/img/20210310210826.png)
顺便说一下，`SoulWebHandler` 判断是否跳过的逻辑在对应插件中实现的，`skip()` 是 `SoulPlugin` 接口定义的方法。
![](https://ced-md-picture.oss-cn-beijing.aliyuncs.com/img/20210312161231.png)


```java
// SoulWebHandler:execute()
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
                    // 跳过逻辑在对应的插件中实现
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
然后会进入到 `AbstractSoulPlugin` 的 `execute()` 方法。如下图所示，该方法实际上继承自 `SoulPlugin`
![](https://ced-md-picture.oss-cn-beijing.aliyuncs.com/img/20210310210852.png)
按照前文经验，`AbstractSoulPlugin` 会在此处判断是否开启了该插件（`ApacheDubboPlugin`），也会判断当前网络包是否能匹配该插件注册的规则。如果上面条件都成立，就会调用该插件的 `doExecute()` 方法，这个方法是该插件继承自 `AbstractSoulPlugin`的。

> 前文：
> [https://solidspoon.xyz/2021/03/06/Soul-%E7%BD%91%E5%85%B3-3%EF%BC%9AHttp-%E8%AF%B7%E6%B1%82%E6%98%AF%E6%80%8E%E4%B9%88%E8%A2%AB%E5%A4%84%E7%90%86%E7%9A%84%EF%BC%9F/#%E8%B0%83%E7%94%A8%E9%93%BE](https://solidspoon.xyz/2021/03/06/Soul-%E7%BD%91%E5%85%B3-3%EF%BC%9AHttp-%E8%AF%B7%E6%B1%82%E6%98%AF%E6%80%8E%E4%B9%88%E8%A2%AB%E5%A4%84%E7%90%86%E7%9A%84%EF%BC%9F/#%E8%B0%83%E7%94%A8%E9%93%BE)

```java
    /**
     * Process the Web request and (optionally) delegate to the next
     * {@code SoulPlugin} through the given {@link SoulPluginChain}.
     *
     * @param exchange the current server exchange
     * @param chain    provides a way to delegate to the next plugin
     * @return {@code Mono<Void>} to indicate when request processing is complete
     */
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
实际调试发现果然和我们预想的一样，来到了 `ApacheDubboPlugin:doExecute()` ,
```java
    @Override
    protected Mono<Void> doExecute(final ServerWebExchange exchange, final SoulPluginChain chain, final SelectorData selector, final RuleData rule) {
        String body = exchange.getAttribute(Constants.DUBBO_PARAMS);
        SoulContext soulContext = exchange.getAttribute(Constants.CONTEXT);
        assert soulContext != null;
        // 取得 Dubbo 请求相关数据
        MetaData metaData = exchange.getAttribute(Constants.META_DATA);
        if (!checkMetaData(metaData)) {
            assert metaData != null;
            log.error(" path is :{}, meta data have error.... {}", soulContext.getPath(), metaData.toString());
            exchange.getResponse().setStatusCode(HttpStatus.INTERNAL_SERVER_ERROR);
            Object error = SoulResultWrap.error(SoulResultEnum.META_DATA_ERROR.getCode(), SoulResultEnum.META_DATA_ERROR.getMsg(), null);
            return WebFluxResultUtils.result(exchange, error);
        }
        if (StringUtils.isNoneBlank(metaData.getParameterTypes()) && StringUtils.isBlank(body)) {
            exchange.getResponse().setStatusCode(HttpStatus.INTERNAL_SERVER_ERROR);
            Object error = SoulResultWrap.error(SoulResultEnum.DUBBO_HAVE_BODY_PARAM.getCode(), SoulResultEnum.DUBBO_HAVE_BODY_PARAM.getMsg(), null);
            return WebFluxResultUtils.result(exchange, error);
        }
        // 向真实后端发送请求，获得结果
        final Mono<Object> result = dubboProxyService.genericInvoker(body, metaData, exchange);
        return result.then(chain.execute(exchange));
    }
```
看一下`genericInvoker()`
```java
// ApacheDubboProxyService:genericInvoker()

    /**
     * Generic invoker object.
     *
     * @param body     the body
     * @param metaData the meta data
     * @param exchange the exchange
     * @return the object
     * @throws SoulException the soul exception
     */
    public Mono<Object> genericInvoker(final String body, final MetaData metaData, final ServerWebExchange exchange) throws SoulException {
        // issue(https://github.com/dromara/soul/issues/471), add dubbo tag route
        String dubboTagRouteFromHttpHeaders = exchange.getRequest().getHeaders().getFirst(Constants.DUBBO_TAG_ROUTE);
        if (StringUtils.isNotBlank(dubboTagRouteFromHttpHeaders)) {
            RpcContext.getContext().setAttachment(CommonConstants.TAG_KEY, dubboTagRouteFromHttpHeaders);
        }
        // metaData.getPath()== "/dubbo/findAll"
        // 大概是用 metaData.getPath() 作为键来获取值
        ReferenceConfig<GenericService> reference = ApplicationConfigCache.getInstance().get(metaData.getPath());
        if (Objects.isNull(reference) || StringUtils.isEmpty(reference.getInterface())) {
            ApplicationConfigCache.getInstance().invalidate(metaData.getPath());
            reference = ApplicationConfigCache.getInstance().initRef(metaData);
        }
        // 生成 RPC 对象
        GenericService genericService = reference.get();
        Pair<String[], Object[]> pair;
        if (ParamCheckUtils.dubboBodyIsEmpty(body)) {
            pair = new ImmutablePair<>(new String[]{}, new Object[]{});
        } else {
            pair = dubboParamResolveService.buildParameter(body, metaData.getParameterTypes());
        }
        // 在此处调用真实后端
        CompletableFuture<Object> future = genericService.$invokeAsync(metaData.getMethodName(), pair.getLeft(), pair.getRight());
        return Mono.fromFuture(future.thenApply(ret -> {
            if (Objects.isNull(ret)) {
                ret = Constants.DUBBO_RPC_RESULT_EMPTY;
            }
            exchange.getAttributes().put(Constants.DUBBO_RPC_RESULT, ret);
            exchange.getAttributes().put(Constants.CLIENT_RESPONSE_RESULT_TYPE, ResultEnum.SUCCESS.getName());
            return ret;
        })).onErrorMap(exception -> exception instanceof GenericException ? new SoulException(((GenericException) exception).getExceptionMessage()) : new SoulException(exception));
    }
```
看一下 `reference.get()` 看到了一个懒加载的代码
```java
// class ReferenceConfig<T> extends ReferenceConfigBase<T>
    public synchronized T get() {
        // 判断是否被毁
        if (destroyed) {
            throw new IllegalStateException("The invoker of ReferenceConfig(" + url + ") has already destroyed!");
        }
        // 延迟加载
        if (ref == null) {
            init();
        }
        return ref;
    }
```
## DubboResponsePlugin
再次从 `SoulWebHandler` 的调用链进来，到达 `DubbleResponsePlugin`
```java
// DubboResponsePlugin
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
            // 从 exchange 中拿到结果
            final Object result = exchange.getAttribute(Constants.DUBBO_RPC_RESULT);
            if (Objects.isNull(result)) {
                Object error = SoulResultWrap.error(SoulResultEnum.SERVICE_RESULT_ERROR.getCode(), SoulResultEnum.SERVICE_RESULT_ERROR.getMsg(), null);
                return WebFluxResultUtils.result(exchange, error);
            }
            Object success = SoulResultWrap.success(SoulResultEnum.SUCCESS.getCode(), SoulResultEnum.SUCCESS.getMsg(), JsonUtils.removeClass(result));
            // 这个方法里使用了之前见过的 wirteWith() 方法返回响应给客户端
            return WebFluxResultUtils.result(exchange, success);
        }));
    }
```
> 这个方法之前见过，之前认为是在 Http 那里用来构建错误响应，注释里也写着 Erro mono，但是似乎在这里不局限于错误响应，先留个疑惑。



> 前文链接
> [https://solidspoon.xyz/2021/03/06/Soul-%E7%BD%91%E5%85%B3-3%EF%BC%9AHttp-%E8%AF%B7%E6%B1%82%E6%98%AF%E6%80%8E%E4%B9%88%E8%A2%AB%E5%A4%84%E7%90%86%E7%9A%84%EF%BC%9F/#DividePlugin](https://solidspoon.xyz/2021/03/06/Soul-%E7%BD%91%E5%85%B3-3%EF%BC%9AHttp-%E8%AF%B7%E6%B1%82%E6%98%AF%E6%80%8E%E4%B9%88%E8%A2%AB%E5%A4%84%E7%90%86%E7%9A%84%EF%BC%9F/#DividePlugin)

```java
// WebFluxResultUtils
    /**
     * Error mono.
     *
     * @param exchange the exchange
     * @param result    the result
     * @return the mono
     */
    public static Mono<Void> result(final ServerWebExchange exchange, final Object result) {
        exchange.getResponse().getHeaders().setContentType(MediaType.APPLICATION_JSON);
        // writeWith() 来自 HttpMessage，如下图
        return exchange.getResponse().writeWith(Mono.just(exchange.getResponse()
                .bufferFactory().wrap(Objects.requireNonNull(JsonUtils.toJson(result)).getBytes())));
    }
```
`writeWith()` 来自 HttpMessage，如下图

![](https://ced-md-picture.oss-cn-beijing.aliyuncs.com/img/20210310210934.png)

## ApacheDubboBodyPlugin
该插件直接继承 `SoulPlugin`，而不是继承 `AbstractSoulPlugin`。
`BodyParamPlugin` 也是直接继承自 `SoulPlugin`，因为 `AbstractSoulPlugin` 主要进行了规则的匹配，可能这几个插件不需要。况且 soul-admin 也没有这些插件的界面。
![](https://ced-md-picture.oss-cn-beijing.aliyuncs.com/img/20210310211108.png)
前文说过修改路径为真实后端服务 RPC 路径，多个 RPC 会有多个相同的这个插件，本次示例 Soul 开启了两个 `DubboBodyParamPlugin`

- ApacheDubboBodyParamPlugin -> Dubbo
- DubboBodyParamPlugin -> SOFA



暂时不知道为什么要开启 SOFA RPC


`ApacheDubboBodyParamPlugin` ：
```java
// ApacheDubboBodyParamPlugin
    @Override
    public Mono<Void> execute(final ServerWebExchange exchange, final SoulPluginChain chain) {
        final ServerHttpRequest request = exchange.getRequest();
        final SoulContext soulContext = exchange.getAttribute(Constants.CONTEXT);
        // DUBBO
        if (Objects.nonNull(soulContext) && RpcTypeEnum.DUBBO.getName().equals(soulContext.getRpcType())) {
            MediaType mediaType = request.getHeaders().getContentType();
            ServerRequest serverRequest = ServerRequest.create(exchange, messageReaders);
            if (MediaType.APPLICATION_JSON.isCompatibleWith(mediaType)) {
                return body(exchange, serverRequest, chain);
            }
            if (MediaType.APPLICATION_FORM_URLENCODED.isCompatibleWith(mediaType)) {
                return formData(exchange, serverRequest, chain);
            }
            return query(exchange, serverRequest, chain);
        }
        return chain.execute(exchange);
    }
```
`BodyParamPlugin` ：
```java
// BodyParamPlugin
    @Override
    public Mono<Void> execute(final ServerWebExchange exchange, final SoulPluginChain chain) {
        final ServerHttpRequest request = exchange.getRequest();
        final SoulContext soulContext = exchange.getAttribute(Constants.CONTEXT);
        // SOFA
        if (Objects.nonNull(soulContext) && RpcTypeEnum.SOFA.getName().equals(soulContext.getRpcType())) {
            MediaType mediaType = request.getHeaders().getContentType();
            ServerRequest serverRequest = ServerRequest.create(exchange, messageReaders);
            if (MediaType.APPLICATION_JSON.isCompatibleWith(mediaType)) {
                return body(exchange, serverRequest, chain);
            }
            if (MediaType.APPLICATION_FORM_URLENCODED.isCompatibleWith(mediaType)) {
                return formData(exchange, serverRequest, chain);
            }
            return query(exchange, serverRequest, chain);
        }
        return chain.execute(exchange);
    }
```
## 总结
插件的调用都类似，所以本节验证了很多上节的猜想，也提出了新的猜想等待日后验证。
