# Macros

Macros, enable you to do compile-time evaluation and type reflection (to a degree) in a convenient way.

Macros are used heavily underneath the hood to handle features like FFI.

## Defining a macro

A simple to string macro could be written like this;

```rust
// lack of () means this doesn't take in any arguments
macro @stringify {
    return fn (context, node) {
        return match (node.kind) {
            // presuming ast nodes have a toString() method
            // i.e. (+ (1) (2)) becoming 1 + 2
            Statement => ConstantNode.String(node.toString()),
            _ => CompilerException("Expected a statement to follow @stringify", node),
        };
    };
}

print(@stringify "2"); // "2" (i.e. escaped quotes)
print(@stringify 2 + 2); // 2 + 2
print(@stringify { x: 2 }) // { x: 2 } (object)

// let's quickly define something to capture compiler errors
// we want to capture a block so we need the `()`, just like how if statements always have `{ }`
// block macros always require `{ }`.  This just removes ambiguity, and keeps the language consistent.
// You can add arguments here too
macro @captureCompilerErrors() {
    return fn (context, block) {
        // we can capture the error very easily and convert it to a real node
        for (let child in block) {
            match child {
                // errors are statements like anything else, so we can just capture them and convert it
                Error => context.replace(child, ConstantNode.String(child.message)),
            };
        }
    };
}

// this way we capture *some* errors, some ones like tokenisation are way earlier and won't be caught
print(@captureCompilerErrors() {
    const x = @stringify() { 2; }                   // Error: Can't pass a block into @stringify
    @stringify if (2 > 1) { return "No Errors"; }   // Error: Expected a statement to follow @stringify

    // unreachable (hopefully), if we reach here it means that our stringify exceptions weren't thrown
    return "No errors";
});
```
