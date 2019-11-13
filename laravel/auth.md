# laravel 源码阅读 Auth

## Index
 - [Auth模块](#Auth模块)
    - [AuthManager](#AuthManager)
    - [Guard](#Guard)
        - [SessionGuard与TokenGuard](#SessionGuard与TokenGuard)
    - [UserProvider](#UserProvider)
 - [Auth与框架的关系](#Auth与框架的关系)
    - [AuthServiceProvider](#AuthServiceProvider)
    - [路由解析](#路由解析)
    - [路由中间件](#路由中间件)

auth模块用于处理用户认证。在源码中，关于Auth模块，有两处命名空间：
 - Illuminate\Auth: Auth模块合兴代码。这部分的代码都是关于Auth模块的实现原理及逻辑。
 - Illuminate\Foundation\Auth: Auth模块应用功能。这部分是Auth模块在应用层的一些功能的实现。

## Auth模块
三大组成部分
 - AuthManager: 认证管理器
 - Guard: 认证器 or 看守器
 - UserProvider: 用户提供者

### AuthManager
AuthManager是用户认证模块功能的入口，是Auth类指代的实例。AuthManager的职责在于管理及扩展Guard与UserProvider，这是他的“本职工作”，如果在使用过程中，我们不涉及对认证功能的扩展，一般不会用到这部分；AuthManager的另一个职责，在于充当模块功能的入口，转发应用中对于Auth类的调用到Guard，比如：

```
Auth::user();
Auth::check();
Auth::login($user);

// 等同于
Auth::guard('web')->user();
Auth::guard('web')->check();
Auth::guard('web')->login($user);
```

### Guard
Guard用于实现认证功能，在AuthManager实例化Guard时，会绑定一个UserProvider给Guard，用于后续提供用户实例。框架实现了SessionGuard及TokenGuard，分别用于使用session，token做用户认证的场景。Guard的认证逻辑可以概括为：Guard从上下文中获取登陆凭证，将登陆凭证传递给UserProvider，查询出登陆用户的实例返回给Guard。

#### SessionGuard与TokenGuard
Session是非常常用的认证手段，框架实现的SessionGuard除了拥有认证功能外，还赋予了登陆与退出的功能。这里简单描述一下认证，登陆与退出的概念：
 - 登陆：客户端提交认证资料，经服务端验证成功后，生成登陆凭证，保存到相应位置，完成登陆。
 - 认证：服务端检查登陆凭证是否存在及有效，有则完成认证，请求放行。
 - 退出：服务端销毁登陆凭证，完成退出。

只有认证才是Guard的职责，基于不同的认证实现，登陆与退出功能可能会交给其他模块完成。比如基于JWT的Token认证方式，其登陆凭证是保存在客户端的，服务端不保存，所以服务端无法主动销毁登陆凭证，也就没有退出功能。然而基于Session的认证方式，登陆凭证是保存在服务端的，所以基于Session认证的方式，可以提供退出功能。

SessionGuard实现的登陆功能，也就是文档中所指的“手动认证用户”部分。SessionGuard提供attempt接口，用于用户登陆，同时提供了logout接口，实现了退出功能。TokenGuard并没有这两个功能。

### UserProvider
用户提供者接收由Guard传递的用户标识，查询出用户实例并返回。框架实现了EloquentUserProvider与DatabaseUserProvider，分别需要在auth配置中指定用户模型与用户表。大多数情况都是使用EloquentUserProvider。

## Auth与框架的关系
要完整的了解Auth模块的认证过程，需要结合框架的其他模块及细节来解读。

### AuthServiceProvider
和其他模块一样，Auth模块也是由服务提供者注册，在laravel应用生命周期中，处于第二阶段（容器启动）的结束阶段，在这里第一次与Request产生互动：

```php
// Illuminate\Auth\AuthServiceProvider

protected function registerRequestRebindHandler()
{
    $this->app->rebinding('request', function ($app, $request) {
        $request->setUserResolver(function ($guard = null) use ($app) {
            return call_user_func($app['auth']->userResolver(), $guard);
        });
    });
}
```
Auth服务注册时，给request绑定了一个“重绑定”事件，该事件的目的何在？

首先需要知道，request对象的实例化，是在容器启动之前，是一个比较早的阶段，可以说，在request对象第一次被实例化时，应用中基本还没有其他对象的存在，那么，如果在代码后续执行的过程中，需要丰富request对象，该怎么办呢？答案就是重绑定，在合适的时机，更新request对象之后，重新将request对象绑定到容器中。

Auth服务注册时，给request对象重绑定了一个事件，用于给request添加“用户解析”功能，当使用request的“用户解析”功能时，实际上会去找Guard要用户。然而，在服务注册阶段，Guard表示我也还没实例化，你不能立刻来找我要用户，而是要“推迟”找我要用户的时间，所以，最终在这里绑定的是一个闭包，保存的是request解析用户的途径，在合适的时机，通过这一途径，即可找Guard要到用户，但这时机究竟是什么时候呢？这个时机，必须满足两个条件：
 - request发生了重绑定
 - Guard认证用户结束

### 路由解析
路由解析处于laravel应用生命周期的的第三阶段（请求处理）。在第二阶段结束，第三阶段开始时，request进行了重绑定：

```php
// Illuminate\Foundation\Http\Kernel

// request 重绑定的发生过程

// HTTP kernel 捕获request，开始处理
public function handle($request)
{
    try {
        $request->enableHttpMethodParameterOverride();

        $response = $this->sendRequestThroughRouter($request);
    } catch (Exception $e) {
        $this->reportException($e);

        $response = $this->renderException($request, $e);
    } catch (Throwable $e) {
        $this->reportException($e = new FatalThrowableError($e));

        $response = $this->renderException($request, $e);
    }

    $this->app['events']->dispatch(
        new Events\RequestHandled($request, $response)
    );

    return $response;
}

// 2. HTTP kernel 发送request通过路由
protected function sendRequestThroughRouter($request)
{
    $this->app->instance('request', $request);

    Facade::clearResolvedInstance('request');

    $this->bootstrap();

    return (new Pipeline($this->app))
        ->send($request)
        ->through($this->app->shouldSkipMiddleware() ? [] : $this->middleware)
        ->then($this->dispatchToRouter());
}

// 3. HTTP kernel 准备解析路由
protected function dispatchToRouter()
{
    return function ($request) {
        // request 在这里重新被绑定，触发重绑定事件
        $this->app->instance('request', $request);

        return $this->router->dispatch($request);
    };
}
```

框架选择在此处对request进行重绑定，是因为，刚刚结束的第二阶段，已经完成了所有服务提供者的注册与启动，此时容器中已经存在所有的服务对象，通过服务对象来丰富request对象成为可能。

在request重新绑定之后，执行了这么一段代码：

```php
// Illuminate\Auth\AuthServiceProvider

return $this->router->dispatch($request);
```

这段代码的后文比较长，我简单概况一下：

```
匹配并命中路由 -> 通过路由解析并实例化控制器对象 -> 收集路由与控制器中定义的中间件 -> 执行路由中间件 -> 执行控制器方法
```

### 路由中间件
认证的行为，在中间件中触发。触发认证行为的中间件是`\Illuminate\Auth\Middleware\Authenticate::class`，在request通过该中间件时，Guard检查request是否已通过认证，通过则放行，否则抛出`AuthenticationException`未认证异常。在通过认证之后，用户实例会保存在Guard对象中，后续所有找Guard要用户的行为，都可以得到相同的用户实例。至此，认证完成。

综上所述，用户的认证，发生在执行路由中间的过程中，在此之前，是无法通过Auth来获取认证用户的，需要特别注意的是，控制器的实例化过程，发生在路由中间件执行之前，所以无法在控制器的构造函数中获取用户的登陆状态。
