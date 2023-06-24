# OOP

Object oriented programming (no matter how divisive it is) is still a common way to structure a program.

We don't have classes in Porc, but that doesn't mean that we don't have nice OOP tooling.  This means that you can still represent OOP like problems with low effort easily!

For example, let's say you have some sort of HTML Anchor which looks like this in IDL (https://html.spec.whatwg.org/multipage/text-level-semantics.html#the-a-element);

```c#
[Exposed=Window]
interface HTMLAnchorElement : HTMLElement {
  [HTMLConstructor] constructor();

  [CEReactions] attribute DOMString target;
  [CEReactions] attribute DOMString download;
  [CEReactions] attribute USVString ping;
  [CEReactions] attribute DOMString rel;
  [SameObject, PutForwards=value] readonly attribute DOMTokenList relList;
  [CEReactions] attribute DOMString hreflang;
  [CEReactions] attribute DOMString type;

  [CEReactions] attribute DOMString text;

  [CEReactions] attribute DOMString referrerPolicy;

  // also has obsolete members
};
HTMLAnchorElement includes HTMLHyperlinkElementUtils;
```

You would represent this in Porc pretty simply like this;

```rust
// There is a big difference from a field lookup and having to run some sort of "getter" function
// in some cases persisting all the various options and updating them on a change may be clean
// but definitely not when you have this many that are all somewhat "entangled"
// 
// This is where properties come in
type HTMLHyperlinkElementUtils {
    // you can write it as a long form
    // `op` just means operator, and is used for index operators and just generate setters/geters
    href: { op get() -> String, op set(value: String) };

    // you can also use macros!
    protocol: @Prop(String) { get; set; };
    // let's say this just has a getter
    host: @Prop(String) { get; };

    // and so on
}

let ElementUtils = {
    fn createElementUtils(href: String) -> HTMLHyperlinkElementUtils {
        // presuming a nice parser exists
        let parsedUri = parseUri(href);

        // uri already supports everything we need (and more)
        // so we can just use it.
        return parsedUri;
    }
}

let HTMLElement = {
    // even though this is returning an HTML Element this doesn't mean that it can't define new fields
    // this is more just a "constraint"
    // you can of course define multiple constructors, make it take in an object, and so on
    fn createAnchor(href: String, target: String) -> HTMLElement {
        return {
            // this repeated all the way up the call tree
            @Extend(createHTMLElement()) {},
            target,
            // NOTE: this just exposes all the required properties of the element utils
            //       through the object we created here.  This actually works *really* well
            //       and it supports fields & properties & methods.
            @Mixin(createElementUtils(href)) {},
        };
    },
};
```

Thus we have really easy ways to extend the behaviour of other classes, and you can extend it *super* easy, for example

```rust
fn Duck() {
    return {
        fn quack() {
            return "quack";
        }
    }
}

fn MadDuck() {
    return {
        @Extend(Duck()) {
            fn quack() {
                // "base" refers to the extended implementation
                return base.quack().toUpper();
            }
        }
    }
}
```

The way this works is quite cool, there are 2 "class" definitions globally, one for MadDuck and the other for Duck, when you add `@Extend` or `@Mixin` it just adds extra metadata records that redirect directly to the parent implementation (unless you override) meaning there is no overhead cost (outside of the bytes of storage).

How does this work with storage?  For example the parsed uri in the previous example has to be stored somewhere!

Well, it behaves like it looks, each inheritance tree item is treated as a "new" object, however our compiler doesn't allocate each item separately, but instead allocates a large buffer to store the entire heirachy in a single object.  Each function has a small amount of metadata with it, including what is called a `this offset`, this refers to the offset for the "this parameter", for most functions it's 0, but in the case of extended functions it'll be non-zero.

When you refer to variables in an extended object it'll automatically adjust for the required offset for the object (though this is also stored in required metadata since for interfaces we already need to know offsets for all fields).

The `this offset` may seem like a waste of cycles but seriously adding / subtracting is super cheap and isn't noticeable for function calls.  If this becomes an issue in the future we'll just duplicate function bodies and optimise the variable lookup so that they can work off a `this offset` of 0.

A single *instance* can only be used once by an extended object (i.e. you can't share an instance of a parent in two children), this is mainly due to the performance improvements above relying on not having to do another pointer dereference for function calls, and to keep inheritance hierachies simple.  This however means that you can only create objects within the `@Extend()` macros if you want to extend them (they don't accept variables).

> If you want above, then you probably want composition instead which is perfectly supported!
