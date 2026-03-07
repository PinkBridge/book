# Gateway 技术文档（第一版骨架）

## 背景与目标

Gateway 是微服务体系的统一流量入口，用于承接客户端请求并将其安全、稳定地分发到后端服务。  
本文目标是建立一套可落地的 Gateway 设计与运维框架，覆盖从架构设计到故障排查的核心实践。

本章聚焦解决以下问题：

- 统一入口与统一治理（鉴权、限流、路由、灰度）；
- 降低客户端与后端服务的耦合；
- 提升系统的可观测性、可用性与发布安全性。

本章不解决：

- 复杂业务编排与领域规则实现；
- 跨服务强事务与业务一致性设计细节。

## 核心概念

### API Gateway

系统的北向入口，负责流量接入、请求预处理、策略执行与结果返回。

### Route（路由）

定义请求匹配条件与目标服务，例如按路径、方法、Header 或 Host 进行转发。

### Predicate（断言）

Predicate（断言）是 API Gateway 路由规则中的“条件表达式”，用于判断某个请求是否命中某条路由。  
当客户端请求到达网关时，Gateway 会依次对所有路由的 Predicate 进行匹配，只有所有断言都成立时，该请求才会命中此路由、执行对应的后续处理。

常见断言包括请求的路径、方法、Header、参数、Host 域名、来源 IP、时间区间等，允许单独或组合使用。  
例如：“只允许 GET 方法且路径为 /api/user/** 的请求命中某路由”；  
又如：“只有携带特定 Header 或在某时间段才生效的流控规则”。

Predicate 机制为服务治理和安全隔离提供了灵活的粒度和入口控制能力。


| 断言类型         | 说明                   | 常见示例                       |
| ------------ | -------------------- | -------------------------- |
| Path         | 按请求路径匹配              | `/api/user/`**、`/order/`*  |
| Method       | 按 HTTP 方法匹配          | `GET`、`POST`               |
| Header       | 按 Header 键值对匹配       | `X-Env=prod`、`token!=null` |
| Query        | 按查询参数（Query Param）匹配 | `version=2`、`debug=true`   |
| Host         | 按 Host（域名）匹配         | `api.example.com`          |
| Cookie       | 按 Cookie 内容匹配        | `sessionid=xxx`            |
| RemoteAddr   | 按客户端 IP 或网段（CIDR）匹配  | `192.168.1.0/24`           |
| After/Before | 按时间/日期判断请求是否在区间内     | `After=2024-01-01`         |
| Custom       | 按自定义规则逻辑             | 特定业务判断、表达式等                |


> 断言可灵活组合，决定请求是否“命中”该路由，常与过滤器联合实现定制流控与治理策略。

### Filter（过滤器）

在请求和响应链路上执行横切逻辑，如鉴权、限流、日志、Header 改写。

- 全局过滤器：对所有请求生效，常用于跨全局的统一处理场景，例如统一鉴权、全链路日志、统一限流、请求/响应头统一加工、安全加固（如添加安全 Header）、统一异常处理等；

#### 自定义全局过滤器（Global Filter）

在实际场景中，内置过滤器往往无法满足全部业务需求，此时可通过编写自定义全局过滤器来扩展网关链路能力。  
自定义全局过滤器适用于全链路请求的统一处理，例如灰度发布标记、统一日志采集、请求追踪、全局安全检查、自定义限流等。

**实现步骤（以 Spring Cloud Gateway 为例）：**

1. **实现 GlobalFilter 接口并加上 @Component 注解：**

```java
import org.springframework.cloud.gateway.filter.GlobalFilter;
import org.springframework.core.Ordered;
import org.springframework.stereotype.Component;
import org.springframework.web.server.ServerWebExchange;
import reactor.core.publisher.Mono;

@Component
public class TraceIdGlobalFilter implements GlobalFilter, Ordered {
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        // 预处理逻辑（比如注入 Trace-Id、日志采集、权限校验…）
        String traceId = generateTraceId();
        exchange.getRequest().mutate()
                .header("X-Trace-Id", traceId)
                .build();
        // 继续后续过滤器链
        return chain.filter(exchange);
    }

    @Override
    public int getOrder() {
        return 0; // 越小越优先，常用 -1~0 作为高优先全局过滤器
    }

    private String generateTraceId() {
        return java.util.UUID.randomUUID().toString();
    }
}
```

1. **全局生效，无需在配置文件中特别指定，系统启动后自动拦截全部请求流量。**
2. **应用场景：**
  - 统一日志/埋点数据采集
  - 鉴权/协议安全过滤
  - 灰度标识下发
  - Abuse/攻击检测
  - 请求头加工与追踪链路ID注入等

> 自定义过滤器适合落地企业级的横向治理能力，记得合理选择执行顺序（通过 `getOrder()` 控制），避免与内置过滤器冲突。

- 路由过滤器：仅对命中某路由的请求生效，通常用于按业务细分的自定义处理，如特定接口的参数校验、请求数据解密、局部埋点、业务属性补充等。

Spring Cloud Gateway 内置了 31 种常用的路由过滤器工厂（GatewayFilter Factory），可通过简洁配置对请求和响应进行灵活增强。  
每个过滤器工厂对应特定的功能点，可在路由级增强链路能力。常见内置过滤器工厂列表如下：

1. AddRequestHeader
2. AddRequestParameter
3. AddResponseHeader
4. DedupeResponseHeader
5. Filter
6. MapRequestHeader
7. PrefixPath
8. PreserveHostHeader
9. RequestRateLimiter
10. RedirectTo
11. RemoveRequestHeader
12. RemoveResponseHeader
13. RemoveRequestParameter
14. RewritePath
15. RewriteResponseHeader
16. RewriteResponseStatus
17. SaveSession
18. SecureHeaders
19. SetPath
20. SetRequestHeader
21. SetResponseHeader
22. SetStatus
23. StripPrefix
24. Retry
25. ModifyRequestBody
26. ModifyResponseBody
27. CircuitBreaker
28. RequestSize
29. AddResponseParameter
30. CacheRequestBody
31. Name (名称用于逻辑分组或过滤条件, 部分版本也记为 AddHostHeader)

> 具体含义举例：
>
> - `AddRequestHeader`：为请求动态添加 Header；
> - `StripPrefix`：去除请求路径前缀；
> - `RequestRateLimiter`：基于 Redis 实现分布式限流；
> - `ModifyRequestBody`：动态修改请求体；
> - `RewritePath`：正则重写路径；
> - `CircuitBreaker`：集成断路器逻辑；
> - 其余，如添加/删除请求参数、响应头、状态码重写、鉴权、异常恢复等。

实际使用时，可按 YAML 或 Java DSL 灵活组合、配置多种过滤器，满足不同业务场景下的 API 管理和网关治理诉求。

---

### 路由滤器的实现方式过

以 Spring Cloud Gateway 为例，实现自定义路由过滤器（GatewayFilter）通常有以下两种方式：

#### 1. 代码方式实现自定义 GatewayFilter

可通过实现 `GatewayFilter` 接口（配合 `@Component` 或工厂注册），将自定义逻辑植入到特定路由上：

```java
import org.springframework.cloud.gateway.filter.GatewayFilter;
import org.springframework.cloud.gateway.filter.GatewayFilterChain;
import org.springframework.stereotype.Component;
import org.springframework.web.server.ServerWebExchange;
import reactor.core.publisher.Mono;

public class CheckTokenFilter implements GatewayFilter {

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        // 这里只是举例：简单校验请求头 Token 是否存在
        String token = exchange.getRequest().getHeaders().getFirst("Token");
        if (token == null || token.isEmpty()) {
            // 拦截，直接返回 401
            exchange.getResponse().setStatusCode(org.springframework.http.HttpStatus.UNAUTHORIZED);
            return exchange.getResponse().setComplete();
        }
        // 继续过滤器链
        return chain.filter(exchange);
    }
}
```

注册到自定义路由配置：

```yaml
routes:
  - id: secure-route
    uri: lb://user-service
    predicates:
      - Path=/api/secure/**
    filters:
      - name: CheckTokenFilter
```

> 可实现工厂扩展 `GatewayFilterFactory`，自定义配置参数，详见官方文档。

#### 2. 配置式路由过滤器（常用内置）

大部分常用场景可直接通过配置 YAML 内置过滤器：

```yaml
routes:
  - id: add-header-demo
    uri: http://example.com
    predicates:
      - Path=/demo/**
    filters:
      - AddRequestHeader=X-Request-Flag, gateway
      - StripPrefix=1
      - name: RequestRateLimiter
        args:
          redis-rate-limiter.replenishRate: 10
          redis-rate-limiter.burstCapacity: 20
```

- `AddRequestHeader`、`StripPrefix` 等均为路由级过滤器，通过配置即可生效。
- 路由过滤器的执行顺序由 filters 列表先后顺序决定。

#### 3. 自定义 GatewayFilterFactory（进阶）

如需带参数或更复杂逻辑，可实现自定义 `GatewayFilterFactory`：

```java
import org.springframework.cloud.gateway.filter.factory.AbstractGatewayFilterFactory;
import org.springframework.stereotype.Component;

@Component
public class CustomHeaderGatewayFilterFactory extends AbstractGatewayFilterFactory<CustomHeaderGatewayFilterFactory.Config> {
    public CustomHeaderGatewayFilterFactory() { super(Config.class); }

    @Override
    public GatewayFilter apply(Config config) {
        return (exchange, chain) -> {
            // 根据配置动态添加请求头
            exchange.getRequest().mutate()
                .header(config.getName(), config.getValue())
                .build();
            return chain.filter(exchange);
        };
    }
    public static class Config {
        private String name;
        private String value;
        // getter/setter
    }
}
```

使用方式：

```yaml
filters:
  - name: CustomHeader
    args:
      name: X-Trace-Id
      value: abc-123
```

### 小结

- **简单需求优先用配置式（YAML）内置过滤器**
- **定制化、带参数、复杂流程用自定义 GatewayFilter 或 GatewayFilterFactory 实现**
- **只在某一条路由匹配时生效，针对单点业务处理，区别于全局过滤器**
- **开发自定义过滤器时注意性能与副作用，推荐无状态、简洁快速响应**

---

### 路由与过滤器的执行顺序

API Gateway 在处理请求时，路由（Route）和过滤器（Filter）的执行顺序大致如下：

1. **请求接入**：客户端请求到达 Gateway；
2. **全局前置过滤器（Global Pre Filters）**：优先执行所有全局过滤器的前置逻辑，用于统一鉴权、限流、埋点、日志等横向能力；
3. **路由匹配（Route & Predicate）**：根据请求信息依次判断各路由的断言（Predicate），命中第一条匹配的路由；
4. **路由级过滤器（GatewayFilter，按顺序）**：仅针对命中的路由，按配置顺序串行执行该路由上声明的过滤器；
5. **业务服务处理（Backend Service）**：请求被转发到目标后端服务，获取响应结果；
6. **路由级后置过滤器**：依序执行路由过滤器的响应相关逻辑，例如响应头加工、日志收集、异常处理等；
7. **全局后置过滤器（Global Post Filters）**：最后统一处理所有请求的后置逻辑（如添加全局响应头、统计等）；
8. **响应返回客户端**。

总结为请求处理链：

```text
Client
  ↓
[全局前置过滤器]
  ↓
路由匹配 (断言)
  ↓
[路由过滤器]
  ↓
后端服务处理
  ↓
[路由过滤器（响应处理）]
  ↓
[全局后置过滤器]
  ↓
Client
```

**注意点：**

- 全局过滤器会包裹所有路由过滤器（环绕链路），且常用于统一、横向治理能力。
- 路由过滤器只作用于命中的路由，顺序取决于配置文件中声明的先后或实现类的 `getOrder()` 方法。
- Spring Cloud Gateway 的过滤器本质是“责任链”模式，前后处理逻辑通常通过 `filter(chain)` 方式嵌套调用，易于扩展和拦截。

良好的执行顺序管理有助于做到“先鉴权、后路由、再业务、最后统计与异常兜底”，实现高可用可扩展的微服务网关架构。

### 北向与南向流量

- 北向（North-South）：客户端到 Gateway；
- 南向（Southbound）：Gateway 到后端微服务。

## 架构设计

### 架构关系

建议采用以下基础关系：

`Client -> Gateway Cluster -> Service Cluster`

并接入以下基础组件：

- 注册中心：服务发现与实例感知；
- 配置中心：动态路由与策略下发；
- 认证中心：令牌签发与校验；
- 可观测平台：日志、指标、链路追踪。

### 请求流转路径

1. 客户端请求进入 Gateway；
2. 路由匹配（Predicate）；
3. 执行前置过滤器（鉴权、限流、审计）；
4. 负载均衡转发至目标服务；
5. 执行后置过滤器（响应头、日志、统计）；
6. 返回客户端。

## 关键能力设计

### 动态路由

- 路由配置支持热更新；
- 路由变更需版本化并可回滚；
- 路由规则要支持灰度标签。

### 认证与鉴权

- 推荐 JWT/OAuth2 方案；
- Gateway 做令牌合法性校验，业务权限下沉到服务侧；
- 鉴权失败统一返回标准错误码。

### 限流与熔断

- 限流维度：IP、用户、租户、接口；
- 熔断维度：下游服务、接口、异常比例；
- 失败策略：快速失败、降级兜底、友好提示。

### 灰度与发布控制

- 支持按用户、Header、流量比例灰度；
- 发布链路：小流量验证 -> 分批放量 -> 全量切换；
- 异常时可一键回滚灰度策略。

### 负载均衡与重试

- 基于健康状态选择实例；
- 幂等接口可重试，非幂等接口谨慎重试；
- 设置最大重试次数与退避策略，防止雪崩。

### 可观测性

- 统一请求 ID（TraceId）；
- 指标最少覆盖 QPS、错误率、P95/P99 延迟；
- 关键日志可检索、可关联、可追踪。

## 配置与实现

### 配置组织建议

- 按环境分层：dev/test/prod；
- 按能力拆分：路由、鉴权、限流、超时、重试；
- 所有关键配置纳入版本管理。

### 路由配置示例（占位）

```yaml
routes:
  - id: user-service
    uri: lb://user-service
    predicates:
      - Path=/api/users/**
    filters:
      - StripPrefix=1
      - RequestRateLimiter=...
```

### 过滤器链示例（占位）

```text
Request -> AuthFilter -> RateLimitFilter -> RouteFilter -> ResponseLogFilter -> Response
```

### 生产默认值建议（占位）

- 连接超时：1-3s；
- 读超时：3-10s；
- 单次重试：0-2 次（仅幂等请求）；
- 熔断恢复窗口：30-60s。

## 高可用与性能

### 部署策略

- Gateway 无状态化；
- 多实例集群部署；
- 跨可用区部署，避免单点故障。

### 扩缩容策略

- 基于 CPU、延迟、QPS 的自动扩缩容；
- 高峰前预热，避免冷启动抖动；
- 配置变更与扩容动作解耦。

### 压测指标

压测报告建议至少包含：

- 峰值 QPS；
- 平均延迟与 P95/P99；
- 错误率与超时率；
- 资源消耗（CPU/内存/连接数）。

## 安全设计

### 请求安全

- 统一 TLS；
- 敏感 Header 过滤；
- 防重放、防刷、防注入基础策略。

### 访问控制

- 白名单/黑名单策略；
- 高风险接口二次校验；
- 管理接口最小权限原则。

### 审计与合规

- 关键操作留审计日志；
- 保留必要追溯信息；
- 日志脱敏，避免敏感数据泄露。

## 故障处理与排查

### 常见故障

- 404：路由未命中或路径改写错误；
- 401/403：认证鉴权失败；
- 502/504：下游不可用或超时；
- 流量突增导致限流误触发。

### 排查路径

1. 看告警面板（错误率、延迟、流量）；
2. 看 Gateway 日志（路由命中、过滤器执行）；
3. 看下游服务健康与容量；
4. 看网络与 DNS；
5. 看最近配置变更与发布记录。

### 应急预案

- 快速降级非核心能力；
- 关闭高风险新规则；
- 回滚路由与过滤器配置；
- 保留故障现场用于复盘。

## 最佳实践与反模式

### 最佳实践

- Gateway 只做流量治理与横切能力；
- 业务规则放在领域服务，不堆在 Gateway；
- 配置与代码双重评审；
- 发布前做回归与压测。

### 反模式

- 在 Gateway 编写复杂业务编排；
- 对所有请求无差别重试；
- 缺少统一 TraceId，导致问题无法追踪；
- 无灰度直接全量发布。

## 小结

- Gateway 是微服务治理的入口层，不是业务实现层；
- 设计重点是“稳定性、可观测性、安全性、可演进性”；
- 通过标准化配置、灰度发布和故障预案，能显著降低线上风险。

## 落地检查清单（Checklist）

- 是否明确 Gateway 的职责边界；
- 是否具备动态路由、鉴权、限流、熔断能力；
- 是否建立了统一日志、指标与链路追踪；
- 是否完成高可用部署与压测基线；
- 是否具备灰度发布和快速回滚机制；
- 是否形成标准故障排查与复盘流程。

