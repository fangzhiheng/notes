# 0. 核心概念

阅读本文前，请先记住这句话：安全框架的大体逻辑基本都是**先认证授权，再权限校验。**

在Security中，以下概念最为核心：

- Authentication

	>  Authentication， [ɔ.θentɪ'keɪʃ(ə)n]: 认证；鉴定

	Spring Security 中最为核心的概念，贯穿整个Security体系，主要的认证对象。
	
- AuthenticationManager

  认证管理器，用来做授权的部分，要么授权成功后，返回认证对象；要么认证失败，抛出相应的认证异常。

- UserDetails

  用户详情，包含用户名、密码、用户状态、用户权限等。

- UserDetailsService

  用户详情服务，用来查询用户详情。

- GrantedAuthority

  被授予的权限，每个认证对象都会持有若干被授予的权限。

- ConfigAttribute

  保存了与Security相关的安全属性，每一个被Security覆盖到的部分，都有其特有的属性。

- SecurityMetadataSource

  安全元数据源，用来获取特定部分的ConfigAttribute。

- AccessDecisionManager

  访问控制管理器，根据认证对象以及配置属性做最终的访问控制决策。

通常的流程如下所示：

```flow
st=>start: 开始
e=>end: 结束
build_token=>operation: 构建口令
authenticate=>condition: 认证
authorize=>operation: 授权
authentication_success=>operation: 认证成功
authentication_failure=>operation: 认证失败
build_authentication=>operation: 构建认证对象
check_authority=>condition: 校验权限
check_authority_success=>operation: 权限校验成功
check_authority_failure=>operation: 权限校验失败

st->build_token->authenticate
authenticate(yes)->authentication_success
authenticate(no)->authentication_failure
authentication_failure->e

authentication_success->build_authentication->check_authority
check_authority(yes)->check_authority_success
check_authority(no)->check_authority_failure

check_authority_success->e
check_authority_failure->e
```

上面Security这部分核心内容可以脱离Web在做其他类型的应用中也可以使用，而且就算我们自己实现一个简单的Security框架，基本逻辑也是大致如此的。

但是，我们更加常见的场景是Web场景，因此本文将更多地结合着Web场景来分析。

# 1. 合适的认证位置

在Spring Security中，框架把这个位置放在了Filter中去实现，比如`UsernamePasswordAuthenticationFilter`就是用来承担Form表单登录的认证服务，它继承自`AbstractAuthenticationProcessingFilter`，而这个类实现了一套默认的认证逻辑（在`doFilter()`方法中）：

```flow
st=>start: 开始过滤
se=>end: 过滤结束，请求继续
fe=>end: 过滤结束，请求结束

check_require_authentication=>condition: 是否需要认证
attempt_authentication=>condition: 尝试认证
authentication_success=>operation: 认证成功
authentication_failure=>operation: 认证失败
on_success=>operation: 成功后置处理
on_failure=>operation: 失败后置处理
continue_filter=>operation: 继续过滤
can_continue=>condition: 是否可以继续

st->check_require_authentication
check_require_authentication(no)->continue_filter
check_require_authentication(yes)->attempt_authentication
attempt_authentication(yes)->authentication_success
attempt_authentication(no,left)->authentication_failure
authentication_failure->on_failure
authentication_success->can_continue
can_continue(yes)->continue_filter
can_continue(no,left)->fe
on_failure->fe
continue_filter->fe
```

该过滤器里面会持有三个类型的对象：`AuthenticationManager`、`AuthenticationSuccessHandler`、`AuthenticationFailureHandler`，分别用来做认证工作、认证成功的后置处理工作、认证失败后的后置处理工作。

大部分时候，`AuthenticationSuccessHandler`的实现类会是`ProviderManager`，而后者是策略模式的一个实现，策略所实现的接口是`AuthenticationProvider`，查看`AuthenticationProvider`的实现类，可以看到许多熟悉的类，比如说：`DaoAuthenticationProvider`，它内部持有了`UserDetailsService`的实现类，这个类就是大多数博文上会提到的那个接入数据库的实现类。此处话多一句，其实框架里提供了一个`JdbcDaoImpl`的实现类，直接写入SQL就可以使用了。

# 2. 权限校验位置

认证部分是最为简单的部分，结构清晰，容易阅读，但是权限校验部分是一个比较复杂的部分，不足之处，望多赐教。

Web的权限校验分为两部分，一部分是基于URL实现的逻辑，一部分是基于拦截器实现的逻辑。前者提供链接级别的校验，而后者可以提供方法级别的校验。但是不管是哪一种，它们的核心逻辑都是在`AbstractSecurityInterceptor#beforeInvocation()`里，都是通过获取安全属性、获取认证对象然后再把它们交给访问控制管理器来做最终的认证校验。无非是不同的校验方式，获取安全属性的安全元数据源和访问控制器的对象不同而已。

访问控制器的实现依赖于投票器，而它们不同的地方主要在于对投票结果的选择上，框架里内置了三个实现，分别是：积极控制（`AffiramativeBased`，只要有一个通过即认为通过）、多数服从少数（`ConsensusBased`）、全部通过才算通过（`UnanimousBased`）。他们可以覆盖大部分场景，但是在某些业务中，我们会需要一些定制化的控制决策，那个时候可以从这里入手。

## 2.1 基于Web表达式的校验

实现基于Web表达式校验逻辑主要还是依赖于过滤器（`FilterSecurityInterceptor`），通过构建`FilterInvocation`然后通过`FilterInvocationSecurityMetadataSource`来获取安全属性，之后再继续上面的流程。

不过它的访问控制器默认使用的是积极控制（`AbstractInterceptorUrlConfigurer#createDefaultAccessDecisionManager()`），而且使用的是`WebExpressionVoter`，看名字大概知道适合SpEL表达式相关，SpEL表达式的构建场所是`AbstractSecurityExpressionHandler#createEvaluationContext()`，SpEL表达式所评估的就是`WebSecurityConfigurerAdapter#configure()`中所配置的，例如：

```java
((HttpSecurity) http).authorizeRequests() // 这个就表明要开启基于Web表达式的校验了
    .anyRequest()
    .authenticated() // 这里就是来定义所配请求的安全属性的
```

## 2.2 基于方法的校验

而实现基于方法的校验，是基于方法拦截器实现的（`MethodSecurityInterceptor`， 实现了`MethodInterceptor`接口）。核心逻辑与上面基于Web表达式的一致，还是元数据源和访问控制器不同。

这里元数据源主要实现了`MethodSecurityMetadataSource`接口，而且实现类很多，比如`PrePostAnnotationSecurityMetadataSource`。

`PrePostAnnotationSecurityMetadataSource`主要用来处理`PreFilter`、`PreAuthorize`、`PostFilter`、`PostAuthorize`注解，具体使用方式也是基于SpEL表达式，所能使用的方法可以查看`SecurityExpressionRoot`。

需要注意的是，开启基于方法的校验需要在配置上启用`@EnableGlobalMethodSecurity`注解，而且由于开启后会增强类，因此需要注意规避自调用问题。

# 3. 小结

很多人在说Security复杂，不如用Shiro。但是Shiro的代码给我的感觉总是很奇怪，无论是过滤器方法的设计，还是认证时Subject的登录逻辑。

但是不管是哪个框架，认证的逻辑都是大同小异的，不要人云亦云，做什么选择无所谓，重要的是选择的理由得足够充分。

本文只是大概捋了一下认证的整体逻辑，细节之处就不做展开了。

