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

What we want is **raw mode**. Fortunately, there are external libraries available to set the terminal to raw mode. Libraries in Rust are called Crates, and [if you haven't, it's worth reading up on these](https://doc.rust-lang.org/book/ch07-00-managing-growing-projects-with-packages-crates-and-modules.html). Like many other programming languages, Rust comes with a lean core and relies on crates to extend its functionality.

(**Side Note:** *We could have done everything we are doing in this tutorial completely manually. We could, on the other hand, also have done a lot of what we are doing in this tutorial by using various crates. I have decided against adding external libraries where I could, but added them, as in this case, whenever the effort of boilerplate code to get things running outweighed the learning value. However, all crates used are publicly available, and I encourage you to read through their source code just to get an overview over what you missed.*)

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
    return stdout().into_raw_mode().expect("Could not enable raw mode")
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

Second, we are catching the error which might be passed to us from termion, and then we are returning the result to `main` - where we are not using it. Why? Because `into_raw_mode` returns a `RawTerminal` which, once it goes out of scope, will reset the terminal into canonical mode. You can try it out by removing `let _stdout =` - the terminal won't stay in raw mode. We are actually telling the rust compiler (and others reading our code) that we want to hold on to `_stdout` even though we are not using it by prefixing the variable with an underscore.

## Display keypresses

To get a better idea of how input in raw mode works, let's improve on how we print out each byte
that we read. 

```rust
use std::io::{self,  stdout, Write, Stdout};
use termion::raw::{IntoRawMode, RawTerminal};
use termion::input::TermRead;
use termion::event::Key;


fn enable_raw_mode() -> RawTerminal<Stdout>{
    return stdout().into_raw_mode().expect("Could not enable raw mode")
}

fn main() {
    let mut stdout = enable_raw_mode();
    
    for k in io::stdin().lock().keys() {        
        let key = k.unwrap();
        match key {
            Key::Char('q') => break,
            Key::Char(c) => print!("{}\r\n", c),
            _ => print!("{:?}\r\n", key)
        }
        stdout.flush().unwrap();
    }
}
```

That is a big change, so a few explanations are in order:
- Now that we are using termion, we can iterate over `keys` instead of `bytes`. This helps us capture characters which are bigger than one byte.
- `stdout` is no longer unused - we flush it after each iteration, which causes our program to display the characters right away. Try removing that line and see what happens!
- We are using `match` instead of `if`. Match is an extremely useful operator, and it's worth checking out [the docs]((https://doc.rust-lang.org/book/ch06-02-match.html) ) if you haven't already.
- Instead of using `println!`, we are using `print!`, and we are manually adding `\r\n`. This makes sure our output is neatly printed line by line without indendation.

This is a very useful program. It shows us how various keypresses translate
into the characters we read. Most ordinary keys translate directly into the
characters they represent. But try seeing what happens when you press the arrow
keys, or <kbd>Escape</kbd>, or <kbd>Page Up</kbd>, or <kbd>Page Down</kbd>, or
<kbd>Home</kbd>, or <kbd>End</kbd>, or <kbd>Backspace</kbd>, or
<kbd>Delete</kbd>, or <kbd>Enter</kbd>. Try key combinations with
<kbd>Ctrl</kbd>, like <kbd>Ctrl-A</kbd>, <kbd>Ctrl-B</kbd>, etc.

You'll notice a few interesting things:
- Most characters show up as expected, as termion converts the bytes in their correct character counterparts
- Nonstandard characters such as German umlauts work as well
- <kbd>Ctrl-A</kbd> is `Ctrl('a')`, <kbd>Ctrl-B</kbd> is `Ctrl('b')`, <kbd>Ctrl-C</kbd> is ...`Ctrl('c')` and does not exit your program, as you might have expected
- This is also true for other combinations, such as `<kbd>Ctrl-S</kbd>, <kbd>Ctrl-A</kbd> or <kbd>Ctrl-Z</kbd>. Interestingly, though, <kbd>Ctrl-V</kbd> still pastes in (and reads) the clipboard on Windows.
- <kbd>Ctrl-Alt-A</kbd> translates into `Alt('\u{1}')`, and the other variants with <kbd>Ctrl</kbd>  and <kbd>Alt</kbd> behave similarily. This is just an issue on how the default logging for Keys works: `\u{1}` is representing the value for <kbd>Strg+A</kbd>
- <kbd>Ctrl-M</kbd> and <kbd>Ctrl-J</kbd> both seem to produce nothing. However, with our new-found knowledge, we can press <kbd>Ctrl-Alt-M</kbd> to see the representation of <kbd>Ctrl-M</kbd>: it's `\r`, the carriage return. Likewise, <kbd>Ctrl-J</kbd> produces `\n`, a newline.

## Sections

That just about concludes this chapter on entering raw mode. The last thing we’ll do now is split our code into sections. This will allow these diffs to be shorter, as each section that isn’t changed in a diff will be folded into a single line.

```rust
/*** includes ***/
use std::io::{self,  stdout, Write, Stdout};
use termion::raw::{IntoRawMode, RawTerminal};
use termion::input::TermRead;
use termion::event::Key;


/*** terminal***/
fn enable_raw_mode() -> RawTerminal<Stdout>{
    return stdout().into_raw_mode().expect("Could not enable raw mode")
}

/*** init ***/
fn main() {
    let mut stdout = enable_raw_mode();
    
    for k in io::stdin().lock().keys() {        
        let key = k.unwrap();
        match key {
            Key::Char('q') => break,
            Key::Char(c) => print!("{}\r\n", c),
            _ => print!("{:?}\r\n", key)
        }
        stdout.flush().unwrap();
    }
}
```

In the next chapter, we'll do some more terminal input/output handling, and use that to draw to the screen and allow the user to move the cursor around.

