---
currentSection: v1
currentItem: TypeMapping
pageflow_prev_url: index.html
pageflow_prev_text: TypeMapping Introduction
pageflow_next_url: MapStrictTypeToMethodName.html
pageflow_next_text: MapStrictTypeToMethodName class
---

# MapDuckTypeToMethodName

<div class="callout info">
Since v1.2016060501
</div>

## Description

`MapDuckTypeToMethodName` is a [`TypeMapper`](../Interfaces/TypeMapper.html). It uses a variable's duck types and a dispatch table to map a variable onto a suitable method name.

<div class="callout info" markdown="1">
#### What Are Duck Types?

Duck types are the list of types that a variable can be safely treated as. They help you safely handle a wider range of input variables with less code.

For example, you don't need to write one method to handle `array`, another to handle `Traversable` objects, and a third to handle `stdClass` objects. You can write a single method to handle anything with the `Traversable` duck type, as long as that method uses `foreach()` loop.

Duck types are not compatible with the strict type declarations added in PHP 7.0. If you need to work with strict type declarations, use [`MapStrictTypeToMethodName`](MapStrictTypeToMethodName.html) instead.
</div>

## Public Interface

`MapDuckTypeToMethodName` has the following public interface:

```php
// MapDuckTypeToMethodName lives in this namespace
namespace GanbaroDigital\Polymorphism\V1\TypeMapping;

// our base classes and interfaces
use GanbaroDigital\Polymorphism\V1\Interfaces\TypeMapper;

class MapDuckTypeToMethodName implements TypeMapper
{
    /**
     * use an input item's data type to work out which method we should
     * call
     *
     * @param  mixed $item
     *         the item we want to dispatch
     * @param  array $dispatchTable
     *         the list of methods that are available
     * @param  string $fallback
     *         the value to return if there's no suitable entry for $item
     *         in $dispatchTable
     * @return string
     *         the name of the method to call
     */
    public function __invoke(
        $item,
        array $dispatchTable,
        $fallback = TypeMapper::FALLBACK_RESULT
    );

    /**
     * use an input item's data type to work out which method we should
     * call
     *
     * @param  mixed $item
     *         the item we want to dispatch
     * @param  array $dispatchTable
     *         the list of methods that are available
     * @param  string $fallback
     *         the value to return if there's no suitable entry for $item
     *         in $dispatchTable
     * @return string
     *         the name of the method to call
     */
    public static function using(
         $item,
         array $dispatchTable,
         $fallback = TypeMapper::FALLBACK_RESULT
    );
}
```

## How To Use

### Adding Polymorphism To Your Code

Here's an example of how to use `MapDuckTypeToMethodName`.

`TrimWhitespace` is a utility that will remove leading and trailing whitespace from variables. It will work with anything that we can iterate over, or that we can turn into a string.

The first step is to define a _dispatch table_:

```php
use GanbaroDigital\Polymorphism\V1\TypeMapping\MapDuckTypeToMethodName;

class TrimWhitespace
{
    private static $dispatchTable = [
        'Traversable' => 'trimFromTraversable',
        'string' => 'trimFromString'
    ];
}
```

The _dispatch table_ is a list of supported types, and the method we want to call for that type. It's a regular PHP array. The order of the entries in the dispatch table is not important.

<div class="callout info" markdown="1">
#### What Order Are Matches Made In?

`MapDuckTypeToMethodName` uses [`GetDuckTypes`](http://ganbarodigital.github.io/php-the-missing-bits/types/GetDuckTypes.html) internally. `GetDuckTypes` returns an ordered list of matching types. The list order is:

- most specific type - e.g. an object's class name
- substitution types - e.g. an object's parent classes and interfaces
- coercable types - e.g. `callable`, `numeric`, objects that are stringy, or strings that are `double` or `integer`
- generic types - e.g. `object`, `string`

`MapDuckTypeToMethodName` works down the list that `GetDuckTypes` provides until it finds a match in dispatch table that you've provided.

If there is no match, the `$fallback` is returned to you. You'll find more information about that further down this page.
</div>

If you look closely, you'll see that we haven't added any entry for PHP arrays at all. How's that going to work? `MapDuckTypeToMethodName` uses a variable's _duck types_ to make the match. `Traversable` is one of `array`'s duck types.

Next, we define the method that is polymorphic:

```php
    public static function from($item)
    {
        $methodToCall = MapDuckTypeToMethodName::using($item, static::$dispatchTable);
        return self::$methodToCall($item);
    }
```

This method does two things:

1. calls `MapDuckTypeToMethodName::using()` to find out which method to send `$item` to
2. sends `$item` to the method that has been chosen

That's it.

Note how it's `TrimWhitespace::from()` that calls the method, not `MapDuckTypeToMethodName`. This keeps your PHP stack size down, and it keeps your stack traces nice and clean in case the method you're calling (or the methods it calls) throw any exceptions.

With our polymorphic method built, we need to add the methods that are listed in the dispatch table:

```php
    private static function trimFromTraversable($item)
    {
        $retval = [];
        foreach ($item as $key => $value) {
            $retval[$key] = self::from($value);
        }
        return $retval;
    }

    private static function trimFromString($item)
    {
        // PHP will coerce into a string for us
        return trim($item);
    }
```

`::trimFromTraversable()` iterates over `$item`, and calls our polymorphic method `TrimWhitespace::from()` to trim the whitespace from each entry in `$item`. This gives us automatic support for trimming the whitespace from nested arrays, or objects that contain arrays.

<div class="callout warning" markdown="1">
#### Beware Of Recursive Data Structures

If you defined data structures that contain references back to themselves, then our `::trimFromTraversable()` example will loop forever. To find out if your data structure is recursive, try using `var_dump()` or `var_export()` to examine it.
</div>

`::trimFromString()` uses PHP's built-in `trim` function to strip the whitespace for us. If `$item` is a stringy object (ie an object that implements `::__toString()`), PHP will automatically coerce it into a string for us.

We're almost done. There's one thing left to deal with: what happens when there's no suitable entry for `$item` in our dispatch table.

When there's no match, `TypeMapper` objects return the value of the `$fallback` parameter. By default, this is the string `nothingMatchesTheInputType`. You can override it if you ever need to.

For our example `TrimWhitespace`, we're going to stick with the default, and add a method called `nothingMatchesTheInputType`:

```php
    private static function nothingMatchesTheInputType($item)
    {
        return $item;
    }
```

<div class="callout info" markdown="1">
It's for you to decide what the fallback method should do. The fallback method is part of your code.
</div>

For `TrimWhitespace`, we could throw an exception ... but is it really an error to call `TrimWhitespace` with a data type that we can't trim whitespace from? That's a judgement call.

Here's what the final class looks like:

```php
use GanbaroDigital\Polymorphism\V1\TypeMapping\MapDuckTypeToMethodName;

class TrimWhitespace
{
    /**
     * a mapping of types to methods
     * @type array
     */
    private static $dispatchTable = [
        'Traversable' => 'trimFromTraversable',
        'string' => 'trimFromString'
    ];

    /**
     * trim the leading and trailing whitespace from your variable
     *
     * @param  mixed $item
     *         the variable to be trimmed
     * @return array|string
     *         your data, with any whitespace trimmed
     */
    public static function from($item)
    {
        $methodToCall = MapDuckTypeToMethodName::using($item, static::$dispatchTable);
        return self::$methodToCall($item);
    }

    /**
     * trim the leading and trailing whitespace from an array
     * or a traversable object
     *
     * @param  array|object $item
     *         the variable to be trimmed
     * @return array
     *         the trimmed data
     */
    private static function trimFromTraversable($item)
    {
        $retval = [];
        foreach ($item as $key => $value) {
            $retval[$key] = self::from($value);
        }
        return $retval;
    }

    /**
     * trim the leading and trailing whitespace from a string
     *
     * @param  string $item
     *         the string to be trimmed
     * @return string
     *         the trimmed data
     */
    private static function trimFromString($item)
    {
        // PHP will coerce into a string for us
        return trim($item);
    }

    /**
     * called when we're asked to trim data from a datatype
     * that we do not know how to handle
     *
     * @param  mixed $item
     *         the item that we don't know how to trim
     * @return mixed
     *         will return the same data type as $item
     */
    private static function nothingMatchesTheInputType($item)
    {
        // we don't know how to process $item, so let's just
        // send back exactly what we received
        return $item;
    }

}
```

### Speeding Things Up

`MapDuckTypeToMethodName` can be an expensive operation - especially if you're using anything before PHP 7.0. It doesn't implement an internal cache at all, because the results returned are unique to each `$dispatchTable`.

Use a [caching dispatch table object](../DispatchTables/index.html) to speed up repeated calls to your polymorphic method:

```php
use GanbaroDigital\Polymorphism\V1\DispatchTables\AllPurposeDispatchTable;
use GanbaroDigital\Polymorphism\V1\TypeMapping\MapDuckTypeToMethodName;

class TrimWhitespace
{
    // this becomes a DispatchTable object
    private static $dispatchTable;

    // this is new, and is called once when the class is
    // autoloaded
    public static function initDispatchTable()
    {
        self::$dispatchTable = new AllPurposeDispatchTable(
            [
                'Traversable' => 'trimFromTraversable',
                'string' => 'trimFromString'
            ],
            new MapDuckTypeToMethodName
        );
    }

    // this is substantially changed from our previous example
    public static function from($item)
    {
        // instead of calling MapDuckTypeToMethodName directly, we let our new
        // DispatchTable do it for us
        //
        // it will only call MapDuckTypeToMethodName if it has no cached
        // result for $item
        $method = self::$dispatchTable->mapTypeToMethodName($item);

        // this is the same as our earlier example
        return self::$method($item);
    }

    // same as our first example
    private static function trimFromTraversable($item)
    {
        $retval = [];
        foreach ($item as $key => $value) {
            $retval[$key] = self::from($value);
        }
        return $retval;
    }

    // same as our first example
    private static function trimFromString($item)
    {
        // PHP will coerce into a string for us
        return trim($item);
    }

    // same as our first example
    private static function nothingMatchesTheInputType($item)
    {
        // we don't know how to process $item, so let's just
        // send back exactly what we received
        return $item;
    }
}

// NEW IN THIS EXAMPLE
// this will get called when the class is autoloaded
TrimWhitespace::initDispatchTable();
```

### Customising The Fallback Result

Are you adding multiple polymorphic methods to a single class? If you are, you'll probably need each polymorphic method to call a different fallback method when there's no match.

The third parameter to `MapDuckTypeToMethodName::using()` contains the value to return when there's no match. Simply pass in a different value:

```php
use GanbaroDigital\Polymorphism\V1\TypeMapping\MapDuckTypeToMethodName;

// hypothetical business logic class that supports refunding
// different types of purchases
//
// the logic is separate to the business entity so that we can load
// the correct logic for each market we operate in
class RefundPaymentUnitedKingdom implements RefundLogic
{
    // this is polymorphic
    public function isRefundable($item)
    {
        // the contents of $dispatchTable are not
        // important for this example
        $dispatchTable = ...;

        $method = MapDuckTypeToMethodName::using(
            $item,
            $dispatchTable,
            "notRefundable"
        );
        return $this->{$method}($item);
    }

    // this is the fallback method for isRefundable()
    private function notRefundable()
    {
        return false;
    }

    // this is polymorphic
    public function refundTerms($item)
    {
        // the contents of $dispatchTable are not
        // important for this example
        $dispatchTable = ...;

        $method = MapDuckTypeToMethodName::using(
            $item,
            $dispatchTable,
            "noRefundTerms"
        );
        return $this->{$method}($item);
    }

    // this is the fallback method for refundTerms()
    private function noRefundTerms()
    {
        return [];
    }
}
```

In the example above, `::isRefundable()` will call `::notRefundable()` if there's no match for `$item`. Similarly, `::refundTerms()` will call `::noRefundTerms()` if there is no match.

## Class Contract

Here is the contract for this class:

    GanbaroDigital\Polymorphism\V1\TypeMapping\MapDuckTypeToMethodName
     [x] Can instantiate
     [x] is TypeMapper
     [x] Can use as object
     [x] Can call statically
     [x] will match NULL
     [x] can match array as array
     [x] can match array as callable
     [x] can match array as Traversable
     [x] can match true as boolean
     [x] can match false as boolean
     [x] can match double as double
     [x] can match double as numeric
     [x] can match integer as integer
     [x] can match integer as numeric
     [x] can match object as object
     [x] can match object as parent
     [x] can match object as interface
     [x] can match object as callable
     [x] can match object as string
     [x] can match resource
     [x] can match string
     [x] can match string as callable
     [x] can match string as numeric
     [x] can match string as double
     [x] can match string as integer
     [x] can match string as classname
     [x] can match string as interface
     [x] returns nothingMatchesTheInputType when no match found
     [x] can change the default fallback when no match found

Class contracts are built from this class's unit tests.

<div class="callout success">
Future releases of this class will not break this contract.
</div>

<div class="callout info" markdown="1">
Future releases of this class may add to this contract. New additions may include:

* clarifying existing behaviour (e.g. stricter contract around input or return types)
* add new behaviours (e.g. extra class methods)
</div>

<div class="callout warning" markdown="1">
When you use this class, you can only rely on the behaviours documented by this contract.

If you:

* find other ways to use this class,
* or depend on behaviours that are not covered by a unit test,
* or depend on undocumented internal states of this class,

... your code may not work in the future.
</div>

## Notes

None at this time.
