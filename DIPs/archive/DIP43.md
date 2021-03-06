# D/Objective-C

| Section         | Value                                          |
|-----------------|------------------------------------------------|
| DIP:            | 43                                             |
| Author:         | Michel Fortin, Jacob Carlborg                  |
| Implementation: | <https://github.com/dlang/dmd/pull/4287>       |
|                 | <https://github.com/dlang/dmd/pull/4321>       |
|                 | <https://github.com/dlang/druntime/pull/1111>  |
|                 | <https://github.com/dlang/dlang.org/pull/1129> |
| Status:         | Implemented                                    |

## Abstract

This document is an overview of the extensions D/Objective-C brings to the D
programming language. It assumes some prior knowledge of
[Objective-C](http://developer.apple.com/library/mac/#documentation/Cocoa/Conceptual/ObjectiveC/Introduction/introObjectiveC.html).

*Note: Some parts of this document describe features which are not yet
implemented and are very much subject to change. Unimplemented sections of this
**document** are marked as such.*

### Links

* [Objective-C](http://developer.apple.com/library/mac/#documentation/Cocoa/Conceptual/ObjectiveC/Introduction/introObjectiveC.html)

## Description

### Rationale

Currently it's very cumbersome and verbose to use Objective-C libraries from D.
The reason to use these libraries is to be able to use the system frameworks on
Mac OS X and iOS. This proposal adds language extensions to make it
significantly easier to interact and create libraries compatible with
Objective-C.

### Using an existing Objective-C class

To use an existing Objective-C class, we must first write a declaration for
that class, and we must mark this class as coming from Objective-C. Here is an
abbreviated declaration for class `NSComboBox`:

``` d
extern (Objective-C)
class NSComboBox : NSTextField
{
    private ObjcObject _dataSource;
    ...
}
```

This declaration will not emit any code because it was tagged as `extern`
`(Objective-C)`, but it will let know to the compiler that the `NSComboBox`
class exists and can be used. Since `NSComboBox` derives from `NSObject`, the
`NSObject` declaration must also be reachable or we'll get an error.

Declaring members variables of the class is important. Even if we don't plan on
using them, they are needed to properly calculate the size of derived classes.

#### Declaring Instance Methods

Objective-C uses a syntax that greatly differs from D when it comes to calling
member functions -- instance methods and class methods in Objective-C parlance.
In Objective-C, a method is called using the following syntax:

``` objc
[comboBox insertItemWithObjectValue:val atIndex:idx];
```

This will call the method `insertItemWithObjectValue:atIndex:` on the object
`comboBox` with two arguments: `val` and `idx`.

To make Objective-C methods accessible to D programs, we need to map them to a
D function name. This is accomplished by declaring a member function and giving
it a selector:

``` d
extern (Objective-C)
class NSComboBox : NSTextField
{
    private void* _dataSource;

    void insertItem(ObjcObject object, NSInteger value) @selector("insertItemWithObjectValue:atIndex:");
}
```

`@selector` is a compiler recognized UDA (User Defined Attribute) declared in
`core.attribute`. It's publicly imported in the `object` module and therefore
no explicit import is necessary.

Now we can call the method in our D program as if it was a regular member function:

``` d
comboBox.insertItem(val, idx);
```

#### Overloading

Objective-C does not support function overloading, which makes it impossible to
have two methods with the same name. D supports overloading, and we can take
advantage of that in a class declaration:

``` d
extern (Objective-C)
class NSComboBox : NSTextField
{
    private void* _dataSource;

    void insertItem(ObjcObject object, NSInteger value) @selector("insertItemWithObjectValue:atIndex:");
    void insertItem(ObjcObject object) @selector("insertItemWithObjectValue:");
}

comboBox.insertItem(val, idx); // calls insertItemWithObjectValue:atIndex:
comboBox.insertItem(val);      // calls insertItemWithObjectValue:
```

### Defining a Subclass

Creating a subclass from an existing Objective-C class is easy, first we must
make sure the base class is declared:

``` d
extern (Objective-C)
class NSObject
{
    ...
}
```

Then we write a derived class as usual:

``` d
class WaterBucket : NSObject
{
    float volume;

    void evaporate(float celcius)
    {
        if (celcius > 100)  volume -= 0.5 * (celcius - 100);
    }
}
```

WaterBucket being a class derived from an Objective-C class, it automatically
becomes an Objective-C class itself. We can now pass instances of WaterBucket
to any function expecting an Objective-C object.

Note that no Objective-C selector name was specified for the `evaporate`
function above. In this case, the compiler will generate one. If we need the
function to have a specific selector name, then we must write it explicitly:

``` d
void evaporate(float celcius) @selector("evaporate:")
{
    if (celcius > 100)  volume -= 0.5 * (celcius - 100);
}
```

If however we were overriding a function present in the base class, or
implementing a function from an interface, the Objective-C selector would be
inherited.

#### Constructors

To create a new Objective-C object in Objective-C, one would call the allocator
function and then the initializer:

``` objc
NSObject *o = [[NSObject alloc] init];
```

In D, we do this instead:

``` d
auto o = new NSObject();
```

The `new` operator knows how to allocate and initialize an Objective-C object,
it only need helps to find the right selector for a given constructor. When
declaring an Objective-C class, we can map constructor to selector names:

``` d
extern (Objective-C)
class NSSound : NSObject
{
    this(NSURL url, bool byRef) @selector("initWithContentsOfURL:byReference:");
    this(NSString path, bool byRef) @selector("initWithContentsOfFile:byReference:");
    this(NSData data) @selector("initWithData:");
}
```

Like for member functions, omitting the selector will make the compiler
generate one. But if a constructor is inherited from a base class or implements
a constructor defined in an interface, it'll inherit that selector instead.

#### Properties

When not given explicit selectors, property functions are given the appropriate
method names so they can participate in key-value coding.

``` d
class Value : NSObject
{
    @property BigInt number();
    @property void number(BigInt v);
    @property void number(int v);
}
```

Given the above code, the compiler will use the selector `number` for the
getter, `setNumber:` for the setter having the same parameter type as the
getter, and the second alternate setter will get the same compiler-generated
selector as a normal function.

### Objective-C Protocols

Protocols in Objective-C are mapped to interfaces in D. This declares an Objective-C protocol:

``` d
extern (Objective-C)
interface NSCoding
{
    void encodeWithCoder(NSCoder aCoder) @selector("encodeWithCoder:");
    this(NSCoder aDecoder) @selector("initWithCoder:");
}
```

Unlike regular D interfaces, we can define a constructor in an Objective-C protocol.

The protocol than then be implemented in any Objective-C class:

``` d
class Cell : NSObject, NSCoding
{
    int value;

    void encodeWithCoder(NSCoder aCoder)
    {
        aCoder.encodeInt(value, "value");
    }

    this(NSCoder aDecoder)
    {
        value = aDecoder.decodeInt("value");
    }
}
```

{Note: We probably need support for @optional interface methods too.}

### Class Methods

Each class in Objective-C is an object in itself that contains a set of methods
that relates to the class itself, with no access to instances of that class.
The D equivalent is to use a static member function:

``` d
extern (Objective-C)
class NSSound : NSObject
{
    static NSSound soundNamed(NSString *name) @selector("soundNamed:");
}
```

There is one key difference from a regular D static function however.
Objective-C class methods are dispatched dynamically on the class object, so
they have a `this` reference to the class they're being called on. `this` might
be a pointer to a class derived from the one our function was defined in, and
through it we can call a static function from that derived class if it
overrides one in the current class. Here is an example:

``` d
class A : NSObject
{
    static void name() { writeln("A"); }
    static void writeName() { writeln("My name is ", name()); }
}

class B : A
{
    static void name() { writeln("B"); }
}

B.writeName(); // prints "My name is B"
```

This is not possible with regular static functions in D.

#### Class References

In Objective-C, you can get a reference to a class by calling the `class` method:

``` objc
[instance class]; // return the class object for instance
[NSObject class]; // return the class object for the NSObject type
```

This works similarly in D:

``` d
instance.class; // get the class object for instance
NSObject.class; // get the class object for the NSObject type
```

The only difference is that D is strongly-typed, which means that `x.class`
returns a different type depending on the type of `x`.

Inside an instance method, use `this.class` to get the current class object;
you cannot omit `this` like you can for regular members as it would be
ambiguous for the parser.

There is no `classinfo` property for Objective-C objects.

### Class Extensions (also known as Categories) {unimplemented}

With Objective-C it is possible for different compilation units, and even
different libraries, to define new methods that will apply to existing classes.

``` d
extern (Objective-C)
class NSString : NSObject
{
    wchar characterAtIndex(size_t index) @selector("characterAtIndex:");
    @propety size_t length() @selector("length");
}

extern (Objective-C)
__classext LastCharacter : NSString
{
    wchar lastCharacter() @property;
}

unittest
{
    NSString s = "hello";
    assert(s.lastCharacter == 'o');
}
```

The `__classext` `LastCharacter` `:` `NSString` syntax maps to an Objective-C
class extension named `LastCharacter` adding methods to the `NSString` class.
Methods in the extension are dispatched dynamically, so you can override them
in a subclass of `NSString`, or in an extension of that subclass.

Having two extensions defining a function with the same selector will make the
Objective-C runtime use one of the two implementations in both cases.

{Question: should we mangle the extension name in the selector to avoid
conflicts? This would transparently implement Apple's recommendation that
methods in third-party extensions should use a prefix to avoid clashes with
future versions of the extended class and other extensions.}

### `NSString` Literals

D string literals are changed to NSString literals whenever the context
requires it. The following Objective-C code:

``` objc
NSString *str = @"hello";
```

becomes even simpler:

``` d
NSString str = "hello";
```

Automatic conversion only works for strings literals. If the string comes from
a variable, you'll need to construct the `NSString` object yourself.

### Selector Literals

When you need to express a selector, in Objective-C you use the `@selector` keyword:

``` objc
SEL sel = @selector(hasSuffix:);
```

In D, selectors are type-safe. To create a selector type, you must know the
return type and the parameter type this selector should have. You can then

``` d
BOOL __selector(NSString) sel = &NSString.hasSuffix;
```

A selector type can be used just like a delegate, with one difference. When
calling a selector, you need to add the object this selector applies to as the
first argument:

``` d
NSString s = "hello world";
sel(s, "world"); // same as s.hasSuffix("world")
```

### Protocol References

When you need to get a reference to a protocol, in Objective-C you use the
`@protocol` keyword:

``` objc
Protocol *p = @protocol(NSCoding);
```

In D, you use the `protocolof` property of the interface:

``` d
Protocol p = NSCoding.protocolof;
```

### Interface Builder Attributes {unimplemented}

The `@IBAction` attribute forces the compiler generate a function selector
matching the name of the function, making the function usable as an action in
Interface Builder and elsewhere.

The `@IBOutlet` attribute mark fields that should be available in Interface Builder.

``` d
class Controller : NSObject
{
    @IBOutlet NSTextField textField;

    @IBAction void clearField(NSButton sender)
    {
        textField.stringValue = "";
    }
}
```

### Special Considerations

#### Casts

The `cast` operator works the same as for regular D objects: if the object you
try to cast to is not of the right type, you will get a `null` reference.

``` d
NSView view = cast(NSView)object;

// produce the same result as:
NSView view = ( object && object.isKindOfClass(NSView.class) ? object : null );
```

For interfaces, the cast is implemented similarly:

``` d
NSCoding coding = cast(NSCoding)object;

// produce the same result as:
NSCoding coding = ( object && object.conformsToProtocol(NSCoding.protocolof) ? object : null );
```

The compiler will not emit any runtime check when casting to a base type.

#### `NSObject` vs. `ObjcObject` vs. `id`

There are two `NSObject` in Objective-C: `NSObject` the protocol and `NSObject`
the class. Not all classes are derived from the `NSObject` class, but they all
implement the `NSObject` protocol.

In D having, an interface and a class with the same name is less practical. So the `NSObject` protocol is mapped to the `ObjcObject` interface instead.

Because all Objective-C objects implement `ObjcObject` (the `NSObject` protocol), `ObjcObject` is used as the base type to hold a generic Objective-C object instead. The Objective-C language uses `id` for that purpose, but `id` cannot work in D because the correct mapping of selectors requires that we know the class or interface declaration.

So if you have a generic Objective-C object and you need to call one of its functions, you must first cast it to the right type, like this:

``` d
void showWindow(ObjcObject obj)
{
    if (auto window = cast(NSWindow)obj)
        window.makeKeyAndOrderFront();
}
```

#### Memory Management {unimplemented}

Only the reference-counted variant of Objective-C is supported, but reference
counting is automated which makes things much easier.

Assigning an Objective-C object to a variable will automatically call the
`retain` function to increase the reference count of the object, and clearing a
variable will call the `release` function on the reference object. Returning a
variable from a function will call the `autorelease` function.

``` d
auto a = textField.stringValue; // implicit a.retain()
auto b = a;                     // implicit b.retain()
b = null;                       // implicit b.release()
a = null;                       // implicit a.release()
```

The compiler can perform flow analysis when optimizing to elide unnecessary
calls to retain and release.

Functions in `extern` `(Objective-C)` class or interface declarations that
return a retained object reference must be marked with the `@retained`
attribute. The `@retained` attribute is inherited when overriding a function.
Most functions do not need this since they return autoreleased objects.

``` d
interface NSCopying
{
    @retained
    ObjcObject copyWithZone(NSZone* zone) @selector("copyWithZone:");
}
```

Note that casting an Objective-C object reference to some other pointer type
will break this mechanism. `retain` and `release` must be called manually in
those cases.

To create a "weak" object reference that does not change the reference count
and automatically becomes `null` when the referenced object is destroyed, use
the `WeakRef` template in the `objc` module. This is needed to break circular
references that would prevent memory from being deallocated.

{Note: need to check how to implement auto-nulling `WeakRef` efficiently.}

Member variables of Objective-C classes defined in a D module are managed by
the garbage collector as usual.

{Note: need to check how to implement this with Apple's Modern Objective-C
runtime.}

#### Null Objects {unimplemented}

Because of the way the Objective-C runtime handle dynamic dispatch, calling a
function on a `null` Objective-C object does nothing and return a zero value if
the function returns an integral type, or `null` for a pointer type. Struct
return values can contain garbage however.

**Do not count on that behavior in D.** While a D compiler will use the
Objective-C runtime dispatch mechanism whenever it can, it might also call
directly or inline the function when possible.

As a convenience to detect calls to `null` objects, you can use the
`-objcnullcheck` command line directive to make the compiler emit instructions
that check for `null` before each call to an Objective-C method and throw when
it encounters `null`.

{Question: Is disallowing calls on `null` objects desirable? How can we ensure
memory-safety for struct return values?}

### Applying D attributes

You can apply D attributes to Objective-C methods as usual and they'll have the
same effect as on any D function.

``` d
abstract, final
pure, nothrow
@safe, @trusted, @system
```

Type modifiers such as `const`, `immutable`, and `shared` can also be used on
Objective-C classes.

#### Design by Contract, Unit Tests

D features such as `unittest`, `in` and `out` contracts as well as `invariant`
all work as expected when defining Objective-C classes in D.

Note that `invariant` will only be called upon entering public functions
defined in D. External Objective-C function won't check the invariants since
Objective-C is unaware of them.

#### Global Functions

`extern(Objective-C)` global functions use the same ABI as C functions.

#### Inner Classes {unimplemented}

Objective-C classes defined in D can contain inner classes. You can also derive
an inner class from an Objective-C object.

#### Memory Safety

While the Objective-C language provide no construct to guaranty memory safety,
D does. Properly declared external Objective-C objects should be usable in
SafeD and provide the same guaranties.

#### Generated Selectors

When a function has no explicit selector, the compiler generate one in a way
that permits function overloading. To this end, a function with one or more
arguments will have the type of its arguments mangled inside the selector name.
Mangling follows what the `type.mangleof` expression returns.

For instance, here is the generated selector for these member functions:

``` d
int length();                    // generated selector: length
void moveTo(float x, float y);   // generated selector: moveTo_f:f:
void moveTo(double x, double y); // generated selector: moveTo_d:d:
void addSubview(NSView view);    // generated selector: addSubview_C4cocoa6appkit6NSView:
```

You generally don't need to care about this. To get the selector of a function,
take its address and simply assign it to a selector variable:

``` d
    void __selector(NSView view) sel = &NSView.addSubview;
```

### Blocks {unimplemented}

While not strictly speaking part of Objective-C, Apple's block extension for C
and Objective-C is now used at many places through the Mac OS X Objective-C
Cocoa APIs. A block is roughly the same thing as a D delegate, but it is stored
in a different data structure.

The type of a block in D is expressed using the same syntax as a delegate,
except that you must use the `__block` keyword. If an Objective-C function
wants a block argument, you declare it like this:

``` d
extern (Objective-C)
class NSWorkspace
{
    void recycleURLs(NSArray urls, void __block(NSDictionary newURLs, NSError error) handler)
        @selector("recycleURLs:completionHandler:");
}
```

Delegates are implicitly converted to blocks when necessary, so you generally
don't need to think about them.

``` d
workspace.recycleURLs(urls, (NSDictionary newURLs, NSError error) {
    if (error == null)
        writeln("success!");
});
```

Blocks are only available on Mac OS X 10.6 (Snow Leopard) and later.

### Breakage

There should be minimal breakage since most changes are new and only affects
code marked with `extern` `(Objective-C)`. A few cases can break where new
keywords are introduced, like `__selector`, `__classext` (unimplemented) and
`__block` (unimplemented). All these keywords start with two underscores, which
are considered reserved by the compiler. That means, these names shouldn't be
present in user code.

## Copyright & License

Copyright (c) 2016 by the D Language Foundation

Licensed under [Creative Commons Zero 1.0](https://creativecommons.org/publicdomain/zero/1.0/legalcode.txt)
