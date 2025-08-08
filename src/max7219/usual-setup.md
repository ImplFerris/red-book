# Usual Setup

In this section, we'll go through the basic setup for our library.  It's very similar to what we did in the DHT22 chapter. In later chapters, we will skip the explanations for these basic setups and provide only the code.


## No Unsafe

If you noticed in the previous chapter, we never used the `unsafe` keyword in our project.  If you want to take it one step further and make sure no unsafe code can ever slip into your driver, you can enforce it with this flag at the top of your crate:

```rust
#![deny(unsafe_code)]
```


## Mark the Crate as no_std

To make our library compatible with embedded systems that don't have access to the standard library, we need to mark it as no_std.

Open the src/lib.rs file, remove all lines and add the following line at the very top:

```rust
#![cfg_attr(not(test), no_std)]
```

This tells the Rust compiler to avoid linking the standard library when we are not running tests. Instead, the crate will rely only on the core crate, which provides essential Rust features like basic types and operations. As a result, we won't be able to use any features from the std crate - only what's available in core.

The cfg_attr is a conditional attribute. It means "if not testing, then apply no_std." This way, we can still write normal unit tests using std, but the compiled library will work in no_std environments.

## Add Dependencies

Next, open Cargo.toml and add the embedded-hal crate under [dependencies]. This allows your driver to work with any compatible GPIO or delay implementation:

```toml
[dependencies]
embedded-hal = "1.0.0"
```

### Dev dependencies

This time, we add the embedded-hal-mock crate now itself under dev-dependencies, since we are already familiar with it. This lets us write tests by simulating embedded-hal traits:

```toml
[dev-dependencies]
embedded-hal-mock = { version = "0.11.1", "features" = ["eh1"] }
```
