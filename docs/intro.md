# Aura.Di

The Aura.Di package provides a dependency injection container system with the
following features:

- constructor and setter injection

- explicit and implicit auto-resolution of typehinted constructor parameter values

- configuration of setters across interfaces and traits

- inheritance of constructor parameter and setter method values

- lazy-loaded services, values, and instances

- instance factories

Fully describing the nature and benefits of dependency injection, while
desirable, is beyond the scope of this document. For more information about
"inversion of control" and "dependency injection" please consult
<http://martinfowler.com/articles/injection.html> by Martin Fowler.

Finally, please note that this package is intended for use as a **dependency injection** system, not as a **service locator** system. If you use it as a service locator, that's bad, and you should feel bad.

**WARNING:** This 3.x package is still under heavy development. Use at your own risk. You may wish to review the 2.x package for a stable version.

## Instantiation

We instantiate a _Container_ like so:

```php
<?php
use Aura\Di\ContainerBuilder;
$builder = new ContainerBuilder();
$di = $builder->newInstance();
?>
```

We can then set shared services on the container. Additionally, we can add default constructor parameters and setter method values to be used on new class instances, along with other values as well.

## Setting And Getting Services

A "service" is an object stored in the _Container_ under a unique name. Any time you `get()` the named service, you always get back the same object instance.

```php
<?php
// define the Example class
class Example
{
    // ...
}

// set the service
$di->set('service_name', new Example());

// get the service
$service1 = $di->get('service_name');
$service2 = $di->get('service_name');

// the two service objects are the same
var_dump($service1 === $service2); // true
?>
```

That usage is great if we want to create the _Example_ instance at the same time we set the service. However, we generally want to create the service instance at the moment we *get* it, not at the moment we *set* it.

The technique of delaying instantiation until `get()` time is called "lazy loading." To lazy-load an instance, use the `lazyNew()` method on the _Container_ and give it the class name to be created:

```php
<?php
// set the service as a lazy-loaded new instance
$di->set('service_name', $di->lazyNew('Example'));
?>
```

Now the service is created only when we we `get()` it, and not before. This lets us set as many services as we want, but only incur the overhead of creating the instances we actually use.

## Constructor Injection

When we use the _Container_ to instantiate a new object, we often need to inject (i.e., set) constructor parameter values in various ways.

### Default Parameter Values

We can define default values for constructor parameters using the `$di->params` array on the _Container_.

Let's look at a class that takes some constructor parameters:

```php
<?php
class ExampleWithParams
{
    protected $foo;
    protected $bar;
    public function __construct($foo, $bar)
    {
        $this->foo = $foo;
        $this->bar = $bar;
    }
}
?>
```

If we were to try to set a service using `$di->lazyNew('ExampleWithParams')`, the instantiation would fail. The `$foo` param is required, and the _Container_ does not know what to use for that value.

To remedy this, we tell the _Container_ what values to use for each _ExampleWithParams_ constructor parameter by name using the `$di->params` array:

```php
<?php
$di->params['ExampleWithParams']['foo'] = 'foo_value';
$di->params['ExampleWithParams']['bar'] = 'bar_value';
?>
```

Now when a service is defined with `$di->lazyNew('ExampleWithParams')`, the instantiation will work correctly. Each time we create an _ExampleWithParams_ instance through the _Container_, it will apply the `$di->params['ExampleWithParams']` values.

### Instance-Specific Parameter Values

If we want to override the default `$di->params` values for a specific new instance, we can pass a `$params` array as the second argument to `lazyNew()` to merge with the default values. For example:

```php
<?php
$di->set('service_name', $di->lazyNew(
    'ExampleWithParams',
    [
        'bar' => 'alternative_bar_value',
    ]
));
?>
```

This will leave the `$foo` parameter default in place, and override the `$bar` parameter value, for just that instance of the _ExampleWithParams_.

### Lazy-Loaded Services As Parameter Values

Sometimes a class will need another service as one of its parameters. By way of example, the following class needs a database connection:

```php
<?php
class ExampleNeedsService
{
    protected $db;
    public function __construct($db)
    {
        $this->db = $db;
    }
}
?>
```

To inject a shared service as a parameter value, use `$di->lazyGet()` so that the service object is not created until the _ExampleNeedsService_ object is created:

```php
<?php
$di->params['ExampleNeedsService']['db'] = $di->lazyGet('db_service');
?>
```

This keeps the service from being created until the very moment it is needed. If we never instantiate anything that needs the service, the service itself will never be instantiated.

### Auto-Resolution Of Parameter Values

Auto-resolution is disabled by default. You can enable auto-resolution by calling `$di->setAutoResolve(true)`.

#### How Auto-Resolution Works

If there is no `$di->params` value for a parameter, the _Container_ will fill in the constructor default value.

If the parameter is typehinted as an `array` but there is no `$di->params` value and also no default value, the _Container_ will fill in an empty `[]`.

If the parameter is typehinted to a class but there is no `$di->params` value for that parameter and also no default value, the _Container_ will fill in a `lazyNew()` call to the typehinted class.

For example, look at the following class; it has a parameter with a default value, a parameter typehinted as an array with no default, and a parameter typehinted to a class with no default:

```php
<?php
class ExampleForAutoResolution
{
    public function __construct($foo = 'bar', array $baz, Example $dib)
    {
        // ...
    }
}
?>
```

For each relevant `$di->params['ExampleForAutoResolution']` element that is missing, the _Container_ will auto-resolve the missing elements to these values:

```php
<?php
$di->params['ExampleForAutoResolution']['foo'] = 'bar';
$di->params['ExampleForAutoResolution']['baz'] = [];
$di->params['ExampleForAutoResolution']['dib'] = $di->lazyNew('Example');
?>
```

We can set any combination of these explicitly, and those that are not explicitly set will be filled in automatically for us.

(Note that we cannot auto-resolve an `array` typehint; such typehints are always resolved to an empty `[]` when no default value is present.)

#### Explicitly Directing Auto-Resolution Typehints

We can direct the auto-resolution of class-typehinted constructor parameters by using the `$di->types` array.  This allows us to avoid having to specify `$di->params` for every typehinted constructor parameter in every class.  (Note that we can still specify explicit params on a specific class to override the auto-resolution.)

We can specify auto-resolution to a new instance of a class of our choosing ...

```php
<?php
// auto-resolve all 'ExampleInterface' typehints to a new 'Example' instance
$di->types['ExampleInterface'] = $di->lazyNew('Example');

// auto-resolve all 'ExampleParent' typehints to a different concrete class
$di->types['ExampleParent'] = $di->lazyNew('ExampleChild');
?>
```

... or to a shared service (aka singleton) of our own choosing:

```php
<?php
// auto-resolve all 'Db' and 'DbInterface' typehints to a shared service
$di->types['Db'] = $di->lazyGet('db_service');
$di->types['DbInterface'] = $di->lazyGet('db_service');
?>
```

## Setter Injection

This package supports setter injection in addition to constructor injection. (These can be combined as needed.)

### Setter Method Values

After the _Container_ constructs a new instance of an object, we can specify that certain methods should be called with certain values immediately after instantiation by using the `$di->setter` array.  Say we have class like the following:

```php
<?php
class ExampleWithSetter
{
    protected $foo;

    public function setFoo($foo)
    {
        $this->foo = $foo;
    }
}
?>
```

We can specify that, by default, the `setFoo()` method should be called with a specific value after construction like so:

```php
<?php
$di->setter['ExampleWithSetter']['setFoo'] = 'foo_value';
?>
```

The value can be any valid value: a literal, a call to `lazyNew()` or `lazyGet()`, and so on.

Note, however, that auto-resolution *does not apply* to setter methods. This is because the _Container_ does not know which methods are setters and which are "normal use" methods.

Note also that this works only with explicitly-defined setter methods. Setter methods that exist only via magic `__call()` will not be honored.

### Setters In Traits and Interfaces

If a class uses a setter method via a trait, or implements an interface with a setter method, we can specify the default value for that setter method in relation to the trait or interface. That value will then be applied by default in every class that uses that trait or implements that interface.

For example, let's say we have this trait, interface, and class:

```php
<?php
trait ExampleFooTrait
{
    public function setFoo($foo)
    {
        $this->foo = $foo;
    }
}

interface ExampleBarInterface
{
    public function setBar($bar);
}

class ExampleWithTraitAndInterface implements ExampleBarInterface
{
    use ExampleFooTrait;

    protected $foo;
    protected $bar;

    public function setBar($bar)
    {
        $this->bar = $bar;
    }
}
?>
```

We then define the default setter method values on the trait and interface:

```php
<?php
$di->setter['ExampleFooTrait']['setFoo'] = 'foo_value';
$di->setter['ExampleBarInterface']['setBar'] = 'bar_value';
?>
```

When we call `$di->lazyNew('ExampleWithTraitAndInterface')`, those setter methods will be called by the _Container_ with those values.

Note that if we have class-specific `$di->setter` values, those will take precedence over the trait and interface setter values.

### Instance-Specific Setter Values

As with constructor injection, we can note instance-specific setter values to use in place of the defaults. We do so via the third argument to `$di->lazyNew()`. For example:

```php
<?php
$di->set('service_name', $di->lazyNew(
    'ExampleWithSetters',
    [], // no $params overrides
    [
        'setFoo' => 'alternative_foo_value',
    ]
));
?>
```

## Lazy Values

Sometimes we know that a parameter needs to be specified, but we don't know what it will be until later.  Perhaps it is the result of looking up an API key from an environment variable. In these and other cases, we can tell a constructor parameter or setter method to use a "lazy value" and then specify that value elsewhere.

For example, we can configure the _Example_ constructor parameters to use lazy values like so:

```php
<?php
$di->params['Example']['foo'] = $di->lazyValue('fooval');
$di->params['Example']['bar'] = $di->lazyValue('barval');
?>
```

We can then specify at some later time the values of `fooval` and `barval` using the `$di->values` array:

```php
<?php
$di->values['fooval'] = 'lazy value for foo';
$di->values['barval'] = 'lazy value for bar';
?>
```

## Lazy Include and Require

Occasionally we will need to `include` a file that returns a value, such as data file that returns a PHP array:

```php
<?php
// /path/to/data.php
return [
    'foo' => 'bar',
    'baz' => 'dib',
    'zim' => 'gir'
];
?>
```

We could set a constructor parameter or setter method value to `include "/path/to/data.php"`, but that would cause the file to be read filesystem at that moment, instead of at instantiation time.  To lazy-load a file as a value, call `$di->lazyInclude()` or `$di->lazyRequire()` (depending on your preference for warning levels).

```php
<?php
$di->params['ExampleNeedsInclude']['data'] = $di->lazyInclude('/path/to/data.php');
$di->params['ExampleNeedsRequire']['data'] = $di->lazyRequire('/path/to/data.php');
?>
```

## Generic Lazy Calls

It may be that we have a complex bit of logic we need to execute for a value. If none of the existing `$di->lazy*()` methods meet our needs, we can wrap an anonymous function or other callable in a `lazy()` call, and the callable's return will be used as the value.

```php
<?php
$di->params['Example']['foo'] = $di->lazy(function () {
    // complex calculations, and then:
    return $result;
});
?>
```

Beware of relying on this too much; if we do, it probably means we need to separate our configuration concerns better than we are currently doing.

## Instance Factory Objects

Occasionally, a class will need to receive not just an instance, but a factory that is capable of creating a new instance over and over.  For example, say we have a class like the following:

```php
<?php
class ExampleNeedsFactory
{
    protected $struct_factory;

    public function __construct($struct_factory)
    {
        $this->struct_factory = $struct_factory;
    }

    public function getStruct(array $data)
    {
        $struct = $this->struct_factory->__invoke($data);
        return $struct;
    }
}

class ExampleStruct
{
    public function __construct(array $data)
    {
        foreach ($data as $key => $val) {
            $this->$key = $val;
        }
    }
}
?>
```

We can inject an _Factory_ that creates only _ExampleStruct_ objects using `$di->newFactory()`.

```php
<?php
$di->params['ExampleNeedsFactory']['struct_factory'] = $di->newFactory('ExampleStruct');
?>
```

Note that the arguments passed to the factory `__invoke()` method will be passed to the underlying instance constructor sequentially, not by name. This means the `__invoke()` method works more like the native `new` keyword, and not like `$di->lazyNew()`.  These arguments override any `$di->params` values that have been set for the class being factoried; without the overrides, all existing `$di->params` values for that class will be honored. (Values from `$di->setter` for the class will also be honored, but cannot be overriddden.)

Do not feel limited by the _Factory_ implementation. We can create and inject factory objects of our own if we like. The _Factory_ returned by the `$di->newFactory()` method is an occasional convenience, nothing more.

## Inheritance Of Parent Values

Whether by constructor parameters or setter methods, each class "inherits" the values of its parents by default. This means we can set a value on a parent class, and the child class will use it (that is, unless we set an overriding value on the child class).

For example, let's say we have this parent class and this child class:

```php
<?php
class ExampleParent
{
    protected $foo;
    protected $bar;

    public function __construct($foo)
    {
        $this->foo = $foo;
    }

    public function setBar($bar)
    {
        $this->bar = $bar;
    }
}

class ExampleChild extends ExampleParent
{
    protected $baz;

    public function setBaz($baz)
    {
        $this->baz = $baz;
    }
}
?>
```

We can specify the default values for every class that extends _ExampleParent_ through the `$di->params` and `$di->setter` values for the _ExampleParent_.

```php
<?php
$di->params['ExampleParent']['foo'] = 'parent_foo';
$di->setter['ExampleParent']['setBar'] = 'parent_bar';
?>
```

When we call `$di->lazyNew('ExampleChild')`, the child class will have inherited the defaults from the parent.

We can always override the inherited values by specifying them for the child class directly:

```php
<?php
$di->params['ExampleChild']['foo'] = 'child_foo';
$di->setter['ExampleChild']['setBaz'] = 'child_baz';
?>
```

Note that classes extended from the child class will then inherit those new values. In this way, constructor parameter and setter method values are propagated down the inheritance hierarchy.


## Container Builder and Config Classes

The _ContainerBuilder_ helps to build _Container_ objects from _ContainerConfig_ classes and pre-existing service objects. It works using a [two-stage configuration system](http://auraphp.com/blog/2014/04/07/two-stage-config/).

The two stages are "define" and "modify". In the "define" stage, the _ContainerConfig_ object defines constructor parameter values, setter method values, services, and so on. The _ContainerBuilder_ then locks the _Container_ so that these definitions cannot be changed, and begins the "modify" stage. In the "modify" stage, we may `get()` services from the _Container_ and modify them programmatically if needed.

To build a fully-configured _Container_ using the _ContainerBuilder_, we do something like the following:

```php
<?php
use Aura\Di\ContainerBuilder;

// pre-existing service objects as ['service_name' => $object_instance]
$services = [];

// config classes to call define() and modify() on
$config_classes = [
    'Aura\Cli\_Config\Common',
    'Aura\Router\_Config\Common',
    'Aura\Web\_Config\Common',
];

// use the builder to create a container
$container_builder = new ContainerBuilder();
$di = $container_builder->newConfiguredInstance(
    $services,
    $config_classes
);
?>
```

A configuration class looks like the following:

```php
<?php
namespace Vendor\Package\_Config;

use Aura\Di\Container;
use Aura\Di\ContainerConfig;

class Common extends ContainerConfig
{
    public function define(Container $di)
    {
        $di->set('log_service', $di->lazyNew('Logger'));
        $di->params['Logger']['dir'] = '/path/to/logs';
    }

    public function modify(Container $di)
    {
        $log = $di->get('log_service');
        $log->debug('Finished config.');
    }
}
?>
```

Here are some example _ContainerConfig_ classes from earlier Aura packages:

- [Aura.Cli](https://github.com/auraphp/Aura.Cli/blob/2.0.0/config/Common.php)
- [Aura.Html](https://github.com/auraphp/Aura.Html/blob/2.0.0/config/Common.php)
- [Aura.Router](https://github.com/auraphp/Aura.Router/blob/2.0.0/config/Common.php)
- [Aura.View](https://github.com/auraphp/Aura.View/blob/2.0.0/config/Common.php)

## Conclusion

If we construct our dependencies properly with params, setters, services, and factories, we will only need to get one object directly from the _Container_ in our bootstrap file. All object creation will then occur within _Container_ itself or the various factory objects. We will never need to use the _Container_ itself in any of our application objects.