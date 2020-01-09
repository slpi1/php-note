# laravel 模型观察者Observer

# Index
 - [使用](#使用)
    - [定义观察者](#定义观察者)
    - [注册观察者](#注册观察者)
    - [事件机制](#事件机制)
 - [踩过的坑](#踩过的坑)
    - [同时发生的事件](#同时发生的事件)

# 使用
模型观察者，可以在对模型数据进行操作时，监听模型操作的事件，并作出相应的处理。默认可监听的事件有 `retrieved/creating/created/updating/updated/deleting/deleted/saving/saved/restoring/restored`。具体这些事件在什么情况下发生，可以在 `Illuminate\Database\Eloquent\Model::class` 中进行查询。

## 定义观察者
首先需要定义一个观察者，然后定义响应事件的方法，方法名同事件名，如不定义，则不监听发生的相应事件。所有监听方法的参数都是模型对象自身：

```php
<?php

namespace App\Observers;

use App\User;

class UserObserver
{
    /**
     * 监听用户创建的事件。
     *
     * @param  User  $user
     * @return void
     */
    public function created(User $user)
    {
        //
    }

    /**
     * 监听用户删除事件。
     *
     * @param  User  $user
     * @return void
     */
    public function deleting(User $user)
    {
        //
    }
}
```

## 注册观察者
定义观察者后，在服务启动时进行观察者的注册，就可以让观察者发挥作用了：

```php
<?php

namespace App\Providers;

use App\User;
use App\Observers\UserObserver;
use Illuminate\Support\ServiceProvider;

class AppServiceProvider extends ServiceProvider
{
    /**
     * 运行所有应用.
     *
     * @return void
     */
    public function boot()
    {
        User::observe(UserObserver::class);
    }

    /**
     * 注册服务提供.
     *
     * @return void
     */
    public function register()
    {
        //
    }
}
```

## 事件机制
观察者依赖于框架本身的事件机制。原理是在服务启动时，通过 `User::observe(UserObserver::class)` 逐个检查模型的事件方法，并进行事件注册。

```php

// 1. 事件句柄注入Model
class DatabaseServiceProvider extends ServiceProvider
{
    public function boot()
    {
        Model::setConnectionResolver($this->app['db']);

        // 在启动Eloquent服务时，将事件注入到Model中
        Model::setEventDispatcher($this->app['events']);
    }
}


// 2. 事件绑定
public static function observe($class)
{
    $instance = new static;

    $className = is_string($class) ? $class : get_class($class);

    // 这里获取框架原始支持的模型事件与用户自定义事件
    foreach ($instance->getObservableEvents() as $event) {
        if (method_exists($class, $event)) {
            static::registerModelEvent($event, $className.'@'.$event);
        }
    }
}

protected static function registerModelEvent($event, $callback)
{
    if (isset(static::$dispatcher)) {
        $name = static::class;

        // 这里对相应事件进行绑定
        static::$dispatcher->listen("eloquent.{$event}: {$name}", $callback);
    }
}
```

# 踩过的坑

## 同时发生的事件

通过上文的事件名，很容易就能看出，有些事件是成对出现的：
 - `creating/created`   
 - `updating/updated`
 - `deleting/deleted`
 - `saving/saved`
 - `restoring/restored`

这些成对出现的事件是在一次操作的前后分别发生，比如在插入数据时，必然会发生 `creating/created` 两个事件。

设想一个场景，在 `update` 的过程中，我们需要检测某些字段是否被修改，如果被修改，则需要在数据更新完毕后，执行一些操作。这个需求实现的方案大致如下：
 - 监听 `updating` 事件，在事件相应中进行脏检查，如果满足条件，则设置一个 `true` 标记
 - 监听 `updated` 事件，检查上述标记是否为 `true`，是则触发更新后的操作

如果某个操作需要夸事件来完成，则需要保存一个这样的数据状态，很容易就想到，通过以下代码来实现:

```php

class UserObserver
{
    protected $shouldExec = false;

    public function updating(User $user)
    {
        if($user->isDirty()){
            $this->shouldExec = true;
        }
    }

    public function updated(User $user)
    {
        if($this->shouldExec){
            // 执行所需操作
        }
    }
}
```

然而，上面的代码并不会按预期来执行，无论什么情况，`updated` 方法中检查 `$shouldExec` 的值，都是初始化时的 `false`，其原因，就是观察者所依赖的事件机制。

在模型事件被触发时，事件派发器按事件名，解析出所要执行的回调方法，在直到要实例话一个观察者，并执行其方法:

```php

    protected function createClassCallable($listener)
    {
        list($class, $method) = $this->parseClassCallable($listener);

        if ($this->handlerShouldBeQueued($class)) {
            return $this->createQueuedHandlerCallable($class, $method);
        }

        // 这里通过解析出的观察者类名，从容器中解析出一个对象
        return [$this->container->make($class), $method];
    }
```

但是普通的容器解析，每次都会解析出一个新的对象。所以在事件多次发生时，每个观察者，都是重新实例化出来的，因此，观察者对象事件之间无法通过观察者自身的属性来共享数据。

那么如何解决这个问题呢？有两个方法：
 - 将要共享的数据存到别的地方，比如数据库，缓存，全局变量等。
 - 让容器每次解析出的，都是同一个观察者，只需要在服务启动时，对观察者类进行共享绑定

```php
<?php

namespace App\Providers;

use App\User;
use App\Observers\UserObserver;
use Illuminate\Support\ServiceProvider;

class AppServiceProvider extends ServiceProvider
{
    /**
     * Bootstrap any application services.
     *
     * @return void
     */
    public function boot()
    {
        // 在注册观察者前，进行共享绑定，以下三种都能达到相同的效果
        $this->app->bind(UserObserver::class, null, true);
        $this->app->singleton(UserObserver::class);
        $this->app->instance(UserObserver::class, new UserObserver);

        User::observe(UserObserver::class);
    }

    /**
     * Register any application services.
     *
     * @return void
     */
    public function register()
    {
        //
    }
}
```
