---
layout: post
title: "Hecto, Chapter 2: Reading User Input"
categories: [Rust, hecto]
---

Let’s try and read keypresses from the user. Remove the line with "Hello, world" from `main` and change your code as follows:

```rust
use std::io::{self, Read};

fn main() {
    for b in io::stdin().bytes() {
        let c = b.unwrap() as char;
        println!("{}", c);
    }
}
```
Play around with that program and try to find out how it works. To stop it, press <kbd>Ctrl-C</kbd>.

First, we are using `use` to import things into our program. `use std::io::{self, Read}` is short for:

```rust
use std::io;
use std::io::Read;
```
After this, we are able to use `io` in our code, and bringing `Read` into our code enables us to use `bytes()`. Try running your code without importing `Read`, and the compiler will exit with an error explaining that `Read` needs, in fact, to be in scope, because it brings the implementation of `bytes` with it. This concept is called a *trait*, and while we will be using traits a lot by importing them, we won't be working with them directly in this tutorial. The [documentation on traits](https://doc.rust-lang.org/book/ch10-02-traits.html) is definitely something to add on your reading list!

If you are new to Rust, don't worry. We have a bit of learning to do in the beginning, but future code additions won't bring as many new concepts at once as this one.

The next line does a lot of things at once, which can be summarized as "For every byte you can read from the keyboard, bind it to `c` and execute the following block".

Let's unravel that statement. `io::stdin()` calls a method called `stdin` from `io`, which we just imported. `stdin` represents the [Standard Input Stream](https://en.wikipedia.org/wiki/Standard_streams#Standard_input_(stdin)), which, simply put, gives you access to everything that can be put into your program.

Calling `bytes()` next returns something we can *iterate over*, or in other words: Something which lets us perform the same task on a series of elements. In Rust, same as many other languages, [this concept is called an Iterator.](https://doc.rust-lang.org/book/ch13-02-iterators.html)

Using an iterator allows us to build a loop with `for..in`. With `for..in` in combination with `bytes()`, we are asking rust to read byte from the standard input into the variable `b`, and to keep doing
it until there are no more bytes to read. The block after `for..in` returns the byte that it read, and will end when it reaches the end of a file.

We will explain `unwrap` and `println!` later in this tutorial.

When you run `./hecto`, your terminal gets hooked up to the standard input, and so your keyboard input gets read into the `b` variable. However, by default your terminal starts in **canonical mode**, also called **cooked mode**. In this mode, keyboard input is only sent to your program when the user presses <kbd>Enter</kbd>. This is useful for many programs: it lets the user type in a line of text, use <kbd>Backspace</kbd> to fix errors until they get their input exactly the way they want it, and finally press <kbd>Enter</kbd> to send it to the program. But it does not work well for programs with more complex user interfaces, like text editors. We want to process each keypress as it comes in, so we can respond to it immediately.

To exit the above program, press <kbd>Ctrl-D</kbd> to tell Rust that it's reached the end of file. Or you can always press <kbd>Ctrl-C</kbd> to signal the process to terminate immediately.

What we want is **raw mode**. Fortunately, there are external libraries available to set the terminal to raw mode. Libraries in Rust are called Crates, and [if you haven't, it's worth reading up on these](https://doc.rust-lang.org/book/ch07-00-managing-growing-projects-with-packages-crates-and-modules.html). Like many other programming languages, Rust comes with a lean core and relies on crates to extend its functionality. In this tutorial, we will sometimes do things manually first before switching to external functionality, and sometimes we jump directly to the library function.

## Press <kbd>q</kbd> to quit?

To demonstrate how canonical mode works, we'll have the program exit when it
reads a <kbd>q</kbd> keypress from the user.

```rust
use std::io::{self, Read};

fn main() {
    for b in io::stdin().lock().bytes() {
        let c = b.unwrap() as char;
        println!("read char  {}", c);
        if c == 'q' {
            break;
        }
    }
}
```
Note that in Rust, characters require single quotes, `'` , instead of double quotes, `"`, to work!

To quit this program, you will have to type a line of text that includes a `q` in it, and then press enter. The program will quickly read the line of text one character at a time until it reads the `q`, at which point the `for..in` loop will stop and the program will exit. Any characters after the `q` will be left unread on the input queue, and you may see that input being fed into your shell after your program exits.

## Entering raw mode by using termion
Open the `Cargo.toml` file. Change the `[dependencies]` section of that file to look like this:
```toml
[dependencies]
termion = "1"
```
With this, we are telling `cargo` that we want to have a dependency called `termion`, in the version 1. Cargo follows a concept called [Semantic Versioning](https://semver.org/), where a program version usually consists of three numbers (like 0.1.0), and by convention, no breaking change occurs as long as the first number stays constant. That means that if you develop against `termion v1.5.0`, your program will also work with `termion v1.5.1` or even `termion v1.7.0`. This is useful, because it means that we are getting bugfixes and new features, but the existing features can still be used without us having to change our code.
By setting `temion = "1"`, we are making sure we are getting the latest version starting with `1`.

Next time you run `cargo build` or `cargo run`, the new dependency, `termion` will be downloaded and compiled.

`termion` comes with dependencies itself, and `cargo` downloads and compiles them, too. You might notice that the `Cargo.lock` has also changed: It now contains the exact names and versions of all packages and dependencies which have been installed. This is helpful to avoid "Works on my machine" - bugs if you are working on a team, where you are encountering a bug in, say, `termion v1.2.3`, while your co-worker is on `termion v1.2.4` and doesn't see it.

Now change the `main.rs` as follows:
```rust
use std::io::{self,stdout, Read};
use termion::raw::IntoRawMode;

fn main() {
    let _stdout = stdout().into_raw_mode().unwrap();

    for b in io::stdin().bytes() {
        let c = b.unwrap() as char;
        println!("{}", c);
        if c == 'q' {
            break;
        }
    }
}
```

There are a few things to note here. First, we are using `termion` to set `stdout`, the [counterpart of `stdin` from above](https://en.wikipedia.org/wiki/Standard_streams#Standard_output_(stdout))  into raw mode. But why are we calling that method on `stdout` to change how we read from `stdin`? The answer is that terminals have their states controlled by the writer, not the reader. The writer is used to draw on the screen or move the cursor, so it is also used to change the mode as well.

Second, we are not using the value in  `_stdout`. Why? Because this is our first encounter with Rust's [Ownership System](https://doc.rust-lang.org/book/ch04-00-understanding-ownership.html). To brutally summarize a complex concept, functions can *own* certain things. Un-owned things will be removed, and  `into_raw_mode` returns a `RawTerminal` which, once it goes out of scope, will reset the terminal into canonical mode, so we need to keep it around by binding it to `_stdout`. You can try it out by removing `let _stdout =` - the terminal won't stay in raw mode.

By prefixing the variable with a `_`, we are actually telling others reading our code that we want to hold on to `_stdout` even though we are not using it. Compiling the program with a variable name without `_` will print a helpful warning message.

Though the topic of ownership is complex, you don't need to fully understand it at this point. Your understanding will grow over the course of this tutorial.

## Observing keypresses

To get a better idea of how input in raw mode works, let's improve on how we print out each byte that we read.

```rust
use std::io::{self,stdout, Read};
use termion::raw::IntoRawMode;

fn main() {
    let _stdout = stdout().into_raw_mode().unwrap();

    for b in io::stdin().bytes() {
        let b = b.unwrap();
        let c = b as char;
        if c.is_control() {
            print!("{:?} \r\n", b);
        } else {
            print!("{:?} ({})\r\n", b,c);
        }
        if c == 'q' {
            break;
        }
    }
}
```
In case you are wondering, in Rust, it is perfectly legal to declare a variable twice by doing `let b = ...` twice. This is called variable shadowing, and will be immensely useful later. It saves us now from awkward variable names such as `wrapped_byte` vs `byte`. (I promise, we will get to `unwrap` soon!). Try playing around with this concept by dropping the `let` on the second statement.

By the way, the `as` keyword attempts to transform a primitive value into another one, in this case a byte into a single `char`.

`is_control()` tests whether a character is a control character. Control characters are nonprintable characters that we don't want to print to the screen. ASCII codes 0&ndash;31 are all control characters, and 127 is also a control character. ASCII codes 32&ndash;126 are all printable. (Check out the [ASCII table](http://asciitable.com) to see all of the characters.)

`print!()`, same as `println!` before, is a macro which prints something to the screen.
`{}` and `{:?}` are placeholders which are filled with the remaining parameters to the macro. `{}` is a placeholer for elements for which a string representation is known, such as a `char`, and `{:?}` is a placeholder for elements for which a string representation is not known, but a "debug string representation" has been implemented. To understand the difference, try running the code with `{}` instead of `{:?}`. Contrast it with the behaviour if you then use `{:?}` instead of `{}`, even for characters.

Instead of using `println!`, which prints its parameters in a new line, we are now using `print!`, and we are manually adding `\r\n`. This makes sure our output is neatly printed line by line without indendation. The reason is that usually, terminals do some kind of output processing for us and convert a newline (`\n`) to carriage return and newline (`\r\n`). The carriage return moves the cursor back to the beginning of the current line, and the newline moves the cursor down a line, scrolling the screen if necessary. (These two distinct operations originated in the days of typewriters and [teletypes](https://en.wikipedia.org/wiki/Teleprinter).)

By switching to Raw Mode, we have disabled this feature, and therefore a newline (as used by `println!`) only moves the cursor down, but not to the left. Try using `println!` without `\r\n` to see the difference!

This is a very useful program. It shows us how various keypresses translate into the characters we read. Most ordinary keys translate directly into the characters they represent. But try seeing what happens when you press the arrow keys, or <kbd>Escape</kbd>, or <kbd>Page Up</kbd>, or <kbd>Page Down</kbd>, or <kbd>Home</kbd>, or <kbd>End</kbd>, or <kbd>Backspace</kbd>, or <kbd>Delete</kbd>, or <kbd>Enter</kbd>. Try key combinations with <kbd>Ctrl</kbd>, like <kbd>Ctrl-A</kbd>, <kbd>Ctrl-B</kbd>, etc.

You'll notice a few interesting things:
-  Arrow keys, <kbd>Page Up</kbd>, <kbd>Page Down</kbd>, <kbd>Home</kbd>, and <kbd>End</kbd> all input 3 or 4 bytes to the terminal: `27`, `'['`, and then one or two other characters. This is known as an *escape sequence*. All escape sequences start with a `27` byte. Pressing <kbd>Escape</kbd> sends a single `27` byte as input, which explains either the name of the key or the sequence.
- <kbd>Backspace</kbd> is byte `127`.
- <kbd>Enter</kbd> is byte `13`, which is a carriage return character, also known as `'\r'` - and not, as you might expect, a newline, `'\n'`
- Special characters such as German umlauts also produce multiple bytes.
- <kbd>Ctrl-A</kbd> is `1`, <kbd>Ctrl-B</kbd> is `2`, <kbd>Ctrl-C</kbd> is... `3` and doesn't terminate the program as you might have expected. And the rest of the <kbd>Ctrl</kbd> key combinations  seem to map the letters A&ndash;Z to the codes 1&ndash;26.

## Error Handling
It's time to think about how we handle errors. First, we add a `die()` function that prints an error message and exits the program.

```rust
use std::io::{self,stdout, Read};
use termion::raw::IntoRawMode;

fn die(e: std::io::Error) {
    panic!(e);
}

fn main() {
    let _stdout = stdout().into_raw_mode().unwrap();

    for b in io::stdin().bytes() {
        let b = b.unwrap();
        let c = b as char;
        if c.is_control() {
            print!("{:?} \r\n", b);
        } else {
            print!("{:?} ({})\r\n", b,c);
        }
        if c == 'q' {
            break;
        }
    }
}
```
`panic!` is a macro which crashes the program with an error message.  Unlike some other programming languages, Rust does not allow you to add some kind of `try..catch` block around the code to catch any error that might occur. Instead, we are propagating errors up alongside the function return values, which will allow us to treat errors at the highest level.

This propagation works so that a function where an error could happen returns something called a [Result](https://doc.rust-lang.org/book/ch09-02-recoverable-errors-with-result.html), which represents either the return value we originally wanted, or an error. Every value in `b` is originally a `Result`, which either holds the byte we have read in, or an Error object, indicating that something went wrong while reading the byte. To get the value we need, we can call `unwrap`, which is short for: "Return the value, or `panic` if there was an error".

We want to control the crash ourselves, because later on we want to clear the screen before a crash occurs, to not leave the user with half-drawn input. For now, let's simply check for an error and call `die`, which panics for us.

 Let's implement that now.

```rust
use std::io::{self,stdout, Read};
use termion::raw::IntoRawMode;

fn die(e: std::io::Error) {
    panic!(e);
}

fn main() {
    let _stdout = stdout().into_raw_mode().unwrap();

    for b in io::stdin().bytes() {
        match b {
            Ok (b) => {
                let c = b as char;
                if c.is_control() {
                    print!("{:?} \r\n", b);
                } else {
                    print!("{:?} ({})\r\n", b,c);
                }
                if c == 'q' {
                    break;
                }
            },
            Err(err) => die(err)
        }

    }
}
```
Here are a few more things to observe. `match` is like a supersized `if-then-else`, which we use to figure out if we got an actual value (wrapped in `Ok`), or an error (wrapped in `err`). Note that our variable shadowing from above is now handled by `match` - in the `Ok` case, `b` now contains the unwrapped value.
Another thing of interest is that `match` must have cases for all values that `b` can have - in this case, `Ok` or `Err`. Try removing the line with the handling of `Err` to see if the code compiles without it. We will investigate `match` more deeply later. [Here's the link to the docs in case you are interested.](https://doc.rust-lang.org/book/ch06-02-match.html)

We have added some error handling  now - but we are deliberately ignoring the error from `into_raw_mode`. Our error handling is mainly aimed at avoiding garbled output, which can only occur when we are actually repeatedly writing to the screen, so for our purposes, there is no need for any additional error handling before our loop begins.

That concludes this chapter on entering raw mode. In the next chapter, we'll do some more terminal input/output handling, and use that to draw to the screen and allow the user to move the cursor around.