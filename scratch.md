# Goals

- Simple, embeddable language that is high performance and uses low amounts of memory
- Minimal to no FFI layer required, making interpobability insanely easy
- Runs natively on the system but is much more easily sandboxed than a native language
- Typed

# Scratch

```rust
fn HelloWorld() {
    print("Hello World");
}

// variables are immutable by default
let x = 2;
// can't run `x = 3;`

mut y = "Hello";
y += " World";

// Let's define a simple object
fn CreateVec3DInt(x: float, y: float, z: float) {
    // "structs" are just functions that return objects
    return {
        // object definition
        x = x,
        y = y,
        // shortform is just
        z,

        // function definitions (part of type)
        fn sqr_mag() {
            // this refers to the object
            // to refer to parameters you use `param.x`
            return this.x ** 2 + this.y ** 2 + this.z ** 2;
        },
        fn mag() {
            return Math.sqrt(sqr_mag());
        },
    }
}

// But often it's nice to have some sort of nice nesting
let Vec3DInt = {
    CreateVec3DInt, // to save us repeating the function (though you would ideally do it inline)
    Zero = CreateVec3DInt(0, 0, 0),
};

let myVec = Vec3DInt.Zero;
// comparisons are always value based
Test.assert(myVec == { x: 0, y: 0, z: 0 });

// type definition is simple we can print anything with x/y/z components
fn printVec3DInt(vec: { x: int, y: int, z: int }) {
    // $ allows us to use variables in strings
    print($"x = {x}, y = {y}, z = {z}");
}

printVec3DInt(myVec); // x = 0, y = 0, z = 0

// to make typing easier it's common to define a common interface
// this means we can define functions simpler and re-use functions
// we can even define it generically
type[T] Vec3D = {
    // if you want them to be modifiable you have to use `mut`
    mut x: T, mut y: T, mut z: T,

    fn sqr_mag() {
        return this.x ** 2 + this.y ** 2 + this.z ** 2;
    },
    fn mag() {
        // this restricts what T can be (has to be supported by Math.sqrt)
        return Math.sqrt(sqr_mag());
    }
}

// then we can "reimplement" very easily
let Vec3D = {
    fn[T] CreateVec3D(x: T, y: T, z: T) -> Vec3D[T] { return { x, y, z }; }
    fn[T] Zero() { return CreateVec3D[T](0, 0, 0); }
};

fn[T] printVec3D(vec: Vec3D[T]) {
    return vec.x + vec.y + vec.z;
}

// i.e.
let myNewVec = Vec3D.CreateVec3D.Zero[int]();

// and it still fits the requirements of the first vec we created
// so we can check for equality (and it works)
Test.assert(myNewVec == myVec);

// we can also call the original print vec
printVec3DInt(myNewVec);

// and it works vice versa (since they are equivalent types)
printVec3D(myVec);

// though note that above won't work if you mutate vec
fn[T] scaleXBy2(mut vec: Vec3D[T]) {
    vec.x *= 2;
}
// i.e. below works
scaleXBy2(myNewVec);
// but below doesn't work
// scaleXBy2(myVec); (type isn't mutable)

// Let's define a simple optional type next to add nullability
type[T] Optional {
    const has_value(): bool;

    fn[K] map(mapFn: fn(value: T) -> Optional[K]);
    fn else_value(value: T) -> T;
}

// then we just define our "enum" options to implement above
let Optional = {
    fn[T] None() -> Optional[T] = {
        // functions and values are "equivalent" and both work (as long as there is a known interface)
        has_value = false,
        fn map(value) { return Optional.None; }
        fn else_value(value) { return value; }
    },
    fn[T] Some(x: T) -> Optional[T] {
        return {
            // this "value" would be effectively private to anything taking an interface of Optional[T]
            value,
            has_value = true,
            fn map(mapFn) { return mapFn(value); },
            else_value = value,
        }
    }
};

// NOTE: if you want you can define `None` without having to use a function
//       this is because it just needs to satisfy Optional[T] for any given T
//       just define it like this
let NicerNone: Optional = {
    has_value = false,
    fn map(value) { return Optional.None; }
    // you don't even have to type this!  It'll implicity convert it to
    // fn[T] else_value(value: T) -> T
    fn else_value(value) { return value; }
};

// Let's define a simple optional type next to add nullability
// types support generics
type[T] Optional {
    // const means it needs a compile time known value
    const has_value: bool;

    // fn(value: T) -> Optional[K] is a function definition (i.e. function arg)
    fn[K] map(mapFn: fn(value: T) -> Optional[K]) -> Optional[K];
    fn else_value(value: T) -> T;
    fn toString() -> string;
}

// then we just define our "enum" options to implement above
let Optional = {
    None: Optional = {
        has_value = false,
        fn map(value) { return Optional.None; }
        // you don't even have to type this!  It'll implicity convert it to
        // fn[T] else_value(value: T) -> T
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

mut test = Optional.Some(2);
mut mapped = test.map(fn (x) { return x * 2; });
print(mapped); // Some(4)
test = Optional.None;
mapped = test.map(fn (x) { return x * 2; });
print(mapped); // None
```

# Core Differences

- We don't have `any` / `object` type support, this is cause we do all type work at compile time
- No nullability by default, you need to define something as optional
- Reflection is replaced with macros / code generators

