# Tinkerbell Language Extensions

Tinkerbell comes with all kinds of sugar to allow writing terser code,.

The sugar is added on a per-class basis:

```
@:tink class MyClass {
}
```

<!-- That means that you can use `tink_lang` granularly. Also, to get rid of sugar in some context, you can prefix it with `@:diet`. This works at class level, field level and expression level. -->

Generally you should not use identifiers starting with `__tink_` to avoid conflicts between your identifiers and those generated by tink.

### Overview

<!-- START INDEX -->
<!-- END INDEX -->

# Declaration Sugar

A few general notes/concepts apply:

#### Publishing

Tink has the concept of *publishing* members. This means that a member not explicitly declared `private` is promoted to become `public`, which is contrary to the default in Haxe. Tink does not publish everything by default, but certain sugar makes it sensible to *publish* a field.

#### Inference

Tink also tries to infer types that you omit that would be mandatory. However currently it will not be able to infer an expression that uses members from the class itself.

#### Implicit Return

In many cases, it's obvious that an expression should actually `return` something. Tink handles many of these by implicitly adding return statements should you omit them.

The strategy is all-or-nothing, i.e. if you have *no* return statements, tink will add them. If you have one, tink will leave things as they are.

Examples:

```
{
	if (foo) return 5;
	x;
	y;
	z;
}
```

This will not be touched and will ultimately result in a type error like "Void should be Int".

```
if (foo) 5;
else {
	x;
	y;
	z;
}
```

This will be transformed into:

```
if (foo) return 5;
else {
	x;
	y;
	return z;
}
```

When adding implicit return statements

- to a block, they are added to the last statement
- to an `if`, they are added to the if branch and the else branch if present
- to a `switch`, they are added to each branch
- any other expression is returned directly

As a corrolary, an implicit return of a loop will not lead to meaningful code.

## Partial implementation

Tink allows for partial implementations, that are quite similar to traits. Partial implementations are always declared as interfaces, that actually have an implementation. We'll take an example that might be familiar to Ruby programmers:

```
interface Enumerable<T> implements tink.Lang {
	var length(get, never):Int;
	function get_length()
		return fold(0, function (count, _) return count + 1);
		
	function fold<A>(init:A, calc:A->T->A):A {
		forEach(function (v) init = calc(init, v));
		return init;
	}
	function forEach(f:T->Void):Void {
		for (v in this)
			f(v);
	}
	function map<A>(f:T->A):Array<A> {
		var ret = [];
		forEach(ret.push);
		return ret;
	}
	function filter<A>(f:T->Bool):Array<T> {
		var ret = [];
		forEach(function (v) if (f(v)) ret.push(v));
		return ret;
	}
}
```

The implementation will be "cut" and "pasted" into classes that implement the interface without providing their own implementation. It is important to understand this metaphor: The process happens at expression level and in some sense is quite similar to C++ templates. For example the implementation of `forEach` only requires that the final class be eligible as a for loop target. That can mean it's an Iterator, an Iterable or has a length and array access.

The partial implementation can basically refer to any identifier. They only need to exist in the final class scope. Please note that if the "pasted" expression leads to a type error, the final class is the best error position we can give. That is about the same quality as saying that the class does not implement a certain method required by one of its interfaces. None the less, it can still be more misleading.

### On demand implementation

In some cases, you want to say "if you use this implementation, then also add member XYZ to build it on".

To extend the example above:

```
interface Enumerable<T> implements tink.Lang {
	@:usedOnlyBy(iterator) var elements:Array<T>;
	function iterator():Iterator<T> {
		return elements.iterator();
	}
	/* see above for the rest */
}
```

Now what this means is, that *if* the iterator implementation is taken from `Enumerable`, then `elements` will be generated. More generally, it will be generated if *any* of the members listed in the `@:usedOnlyBy` metadata are taken from the partial implementation. Note that `elements` will *not* be part of the interface itself. 

Note that we can go further:

```
interface Enumerable<T> implements tink.Lang {
	@:usedOnlyBy(iterator) 
	var elements:Array<T>;
	@:usedOnlyBy(forEach)
	public function iterator():Iterator<T> {
		return elements.iterator();
	}
	/* see above for the rest */
}
```

### Default initialization

The above is rather hard to use, if `elements` is not initialized. Therefore we also define a default value:

```
interface Enumerable<T> implements tink.Lang {
	@:usedOnlyBy(iterator) 
	var elements:Array<T> = [];
	@:usedOnlyBy(forEach)
	function iterator():Iterator<T> {
		return elements.iterator();
	}
	/* see above for the rest */
}
```

Default initializations are added at the beginning of the final class constructor through [direct initialization](#direct-initialization), if the corresponding field is generated. This doesn't require `@:usedOnlyBy`.

### Partial implementation caveats and use cases

This feature should be used sparsingly. Composition is preferable (check out [syntactic delegation](#syntactic-delegation)). You would use partial implementation when:

1. Performance matters so badly, that you cannot afford the cost of composition. Beware of premature optimization here.
2. What you do is so simple, that composition would complicate it.
3. You have some intricate relationship that is hard, if not impossible, to express in the type system.

To expand on the second case:

```
interface Identifiable {
	var id(default, null):Int = Id.generate();
}
```

Hence if you now implement `Identifiable`, the id variable will be added and initialized automatically.

To expand on the third case: Haxe's `@:generic` can work some wonders, but it cannot really cover everything. For example it demands for type parameters to be physical types (classes/interfaces or enums). Partial implementations don't have that restriction. Also, some constraints cannot be expressed with types, such as "can be iterated over" (which can be satisfied in many ways) or "supports array access" (which is true for `Array`, `ArrayAccess` and any abstract that defines array access) or "supports `+` operator".

One major trip wire is that `import` and `using` in the scope of the partial implementation will be ignored. This is not absolutely unsolvable, but a solution with the means currently provided by the macro API would be quite expensive.

To some extent, this is also an advantage of this feature. You may for example have implemented a `using` extension for some type, that gives it the same interface as some other type. Or you may have two abstracts, that have the same methods. But the Haxe type system does not allow for polymorphism in this case.

Say you have this:

```
class ArrayMapExtension {
	static public function exists<A>(arr:Array<A>, key:Int):Bool
		return key > -1 && key < arr.length;
	static public function keys<A>(arr:Array<A>):Iterator<Int>
		return 0...arr.length;
}
```

If you were `using` this, then an array can easily act as a read-only map.

```
interface PairMaker<K, V, T> {
	function make(target:T):Array<Pair<K, V>>
		return [for (i in target.keys()) new Pair(i, target[i])]
}

class IntMapPairMaker<V> implements PairMaker<Int, V, Map<Int, V>> {}

using ArrayMapExtension;

class ArrayPairMaker<V> implements PairMaker<Int, V, Array<V>> {}
```

Finally, it should be noted that like `@:generic`, partial implementations will cause generation of lots of code.

## Property declaration

### Pure calculated properties

You can declare purely calculated properties like this:

```
@:calculated var field:SomeType = someExpr;
```

Calculated properties are [published](#publishing) and can be [infered](#inference) if you omit `SomeType`.

The above code will simply translate into:

```
public var field(get, never):SomeType;
function get_field():SomeType someExpr;
```

Return statements are [added implicitly](#implicit-return) to the getter. You can also use `inline` on the variable which will cause the generation of an `inline` getter. Also `@:calc` is a recognized shortcut.

Here's what happens if we use all of these together:

```
@:calc inline var data = if (Config.IS_LIVE) Data.LIVE else Data.TEST;
```

Assuming `Data.LIVE` and `Data.TEST` are of type `Foo`, this becomes:

```
public var data(get, never):Foo;
inline function get_data()
	if (Config.IS_LIVE) return Data.LIVE;
	else return Data.TEST;
```

### Direct initialization

Tink allows directly initializing fields with three different options:

```
var a:A = _;
var b:B = (defaultB);
var c:C = constantC;
```

Which are defined as follows:

- `_` : a constructor argument
- `(fallback)` : a constructor argument (or use `fallback` if it is null).
- or an arbitrary expression, that must be valid in the context of the constructor

Using any of these has a number of side effects:

- They will generate a constructor if none exists, with a super call if necessary. This can sometimes lead to subtle issues. If you're getting cryptic error messages in complex inheritance chains, look here.
- In the first two cases, they will add an argument to the constructor's argument list and [publish](#publishing) the constructor. Arguments are *appended* in the order of appearence. If you need them to go elsewhere, you can declare your constructor as `function new(before1, before2, _, after1, after2)`, where they will be inserted in order of appearence.
- Any initialization will cause the field to be get an `@:isVar`.

#### Setter Bypass

Direct initialization will cause setter bypass. That means if your field has a setter, it will not be invoked. This is useful if you have the chicken and egg problem that your setter requires the underlying field to be in a particular state to work correctly, but to set that state you would need to call the setter. Well, here you go.

Beware that technically you can create invalid code with this.

If you don't want setter bypass, initialize the field the old fashioned way - in the constructor body.

### Lazy initialization

You can define lazily initialized fields using the `@:lazy` metadata. The implementation relies on defining an additional `tink.core.Lazy` under `lazy_<fieldName>`. Example:

```
@:lazy var x = [1,2,3,4];
```

This corresponds to:
	
```
@:noCompletion var lazy_x:tink.core.Lazy<Array<Int>> = tink.core.Lazy.ofFunc(function () return [1,2,3,4])
@:calc var x:Array<Int> = lazy_x.get();
```

### Property notation

#### Readonly property

To denote readonly properties with a getter, you can use this syntax:

```
@:readonly var x:X;

@:readonly(someExpr) var y:Y;
```

Which is converted to:

```
public var x(get, null):X;
function get_x():X return x;

public var y(get, null):Y;
function get_y():Y someExpr;
```

Readonly properties are [published](#publishing), and the getters use [implicit returns](#implicit-return).  
Also, `@:read` is a recognized shortcut and you can use `inline` to cause the getter to be inlined.

#### Readwrite properties

Similarly, you can define properties with both getter and setter:

```
@:property var a:A;

@:property(guard) var b:B;

@:property(readC, writeC) var c:C; 
```

This will be converted into:

```
@:isVar public var a(get, set):A;
function get_a() return this.a;
function set_a(param) return this.a = param;

@:isVar public var b(get, set):B;
function get_b() return this.b;
function set_b(param) return this.b = guard;

public var c(get, set):C; 
function get_c() readC;
function set_c(param) writeC;
```

These properties are also [published](#publishing), and the getters and setters use [implicit returns](#implicit-return). Also, `@:prop` is a recognized shortcut and you can use `inline` to cause the getter and setter to be inlined.

We have 3 different cases here:

- default properties - the actual value is stored in the underlying field and the getter and setter do nothing but access it
- guarded properties - the actual value is stored in the underlying field and while the getter just retrieves it, the setter uses a guard expression
- full properties - here getter and setter are really just what you define them to be. If you want to store values in the underlying field, don't forget to add `@:isVar`

Real world example:

```
import Math.*;

class Point implements tink.Lang {
	static var counter = 0;
	
	@:property(max(param, 0)) var radius = .0;
	
	@:property(param % (PI * 2)) var angle = .0;
	
	@:property var name:String = 'P'+counter++;
	
	@:property(cos(angle) * radius, { setCartesian(param, y); param; }) var x:Float;
	
	@:property(sin(angle) * radius, { setCartesian(x, param); param; }) var y:Float;
	
	function setCartesian(x, y) {
		this.angle = atan2(y, x);
		this.radius = sqrt(x*x + y*y);
	}
}
```

So here we have a point that is internally represented in polar coordinates, that we can get and set. When setting these, some guards are applied, to ensure the radius never becomes negative and that the angle always stays within a certain interval. We give the point a name that can be changed. And we implement x and y as calculated settable properties.

## Complex Default Arguments

Tink allows for arbitrary default arguments. Note that the expression will be evaluated for every function call. The generated code is similar to that generated by Haxe for simple default arguments (i.e. when `null` is passed, then the default value is applied). Example:

```
function prophanityFilter(text:String, blacklist:Array<String> = haxe.Resource.getString('blacklist').split('\n)) {
	//...
}
```

As you can see, that works well.

### Function options

Default arguments that are anonymous objects are treated specially in that each of their properties can be optional and those that are passed are merged with the defaults.

```
function mysqlConnect(options = { host: 'localhost', port: 3306, user: _, password: _, database: 'test' }) {}

//which can be called like so:

mysqlConnect({ user: 'root', password: '' });
mysqlConnect({ user: 'u3405', password: 'cpej3051', host: 'foo.dbserver.com' });


//because it is transformed to this:

function mysqlConnect(options: { ?host: String, ?port: Int, user:Unknown, password:Unknwon, ?database:String }) {
	if (options.host == null) options.host = 'localhost';
	if (options.port == null) options.port = 3306;
	if (options.database == null) options.database = 'test';
}
```

This kind of signature is useful to pass many options to a single function, hence the name "function options". The following rules apply

- To have a mandatory option (that's a bit of an oxymoron), you can use `_`. 
- To give an option a type, use the `($expr : $type)` syntax.
- If you have a mandatory options, then the whole options object becomes mandatory.
- You can specify multiple options if you wish to, although there's no real point in doing that.

#### Direct options

If you don't wish to actually have an object holding the options, but rather variables directly, you can use this:

```
function bar(_ = { x: someX, y:someY, ...}) 
	body;

//becomes

function bar(?_:{?x:X,?y:Y}) {
	var x = if (_ == null || _.x == null) someX else _.x;
	var y = if (_ == null || _.y == null) someY else _.y;
	body;
}
```

This comes pretty close to [named parameters](http://en.wikipedia.org/wiki/Named_parameter). 

## Notifiers

To make defining signals and futures (and usually the associated triggers) easy, you can use the following syntax:

```
class Observable implements tink.Lang {
	@:signal var click:MouseEvent;
	@:future var data:Bytes;
	@:signal var clickLeft = this.click.filter(function (e:MouseEvent) return e.x < this.width / 2);
	@:future var jsonData = this.data.map(function (b:Bytes) return b.toString()).map(haxe.Json.parse);
}
```

This will be converted as follows:

```
class Observable implements tink.Lang {
	private var _click:SignalTrigger<MouseEvent>;
	private var _data:FutureTrigger<Bytes>;
	@:readonly var click:Signal<MouseEvent> = _click.toSignal();
	@:readonly var data:Future<Bytes> = _data.toFuture();
	@:readonly var clickLeft = this.click.filter(function (e) e.x < this.width / 2);
	@:readonly var jsonData = this.data.map(function (b) return b.toString()).map(haxe.Json.parse);
}
```

As we see, not specifying an initialization will cause generation of a trigger. If you do specify an initialization, you might just as well use normal property notation. This syntax however allows for a consistent notation in both cases, that allows users to see signals and futures at a single glance.

### Signal/Future on interfaces

You can use this syntax on interfaces also, which causes [partial implementations](#partial-implementation). If a trigger is generated, it will get a `@:usedOnlyBy`-clause.

## Syntactic Delegation

Tinkerbell supports syntactic delegation for both fields and methods. The basic idea is, that you can automatically have the delegating class call methods or access properties on the objects it is delegating to. In the simpler of two cases, the class delegates to one of its members. A very simple example:

```
class Stack<T> implements tink.Lang {
	@:forward(push, pop, iterator, length) var elements:Array<T>;
	public function new() {
		this.elements = [];
	}
}
```

Here, we are forwarding the calls `push`, `pop`, `iterator` as well as the field `length` to the underlying data-structure. 

Another example:

```
class OrderedStringMap<T> implements tink.Lang {
	var keyList:Array<String> = [];
	@:forward var map:haxe.ds.StringMap<T> = new haxe.ds.StringMap<T>();
	public function new() {}
	public function set(key:String, value:T) 
		if (!exists(key)) {
			keyList.push(key);
			map.set(key, value);
		}
	public function remove(key:String) 
		return map.remove(key) && keyList.remove(key)
	public function keys() 
		return keyList.iterator()
}
```

### Delegation filters

As you have seen in the above example, we chose which fields to forward. What we are doing here is matching a field against a filter. The rules:

* An identifier matches the field with the same name
* A regular expression matches all fields with matching names
* A string matches all fields matching it, with the `*`-character being matching any character sequence, i.e. `do*` would match all members starting with "do" and `*Slot` matches all members ending with "Slot"
* `filter_1 | filter_2` and `filter_1 || filter_2` match if either filter matches
* `[filter_1, ..., filter_n]` matches if either of the filters match
* `filter_1 & filter_2` and `filter_1 && filter_2` match if both filters match
* `!filter` matches if `filter` doesn't match

If the `@:forward`-tag has no arguments, then all fields are matched. Otherwise all fields matching either argument are matched.

Also `@:fwd` is a recognized shortcut for `@:forward`.

### Delegation to member

Usage example:

```
//let's take two sample classes
class Foo {
	public function fooX(x:X):Void;
	public function yFoo():Y;
}
class Bar {
	public var barVar:V;
	public function doBar(a:A, b:B, c:C):R;
}
//and now we can do
class FooBar implements tink.Lang {
	@:forward var foo:Foo;
	@:forward var bar:Bar;
}
//which corresponds to
class FooBar implements tink.Lang {
	var foo:Foo;
	var bar:Bar;
	public function fooX(x) return foo.fooX(x)
	public function yFoo() return foo.yFoo()
	@:prop(bar.barVar, bar.barVar = param) var barVar:V;//see property notation
	public function doBar(a, b, c) return bar.doBar(a,b,c)
}
```

### Delegation to method

This kind of forwarding may appear a little strange at first, but let's see it in action:

```
//Foo and Bar defined in the example above
class FooBar2 implements tink.Lang {
	var fields:Hash<Dynamic>;
	@:forward function anyName(foo:Foo, bar:Bar) {
		get: fields.get($name),
		set: fields.set($name, param),
		call: trace('calling '+$name+' on '+$id+' with '+$args)
	}
}
```

This becomes (actually this is simplified for your convenience):

```
class Foobar2 implements tink.Lang {
	var fields:Hash<Dynamic>;
	public function fooX(x:X) trace('calling '+'fooX'+' on '+'foo'+' with '+[x])
	public function yFoo() trace('calling '+'yFoo'+' on '+'foo'+' with '+[])
	@:prop(fields.get('barVar'), fields.set('barVar', param)) var barVar:V;//see accessor generation
	public function doBar(a:A, b:B, c:C) trace('calling '+'doBar'+' on '+'bar'+' with '+[a, b, c])
}
```

This feature is quite exotic. It's intention is to allow building full proxies, such as `haxe.remoting.Proxy`.

### Delegation rules

- Forward is generated per member in order of appearance
- If a member with a given name already exists, no forward statement is generated (i.e. if `FooBar` already had a method `fooX` in the above statement, the forwarding method would not be generated). This applies also if the member is defined in a super class.

### Delegation on interfaces

Using this syntax on interfaces will cause sensible [partial implementations](#partial-implementation) most of the time. Consider it experimental.

# Implementation Sugar

This kind of syntactic sugar works at expression level, i.e. in function bodies.

## Extended For Loops

### Arbitrary steps

Loops with arbitrary steps are denoted as follows:

```
//upward
for (i += step in min...max) body;
//downward
for (i -= step in max...min) body;
```

This also works for float loops. The type of `step` will determine whether this is a `Float` loop or an `Int` loop. The use of `+=` or `-=` determines whether you want an upward or downward loop. 

The downward loop is symmetrical to the upward loop, i.e. it will yield the same values, only in backward order. A upward loop will always start with min and stop just before max (except in the case of float precision issues), while an downward loop will always end with min, starting just "after" max.

Using this syntax will cause generation of a while loop.

### Key-value loops

This syntax is also supported:

```
for (key => value in target) body;
```

It will just be translated into:

```
for (key in target.keys()) {
    var value = target.get(key);
    body;
}
```

If `target` doesn't actually have a compatible `keys` or `get` method a type error will be generated at the position of where the `key => value` was found.

### Destructuring loops

If you wish to run a loop only to destructure the items right away, you can use this syntax:

```
for (pattern in target) body;
//which is equivalent to
for (tmp in target)
  switch tmp {
		case pattern: body;
		default:
	}
```

Here is an example:

```
var a = [Left(4), Right('foo'), Right('bar'), Left(5), Left(6)];

trace([for (Left(x) in a) x]);//[4, 5, 6]
trace([for (Right(x) in a) x]);//['foo', 'bar']
```

You may notice that `pattern` being an identifier is indeed just a special case of this rule.

```
for (v in 0...100) {}
//is equivalent to:
for (tmp in 0...100) 
	switch tmp {
		case v: //this matches everything
	}
```

However, to keep the code simple, tink does not generate the switch statement for mere identifiers.

### Parallel loops

Sometimes you want to iterate over multiple targets at once. Tink supports this syntax:

```
for ([head1, head2, head3]) body;
```

Here `head1`, `head2` and `head3` can be normal loop heads (`variable in expression`) or loop heads for arbitrary step or key-value loops (please note that using parallel loops for key-value loops only makes sense if key order is deterministic, i.e. you're using an ordered map or something).

Example:

```
for ([ship in ships, i -= 1 in ships.length...0])
	ship.x = 30 * i;
```

This will order the ships in your array from right to left.

By default, a parallel loop will stop as soon as any head is "depleted". Another example, to show just that:

```
var girls = ['Lilly', 'Peggy', 'Sue'];
var boys = ['Peter', 'Paul', 'Joe', 'John', 'Jack'];
for ([girl in girls, boy in boys])
    trace(girl + ' loves ' + boy);
-- OUTPUT:
Lilly loves Peter
Peggy loves Paul
Sue loves Joe
```

Now that's really unfortunate for John and Jack. Luckily there's one person they can always lean on: 

```
var girls = ['Lilly', 'Peggy', 'Sue'];
var boys = ['Peter', 'Paul', 'Joe', 'John', 'Jack'];
for ([girl in girls || 'Mommy', boy in boys])
    trace(girl + ' loves ' + boy);
-- OUTPUT:
Lilly loves Peter
Peggy loves Paul
Sue loves Joe
Mommy loves John
Mommy loves Jack
```

#### Loop Fallbacks

As we see in the example just above, we can provide *fallbacks* for parallel loops. We simply use `||` for this. As soon as a loop target is depleted, the fallback expression is used instead. Please note that the expression is evaluated *every time* a fallback value is needed. Example:

```
var girls = ['Lilly', 'Peggy', 'Sue'];
var boys = ['Peter', 'Paul', 'Joe', 'John', 'Jack', 'Jeff', 'Josh'];
var index = 0;
var family = ['Mommy', 'Grandma', 'Aunt Lilly'];
for ([girl in girls || family[index++ % family.length], boy in boys])
    trace(girl + ' loves ' + boy);
-- OUTPUT:
Lilly loves Peter
Peggy loves Paul
Sue loves Joe
Mommy loves John
Grandma loves Jack
Aunt Lilly loves Jeff
Mommy loves Josh
```

This is very powerful, but it's also a great way to shoot yourself in the foot. Please use non-constant expressions with care.

If you specify fallbacks for all targets, the loop will stop as soon as all targets are depleted and only fallbacks are available.

### Extended comprehensions

Tink generalizes the concept of for comprehensions in two ways. It deals with more complex loop bodies and it allows to construct things other than arrays.

#### Complex bodies

Haxe comprehensions are rather narrow in what they accept as bodies. In a number of cases the behavior is unintuitive:

Example with `switch`:

```
var x = [true, false, true];

trace([for (x in x)
	if (x) 1;
]);//[1, 1]

var x = [true, false, true];
trace([for (x in x)
	switch x {
		case true: 1;
		default:
	}
]);//[1, 1] with tink_lang, compiler error "Void should be Int" with vanilla Haxe 
```

Example with arbitrary `if`:

```
typedef Person = { name: String, age:Int, male:Bool }
enum Rescued {
	Woman(person:Person);
	Child(person:Person);
}

var crew:Array<Person> = [
	{ name : 'Joe', age: 25, male: true }, 
	{ name : 'Jane', age: 24, male: false }, 
	{ name: 'Timmy', age: 8, male: true }
];

var womenAndChildren = [for (person in crew)
	if (person.age < 18) Child(person)
	else if (!person.male) Woman(person)
];
```

With plain Haxe this will not compile saying "Void should be Rescued".

The vanilla Haxe behavior helps avoiding mistakenly empty branches. The idea has merit. This library has a different approach. By default, tink_lang will just follow down all paths to see if there is something to be returned. You can always use [manual yielding](#manual-yielding) if you need more control.

#### Alternative output

Haxe comprehensions can only construct maps or arrays. Tink comprehensions have a broader spectrum and deal with maps and arrays as special cases.

The general structure of a tink comprehension is:

```
target.method(for (head) body)
```

This gets translated to something like

```
{
	var tmp = target;
	for (head) bodyCallingMethod;
	tmp;
}
```

Where the body is transformed so that the leaf expressions call `tmp.method`.

If the method requires more than one argument, you can use `_(arg1, arg2, arg3)` to yield multiple values. Example:

```
var peopleByName = new Map().set(for (person in people) _(person.name, person));
```

This is translated into:

```
var peopleByName = {
	var tmp = new Map();
	for (person in people) 
		tmp.set(person.name, person);
	tmp;
}
```

When tink encounters `[for (head) body]` it will simply translate it into `[].push(for (head) body)` before processing, and when it encounters something like `[for (head) key => val]` it will translate it into `new Map().set(for (head) _(key, val))`, and they will thus work as though transformed by the Haxe compiler.

But if you need to output a list, you can do:

```
new List().add(for (i in 0...100) i)
```

But you needn't *construct* the target. You can use an existing one. For example to draw a couple of rectangles on the same sprite:

```
sprite.graphics.drawRect(
	for (i in 0...10) 
		_(0, i*20, 100, 10)
)
```

Also, because the target is returned, you can chain stuff:

```
var upAndDown = new List()
	.add(for (i in 0...5) i)
	.add(for (i -= 1 in 5...0) i)
trace(upAndDown);//{0, 1, 2, 3, 4, 4, 3, 2, 1, 0}
```

#### Manual yielding

If your loop body contains an expression such as `@yield $value`, then instead of gathering the result automatically, the comprehension will only add to the output what you yield, which allows you to have multiple results per loop iteration.

```
var ret = [for (i in 1...5) {
	if (i == 1) @yield 0;
	@yield -i;
	@yield i;
}];
trace(ret);//[0, -1, 1, -2, 2, -3, 3, -4, 4];
```

## Trailing arguments

Because of Haxe's call syntax you can often find yourself in a situation where a closing `)` corresponds to something *high* up. Tink has a notation for trailing arguments to deal with that, which transforms `someFunc(...args) => lastArg` to `someFunc(...args, lastArg)` and `new SomeClass(...args) => lastArg` to `new SomeClass(...args, lastArg)`.

Example use cases:

```
myButton.on('click') => function () {
	trace('click!');
	triggerSomeAction();
}

sys.db.Mysql.connect() => { 
  host : "localhost",
  port : 3306,
  user : "root",
  pass : "",
  socket : null,
  database : "MyBase"
};
```

## Short lambdas

Tink supports a multitude of notations for short lambdas. Generally, two different kinds of functions are distinguished: those that return values and those that don't. The distinction is necessary since Haxe no longer allows values of type `Void`. We'll be calling them functions and procedures respectively (as is the case in Pascal).

Currently, Haxe does not support short lambdas, the rationale being that they are harder to read to new comers. This concern does have its value. Use this notation to increase readability and not to obfuscate code for the sake of saving a few key strokes. As the name would suggest, short lambdas should be *short*, the motivation here being to write function inline with minimal noise, which by nature is not compatible with complex bodies. If you have some complex, **give it a name** (you can always use a nested function and declare it `inline`).

### Arrow lambda

The notation looks like `[...args] => body`, with a shortcut for exactly one argument `arg => body`. Examples:

* `[] => true` becomes `function () return true`
* `[x] => 2 * x` becomes `function (x) return 2 * x`
* `x => 2 * x` (special case) becomes `function (x) return 2 * x`
* `[x, y] => x + y` becomes `function (x, y) return x + y`

Arrow lambdas are always funtions, since the arrow is conventionally used to represent a mapping (as in map literals, map comprehensions and extractors). A procedure does not define a mapping.

### Do procedures

This notation uses inline metadata to add a "keyword" as follows.

* `@do trace('foo')` becomes `function () trace('foo')`
* `@do(who) trace('hello $who')` becomes `function (who) trace('hello $who')`

Please note that metadata has precedence over binary operations. So `@do x = 5` will become `(function () x) = 5` which is an invalid statement. It's best to use `@do` with a block for a body, as that will assure the right precendence and should also look familiar to Ruby programmers.

Combined with [trailing arguments](#trailing-arguments), you can write things like:

```
myButton.on('click') => @do {
	trace('click');
	triggerSomeAction();
}
```

Or why not some nodejs code:

```
fs.readFile('config') => @do(error, data)
	if (error != null) panic(error);
	else
		http.get(Json.parse(data).someURL) => @do(error, data)
			if (error != null) panic(error);
			else {
				trace('we have the data')
			}
```

### F functions

Similarly to [do procedures](#do-procedure), `@f` will create a function:

* `@f 4` becomes `function () return 4`
* `@f(who) 'hello $who'` becomes `function (who) return 'hello $who')`

### Matchers

Another kind of short lambdas are "matchers", where the arguments are directly piped into a switch statement and therefore needn't be named (since you will capture the values you need in the respective case statements).

```
@do switch _ {
	/* cases */
}

switch _ {
	/* cases */
}
```

Which become:

```
function (tmp) switch tmp {
	/* cases */
}

function (tmp) return switch tmp {
	/* cases */
}
```

For the sake of consistency `@f switch _ {}` is treated like `switch _ {}`.

#### Multi argument matchers

If you expect more than one argument, you can use `[_,_]`, `[_, _, _]` and so on:

```
// or alternatively
@do switch [_, _] {
	/* cases */
}
```

Each of which becomes:

```
function (tmp1, tmp2) switch [tmp1, tmp2] {
	/* cases */
}
```

Put together with [trailing arguments](#trailing-arguments), you can write code like this:

```
someOp() => switch _ {
	case Success(result):
	case Failure(error):
}
```

## Handlers

As a counterpart to notifiers, you can use the following syntax to register handlers:

### When and Whenever

```
@when(someFuture) handler;
@whenever(someSignal) handler;
@until(someFutureOrSignal) someLink;
```

These are shortcuts for:

```
(someFuture : Future<Unknown>).handle(handler);
(someSignal : Signal<Unknown>).handle(handler);
(someFuture : Future<Unknown>).handle((someLink : CallbackLink));
```

If you want to only listen to the next occurrence of a `Signal` here's how:

```
@when(someSignal.next()) handler;
```

Here is how you would implement drag and drop in flash/NME/OpenFL:

```
class EventTools {
  static public function gets(target:EventDispatcher, event:String) {
  	return Signal.ofClassical(
  		target.addEventListener.bind(event),
  		target.removeEventListener.bind(event),
  		false
  	);
  }
}

import flash.events.MouseEvent.*;
using EventTools;

@whenever(target.gets(MOUSE_DOWN)) @do {
  
  var x0 = stage.mouseX - target.x,
      y0 = stage.mouseY - target.y;
      
  @until(stage.gets(MOUSE_UP).next()) 
    @whenever(stage.gets(MOUSE_MOVE)) {
      target.x = stage.mouseX - x0;
      target.y = stage.mouseY - y0;
    }
    
}
```

### In and Every

```
@in(delay) handler;

@every(interval) handler;
```

These get translated to:

```
(haxe.Timer.delay(handler, Std.int(delay * 1000)) : CallbackLink);

{
  var timer = new haxe.Timer(Std.int(interval * 1000));
  timer.run = handler;
  (timer : CallbackLink);
}
```

Notice that the expression becomes a `CallbackLink` which allows us to use it with `@until`.

```
@whenever(button.pressed) @do {
  @until(button.released.next()) 
  	@every(1) @do {
      trace('tick');
    }
}
```

Which reads as "whenever the button is pressed, until it is released the next time, every second trace tick". Slightly awkward, but consider spelling it out.

## Array Rest Pattern

Sometimes you want to pattern match against an array with specific entries at the start or the end and capture the rest. You can do this like so:

```
var uris = [
	'foo/bar/x/y/z',
	'foo/baz/bar',
	'foo/bar',
];
for (i in 0...uris.length)
	switch uris[i].split('/') {
	  case ['foo', 'bar', @rest rest]:
	  	trace(i+':'+rest);
	  case ['foo', @rest rest, 'bar']:
	  	trace(i+':'+rest);
	  case [@rest rest, 'foo', 'bar']:
	  	trace(i+':'+rest);
	  default:
	}
```

The code will output:

```
0:[x,y,z]
1:[baz]
2:[]
```

## Type switch

With tink you can switch over an expression's type like so:
	
```
switch expr {
	case (name1 : Type1):
	case (name2 : Type2):
	case (name3 : Type3):
		...
	default:
}
```

A default clause is mandatory. Also expr must be of the type you are switching against, so if for example you want to use this for downcasting, you will need to do `switch (expr : Dynamic) { ... }` or something equivalent.

Simple example:
	
```
var value:haxe.extern.EitherType<Int, String> = 5;

switch value {
	case (i : Int): trace('int $i');
	case (s : String): trace('string $s');
	default:
}
```

Put together with a destructuring loop:

```
var fruit:Array<Any> = [new Apple(), new Apple(), new Banana(), new Apple(), new Kiwi()];
var apples = [for ((a : Apple) in fruit) a];
trace(apples.length);//3
```

## Default

Default allows you to deal with sentinel or default values (such as `null`, `-1`, `0`). Instead of writing this code:

```
var x = someComplexExpression;
if (x == null) x = defaultValue;
doSomething(x)
```

You would write:

```
doSomething(someComplexExpression | if (null) defaultValue);
```

Read this syntax as "use `someComplexExpression` or if `null` use `defaultValue`". There's really not much to it. It helps avoiding additional variables. If you need to check against more than one value a switch statement is more appropriate.