---
layout: post
title: "Hecto, Chapter 1: Setup"
categories: [Rust, hecto]
---
Ahh, step 1. Don't you love a fresh start on a blank slate? And then selecting that singular brick onto which you will build your entire palatial estate?

Unfortunately, when you're building a *computer program*, step 1 can get... complicated. And frustrating. You have to make sure your environment is set up for the programming language you're using, and you have to figure out how to
compile and run your program in that environment.

Fortunately, it's very fairly to set up a development environment with Rust, and we won't be needing anything besides a text editor, `rust` and `cargo`. To install these programs, we will be using a program called `rustup`, but there are other ways available to get Rust up and running.

It's usually a bit easier to install and run rust on Linux environments than on Windows, and the easiest way to have a Linux environment *on* Linux is using the Windows Subsystem for Linux (WSL), which [is available for Windows 10](https://docs.microsoft.com/en-us/windows/wsl/install-win10). This tutorial was written and tested on macOS and on Windows Subsystem for Linux, so you might run into issues when using it under Windows.

The installation will use a terminal on Linux and `cmd` on Windows, which we will also use to run our programs later on.

## How to install rust with `rustup`
If you visit the [rustup web site](https://rustup.rs/), it tries to auto-detect your operating system and displays the best way to get rustup installed. Usually, you download and execute a script, `rustup-init`, which does the installation for you.

However, if downloading and executing a remote script is a red flag to you, you can click on [other installation options](https://github.com/rust-lang/rustup.rs/#other-installation-methods) and download `rust-init` directly for your platform to start the installation.

### Finishing the installation on Linux, macOS or Windows Subsystem for Linux (WSL)
If you are on Linux or macOS, or if you are using WSL, the installer will tell you that it's finished by printing:
```
Rust is installed now. Great!
```
To start using Rust, you either need to restart your terminal or type
```bash
source $HOME/.cargo/env
```

### Finishing the installation on Windows
If you are installing on Windows, at some point during the installation you'll be told that you need the C++ build tools for Visual Studio 2013 or later. The easiest way of getting these is to [download and install the Build Tools for Visual Studio 2019](https://www.visualstudio.com/downloads/#build-tools-for-visual-studio-2019).

After the installation is done, make sure rust is in your `%PATH%` system variable.

### Adding a linker
Rust needs a linker of some kind to operate. It's likely that it's already installed, but when you try to compile Rust problems and get errors telling you that the linker could not be executed, you need to install one manually. C comes with a correct linker, so it's worth following the [original tutorial](https://viewsourcecode.org/snaptoken/kilo/index.html) on how to install C.

### Check your installation
To verify that Rust has been installed correctly, run the following command (On Windows, you need to add `.exe` to the program name):
```bash
rustc --version
```

For Cargo, run:
```bash
cargo --version
```
In both cases, you should see the program name, the version number and a few other information. If not, please refer to the Installation chapter of the [official Rust book](https://doc.rust-lang.org/book/ch01-01-installation.html) to troubleshoot your installation.

## The main function
Go to a directory where you would like to start building and type

```bash
cargo init hecto
```
`hecto` is the name of the text editor we will be building. Executing this command will create a folder called `hecto` which has already set up git (and therefore includes a folder called `.git` and a file called `.gitignore`). We are not going to use git for this tutorial, so you can ignore this for now.

It also creates a file called `Cargo.toml`, which is pre-filled with some information like the author or application name. If you are familiar with JavaScript, the `Cargo.toml` is comparable to the `package.json`. It describes your package as well as its dependencies to other packages.

Finally, there is a file called `src/main.rs`. When you open it, it already contains the following code:
```rust
fn main() {
    println!("Hello, world!");
}
```
This code defines a function. The `main()` function is special. It is the default starting point when you run your program. When you return from the main() function, the program exits and passes the control back to the operating system.

Rust is a compiled language. That means we need to run our program through a Rust compiler to turn it into an executable file. We then run that executable like we would run any other program on the command line.

To compile `main.rs`, make sure you are in the `hecto` folder (Run `cd hecto` after `cargo init hecto`), and then run `cargo build` in your shell. The output will look similar to this:
```
    Compiling hecto v0.1.0 (/Users/flenker/Documents/Repositories/hecto)
    Finished dev [unoptimized + debuginfo] target(s) in 0.45s
```
This will produce an executable named `hecto` and place it in a new folder called `target/debug/`.
Additionally, a new file will be created called `Cargo.lock`. It is automatically generated and should not be touched. When we add dependencies later to the project, the `Cargo.lock` will also be updated.

If you look further at the contents of `target/debug`, you will find that several more files have been generated.  These files are mostly needed by `cargo`to make rebuilding the code more efficient (more on that in a second)
You won't need any of these files to run `hecto` except for the executable itself, to run `hecto`, type `./target/debug/hecto` and press <kbd>Enter</kbd>. The program should output `Hello, world!` and then exit.

### Compiling and running
Since it's very common that you want to compile and run your program, Rust combines both steps with the command `cargo run`.
If you run that command now after `cargo build`, you might notice that the output changes a bit. It now looks similar to this:
```
    Finished dev [unoptimized + debuginfo] target(s) in 0.00s
     Running `target/debug/hecto`
Hello, world!
```
As you might have noticed, `cargo` does not output a line starting with `Compiling`, as it dit before.
That it because rust can tell that the current version of `main.rs` has already been compiled. If  the `main.rs` was not modified since the last compilation, then `rust` doesn’t bother running the compilation again. If `main.rs` was changed, then `rust` recompiles `main.rs`. This is more useful for large projects with many different components to compile, as most of the components shouldn’t need to be recompiled over and over when you’re only making changes to one component’s source code.

Try changing the return value in `main.rs` to a string other than `Hello, World`. Then run `cargo run`, and you should see it compile. Check the result to see if you get the string you changed it to. Then change it back to `Hello, World`, recompile,and make sure it's back to returning `Hello, World`.

After each step in this tutorial, you will want to recompile `main.rs`, see if it finds any errors in your code, and then run `hecto` by calling `cargo run`.

In the next chapter, we'll work on reading individual keypresses from the user.