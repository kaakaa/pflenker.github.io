---
layout: post
title: "Hecto, Chapter 2: Reading User Input"
categories: [Rust, hecto]
---

Let’s try and read keypresses from the user. Remove the line with "Hello, world" and change your code as follows:

```rust
use std::io::{self, Read};

fn main() {
    for b in io::stdin().lock().bytes() {
        let c = b.unwrap() as char;
        println!("read char  {}", c.unwrap());
    }
}
```

We are importing `stdin` from `std::io`. With `for..in` in combination with `bytes()`, we are asking rust
to read `1` byte from the standard input into the variable `b`, and to keep doing
it until there are no more bytes to read. The block after `for..in` returns the byte
that it read, and will end when it reaches the end of a file.

When you run `./hecto`, your terminal gets hooked up to the standard input, and
so your keyboard input gets read into the `b` variable. However, by default
your terminal starts in **canonical mode**, also called **cooked mode**. In
this mode, keyboard input is only sent to your program when the user presses
<kbd>Enter</kbd>. This is useful for many programs: it lets the user type in a
line of text, use <kbd>Backspace</kbd> to fix errors until they get their input
exactly the way they want it, and finally press <kbd>Enter</kbd> to send it to
the program. But it does not work well for programs with more complex user
interfaces, like text editors. We want to process each keypress as it comes in,
so we can respond to it immediately.

What we want is **raw mode**. Fortunately, there are external libraries available to set the terminal to raw mode. Libraries in Rust are called Crates, and [if you haven't, it's worth reading up on these](https://doc.rust-lang.org/book/ch07-00-managing-growing-projects-with-packages-crates-and-modules.html). Like many other programming languages, Rust comes with a lean core and relies on crates to extend its functionality. In this tutorial, we will sometimes do things manually first before switching to external functionality, sometimes we jump directly to the library function (as in this case), and sometimes we won't use external libraries at all..

To exit the above program, press <kbd>Ctrl-D</kbd> to tell `read()` that it's
reached the end of file. Or you can always press <kbd>Ctrl-C</kbd> to signal
the process to terminate immediately.

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
Note that characters require single quotes, `'` , instead of double quotes, `"`, to work!

To quit this program, you will have to type a line of text that includes a `q`
in it, and then press enter. The program will quickly read the line of text one
character at a time until it reads the `q`, at which point the `for..in` loop
will stop and the program will exit. Any characters after the `q` will be left
unread on the input queue, and you may see that input being fed into your shell
after your program exits.

## Entering raw mode by using termion
Open the `Cargo.toml` file. Change the `[dependencies]` section of that file to look like this:
```toml
[dependencies]
termion = "*"
```

Next time you run `cargo run`, the new dependency, `termion` , and its dependencies will be downloaded and compiled.

Now change the `main.rs` as follows:
```rust
use std::io::{self,  stdout, Read, Stdout};
use termion::raw::{IntoRawMode, RawTerminal};


fn enable_raw_mode() -> RawTerminal<Stdout>{
    return stdout().into_raw_mode().unwrap();
}

fn main() {
    let _stdout = enable_raw_mode();

    for b in io::stdin().lock().bytes() {
        let c = b.unwrap() as char;
        println!("read char {}", c);
        if c == 'q' {
            break;
        }
    }
}
```

There are a few things to note here. First, in `enable_raw_mode`, we are using `termion` to set the standard output into raw mode. But why are we modifying `stdout` to change how we read from `stdin`? The answer is that terminals have their states controlled by the writer, not the reader. The writer is used to draw on the screen or move the cursor, so it is also used to change the mode as well.

Second, we are unwrapping the value retunred from termion, and then we are returning the result to `main` - where we are not using it. Why? Because `into_raw_mode` returns a `RawTerminal` which, once it goes out of scope, will reset the terminal into canonical mode, so we need to keep it around. You can try it out by removing `let _stdout =` - the terminal won't stay in raw mode. We are actually telling the rust compiler (and others reading our code) that we want to hold on to `_stdout` even though we are not using it by prefixing the variable with an underscore.

## Display keypresses

To get a better idea of how input in raw mode works, let's improve on how we print out each byte
that we read.

```rust
use std::io::{self,  stdout, Read, Stdout};
use termion::raw::{IntoRawMode, RawTerminal};


fn enable_raw_mode() -> RawTerminal<Stdout>{
    return stdout().into_raw_mode().unwrap();
}

fn main() {
    let _stdout = enable_raw_mode();

    for byte in io::stdin().lock().bytes() {
        let byte = byte.unwrap();
        let c = byte as char;
        if c.is_control() {
            print!("{:?} \r\n", byte);
        } else {
            print!("{:?} ({})\r\n", byte,c);
        }
        if c == 'q' {
            break;
        }
    }
}
```
`is_control()` tests whether a character is a control character. Control characters are nonprintable characters that we don't want to print to the screen. ASCII codes 0&ndash;31 are all control characters, and 127 is also a control character. ASCII codes 32&ndash;126 are all printable. (Check out the [ASCII table](http://asciitable.com) to see all of the characters.)

`print!()`, same as `println!` before, is a macro which prints something to the screen.
`{}` and `{:?}` are placeholders which are filled with the remaining parameters to the macro. `{}` is a placeholer for elements for which a string representation is known, such as a `char`, and `{:?}` is a placeholder for elements for which a string representation is not known. To understand the difference, try running the code with `{}` instead of `{:?}`. Contrast it with the behaviour if you then use `{:?}` instead of `{}`, even for characters.

Instead of using `println!`, which prints its parameters in a new line, we are now using `print!`, and we are manually adding `\r\n`. This makes sure our output is neatly printed line by line without indendation. The reason is that usually, terminals do some kind of output processing for us and convert a newline (`\n`) to carriage return and newline (`\r\n`). The carriage return moves the cursor back to the beginning of the current line, and the newline moves the cursor down a line, scrolling the screen if necessary. (These two distinct operations originated in the days of typewriters and [teletypes](https://en.wikipedia.org/wiki/Teleprinter).)

By switching to Raw Mode, we have disabled this feature, and therefore a newline (as used by `println!`) only moves the cursor down, but not to the left. Try using `println!` without `\r\n` to see the difference!

This is a very useful program. It shows us how various keypresses translate into the characters we read. Most ordinary keys translate directly into the characters they represent. But try seeing what happens when you press the arrow keys, or <kbd>Escape</kbd>, or <kbd>Page Up</kbd>, or <kbd>Page Down</kbd>, or <kbd>Home</kbd>, or <kbd>End</kbd>, or <kbd>Backspace</kbd>, or <kbd>Delete</kbd>, or <kbd>Enter</kbd>. Try key combinations with <kbd>Ctrl</kbd>, like <kbd>Ctrl-A</kbd>, <kbd>Ctrl-B</kbd>, etc.

You'll notice a few interesting things:
-  Arrow keys, <kbd>Page Up</kbd>, <kbd>Page Down</kbd>, <kbd>Home</kbd>, and <kbd>End</kbd> all input 3 or 4 bytes to the terminal: `27`, `'['`, and then one or two other characters. This is known as an *escape sequence*. All escape sequences start with a `27` byte. Pressing <kbd>Escape</kbd> sends a single `27` byte as input, which explains either the name of the key or the sequence.
- <kbd>Backspace</kbd> is byte `127`.
- <kbd>Enter</kbd> is byte `13`, which is a carriage return character, also known as `'\r'` - and not, as you might expect, a newline, `'\n'`
- Nonstandard characters such as German umlauts also produce multiple bytes.
- <kbd>Ctrl-A</kbd> is `1`, <kbd>Ctrl-B</kbd> is `2`, <kbd>Ctrl-C</kbd> is... `3` and doesn't terminate the program as you might have expected. And the rest of the <kbd>Ctrl</kbd> key combinations  seem to map the letters A&ndash;Z to the codes 1&ndash;26.

## Error Handling
It's time to think about how we handle errors. First, we add a `die()` function that prints an error message and exits the program.

```rust
use std::io::{self,  stdout, Read, Stdout};
use termion::raw::{IntoRawMode, RawTerminal};

fn die(e: std::io::Error) {
    panic!(e);
}

fn enable_raw_mode() -> RawTerminal<Stdout>{
    return stdout().into_raw_mode().unwrap();
}

fn main() {
    let _stdout = enable_raw_mode();

    for byte in io::stdin().lock().bytes() {
        let byte = byte.unwrap();
        let c = byte as char;
        if c.is_control() {
            print!("{:?} \r\n", byte);
        } else {
            print!("{:?} ({})\r\n", byte,c);
        }
        if c == 'q' {
            break;
        }
    }
}
```
`panic!` is a macro which crashes the program with an error message. This is the same as what happens if calls like `unwrap` go wrong, so our `die()` function is useless for now - but we want to do some more cleanup later in case of an error. so we start thinking about errors now. Unlike some other programming languages, Rust does not allow you to add some kind of `try..catch` block around the code to catch any error that might occur. Instead, we are propagating errors up alongside the function return values, which will allow us to treat errors at the highest level - in `main`. Let's implement that now.

```rust
use std::io::{self,  stdout, Read, Stdout};
use termion::raw::{IntoRawMode, RawTerminal};

fn die(e: std::io::Error) {
    panic!(e);
}

fn enable_raw_mode() -> io::Result<RawTerminal<Stdout>>{
    return stdout().into_raw_mode();
}


fn main() {
    let mut _stdout = None;
    match  enable_raw_mode() {
        Ok(s) => _stdout = Some(s),
        Err(err) => die(err)
    }

    for byte in io::stdin().lock().bytes() {
        match byte {
            Ok(byte) => {
                let c = byte as char;
                if c.is_control() {
                    print!("{:?} \r\n", byte);
                } else {
                    print!("{:?} ({})\r\n", byte,c);
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
We have converted `_stdout` to a variable which can hold either `None` or the Raw Terminal from before, and we are using `match` in two places to check against the return type and call `die` on error.

## Sections

That just about concludes this chapter on entering raw mode. The last thing we’ll do now is split our code into sections. This will allow these diffs to be shorter, as each section that isn’t changed in a diff will be folded into a single line.

```rust
/*** includes ***/
use std::io::{self,  stdout, Read, Stdout};
use termion::raw::{IntoRawMode, RawTerminal};

/*** terminal***/
fn die(e: std::io::Error) {
    panic!(e);
}

fn enable_raw_mode() -> io::Result<RawTerminal<Stdout>>{
    return stdout().into_raw_mode();
}

/*** init ***/
fn main() {
    let mut _stdout = None;
    match  enable_raw_mode() {
        Ok(s) => _stdout = Some(s),
        Err(err) => die(err)
    }

    for byte in io::stdin().lock().bytes() {
        match byte {
            Ok(byte) => {
                let c = byte as char;
                if c.is_control() {
                    print!("{:?} \r\n", byte);
                } else {
                    print!("{:?} ({})\r\n", byte,c);
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

In the next chapter, we'll do some more terminal input/output handling, and use that to draw to the screen and allow the user to move the cursor around.
