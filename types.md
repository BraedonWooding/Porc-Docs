# Interfaces

> There is some *slightly* more complicated cases such as multiple interfaces and so on, but we'll cover that later.

Interfaces are defined by using the `type` keyword to define a set of constraints.

For example, a simple animal interface may look like this;

```rust
type Animal {
    kind: String;
    name: String;
    age: Int;

    fn speak() -> String;
}
```

Porc doesn't have classes, instead objects "constrain" to interfaces, for example I could do the following;

```rust
fn animal_speak(animal: Animal) {
    print($"{animal.name} {animal.speak()}");
}

animal_speak({ kind: "Cat", name: "Whiskers", age: 3, speak: "Meow" }); // Whiskers Meow

let Dog = {
    createDog(name: String, age: Int) -> Animal {
        return {
            kind: "Dog",
            name,
            age,
            fn speak() {
                return if this.age <= 1 { "Yip" } else { "Bark" };
            }
        }
    }
}

animal_speak(Dog.createDog("Chippie", 1)); // Chippie Yip
animal_speak(Dog.createDog("Amelia", 5)); // Amelia Bark
```

`Dog` is sort of "inheriting" from `Animal` and adding it's own behaviour, but it's just defining an object that is constrained to `Animal`.  `Dog` in itself is literally just an object that has a single function method `createDog`.  This sort of pattern is quite common to implement common "classes".

# Inheritance chains vs composition

The main reason why we don't have classes and explicit inheritance is to decentivise deep inheritance chains.  In most cases composition is a cleaner approach.  For example, let's say you have some sort of "Load data" module that needs to be generalised across a few cases (load from table / load from api / load from csv file / load from excel file / ...).

You can easily define a type like this;

```rust
type[T] Load {
    // presumably you'll have other generalised fields here
    kind: String,
    // some sort of set of renames i.e. rename field X -> Y
    renames: Map[string, string],
    // maybe you also have some sort of other "wrangling" i.e. concatenate fields together and so on
    // probably a generalised list of "Transformations" or something.

    fn next() -> Result[Option[T]] {
        let next = raw_next();
        // notice that we don't need to
        // 1) specify a type for Optional
        // 2) specify Result at all
        if !next { return Option.None; }

        let val = Json.DeserializeObject[T](next);
        // We can check for errors by just writing it like this
        // then afterwards we can just use val as if it wasn't an error
        if !val { return val }
        
    },

    // just cause it's a simple format
    fn raw_next() -> Optional[Json.Object],
}
```

This makes the interface also implement a form of `iterator` which is just;

```rust
// there is also one for non-fallible next methods
type[T] FallibleForwardIterator {
    fn next() -> Error[Optional[T]]
}
```

### `@try`

We have the macro `@try` i.e. `let next = @try raw_next` which converts it to this form

```rust
let next = match raw_next() as x {
    true => x,
    false => return x,
};
```

This `@try` macro then works with anything that is coalescible to a boolean value.  The macro is really defined as above (it's that simple).

# Primitives

Primitives are classified as non-object types in Porc and represent your `int/float/...`.

You've probably noticed that `Int` is used in most places, this is because `Int` is a type!  It represents integers generally and supports arbitrary sized ints.

In some cases (like FFI / performance edge cases) it may be nicer to use a primitive such as `i32` (32 bit integer) or `i64` (64 bit integer).  These are referred to as "primitives" since they have a series of restrictions on them that objects don't.

- They can't be used in generics
- They can't be constrained to an interface
- They are stack allocated and thus there is an effective limit to their length

Let's begin by defining a constant object, a constant object is stack allocated (currently we don't allow mutable objects to be stack allocated).

```rust
// To not define a circular definition the constructor should take in a "constant" integer
// i.e. 3 or 3 + 2, but it can't take in a variable value (that isn't a constant).
// We want to constrain this to a Constant.Int (i.e. 3 not 3.0)
fn myI32(const value: Constant.Number.Int) {
    // we need to assert that the value is coercible to i32
    // that is in the range of -2^31 to 2^31 - 1
    // a standard assert is fine, since it's a const it'll be compile time!
    assert(-(2 ** 31) <= value <= (2 ** 31) - 1, "Value needs to be represented as a 32 bit integer");

    // extract the array of bytes of value if we are taking it as a signed twos complement value
    // often this is just a "reintepretation" of it's underlying representation (free in this case)
    // 
    // This is fallible since it could not fit in the length potentially (thus the assert above)
    // so in our case we can just convert that possible error into an error, since we don't want to make
    // each user of this function have to handle this error we can just use a panic (i.e. unwrap)
    const bytes = @unwrap(value.as_bytes(.Signed, .TwosComplement, length: 4), "We've already asserted the range of values for this conversion");

    return const {
        _rep: bytes,

        fn toString() {
            return $"${this._rep:int}";
        }
    };
}
```

> The same optimisation that applies to boxing applies here so it makes single item sized objects pretty efficient (even if nested).

Of course, we would need to define addition / subtraction and other sorts of operations, you might state that we can presume integer operations exist but this isn't always true.

For example, Lua supports 64 bit signed integer/floats (presuming > 5.3) but not unsigned integers or smaller integers/floats (without changing the number type for the entire language).

For this reason, we recommend using just `Int` and `Float` types (or just the `Number` one), however we do have options for the few cases where smaller ints are required.

# Integers vs Floats ("number" type)

A few languages that inspired the decisions made here are:
- Lua, which has it's Number type be a union of Integers and Floats.
  - If you initialise a number with an integer value it does integer math.
  - If you combine an int and a float it becomes a float (transparently)
  - It does not promote an integer to a float when it gets too large it instead overflows/underflows
  - It has 2 operations for division (int and float)
  - Exponentiation is always a float operation
  - For loops never overflow (a bit of a weird case meant to reduce infinite loops)
- Python, which has 2 different types (ints & floats)
  - Integers have arbitrary precision (no overflow/underflow) i.e. they are bignums
  - Floats have restricted precision (are just doubles)
- Javascript just has a `Number` type which is just a double
  - It does have optimisations on a JIT level for integer math (i.e. `((x | 0) + (10 | 0)) | 0` is optimised to do integer operations).

Our solution here is to follow a little bit of a mix.

- We do have a `Number` type (that is a union of ints/floats) but is implemented as purely just a convenient type (for generalising / restricting generics).
- We have 2 key types `Int` and `Float` (like Python) but they are both 64 bit values (like Lua)
  - We don't have the typical memory overhead that other scripting languages have (so 8 bytes for us on the stack is very often literally just 8 bytes, rather than the 28 bytes that an integer takes in python).
  - This also reduces the impact of overflowing
- We offer variants for common integer/float sizes i.e. `i8 / u8 / f32`
  - We support 8/16/32/64/128 for signed & unsigned integers
  - We support 16/32/64 for floats
- You can configure the default size for an integer/float (to any of the common variants above)
    - You can access the global `Int.byteSize` and `Float.byteSize` to get the length (in bytes) of the integers in the current environment.

## Integer Overflow

As a good example let's say you have some sort of FFI that acts as a counter, if the player plays for long enough that counter could get too large.  The question is what should happen here?

- Should we truncate integers/floats and accept the loss of precision / overflowing?
- If overflow/underflow happens what should we do?
  - Wrap back around (i.e. Max -> Min)
  - Saturate at the bounds (i.e. 1 + Max == Max)
  - Make it fallible (return 'null'/'nil'/'none')
  - Panic?  (crash application)
- Should we make all FFIs that perform implicit casts fallible
- Should it panic?

Quite a few languages were considered that handle overflow differently;
- Lua performs wrapping behaviour on overflowed values
- Rust panics on overflow for debug builds only
- C & C++ only define overflow as valid for unsigned integers (wrapping) but *typically* the behaviour for signed integers is wrapping too.
- C# throws an exception in checked contexts
- Swift panics in both debug & release builds (though supplies a flag if you wish to disable checking)

---

What's important is that behaviour is consistent, if we decide to do wrapping behaviour for our `Int`s then we should also do it for our typed variants (i.e. `i8`).

To go into more detail for each option (noting that our context is for scripting and not necessarily system programming)
- Saturating is weird behaviour in most cases (may make sense for something like a "money" counter but in other cases is often not desired) and given that our integers are 64 bit by default it's unlikely that they are going to overflow under "typical" use.  It's also more expensive compared to overflow.  It also is very hard to track down (it's not an obvious bug)
- Fallible integer operations is an awful developer experience and leads to very verbose code
- Panic is a great idea for a system programming language where the code is critical, but for scripting it often can be annoying, having a game crash cause you have too much money (or a mod overflowed) is annoying.
- Overflow is the most common solution cause it's the cheapest, and it is "reasonably" obvious that it's occurred (i.e. going from positive money -> negative money is pretty obvious).  However, it can still lead to very odd glitches and issues.

Other context:
- Overflow is a rare case for most applications
- We run within a runtime and have a global logger

Thus the solution we went for is:
- Overflow for all integers
- Overflowing results in a warning being pushed to the global logger (though this can be disabled).

This does mean that our performance is impacted for integer math (since we are effectively panic'ing).  In cases where it matters we support changing overflow for a region of code (to improve performance).

```rust
mut x: u8 = 255;

@IntegerOverflow(setTo: .Wrapping) {
    x += 1;
}
print(x); // 0

x = 255;
@IntegerOverflow(setTo: .Saturating) {
    x += 1;
}
print(x); // 255

// and so on
```

> We also have `Int.saturating_X()`, `Int.wrapping_X()`, `Int.panicking_X()`, and `Int.try_X()` (fallible).

## Compiling to Javascript

Javascript is the outlier for math behaviour, it doesn't support 64 bit integers (32 bit integers are supported for the sake of math operations and fit in the range of a double but 64 bit integers aren't).

For this reason we don't fully support conversion to javascript.  We recommend using WebAsm instead.

If javascript is the target then the default Integer size used is 32 bits (not 64).  If any 64 bit (or higher) integers are used then a compilation exception will be raised.

> We may support a sort of polyfill for 64 bit math, but likely will result in significantly impacted performance of math.

# Union Types

Union (or sum) types are very useful!  Our solution is very simple (we don't have a union type or anything).

## Example 1. Optional

```rust
type[T] Optional {
    // const means it needs a compile time known value
    const has_value: bool;

    fn[K] map(mapFn: fn(value: T) -> Optional[K]) -> Optional[K];
    fn else_value(value: T) -> T;
    fn toString() -> string;
}

// then we just define our "enum" options to implement above
let Optional = {
    None: Optional = {
        has_value = false,
        fn map(value) { return Optional.None; }
        fn else_value(value) { return value; },
        fn toString() { return "None"; },
    },
    fn[T] Some(x: T) -> Optional[T] {
        return {
            // this "value" would be effectively private to anything taking an interface of Optional[T]
            value,
            has_value = true,
            fn map(mapFn) { return mapFn(value); },
            else_value = value,
            fn toString() { return $"Some({value})"; },
        }
    }
};
```

## Example 2. Number

```rust
type Number {
    // all numbers have operators
    op add(other: Number) -> Number;
    op multiply(other: Number) -> Number;
    // so on...

    // typically numbers have a cast method too!
    // notice that we use `Self` here, since
    // ideally a cast value should convert to the implementation
    // of this type *not* just a generic number.
    fn cast(other: Number) -> Self;

    // there are other fields as well that indicate stuff like
    // number of bytes, signedness, "type" (float/int/decimal), ...
}

let Number = {
    // notice that we could use compile time flags to change these
    Int: Number = i64,
    Float: Number = f64,
};
```

> We don't use Self for add/multiply cause in some cases you would want it to be the other type i.e. int + float = float.  The actual real definition of Number is a little more complicated to help with type hinting this, we use a compile time function to figure out what the output type is.

As you can see it's really that simple!  However, you might say, but how do we do match statements to figure out what number type it is and how would you guard against other possible values (i.e. make sure it's exhaustive).

### Guarding/Exhaustiveness

Since our typing system is meant to be "duck typing" (for flexibility) you can't restrict an interface to only a set of possible values.  However, notice that the object definition for `Number` only has 2 possible fields, you could instead restrict an argument to have to be inside the Number object.

i.e. you could write it like this

```rust
fn is_int(value: Number) {
  return @typeId(value) match {
    Int => true,
    Float => false,
    _ => @compilerError("Missing case for {@typeName(value)}"),
  }
}
```

Notice that we are using the type of value here, if you are missing any values then you'll get a nice compiler error!  For example `i8` is not `Int` (presuming you've set it to a default value of `i64`) so passing that in will raise a compiler error.

```rust
is_int(i8.cast(2)); // compiler error: Missing case for i8
```

> `i8` satisfies the interface requirements of `Number`

# Casting

```rust
let x: u32 = 255;

// non-fallible since u8 fits in u64
let y = u64.cast(x);

// fallible variant since casting from u32 -> u8 is potentially fallible
let z = @try u8.cast(x);

const x2: u32 = 255;
// valid since x2 is const
let z2 = u8.cast(x2);
```

Note that we don't support overloading functions, so instead we use generics & compile time execution to handle the cases of overloading.

For example, `u8` would be implemented like this;

```rust
let u8 = {
  // const since this can be executed at compile time
  // notice this isn't fallible (try_u8 is for that)
  fn cast(const value: Number) -> u8 {
    // value is const allows us to run specific code if it's running at compile time
    if value is u8 or value is const {
      // non fallible u8 should fit in u8
      // we don't care about the conversion for this snippet
      return @unwrap(try_u8(value), $"Conversion of value failed, {value} does not fit in a u8 (0 <= x <= 255).");
    } else {
      // unhandled case
      @compilerError($"Use {try_u8} if you want a fallible conversion method, otherwise pass either u8 or a constant value");
    }
  }
}
```

> Side note: we prefer the use of generics/compile time execution rather than overloading functions since the errors are a lot nicer, as you can see above we can refer them to the right function and give context rather than just listing all the possible overloading function variants with some obscure type error.

> Another side note: we prefer unwraps for these sorts of "compile time functions" since the panics result in compiler errors but are a bit cleaner than catching and logging it (i.e. it's not a syntax error but a code error so you want a stack trace / the ability to debug it).

# Boxing

Boxing isn't actually a conversion to `Int` (though optimisations mean this is what often happens).  It instead is just defining an object to 'wraps' your primitive value.  For example, a very poor boxing of a 32 bit int would be;

```rust
fn createBadBox(value: i32) {
    return {
        _value: value,
        
        multiply(other: BadBox) -> BadBox {
            return createBadBox(_value * other._value);
        }
        // ... and so on
    };
}
```

One of the key optimisations that is performed is around "dynamic dispatch", the idea is to avoid large objects with dynamic functions.  This is done automatically on most objects (that constraint to a given interface).

This results in something that originally looks like this;

```rust
fn[T] foo(value: { fn multiply(other: T) -> T }, otherValue: T) {
    return foo.multiply(otherValue);
}

print(foo(3, 2)); // 6
```

Mapping to an output that behaves like this;

```rust
fn i32_multiply(this: { _value: i32 }, other: { _value: i32 }) {
    return this._value * other._value;
}

let i32_methods = {
    multiply: i32_multiply,
    // ... and others
}

fn createBadBox(value: i32) {
    return {
        _value: value,
        _methods: i32_methods,
    }
}

fn[T] foo(value: { _methods: { fn multiply(other: T) -> T } }, otherValue: T) {
    // we pass in 'this'
    return value._methods.multiply(value, otherValue);
}

print(foo(createBadBox(3), createBadBox(2))); // 6
```

This optimisation happens for pretty much *every* object, in most cases this is very cheap we just need some sort of pointer on the object to point to it's function table, however in the cases of primitives this results in higher costs since boxing is always non-zero cost (involving an allocation).

> This is why our approach of "everything being objects" isn't inefficient, we are able to form a "class" equivalent at compile-time.  The only time this wouldn't occur is for closure arguments that aren't compile-time (one of the reasons why we don't support implicit closures, you need to explicitly specify it).

However, further optimisations exist for primitive "like" objects (objects with just a single field), this effects objects like Option/Result/Int and so on, and often result in nice speed improvements (completely removing the need for wrapping).

The idea, is that we pass in another parameter that holds the methods

```rust
fn i32_multiply(this: { _value: i32 }, other: { _value: i32 }) {
    return this._value * other._value;
}

let i32_methods = {
    multiply: i32_multiply,
    // ... and others
}

// weirdly enough since the typing of otherValue wasn't including the multiply function (intentional for this example)
// we generate it with another generic parameter, this does make sense though you could have something like
// foo(3.9, 2) and that would work as long as if float had a multiply function that accepted an int
fn[T, TValue] foo(value: TValue, otherValue: T, __methods: { fn multiply(self: TValue, other: T) -> T }) {
    return __methods.multiply(value, otherValue);
}

// no need for boxing here
print(foo(3, 2, i32_methods)); // 6
```

Thus, we have cheap and free wrappers!

## Inlining (quick aside)

We don't inline anything, we leave that up to the native output compilers to do, while you might argue that the boxing would be easily inlineable by compilers (and thus this optimisation isn't useful), that's not always true;

- Our objects are again only allocated in the heap, so this would not be able to be optimised away right now
- If the function you are calling isn't inlineable it won't remove the boxing
- The more complicated code is the less you can trust a compiler (look at how bad debug optimisations perform in langauges like C++ for wrapping classes/types).
