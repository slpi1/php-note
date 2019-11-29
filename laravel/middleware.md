# Laravel 中间件

# Index
 - [中间件全家福](#中间件全家福)
 - [Authenticate](#Authenticate)
 - [CheckForMaintenanceMode](#CheckForMaintenanceMode)
 - [EncryptCookies](#EncryptCookies)
 - [RedirectIfAuthenticated](#RedirectIfAuthenticated)
 - [TrimStrings](#TrimStrings)
 - [TrustProxies](#TrustProxies)
 - [VerifyCsrfToken](#VerifyCsrfToken)
 - 待续...

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

## Authenticate
   
### 源文件

`app\Http\Middleware\Http\Middleware\Authenticate.php`

```php
<?php

namespace App\Http\Middleware;

use Illuminate\Auth\Middleware\Authenticate as Middleware;

class Authenticate extends Middleware
{
    /**
    * Get the path the user should be redirected to when they are not authenticated.
    *
    * @param \Illuminate\Http\Request $request
    * @return string
    */
    protected function redirectTo($request)
    {
        if (! $request->expectsJson()) {
            return route('login');
        }
    }
}
```

### 作用

用户身份验证。可修改 `redirectTo` 方法，返回未经身份验证的用户应该重定向到的路径。


## CheckForMaintenanceMode

### 源文件

`app\Http\Middleware\CheckForMaintenanceMode.php`

```php
<?php

namespace App\Http\Middleware;

use Illuminate\Foundation\Http\Middleware\CheckForMaintenanceMode as Middleware;

class CheckForMaintenanceMode extends Middleware
{
    /**
    * The URIs that should be reachable while maintenance mode is enabled.
    *
    * @var array
    */
    protected $except = [
        //
    ];
}
```

### 作用

检测项目是否处于 **维护模式**。可通过 `$except` 数组属性设置在维护模式下仍能访问的网址。


## EncryptCookies

### 源文件

`app\Http\Middleware\EncryptCookies.php`

```php
<?php

namespace App\Http\Middleware;

use Illuminate\Cookie\Middleware\EncryptCookies as Middleware;

class EncryptCookies extends Middleware
{
    /**
    * The names of the cookies that should not be encrypted.
    *
    * @var array
    */
    protected $except = [
        //
    ];
}
```

### 作用

对 Cookie 进行加解密处理与验证。可通过 `$except` 数组属性设置不做加密处理的 cookie。



## RedirectIfAuthenticated

### 源文件

`app\Http\Middleware\RedirectIfAuthenticated.php`

```php
<?php

namespace App\Http\Middleware;

use Closure;
use Illuminate\Support\Facades\Auth;

class RedirectIfAuthenticated
{
    /**
    * Handle an incoming request.
    *
    * @param \Illuminate\Http\Request $request
    * @param \Closure $next
    * @param string|null $guard
    * @return mixed
    */
    public function handle($request, Closure $next, $guard = null)
    {
        if (Auth::guard($guard)->check()) {
            return redirect('/home');
        }

        return $next($request);
    }
}
```

### 作用

当请求页是 `注册、登录、忘记密码` 时，检测用户是否已经登录，如果已经登录，那么就重定向到首页，如果没有就打开相应界面。可以在 `handle` 方法中定制重定向到的路径。


## TrimStrings

### 源文件

`app\Http\Middleware\TrimStrings.php`

```php
<?php

namespace App\Http\Middleware;

use Illuminate\Foundation\Http\Middleware\TrimStrings as Middleware;

class TrimStrings extends Middleware
{
    /**
    * The names of the attributes that should not be trimmed.
    *
    * @var array
    */
    protected $except = [
        'password',
        'password_confirmation',
    ];
}
```

### 作用

对请求参数内容进行 **前后空白字符清理**。可通过 `$except` 数组属性设置不做处理的参数。


## TrustProxies

### 源文件

`app\Http\Middleware\TrustProxies.php`

```php
<?php

namespace App\Http\Middleware;

use Illuminate\Http\Request;
use Fideloper\Proxy\TrustProxies as Middleware;

class TrustProxies extends Middleware
{
    /**
    * The trusted proxies for this application.
    *
    * @var array|string
    */
    protected $proxies;

    /**
    * The headers that should be used to detect proxies.
    *
    * @var int
    */
    protected $headers = Request::HEADER_X_FORWARDED_ALL;
}
```

### 作用

配置可信代理。可通过 `$proxies` 属性设置可信代理列表，`$headers` 属性设置用来检测代理的 HTTP 头字段。


## VerifyCsrfToken

### 源文件

`app\Http\Middleware\VerifyCsrfToken.php`

```php
<?php

namespace App\Http\Middleware;

use Illuminate\Foundation\Http\Middleware\VerifyCsrfToken as Middleware;

class VerifyCsrfToken extends Middleware
{
    /**
    * Indicates whether the XSRF-TOKEN cookie should be set on the response.
    *
    * @var bool
    */
    protected $addHttpCookie = true;

    /**
    * The URIs that should be excluded from CSRF verification.
    *
    * @var array
    */
    protected $except = [
        //
    ];
}
```

### 作用

验证请求里的令牌是否与存储在会话中令牌匹配。可通过 `$except` 数组属性设置不做 CSRF 验证的网址。