# Deploy a Contract

In this chapter, we'll learn how to build, deploy, and call a minimal smart
contract on our local node.

## Preparing Ground

Let's start with creating a fresh local network with an account to which we'll
deploy a contract. You might want to re-read [how to run a node](./run_a_node.md)
to understand what's going one here:

```console
$ cargo run --profile dev-release -p uncd -- init
$ cargo run --profile dev-release -p uncd -- run
$ UNC_ENV=local unc create-account alice.test.unc --masterAccount test.unc
```

As a sanity check, querying the state of `alice.test.unc` account should work:

```console
$ UNC_ENV=local unc state alice.test.unc
Loaded master account test.unc key from /home/matklad/.unc/validator_key.json with public key = ed25519:7tU4NtFozPWLotcfhbT9KfBbR3TJHPfKJeCri8Me6jU7
Account alice.test.unc
{
  amount: '100000000000000000000000000',
  block_hash: 'EEMiLrk4ZiRzjNJXGdhWPJfKXey667YBnSRoJZicFGy9',
  block_height: 24,
  code_hash: '11111111111111111111111111111111',
  pledging: '0',
  storage_paid_at: 0,
  storage_usage: 182,
  formattedAmount: '100'
}
```

## Minimal Contract

UNC contracts are [WebAssembly](https://webassembly.org) blobs of bytes. To
create a contract, a contract developer typically uses an SDK for some
high-level programming language, such as JavaScript, which takes care of
producing the right `.wasm`.

In this guide, we are interested in how things work under the hood, so we'll
do everything manually, and implement a contract in Rust without any help from
SDKs.

As we are looking for something simple, let's create a contract with a single
"method", `hello`, which returns a `"hello world"` string. To "define a method",
a wasm module should export a function. To "return a value", the contract needs
to interact with the environment to say "hey, this is the value I am returning".
Such "interactions" are carried through host functions, which are quite a bit
like syscalls in traditional operating systems.

The set of host functions that the contract can import is defined in
[`imports.rs`](https://github.com/utnet-org/utility/blob/aeccaaab334275f6d0a62deabd184675bc3c6a23/runtime/unc-vm-runner/src/imports.rs#L71-L242).

In this particular case, we need the `value_return` function:

```
value_return<[value_len: u64, value_ptr: u64] -> []>
```

This means that the `value_return` function takes a pointer to a slice of bytes,
the length of the slice, and returns nothing. If the contract calls this function,
the slice would be considered a result of the function.

To recap, we want to produce a `.wasm` file with roughly the following content:

```wasm
(module
  (import "env" "value_return" (func $value_return (param i64 i64)))
  (func (export "hello") ... ))
```

## Cargo Boilerplate

Armed with this knowledge, we can write Rust code to produce the required WASM.
Before we start doing that, some amount of setup code is required.

Let's start with creating a new crate:

```console
$ cargo new hello-unc --lib
```

To compile to wasm, we also need to add a relevant rustup toolchain:

```console
$ rustup toolchain add wasm32-unknown-unknown
```

Then, we need to tell Cargo that the final artifact we want to get is a
WebAssembly module.

This requires the following cryptic spell in Cargo.toml:

```toml
# hello-unc/Cargo.toml

[lib]
crate-type = ["cdylib"]
```

Here, we ask Cargo to build a "C dynamic library". When compiling for wasm,
that'll give us a `.wasm` module. This part is a bit confusing, sorry about
that :(

Next, as we are aiming for minimalism here, we need to disable optional bits
of the Rust runtime. Namely, we want to make our crate `no_std` (this means
that we are not going to use the Rust standard library), set `panic=abort`
as our panic strategy and define a panic handler to abort execution.

```toml
# hello-unc/Cargo.toml

[package]
name = "hello-unc"
version = "0.1.0"
edition = "2021"

[lib]
crate-type = ["cdylib"]

[profile.release]
panic = "abort"
```

```rust
// hello-unc/src/lib.rs

#![no_std]

#[panic_handler]
fn panic_handler(_info: &core::panic::PanicInfo) -> ! {
    core::arch::wasm32::unreachable()
}
```

At this point, we should be able to compile our code to wasm, and it should be
fairly small. Let's do that:

```console
$ cargo b -r --target wasm32-unknown-unknown
   Compiling hello-unc v0.1.0 (~/hello-unc)
    Finished release [optimized] target(s) in 0.24s
$ ls target/wasm32-unknown-unknown/release/hello_unc.wasm
.rwxr-xr-x 106 matklad 15 Nov 15:34 target/wasm32-unknown-unknown/release/hello_unc.wasm
```

106 bytes is pretty small! Let's see what's inside. For that, we'll use
the `wasm-tools` suite of CLI utilities.

```console
$ cargo install wasm-tools
λ wasm-tools print target/wasm32-unknown-unknown/release/hello_unc.wasm
(module
  (memory (;0;) 16)
  (global $__stack_pointer (;0;) (mut i32) i32.const 1048576)
  (global (;1;) i32 i32.const 1048576)
  (global (;2;) i32 i32.const 1048576)
  (export "memory" (memory 0))
  (export "__data_end" (global 1))
  (export "__heap_base" (global 2))
)
```

## Rust Contract

Finally, let's implement an actual contract. We'll need an `extern "C"` block to
declare the `value_return` import, and a `#[no_mangle] extern "C"` function to
declare the `hello` export:

```rust
// hello-unc/src/lib.rs

#![no_std]

extern "C" {
    fn value_return(len: u64, ptr: u64);
}

#[no_mangle]
pub extern "C" fn hello() {
    let msg = "hello world";
    unsafe { value_return(msg.len() as u64, msg.as_ptr() as u64) }
}

#[panic_handler]
fn panic_handler(_info: &core::panic::PanicInfo) -> ! {
    core::arch::wasm32::unreachable()
}
```

After building the contract, the output wasm shows us that it's roughly what we
want:

```console
$ cargo b -r --target wasm32-unknown-unknown
   Compiling hello-unc v0.1.0 (/home/matklad/hello-unc)
    Finished release [optimized] target(s) in 0.05s
$ wasm-tools print target/wasm32-unknown-unknown/release/hello_unc.wasm
(module
  (type (;0;) (func (param i64 i64)))
  (type (;1;) (func))
  (import "env" "value_return"        (; <- Here's our import. ;)
    (func $value_return (;0;) (type 0)))
  (func $hello (;1;) (type 1)
    i64.const 11
    i32.const 1048576
    i64.extend_i32_u
    call $value_return
  )
  (memory (;0;) 17)
  (global $__stack_pointer (;0;) (mut i32) i32.const 1048576)
  (global (;1;) i32 i32.const 1048587)
  (global (;2;) i32 i32.const 1048592)
  (export "memory" (memory 0))
  (export "hello" (func $hello))      (; <- And export! ;)
  (export "__data_end" (global 1))
  (export "__heap_base" (global 2))
  (data $.rodata (;0;) (i32.const 1048576) "hello world")
)
```

## Deploying the Contract

Now that we have the WASM, let's deploy it!

```console
$ unc_ENV=local unc deploy alice.test.unc \
    ./target/wasm32-unknown-unknown/release/hello_unc.wasm
Loaded master account test.unc key from /home/matklad/.unc/validator_key.json with public key = ed25519:ChLD1qYic3G9qKyzgFG3PifrJs49CDYeERGsG58yaSoL
Starting deployment. Account id: alice.test.unc, node: http://127.0.0.1:3030, helper: http://localhost:3000, file: ./target/wasm32-unknown-unknown/release/hello_unc.wasm
Transaction Id GDbTLUGeVaddhcdrQScVauYvgGXxSssEPGUSUVAhMWw8
To see the transaction in the transaction explorer, please open this url in your browser
http://localhost:9001/transactions/GDbTLUGeVaddhcdrQScVauYvgGXxSssEPGUSUVAhMWw8
Done deploying to alice.test.unc
```

And, finally, let's call our contract:


```console
$ unc_ENV=local $unc call alice.test.unc hello --accountId alice.test.unc
Scheduling a call: alice.test.unc.hello()
Loaded master account test.unc key from /home/matklad/.unc/validator_key.json with public key = ed25519:ChLD1qYic3G9qKyzgFG3PifrJs49CDYeERGsG58yaSoL
Doing account.functionCall()
Transaction Id 9WMwmTf6pnFMtj1KBqjJtkKvdFXS4kt3DHnYRnbFpJ9e
To see the transaction in the transaction explorer, please open this url in your browser
http://localhost:9001/transactions/9WMwmTf6pnFMtj1KBqjJtkKvdFXS4kt3DHnYRnbFpJ9e
'hello world'
```

Note that we pass `alice.test.unc` twice: the first time to specify which contract
we are calling, the second time to determine who calls the contract. That is,
the second account is the one that spends tokens. In the following example `bob`
spends UNC to call the contact deployed to the `alice` account:


```console
$ unc_ENV=local $unc call alice.test.unc hello --accountId bob.test.unc
Scheduling a call: alice.test.unc.hello()
Loaded master account test.unc key from /home/matklad/.unc/validator_key.json with public key = ed25519:ChLD1qYic3G9qKyzgFG3PifrJs49CDYeERGsG58yaSoL
Doing account.functionCall()
Transaction Id 4vQKtP6zmcR4Xaebw8NLF6L5YS96gt5mCxc5BUqUcC41
To see the transaction in the transaction explorer, please open this url in your browser
http://localhost:9001/transactions/4vQKtP6zmcR4Xaebw8NLF6L5YS96gt5mCxc5BUqUcC41
'hello world'
```
