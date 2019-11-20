# Laravel 中间件

## 中间件全家福
先在这里列一下框架中用到的中间件。

```php
/**
 * 全局中间件
 */

// 检测项目是否处于 维护模式。
\Illuminate\Foundation\Http\Middleware\CheckForMaintenanceMode::class

// 检查请求数据大小是否超过限制
\Illuminate\Foundation\Http\Middleware\ValidatePostSize::class

// 清理请求字段值首位的空格
\App\Http\Middleware\TrimStrings::class

// 将请求中的空字段转化为null值
\Illuminate\Foundation\Http\Middleware\ConvertEmptyStringsToNull::class

// 设置可信任代理
\App\Http\Middleware\TrustProxies::class

/**
 * web 路由中间件
 */
// 获取请求中的Cookies，解密Cookies的值，加密Headers中的Cookies
\App\Http\Middleware\EncryptCookies::class

// 添加Cookies到Headers
\Illuminate\Cookie\Middleware\AddQueuedCookiesToResponse::class

// 启用Session，在Headers中添加Session相关的Cookies
\Illuminate\Session\Middleware\StartSession::class

// 将闪存到Session中的错误信息，共享到视图 - 闪存是指在上一次请求时将数据存入Session的过程，数据会在下一次请求时取出并销毁
\Illuminate\View\Middleware\ShareErrorsFromSession::class

// csrf 令牌验证
\App\Http\Middleware\VerifyCsrfToken::class

// 路由参数模型绑定检查
\Illuminate\Routing\Middleware\SubstituteBindings::class

/**
 * 其他路由中间件
 */
[
    // 认证中间件
    'auth'       => \Illuminate\Auth\Middleware\Authenticate::class,

    // http基础认证
    'auth.basic' => \Illuminate\Auth\Middleware\AuthenticateWithBasicAuth::class,

    // 路由参数模型绑定检查
    'bindings'   => \Illuminate\Routing\Middleware\SubstituteBindings::class,

    // 授权检查
    'can'        => \Illuminate\Auth\Middleware\Authorize::class,

    // 访客认证
    'guest'      => \App\Http\Middleware\RedirectIfAuthenticated::class,

    // 请求节流
    'throttle'   => \Illuminate\Routing\Middleware\ThrottleRequests::class,
]
```