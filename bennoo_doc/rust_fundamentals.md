# Rust Fundamentals

This guide provides the essential Rust concepts needed to understand and contribute to the Codex codebase.  
For a deeper dive, consult the [official Rust Book](https://doc.rust-lang.org/book/).

## Language Basics

### Variables and Mutability

```rust
let x = 5;        // immutable
let mut y = 6;    // mutable
```

Rust defaults to immutability, encouraging safer code. Use `mut` only when necessary.

### Primitive Types

Rust offers familiar primitives (integers, floats, booleans, chars) and compound types:

```rust
let tup: (i32, f64, bool) = (1, 2.0, true);
let arr: [u8; 3] = [1, 2, 3];
```

### Control Flow

`if`, `loop`, `while`, and `for` operate as in other languages. The final expression in a block is returned implicitly:

```rust
let result = if condition { 10 } else { 20 };
```

## Ownership, Borrowing, and Lifetimes

These concepts prevent data races and dangling pointers at compile time.

- **Ownership**: Each value in Rust has a single owner. When the owner goes out of scope, the value is dropped.
- **Borrowing**: References allow using a value without taking ownership. Mutable (`&mut T`) and immutable (`&T`) borrows are tracked so that only one mutable borrow or many immutable borrows exist at a time.
- **Lifetimes**: The compiler ensures references remain valid. Most lifetimes are inferred, but explicit annotations (`<'a>`) appear in complex code.

Example:

```rust
fn greet(name: &str) {
    println!("Hello {name}!");
}
```

`name` is borrowed immutably, so the caller keeps ownership.

## Modules and Crates

- A **crate** is a compilation unit (library or binary). The workspace in `codex-rs` contains multiple crates, each under its own directory.
- A **module** (`mod`) organizes code within a crate. Files correspond to modules.
- Use `use` statements to bring items into scope.

```rust
pub mod utils {
    pub fn add(a: i32, b: i32) -> i32 { a + b }
}

use utils::add;
```

## Cargo

Cargo is Rust’s build system and package manager.

Key commands:

- `cargo build` – compile the crate.
- `cargo test` – run tests.
- `cargo run` – build and execute a binary crate.
- `cargo fmt` – format code (delegated to `rustfmt`).

In this repository, `just` scripts wrap common `cargo` tasks. See [Development Workflow](./development_workflow.md) for details.

## Error Handling

Rust distinguishes between recoverable (`Result<T, E>`) and unrecoverable errors (`panic!`). Codex code tends to use the [`anyhow`](https://docs.rs/anyhow) crate for ergonomic error propagation:

```rust
use anyhow::{Result, Context};

fn load() -> Result<String> {
    std::fs::read_to_string("config.toml").context("reading config")
}
```

## Traits and Generics

Traits define shared behavior. Generics allow implementations for multiple types:

```rust
trait Displayable {
    fn display(&self);
}

impl<T: std::fmt::Debug> Displayable for T {
    fn display(&self) {
        println!("{:?}", self);
    }
}
```

## Asynchronous Programming

Codex relies heavily on async code via `tokio`:

```rust
use tokio::task;

#[tokio::main]
async fn main() {
    let handle = task::spawn(async { 42 });
    println!("{}", handle.await.unwrap());
}
```

Async functions return `Future`s that are polled to completion by an executor.

## Macros

Macros like `println!` or `format!` generate code at compile time. Use `format!("Hello {name}")` to inline variables directly, as required by repo guidelines.

## Testing

Unit tests live in `tests` modules or the `tests/` directory. Snapshot tests use the [`insta`](https://insta.rs) crate, especially in `tui`. Run tests with:

```bash
cargo test -p codex-tui           # specific crate
cargo test --all-features         # workspace-wide (for shared crates)
```

## Further Reading

- [Rust By Example](https://doc.rust-lang.org/rust-by-example/)
- [Tokio Guide](https://tokio.rs/tokio/tutorial)
- [Clippy Lints](https://rust-lang.github.io/rust-clippy/) – enforced via `just fix`

Mastering these fundamentals will make navigating the Codex codebase much easier.
