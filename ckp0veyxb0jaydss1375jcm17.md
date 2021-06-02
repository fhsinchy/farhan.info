## Laravel Service Container and Service Providers Explained

Laravel's service container is one of the most important pieces of the framework yet it gets so little attention from a lot of developers. Being interviewed a large number of candidates, I've realized that there are two main reasons behind this ignorance.

- They find the idea of dependency injection hard to understand. Let alone the idea of IoC and IoC container.
- They don't understand if they're ever going to use the container or not.

So in this article, I'll take you through step by step in understanding this magical concept. I'll begin with the fundamental concepts and finally, by the end, you should get an idea of how the different pieces fit together.

Going forward I am assuming you have a sound understanding of object-oriented programming, classes, objects, interfaces, and namespaces.

# Table of Content

- [Dependency Injection and IoC](#dependency-injection-and-ioc)
- [IoC Container](#ioc-container)
- [Service Container and Service Providers](#service-container-and-service-providers)
- [The Full Picture](#the-full-picture)
- [To Bind or Not To Bind](#to-bind-or-not-to-bind)
- [Suggested Reads](#suggested-reads)
- [Closing Thoughts](#closing-thoughts)

# Dependency Injection and IoC

An oversimplified definition of dependency injection is **the process of passing a class dependency as an argument to one of its methods** (usually the constructor).

Take a look at the following piece of code with no dependency injection:

```php

<?php

namespace App;

use App\Models\Post;
use App\Services\TwitterService;

class Publication {

    public function __construct()
    {
        // dependency is instantiated inside the class
        $this->twitterService = new TwitterService();
    }

    public function publish(Post $post)
    {
        $post->publish();

        $this->socialize($post);
    }

    protected function socialize($post)
    {
        $this->twitterService->share($post);
    }

}

```

This class which can be a part of an imaginary blogging platform is responsible for publishing a post and sharing it on social media.

The `socialize()` method uses an instance of the `TwitterService` class, which contains a single public method called `share()`.

```php

<?php

namespace App\Services;

use App\Models\Post;

class TwitterService {
    public function share(Post $post)
    {
        dd('shared on Twitter!');
    }
}

```

In the constructor, as you can see, a new instance of the `TwitterService` class has been created. Instead of performing the instantiation inside the class, you can inject an instance into the constructor as an argument from the outside.

```php

<?php

namespace App;

use App\Models\Post;
use App\Services\TwitterService;

class Publication {

    public function __construct(TwitterService $twitterService)
    {
        $this->twitterService = $twitterService;
    }

    public function publish(Post $post)
    {
        $post->publish();

        $this->socialize($post);
    }

    protected function socialize($post)
    {
        $this->twitterService->share($post);
    }

}

```

For this simple demonstration, you may use the `/` route callback in the `routes/web.php` file.

```php

<?php

// routes/web.php

use App\Publication;
use App\Models\Post;
use App\Services\TwitterService;
use Illuminate\Support\Facades\Route;

Route::get('/', function () {
    $post = new Post();

    // dependency injection
    $publication = new Publication(new TwitterService());

    dd($publication->publish($post));

    // shared on Twitter!
});

```

This is dependency injection at a basic level. Applying dependency injection to a class causes an inversion of control. Previously, the dependent class i.e. `Publication` was in control of instantiating the dependency class i.e. `TwitterService` whereas later, the control has been handed over to the framework.

# IoC Container

In the previous section, I've introduced you to the idea of dependency injection and showed you how it causes a class to hand over instantiation control to the framework.

An IoC container can make the process of dependency injection more efficient. It is a simple class capable of saving and providing pieces of data when asked for. A simplified IoC container can be written as follows:

```php

<?php

namespace App;

class Container {

    // array for keeping the container bindings
    protected $bindings = [];

    // binds new data to the container
    public function bind($key, $value)
    {
        // bind the given value with the given key
        $this->bindings[$key] = $value;
    }

    // returns bound data from the container
    public function make($key)
    {
        if (isset($this->bindings[$key])) {
            // check if the bound data is a callback
            if (is_callable($this->bindings[$key])) {
                // if yes, call the callback and return the value
                return call_user_func($this->bindings[$key]);
            } else {
                // if not, return the value as it is
                return $this->bindings[$key];
            }
        }
    }

}

```

You can bind any data to this container using the `bind()` method:

```php

<?php

// routes/web.php

use App\Container;
use Illuminate\Support\Facades\Route;

Route::get('/', function () {
    $container = new Container();

    $container->bind('name', 'Farhan Hasin Chowdhury');

    dd($container->make('name'));

    // Farhan Hasin Chowdhury
});

```

You can bind classes to this container by passing a callback function that returns an instance of the class as the second argument.

```php

<?php

// routes/web.php

use App\Container;
use App\Service\TwitterService;
use Illuminate\Support\Facades\Route;

Route::get('/', function () {
    $container = new Container;

    $container->bind(TwitterService::class, function(){
        return new App\Services\TwitterService;
    });

    ddd($container->make(TwitterService::class));

    // App\Services\TwitterService {#269}
});

```

Assume your `TwitterService` class needs an API key for authentication. In that case, you can do something as follows:

```php

<?php

// routes/web.php

use App\Container;
use App\Service\TwitterService;
use Illuminate\Support\Facades\Route;

Route::get('/', function () {
    $container = new Container;

    $container->bind('ApiKey', 'very-secret-api-key');

    $container->bind(TwitterService::class, function() use ($container){
        return new App\Services\TwitterService($container->make('ApiKey'));
    });

    ddd($container->make(TwitterService::class));

    // App\Services\TwitterService {#269 ▼
    //     #apiKey: "very-secret-api-key"
    // }
});

```

Once you bind a piece of data to the container, you can ask for it whenever necessary. This way you have to use the `new` keyword only once.

I'm not saying that using `new` is bad. But every time you use `new`, you'll have to be careful about passing the correct dependencies to the class. With an IoC container, however, the container takes care of injecting the dependencies.

An IoC container can make your code a lot more flexible. Consider a situation where you want to swap the `TwitterService` class with something else like a `LinkedInService` class.

The current implementation of the system is not very suitable for that. To replace the `TwitterService` class, you'll have to make a new class, bind that to the container, and replace all references to the previous class.

It doesn't have to be like that. You can make this process much easier by utilizing interfaces. Start by creating a new `SocialMediaServiceInterface`.

```php

<?php

namespace App\Interfaces;

use App\Models\Post;

interface SocialMediaServiceInterface {
    public function share(Post $post);
}

```

Now make your `TwitterService` class implement this interface.

```php

<?php

namespace App\Services;

use App\Models\Post;
use App\Interfaces\SocialMediaServiceInterface;

class TwitterService implements SocialMediaServiceInterface {
    protected $apiKey;

    public function __construct($apiKey)
    {
        $this->apiKey = $apiKey;
    }

    public function share(Post $post)
    {
        dd('shared on Twitter!');
    }
}

```

Instead of binding the concrete class to the container, bind the interface. In the callback, return an instance of the `TwitterService` class just like before.

```php

<?php

// routes/web.php

use App\Container;
use Illuminate\Support\Facades\Route;
use App\Interfaces\SocialMediaServiceInterface;

Route::get('/', function () {
    $container = new Container;

    $container->bind('ApiKey', 'very-secret-api-key');

    $container->bind(SocialMediaServiceInterface::class, function() use ($container){
        return new App\Services\TwitterService($container->make('ApiKey'));
    });

    ddd($container->make(SocialMediaServiceInterface::class));

    // App\Services\TwitterService {#269 ▼
    //     #apiKey: "very-secret-api-key"
    // }
});

```

So far the code is working just as before. The fun begins when you want to use LinkedIn instead of Twitter. With the interface in place, you can do that in two simple steps.

Create a new `LinkedInService` class that implements the `SocialMediaServiceInterface`.

```php

<?php

namespace App\Services;

use App\Models\Post;
use App\Interfaces\SocialMediaServiceInterface;

class LinkedInService implements SocialMediaServiceInterface {
    public function share(Post $post)
    {
        dd('shared on LinkedIn!');
    }
}

```

Update the call to the `bind()` method to return an instance of the `LinkedInService` class instead.

```php

<?php

// routes/web.php

use App\Container;
use App\Interfaces\SocialMediaServiceInterface;
use Illuminate\Support\Facades\Route;

Route::get('/', function () {
    $container = new Container;

    $container->bind(SocialMediaServiceInterface::class, function() {
        return new App\Services\LinkedInService();
    });

    ddd($container->make(SocialMediaServiceInterface::class));

    // App\Services\LinkedInService {#269}
});

```

Now you're getting an instance of the `LinkedInService` class. The beauty of this approach is that all the codes in other places remain the same. You only have to update the `bind()` method call. As long as a class implements the `SocialMediaServiceInterface` it can be bound to the container as a valid social media service.

# Service Container and Service Providers

Laravel comes with a more powerful IoC container, known as the service container. You can rewrite the example from the previous section using the service container as follows:

```php

<?php

// routes/web.php

use App\Container;
use Illuminate\Support\Facades\Route;

Route::get('/', function () {
    app()->bind('App\Interfaces\SocialMediaService', function() {
        return new App\Services\LinkedInService();
    });

    ddd(app()->make('App\Interfaces\SocialMediaService'));

    // App\Services\LinkedInService {#262}
});

```

In every Laravel application, the `app` instance is the container. The `app()` helper returns an instance of the container.

Just like your custom container, the Laravel service container has a `bind()` and a `make()` method used for binding services and retrieving services.

There is another method called `singleton()`. When you bind a class as a singleton, there can be only one instance of that class.

Let me show you an example. Update your code to make two instances of a given class.

```php

<?php

// routes/web.php

use App\Container;
use Illuminate\Support\Facades\Route;

Route::get('/', function () {
    app()->bind('App\Interfaces\SocialMediaService', function() {
        return new App\Services\LinkedInService();
    });

    ddd(app()->make('App\Interfaces\SocialMediaService'), app()->make('App\Interfaces\SocialMediaService'));

    // App\Services\LinkedInService {#262}
    // App\Services\LinkedInService {#269}
});

```

Indicated by the numbers (#262 and #269) at the end, the two instances are different from each other. If you bind the class as a singleton, you'll see something different.

```php

<?php

// routes/web.php

use App\Container;
use Illuminate\Support\Facades\Route;

Route::get('/', function () {
    app()->singleton('App\Interfaces\SocialMediaService', function() {
        return new App\Services\LinkedInService();
    });

    ddd(app()->make('App\Interfaces\SocialMediaService'), app()->make('App\Interfaces\SocialMediaService'));

    // App\Services\LinkedInService {#262}
    // App\Services\LinkedInService {#262}
});

```

As you can see, now the two instances are numbered the same indicating they are the same instance.

Now that you've learned about the `bind()`, `singleton()`, and `make()` methods, The next thing you will have to learn about is where to put these method calls. You certainly can not put them in your controllers or models.

The right place to put your bindings is service providers. Service providers are classes that reside inside the `app/Providers` directory. These are the bedrock of the framework, responsible for bootstrapping the majority of the framework services.

Every new Laravel project comes with **five** service provider classes by default. Among them, the `AppServiceProvider` class comes as empty by default with two methods. They are `register()` and `boot()`.


The `register()` method is used for registering new services to the application. This is where you put your `bind()` and `singleton()` method calls.

```php

<?php

namespace App\Providers;

use App\Interfaces\SocialMediaService;
use Illuminate\Support\ServiceProvider;

class AppServiceProvider extends ServiceProvider
{
    /**
     * Register services.
     *
     * @return void
     */
    public function register()
    {
        $this->app->bind(SocialMediaService::class, function() {
            return new \App\Services\LinkedInService;
        });
    }

    /**
     * Bootstrap services.
     *
     * @return void
     */
    public function boot()
    {
        //
    }
}

```

Inside providers, you get access to the container by simply writing `$this->app` instead of calling the `app()` helper function. But you can do that as well.

The `boot()` method is used for the logic necessary to bootstrap the registered service. A good example is the `BroadcastingServiceProvider` class.

```php

<?php

namespace App\Providers;

use Illuminate\Support\Facades\Broadcast;
use Illuminate\Support\ServiceProvider;

class BroadcastServiceProvider extends ServiceProvider
{
    /**
     * Bootstrap any application services.
     *
     * @return void
     */
    public function boot()
    {
        Broadcast::routes();

        require base_path('routes/channels.php');
    }
}

```

As you can see, it makes a call to the `Broadcast::routes()` method and requires the `routes/channels.php` file, making the broadcasting routes active in the process.

For one or two simple bindings like in this example, you can use the `AppServiceProvider` but in the case of services that require more complex logic to be executed, you can create new service providers using the `php artisan make:provider <provider name>` command.

# The Full Picture

In the previous sections, I've introduced you to different concepts such as dependency injection, inversion of control, service container, service providers. By now you should have a solid idea of what the container is, how to bind classes to it and retrieve them when necessary. In this section, I'll show you how all these ideas work in harmony.

Let's go back to the `Publication` class you worked with a few sections ago. I hope you remember that the `Publication` class was previously dependent on the `TwitterService` class. But now that you've introduced interfaces to the project, let's update `Publication` to make use of that.

```php

<?php

namespace App;

use App\Models\Post;
use App\Interfaces\SocialMediaServiceInterface;

class Publication {
    protected $socialMediaService;

    public function __construct(SocialMediaServiceInterface $socialMediaService)
    {
        $this->socialMediaService = $socialMediaService;
    }

    public function publish(Post $post)
    {
        $post->publish();

        $this->socialize($post);
    }

    protected function socialize($post)
    {
        $this->socialMediaService->share($post);
    }

}

```

Now instead of depending on a single class, this `Publication` will accept any class that implements the `SocialMediaServiceInterface`.

You've also bound the `SocialMediaServiceInterface` to an implementation of the `LinkedInService` class, so executing `app()->make(SocialMediaServiceInterface::class);` should return an instance of the `LinkedInService` class.

One class that you haven't bound to the container however is the `Publication` class itself. But what if you execute `app()->make(Publication::class);` without binding it? Let's try this out.

```php

<?php

// routes/web.php

use App\Publication;
use Illuminate\Support\Facades\Route;

Route::get('/', function () {
    $publication = app()->make(Publication::class);

    ddd($publication);

    // App\Publication {#273 ▼
    //     #socialMediaService: App\Services\LinkedInService {#272}
    // }      
});

```

Turns out it works. Laravel has somehow successfully instantiated an instance of the `Publication` class without it being bound to the container.

At first glance, this may seem like magic. But in reality, this is the result of all the previously explained concepts working in harmony.

When Laravel encounters the `app()->make(Publication::class);` line, it goes looking for a corresponding entry in the container. When it doesn't find a binding this class, it looks at the class constructor.

```php
<?php

namespace App;

use App\Models\Post;
use App\Interfaces\SocialMediaServiceInterface;

class Publication {
    protected $socialMediaService;

    public function __construct(SocialMediaServiceInterface $socialMediaService)
    {
        $this->socialMediaService = $socialMediaService;
    }

    //

}

```

Laravel realizes that the `Publication` class has a dependency `$socialMediaService` of type `SocialMediaServiceInterface` and goes looking for this interface in the container.

```php

<?php

namespace App\Providers;

use Illuminate\Support\ServiceProvider;
use App\Interfaces\SocialMediaServiceInterface;

class AppServiceProvider extends ServiceProvider
{
    /**
     * Register any application services.
     *
     * @return void
     */
    public function register()
    {
        $this->app->bind(SocialMediaServiceInterface::class, function() {
            return new \App\Services\LinkedInService;
        });
    }

    //
}

```

Upon finding the desired entry, Laravel instantiates a new social media service, feeds that to the `Publication` class, and successfully creates a new instance.

It should be clear to you that Laravel can instantiate type-hinted dependencies automatically as long as they are not implementing any interface. Keeping this in mind, update your `routes/web.php` as follows:

```php

<?php

// routes/web.php

use App\Publication;
use Illuminate\Support\Facades\Route;

Route::get('/', function (Publication $publication) {
    ddd($publication);

    // App\Publication {#276 ▼
    //     #socialMediaService: App\Services\LinkedInService {#275}
    // }
});

```

And the project works just as expected. You see, due to the automatic resolution capability of the container you'll hardly ever take out instances from the container manually. As long as you're binding interfaces properly and type-hinting your dependencies, Laravel will do the heavy lifting.

# To Bind or Not To Bind

The final question to answer in this article when should you bind a service to the container. Given the container is capable of instantiating classes without the need for binding, do you even need to consider binding.

The answer should be clear by now but here it is - 

> You bind a class to the container when it implements an interface.

In the example above, the `LinkedInService` class implements an interface, so you have to bind the interface to an implementation of the class.

The `Publication` class, however, doesn't implement any interface. Its only dependency is an implementation of the `SocialMediaServiceInterface` which is already bound to the container. Hence binding this class was not necessary.

# Suggested Reads

* [Service Container (Laravel Docs)](https://laravel.com/docs/8.x/container)
* [Service Providers (Laravel Docs)](https://laravel.com/docs/8.x/providers)
* [Service Container (Symfony Docs)](https://symfony.com/doc/current/service_container.html)
* [Service Container Fundamentals (Laracasts)](https://laracasts.com/series/laravel-6-from-scratch/episodes/38)
* [Automatically Resolve Dependencies (Laracasts)](https://laracasts.com/series/laravel-6-from-scratch/episodes/39)
* [Service Providers are the Missing Piece (Laracasts)](https://laracasts.com/series/laravel-6-from-scratch/episodes/41)
* [Inversion of Control Containers and the Dependency Injection pattern (Martin Fowler)](https://martinfowler.com/articles/injection.html)
* [Inversion of Control – The Hollywood Principle (Alejandro Gervasio)](https://www.sitepoint.com/inversion-of-control-the-hollywood-principle/)

# Closing Thoughts

The service container is a tough concept to understand, there is no denying that. The fact that you don't use it directly very often, makes it even less important to many developers. But understanding the service container is one of the most important steps of mastering Laravel.

I hope I was able to make the various concepts regarding the service container a bit clearer to you. Knowing what's going on behind the scenes will help you in the long run.

From now on, you should be a tad bit more confident when type-hinting dependencies in your class constructors.