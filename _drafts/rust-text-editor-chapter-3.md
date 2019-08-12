---
layout: post
title: "Hecto, Chapter 3: Raw input and output"
categories: [Rust, hecto]
---
In this chapter, we will tackle reading from and writing to the terminal. But first, we need to make our code more idiomatically.

## Writing idiomatic code
Whenever you deal with newer languages such as Rust, you will often hear about _idiomatic_ code. Apparently, it is not enough to make the code solve a problem, it should be _idiomatic_ as well. Let's discuss first why this makes sense. We start off with a linguistic example: If I told you that with this tutorial, you could _kill two flies with one swat_, because you learn Rust and write your very own editor at the same time, would you understand my meaning?

If you are a German, you would probably not have any trouble, because "Killing to flies with one swat" is a near-literal translation of a German saying. If you are Russian, "killing two rabbits with one shot" would be more understandable for you. But if you are not familiar with German or Russian, you would have to try and extract the meaning of these sentences out of the context. It's not terribly difficult, but it takes a bit of time, and if I would be talking to you, you would probably be missing my super important point while thinking about flies, rabbits and stones. The _idiomatic_ way of saying this in English is, of course, "To kill two birds with one stone". 

Writing idiomatic code is similar. It's easier to maintain for others, since it sticks to certain rules and conventions, which are what the designers of the language had in mind. Remember, your code is only reviewed when it doesn't work - either because a feature is missing and someone wants to extend it, or because it has a bug. Making it easier for others to read and understand your code is generally a good idea. 

We saw before that the Rust compiler can give you some advise on idiomatic code - for instance, when we prefixed a variable that we where not using with an underscore. My advise is to not ignore compiler warnings, your final code should always compile without warnings.

In this tutorial, though, we are sometimes adding functionality a bit ahead of time, which will cause Rust to complain about unreachable code. This is usually fixed a step or two later.

Let's start this chapter by making our code a bit more idiomatic.

## Read keys instead of bytes
In the previous steps, we have worked directly on bytes, which was both fun and valuable. However, at a certain point you should ask yourself if the funtionality you are implementing could not be replaced by a library function, as in many cases, someone else has already solved your problem, and probably better.
For me, handling with bit operations is a huge red flag that tells me that I am probably too deep down the rabbit hole.
Fortunately for us, our dependency, `termion`, makes things already a lot easier, as it can already group individual bytes to keypresses and pass them to us. Let's implement this.



```rust
use std::io::{self,stdout};
use termion::raw::IntoRawMode;
use termion::input::TermRead;
use termion::event::Key;

fn die(e: std::io::Error) {
    panic!(e);
}

fn main() {
    let _stdout = stdout().into_raw_mode().unwrap();
    for key in io::stdin().keys() {
        match key {
            Ok(key) => {
                match key {
                    Key::Char(c) => {
                        if c.is_control() {
                            print!("{:?}\r\n", c as u8);
                        } else {
                            print!("{:?} ({})\r\n", c as u8,c);
                        }
                    },
                    Key::Ctrl('q') => break,   
                    _ => print!("{:?}\r\n", key)
                }
            },
            Err(err) => die(err)
        }
        
    }
}
```
This still does not look great, as we have a lot of nested things going on - but we are now working with keys instead of bytes. With that change, we where able to get rid of manually checking if <kbd>Ctrl</kbd> has been pressed, as all keys are now properly handled for us. Note the subtle difference in the inner match: `Key::Char(c)` matches _any_ Character and binds it to `c`, ` Key::Ctrl('q')` matches the specific character `'q'`.
Another addition here is that we have a match against `_`. Matches need to be _exhaustive_, so every possibility must be handled. `_` is the default option for everything that has not been handled before. In this case, if anything is pressed that is neither a character nor <kbd>Ctrl-Q</kbd>, we just print it out.

Note that we had to import a few things to make our code working. Similar as with `into_raw_mode`, we need to import `TermRead` so that we can use the `keys` method on `stdin`, but we could remove `std::io::Read` in return.

## Separate the code into multiple files
It's idiomatic that the main method itself does not do much more than providing the entry point for the app. This is the same in many programming languages, and Rust is no exception. We want the code to be placed where it makes sense, so that it's easier to locate and maintain code later. There are more benefits as well, which I will explain when we encounter them.

The code we have is too low-level for now. We have to understand the whole code to understand that in essence, it prints out key input and quits if <kbd>Ctrl-Q</kbd> is being pressed. Let's improve this code by creating a code representation of our editor.

Create a new file in `src` called `editor.rs` and add the following code:
```rust
pub struct Editor {
    
}

```
A `struct` is a collection of variables and, eventually, functions which are grouped together to form a meaningful entity - in our case, the Editor (It's not very meaningful yet, but we'll get to that!). The `pub` keyword tells us that this struct can be accessed from outside the `editor.rs`. We want to use it from `main`, so we use `pub`. This is already the next advantage of separating our code: We can make sure that certain functions are only called internally, while we expose others to other parts of the system.

Now, our editor needs some functionality. Let's provide it with a `run()` function, like this: 
```rust
use std::io::{self,stdout};
use termion::raw::IntoRawMode;
use termion::input::TermRead;
use termion::event::Key;

pub struct Editor {
    
}


impl Editor {
    pub fn run(&self) {    
        let _stdout = stdout().into_raw_mode().unwrap();
        for key in io::stdin().keys() {
            match key {
                Ok(key) => {
                    match key {
                        Key::Char(c) => {
                            if c.is_control() {
                                print!("{:?} \r\n", c as u8);
                            } else {
                                print!("{:?} ({})\r\n", c as u8,c);
                            }
                        },
                        Key::Ctrl('q') => break,   
                        _ => print!("{:?}\r\n", key)
                    }
                },
                Err(err) => die(err)
            }
            
        }
    }
}

fn die(e: std::io::Error) {
    panic!(e);
}
```

You already know the implementation of `run` - it's copy-pasted from our `main`, and so are the imports and `die` (If you are familiar with other languages, note how it is possible to define `die` after it is used for the first time!). Let's focus on what's new:
The `impl` block contains function definition which can be called on the struct (We see how this works in a second). The function gets the `pub` keyword, so we can call it from the outside. And the `run` function accepts a parameter called `self`, which will contain a reference to the struct it was called upon. This is equivalent to having a function outside of the `impl` block which accepts a `&Editor` as the first parameter. If this sounds a bit too theoretical, let's see it in action by changing the `main.rs` as follows:

```rust
mod editor;

fn main() {
    let editor = editor::Editor{};
    editor.run();
}
```

As you can see, we have removed nearly everything from the `main.rs`. We are creating a new instance of `Editor` and we call `run()` on it. If you run the code now, you should see that it works just fine.
Now let's make the last two remaining lines of the `main` a bit more idiomatic. Structs allow us to group variables, but for now, our struct is empty. As soon as we start adding things to the struct, we have to set all the fields as soon as we create a new one. This means that for every new entry in `Editor`, we have to go back to the `main` and change the line `let editor = editor::Editor{};` to set the new field values. Let's refactor that again.

Here is the `editor.rs`:
```rust
use std::io::{self,stdout};
use termion::raw::IntoRawMode;
use termion::input::TermRead;
use termion::event::Key;

pub struct Editor {
    
}


impl Editor {
    pub fn run(&self) {    
        let _stdout = stdout().into_raw_mode().unwrap();
        for key in io::stdin().keys() {
            match key {
                Ok(key) => {
                    match key {
                        Key::Char(c) => {
                            if c.is_control() {
                                print!("{:?} \r\n", c as u8);
                            } else {
                                print!("{:?} ({})\r\n", c as u8,c);
                            }
                        },
                        Key::Ctrl('q') => break,   
                        _ => print!("{:?}\r\n", key)
                    }
                },
                Err(err) => die(err)
            }
            
        }
    }
    pub fn new() -> Editor {
        Editor{}
    }
}

fn die(e: std::io::Error) {
    panic!(e);
}
```

The `main.rs` now reads like this:

```rust
mod editor;

fn main() {
    editor::Editor::new().run();
}
```

We have now created a new function called `new`, which constructs a new `Editor` for us. Note that the one line in `new` does not contain the keyword `return`, and it does not end with a `;`. Rust treats the result of the last line in a function as its output, and by omitting the `;`, we are telling rust that we are interested in the value of that line, and not only in executing it. Play around with that by adding the `;` and seeing what happens.

 Unlike `run`, `new` is not called on an already-existing `Editor`. This is called a `static` method, and how these are called can be seen in the `main.rs`: `editor::Editor::new()`.

Now, we can leave the `main.rs` alone while we focus on the `editor.rs` .

## Separate reading from evaluating

Let's make a function for keypress reading, and another function for mapping keypresses to editor operations. We'll also stop printing out keypresses at this point. The code blocks will now only display changed functions instead of the whole thing.

```rust
impl Editor {
    pub fn run(&self) {    
        let _stdout = stdout().into_raw_mode().unwrap();
        loop {
            if let Err(error) = self.process_keypress() {
                die(error);
            }
        }
    }
    pub fn new() -> Editor {
        Editor{}
    }

    fn process_keypress(&self) -> Result<(), std::io::Error> {
        let pressed_key = read_key()?;
        match pressed_key {
             Key::Ctrl('q') => panic!("Program end"),
             _ => ()
        }
        Ok(())
    }
    
}

fn read_key() -> Result<Key, std::io::Error> {
    loop {
        if let Some(key) = io::stdin().lock().keys().next(){
            return key;
        }
    }
}
```

We have now added a loop in `run`, which we use in combination with another feature of Rust: `If..Let`. This is a shortcut for using a `match` where we only want to handle one case and ignore all the rest. We execute self.process_keypress() and see if the result matches to `Err`. If so, we pass that error to `die`, if not, nothing happens. This is our error propagation in progress, as  `process_keypress` returns a `Result` to us, which is either empty (`()`), or an error.

`process_keypress()` waits for a keypress, and then handles it. Later, it will map various <kbd>Ctrl</kbd> key combinations and other special keys to different editor functions, and insert any alphanumeric and other printable keys' characters into the text that is being edited. That's why we are using `match` instead of `if let` here. To understand the question mark after `read_key`, let's look at what this function does by starting to read it from the inside.

Similar to `Result`, which is used to return either a value or an error, Rust has a concept of an `Option`, which either holds a value or `None`, to signify that nothing is there. This is Rust's way of avoiding `undefined` or `null` values. With `if let`, we get a match as soon as some value is returned from `stdin`. Else, we loop until we get a proper keypress. 

However, `key` itself is not directly a keypress, it is a `Result` which could also contain an Error. In `process_keypress`, we want to continue with the `Result` and propagate the error up to the top if there is one. That's what `?` does for us - if there is an error, return it, if not, continue with the unwrapped value.

Because of that, `process_keypress` does not return nothing - it returns a Result which can either be nothing or an Error. this is why we have to return a result which contains the empty result in the last line with `Ok(())`. 

Try playing around with these concepts by removing the `?`, or the `Ok(())`, or by changing the return value.

That is how the error makes its way into `run`, where it is finally handled by `die`. Speaking of `die`, there is a new ugly wart in our code: Because we don't know yet how to exit our code from within the program, we are `panic`king now when the user uses <kbd>Ctrl-Q</kbd>.  

We could instead call the proper method to end the program (`std::process::exit`, in case you are interested), but similar to how we do not want our program to crash randomly deep within our code, we also don't want it to exit somewhere deep down, but in `run`. We solve this by adding our first element to the `Editor` struct: a boolean which indicates if the user wants to quit.

```rust
pub struct Editor {
    should_quit: bool
}

//...
    pub fn new() -> Editor {
        Editor{
            should_quit: false
        }
    }
```
We have to initialize it in `new` right away, or we won't be able to compile our code. Let's set the boolean now and quit the program when it is `true`.

```rust
    pub fn run(&mut self) {    
        let _stdout = stdout().into_raw_mode().unwrap();
        loop {
            if let Err(error) = self.process_keypress() {
                die(error);
            }
            if self.should_quit {
                break;
            }
        }
    }

    fn process_keypress(&mut self) -> Result<(), std::io::Error> {
        let pressed_key = read_key()?;
        match pressed_key {
             Key::Ctrl('q') => self.should_quit = true,
             _ => ()
        }
        Ok(())
    }
    
```
Note that we had to change `&self` to `&mut self` in the function signature. The reason is that our methods now mutate (change) our `Editor`, something which must be visible in the signature. While `&foo` means "A reference to a `foo`", `&mut foo` means "A mutable reference to a `foo`". That way, you always have an idea whether or not a method call changes the parameters you hand in.

Now we have simplified `run()`, and we will try to keep it that way.

## Clear the screen

We're going to render the editor's user interface to the screen after each keypress. Let's start by just clearing the screen.

```rust
    pub fn run(&mut self) {    
        let _stdout = stdout().into_raw_mode().unwrap();
        loop {
            if let Err(error) = self.refresh_screen() {
                die(error);
            }
            if self.should_quit {
                break;
            }
            if let Err(error) = self.process_keypress() {
                die(error);
            }

        }
    }
    fn refresh_screen(&self) -> Result<(), std::io::Error>{
        print!("\x1b[2J");
        io::stdout().flush()
    }
```

We plan to access the state of the editor later, so we make `refresh_screen` a method on `Editor`. We also rearrange the code where we quit, so that we have the chance to redraw the screen one more time - for example with a goodbye message - before we end the program.

`print` is a macro which writes its parameters to `stdout` - we have used this before. However, since `stdout` is buffered, the results of `print` are not always written to the screen immediately. We need to call `flush()` manually to force Rust to write everything in its buffer to the terminal.

To clear the screen, we are writing `4` bytes out to the terminal. The first byte is `\x1b`, which is the escape character, or `27` in decimal. (Try and remember `\x1b`, we will be using it a lot.) The other three bytes are `[2J`.

We are writing an *escape sequence* to the terminal. Escape sequences always start with an escape character (`27`, which, as we saw earlier, is also produced by <kbd>Esc</kbd>) followed by a `[` character. Escape sequences instruct the terminal to do various text formatting tasks, such as coloring text, moving the cursor around, and clearing parts of the screen.

We are using the `J` command ([Erase In Display](http://vt100.net/docs/vt100-ug/chapter3.html#ED)) to clear the screen. Escape sequence commands take arguments, which come before the command. In this case the argument is `2`, which says to clear the entire screen. `<esc>[1J` would clear the screen up to where the cursor is, and `<esc>[0J` would clear the screen from the cursor up to the end of the screen.
Also, `0` is the default argument for `J`, so just `<esc>[J` by itself would also clear the screen from the cursor to the end.

In this tutorial, we will be mostly looking at [VT100](https://en.wikipedia.org/wiki/VT100) escape sequences, which are supported very widely by modern terminal emulators. See the [VT100 User Guide](http://vt100.net/docs/vt100-ug/chapter3.html) for complete documentation of each escape sequence.

`termion` eliminates the need for us to write the escape sequences directly to the terminal ourselves, so let's change our code as follows:

```rust
    fn refresh_screen(&self) -> Result<(), std::io::Error>{
        print!("{}",  termion::clear::All);
        io::stdout().flush()
    }
```

From here on out, we will be using `termion` directly in the code instead of the escape characters.

By the way, since we are now clearing the screen every time we run the program, we might be missing out on valuable tips the compiler might give us. Remember that you can run `cargo build` seperately to take a look at the warnings. Remember, though, that Rust does not recompile your code if it hasn't changed, so running `cargo build` immediately after `cargo run` won't give you the same warnings. Run `cargo clean` and then run `cargo build` to recompile the whole project and get all the warnings.

## Reposition the cursor

You may notice that the `<esc>[2J` command left the cursor at the bottom of the screen. Let's reposition it at the top-left corner so that we're ready to draw the editor interface from top to bottom.

``` rust
    fn refresh_screen(&self) -> Result<(), std::io::Error>{
        print!("{}{}", termion::clear::All, termion::cursor::Goto(1, 1));
        io::stdout().flush()
    }
```

The escape sequence behind `termion::clear::All`  uses the `H` command ([Cursor Position](http://vt100.net/docs/vt100-ug/chapter3.html#CUP)) to position the cursor. The `H` command actually takes two arguments: the row number and the column number at which to position the cursor. So if you have an 80&times;24 size terminal and you want the cursor in the center of the screen, you could use the command `<esc>[12;40H`. (Multiple arguments are separated by a `;` character.) As rows and columns are numbered starting at `1`, not `0`, the `termion` method is also 1-based.

## Clear the screen on exit

Let's clear the screen and reposition the cursor when our program crashes. If an error occurs in the middle of rendering the screen, we don't want a bunch of garbage left over on the screen, and we don't want the error to be printed wherever the cursor happens to be at that point.

```rust
fn die(e: std::io::Error) {
    print!("{}", termion::clear::All);
    panic!(e);
}
```

We have two exit points where we want to clear the screen at: When the user presses <kbd>Ctrl-Q</kbd> to quit, or when an error occurs. We have now extended `die`, but the other exit point is only handled implicitly. If we are going to extend ` refresh_screen`, we have to make sure that we don't draw more on the screen in case the user wants to quit. We can combine that with a nice farewell message:

```rust
    fn refresh_screen(&self) -> Result<(), std::io::Error>{
        print!("{}{}", termion::clear::All, termion::cursor::Goto(1, 1));
        if self.should_quit {
            print!("Goodbye!\r\n");
        }
        io::stdout().flush()
    }
```

## Tildes
It's time to start drawing. Let's draw a column of tildes (`~`) on the left hand side of the screen, like [vim](http://www.vim.org/) does. In our text editor, we'll draw a tilde at the beginning of any lines that come after the end of the file being edited.

```rust
    fn draw_rows(&self) {
        for _i in 0..24 {
            print!("~\r\n");
        }
    }
    fn refresh_screen(&self) -> Result<(), std::io::Error>{
        print!("{}{}", termion::clear::All, termion::cursor::Goto(1, 1));
        if self.should_quit {
            print!("Goodbye!\r\n");
        } else {
            self.draw_rows();
            print!("{}", termion::cursor::Goto(1, 1));
        }
        io::stdout().flush()
    }
```

`draw_rows()` will handle drawing each row of the buffer of text being edited. For now it draws a tilde in each row, which means that row is not part of the file and can't contain any text.

We don't know the size of the terminal yet, so we don't know how many rows to draw. For now we just draw `24` rows.

After we're done drawing, we reposition the cursor back up at the top-left corner.

## Window size
Our next goal is to get the size of the terminal, so we know how many rows to draw in `editor_draw_rows()`. It turns out that `termion`  provides us with a method to get the screen size. We are going to use that in a new data structure which represents the terminal. We place it in a new file called `terminal.rs`.

```rust
struct Size {
    width: u16,
    height: u16
}
pub struct Terminal {
    size: Size
}

impl Terminal {
   pub fn new() -> Result<Terminal, std::io::Error>{
        let size =  termion::terminal_size()?;
        Ok(Terminal {
            size: Size {
                width: size.0, 
                height: size.1
            }
        })
    }
}
```
We are also using a helper struct called `Size`, which is more expressive than using a Tuple as `termion` does. We also briefly need to talk about data types: termion returns the cursor coordinates as `u16`, which is an unsigned 16 bit integer and ends at around 65,000. That makes sense for the terminal, at least for virtually every terminal I have ever seen. We will be using other data types for integers later, though, as we definitely want our editor to be able to handle more than 65,000 entries per file.

Now, let's use it in `editor.rs` by adding the following to the top:

```rust
mod terminal;
```

Turns out that this doesn't compile - Rust does not find our new file. Rust wants us to group files in a specific way, so that files referenced by the `editor` module can be found at `editor/`. Let's move our file there, then. We also move our editor.rs to that folder and rename it to `mod.rs`. Run `cargo build` to confirm that it works.

This mechanism is very similar to modern JavaScript, where you can import a file called `foo/index.js` in another file with `import bar from "foo"`. Some people don't like to have multiple files called `mod.rs` open in their editor. I, for one, don't mind, since my editor shows me the path alongside the filename as soon as multiple files with the same name are opened, and I prefer to have files co-located in the same directory.

Now that our code runs again, let's try to use our new struct as follows:


```rust
pub struct Editor {
    should_quit: bool,
    terminal: Terminal
}
pub fn new() -> Result<Editor, std::io::Error> {
        Ok(Editor{
            should_quit: false,
            terminal: Terminal::new()?
        })
}
```
This code doesn't compile yet, and, surprisingly, we have added a lot of extra code. So what's going on? Turns out that retrieving the display size can also create an error. True to our earlier decisions, we propagate it up from the `new()` function of `Terminal`, which means that the signature for our `new` function of the Editor also changes. This means that we have to touch our main.rs again, to look like this:

```rust
mod editor;

fn main() {
    if let Ok(mut editor) = editor::Editor::new() {
        editor.run();
    } else {
        print!("Failed to initialize hecto");
    }
}
```
Here, we use `if let` again to check if we successfully got a new Editor instance. If not, we print an error. Rust forces us to think about these things earlier and not just let it happen. There are multiple other strategies which we could have taken:
- `panic` right then an there in `terminal.rs` 
- Defer the error handling to the `run` function. This would mean that the `terminal` property of `editor` needs to be an `Option`, to indicate if we have a terminal object or not.
- maybe more?

The point here is not that one strategy should be preferred over the other, the point is that with Rust, we need to take a conscious decision which path we take. We can't ignore it - the code just won't compile then.

Now that our code compiles, let's use our new struct.
 
```rust
    fn draw_rows(&self) {
        for _i in 0..self.terminal.size.height {
            print!("~\r\n");
        }
    }
```
This doesn't compile - again. This time, the error is that we haven't marked `size` as `pub`. But we actually don't want to do this. In Rust, there is no way to make fields public without making them mutable. But once we have returned a `Terminal`, we don't want others to change our dimensions. After all, the point of `Terminal` is to encapsulate this kind of functionality. So we build a getter in `Terminal` instead. Also, we need to make `Size` and its members public as well.

```rust
pub struct Size {
    pub width: u16,
    pub height: u16
}
// ...
    pub fn size(&self) -> &Size {
        &self.size
    }
```

We access it from `mod.rs` like this:
```rust
    fn draw_rows(&self) {
        for _i in 0..self.terminal.size().height {
            print!("~\r\n");
        }
    }
```

Now that we have a representation of the terminal in our code, let's move a bit of code there.
First the new `terminal.rs`:
```rust
use std::io::{self, stdout, Write};
use termion::raw::{IntoRawMode, RawTerminal};
use termion::input::TermRead;
use termion::event::Key;

pub struct Size {
    pub width: u16,
    pub height: u16
}
pub struct Terminal {
    size: Size,
    _stdout: RawTerminal<std::io::Stdout>
}

impl Terminal {
   pub fn new() -> Result<Terminal, std::io::Error>{
        let size =  termion::terminal_size()?;
        let stdout = stdout().into_raw_mode()?;
        Ok(Terminal {
            size: Size {
                width: size.0 as u16, 
                height: size.1 as u16,
            },
            _stdout: stdout
        })
    }
    pub fn size(&self) -> &Size {
        &self.size
    }
    pub fn clear_screen() {
        print!("{}", termion::clear::All);
    }
    pub fn cursor_position(x: u16, y: u16) {
        let mut x = x;
        let mut y = y;
        if x < std::u16::MAX {
            x = x + 1;
        }
        if y < std::u16::MAX {
            y = y + 1;
        }
        print!("{}", termion::cursor::Goto(x, y));
    }
    pub fn flush() -> Result<(), std::io::Error> {
        io::stdout().flush()
    }
    pub fn read_key() -> Result<Key, std::io::Error> {
        loop {
            if let Some(key) = io::stdin().lock().keys().next(){
                return key;
            }
        }
    }
}
```

Now the slimmed `mod.rs`:
```rust
mod terminal;
use terminal::Terminal;
use termion::event::Key;

pub struct Editor {
    should_quit: bool,
    terminal: Terminal
}

impl Editor {
    pub fn new() -> Result<Editor, std::io::Error> {
        Ok(Editor{
            should_quit: false,
            terminal: Terminal::new()?
        })
    }

    pub fn run(&mut self) {    
        loop {
            if let Err(error) = self.refresh_screen() {
                die(error);
            }
            if self.should_quit {
                break;
            }
            if let Err(error) = self.process_keypress() {
                die(error);
            }

        }
    }
    fn draw_rows(&self) {
        for _i in 0..self.terminal.size().height {
            print!("~\r\n");
        }
    }
    fn refresh_screen(&self) -> Result<(), std::io::Error>{
        Terminal::clear_screen(); 
        Terminal::cursor_position(0,0);
        if self.should_quit {
            print!("Goodbye!\r\n");
        } else {
            self.draw_rows();
            Terminal::cursor_position(0,0);
        }
        Terminal::flush()
    }
    fn process_keypress(&mut self) -> Result<(), std::io::Error> {
        let pressed_key = Terminal::read_key()?;
        match pressed_key {
             Key::Ctrl('q') => self.should_quit = true,
             _ => ()
        }
        Ok(())
    }
    
}


fn die(e: std::io::Error) {
    Terminal::clear_screen();
    panic!(e);
}
```

What did we do? We moved all the low-level terminal stuff to `Terminal`, leaving all the higher-level stuff in the `mod.rs`. Along the way, we have cleaned up a few things:

- We do not need to keep track of `stdout` for the raw mode in the editor. This is handled internally in `Terminal` now - as long as the `Terminal` struct lives, `_stdout` will be present.
- We have hidden the fact that the terminal is 1-based from the caller by making `Terminal::cursor_position` 0-based.
- We are checking if we are going to overflow our `u16` in `cursor_position`. If that happens, we will simply refuse to go any further.

By the way, the normal handling of overflows in Rust is as follows: In debug mode (which we are using by default), the program crashes. This is what you want: The compiler should not try to keep your program alive, but slap the bug in your face. In production mode, an overflow occurs, so `y+1` would return `0`. This is also what you want, because you don't want to crash your application unexpectedly in production, instead it could make sense continuing with the overflown values. In our case, it would mean that the cursor is not placed at the bottom or right of the screen, but at the laft, which would be annoying, but not enough to warrant a crash. Luckily, our new code avoids this anyways.

You can build your application for production with `cargo build --release`, which will place the production executable in `target/release`. 

## The last line

Maybe you noticed the last line of the screen doesn't seem to have a tilde. That's because of a small bug in our code. When we print the final tilde, we then print a `"\r\n"` like on any other line, but this causes the terminal to scroll in order to make room for a new, blank line. Since we want to have a status bar at the bottom later anyways, let's just change the range in which we are drawing rows for now. We will revisit this later and make this more robust against overflows/underflows, but for now let's focus on getting this working.

```rust
    fn draw_rows(&self) {
        let height = self.terminal.size().height;
        for row in 0..height - 1 {
            print!("~\r\n");
        }
    }
```

## Hide the cursor when repainting

There is another possible source of the annoying flicker effect we will take care of now. It's possible that the cursor might be displayed in the middle of the screen somewhere for a split second while the terminal is drawing to the screen. To make sure that doesn't happen, let's hide the cursor before refreshing the screen, and show it again immediately after the refresh finishes.

In `terminal.rs`:
```rust
    pub fn cursor_hide() {
         print!("{}", termion::cursor::Hide);
    }
    pub fn cursor_show() {
         print!("{}", termion::cursor::Show);
    }    
```


In `mod.rs`:
```rust
    fn refresh_screen(&self) -> Result<(), std::io::Error>{
        Terminal::cursor_hide();
        Terminal::clear_screen();        
        Terminal::cursor_position(0,0);

        if self.should_quit {
            print!("Goodbye!\r\n");
        } else {
            self.draw_rows();
            Terminal::cursor_position(0,0);
        }
        
        Terminal::cursor_show();
        Terminal::flush()
    }

```

Under the hood, we use escape sequences to tell the terminal to hide and show the cursor by writing `\x1b[?25h`,  the `h` command ([Set Mode](http://vt100.net/docs/vt100-ug/chapter3.html#SM)) and `\x1b[?25l`, the `l` command ([Reset Mode](http://vt100.net/docs/vt100-ug/chapter3.html#RM)). These commands are used to turn on and turn off various terminal features or ["modes"](http://vt100.net/docs/vt100-ug/chapter3.html#S3.3.4). The VT100 User Guide just linked to doesn't document argument `?25` which we use above. It appears the cursor hiding/showing feature appeared in [later VT models](http://vt100.net/docs/vt510-rm/DECTCEM.html).

## Clear lines one at a time

Instead of clearing the entire screen before each refresh, it seems more optimal to clear each line as we redraw them. Let's replace `termion::clear::All` (clear entire screen) escape sequence with `<esc>[K` sequence at the beginning of each line we draw with `termion::clear::CurrentLine`.

In `terminal.rs`:
```rust
    pub fn clear_current_line() {
        print!("{}", termion::clear::CurrentLine);
    }
```

In `mod.rs`:

```rust
    fn draw_rows(&self) {
        let height = self.terminal.size().height;
        for row in 0..height {
            Terminal::clear_current_line();
            print!("~\r\n");
        }
    }
    fn refresh_screen(&self) -> Result<(), std::io::Error>{
        Terminal::cursor_hide();      
        Terminal::cursor_position(0,0);

        if self.should_quit {
            Terminal::clear_screen();
            print!("Goodbye!\r\n");
        } else {
            self.draw_rows();
            Terminal::cursor_position(0,0);
        }

        Terminal::cursor_show();
        Terminal::flush()
    }
```

Note that we are now clearing the screen before displaying our goodbye message, to avoid the effect of showing the message on top of the other lines before the program finally terminates.

## Welcome message

Perhaps it's time to display a welcome message. Let's display the name of our editor and a version number a third of the way down the screen.

```rust
const VERSION: &'static str = env!("CARGO_PKG_VERSION");
//

    fn draw_rows(&self) {
        let height = self.terminal.size().height;
        for row in 0..height {
            Terminal::clear_current_line();
            if row == height / 3 {
                print!("Hecto editor -- version {}\r\n", VERSION)
            } else {
                print!("~\r\n");
            }
            
            if row < height-1 {
                print!("\r\n");
            }
        }
    }
```

Since our `Cargo.toml` already contains our version number, we use the `env!` macro to get it. We add it to our welcome message. However, we need to deal with the fact that our message might be cut off due to the terminal size. Also, we want our `Terminal` struct to handle the writing for us, since it knows about the size of the terminal.

In `terminal.rs`:
```rust
    pub fn println(&self, text: &String) {
        let mut text = text.clone();
        text.truncate(self.size.width as usize);
        print!("{}\r\n", text)
    }
    pub fn println_str(&self, text: &str) {
        self.println(&text.to_string())
    }
```
We are defining two functions here, one takes a string, makes a copy with `clone` (we don't want to modify the original string by printing it out) and then printing it. The other one takes a `&str`, which is the data type you get when you type literal strings. It converts it into a string and passes it to `println`. In the upcoming steps, working with Strings will become the norm, and working with `str`s will be the exception, so we do not bother optimizing `println_str` to not create a string, but printing its contents directly.


In `main.rs`:
```rust
 fn draw_rows(&self) {
        let height = self.terminal.size().height;
        for row in 0..height-1 {
            Terminal::clear_current_line();
            if row == height / 3 {
                let welcome_message = format!("Hecto editor -- version {}", VERSION);
                self.terminal.println(&welcome_message);
            } else {
                self.terminal.println_str("~");
            }
        }
    }
    fn refresh_screen(&self) -> Result<(), std::io::Error>{
        Terminal::cursor_hide();      
        Terminal::cursor_position(0,0);

        if self.should_quit {
            Terminal::clear_screen();
            self.terminal.println_str("Goodbye!");
        } else {
            self.draw_rows();
            Terminal::cursor_position(0,0);
        }

        Terminal::cursor_show();
        Terminal::flush()
    }
```

Now let's center the welcome message.

```rust
    fn draw_empty_row(&self) {
        self.terminal.println_str("~");
    }
    fn draw_welcome_message(&self) {
        let width = self.terminal.size().width as usize;
        let mut welcome_message = format!("Hecto editor -- version {}", VERSION);
        let len = welcome_message.len();
        
        if width > len {
            let padding  = (width - len)/2;
            let prefix = "~";
            let spaces = " ".repeat(padding - 1);
            welcome_message = format!("{}{}{}", prefix, spaces, welcome_message);
        }
        self.terminal.println(&welcome_message);
    }
    fn draw_rows(&self) {
        let height = self.terminal.size().height;
        for row in 0..height-1 {
            Terminal::clear_current_line();
            if row == height / 3 {
                self.draw_welcome_message()
            } else {
                self.draw_empty_row();
            }
        }
    }
```
To retain code clarity, we have extracted `draw_welcome_message` - and while we were doing that, we also did the same with `draw_empty_row`. 

To center a string, you divide the screen width by `2`, and then subtract half of the string's length from that. In other words: `width/2 - welcome_len/2`, which simplifies to `(width - welcome_len) / 2`. That tells you how far from the left edge of the screen you should start printing the string. So we fill that space with space characters, except for the first character, which should be a tilde. `repeat` is a nice helper function which repeats the character we hand in.

We are handling the centering in the `Editor`, not in the `Terminal`. We are not planning to center another screen for now, so adding a function in the `Terminal` seemed a bit like overkill. As usual, this comes back to personal preference, so if you want to include this method into `Terminal`, I am not going to stop you.

## Move the cursor

Let's focus on input now. We want the user to be able to move the cursor around. The first step is to keep track of the cursor's `x` and `y` position in the editor state. We're going to add another `struct` to help us with that.

```rust
struct Position {
    x: usize,
    y: usize
}

pub struct Editor {
    should_quit: bool,
    terminal: Terminal,
    cursor_position: Position
}

impl Editor {
    pub fn new() -> Result<Editor, std::io::Error> {
        Ok(Editor{
            should_quit: false,
            terminal: Terminal::new()?,
            cursor_position: Position {
                x: 0,
                y: 0
            }
        })
    }
//...
}
```
`cursor_position` is a struct where `x` will hold the horiozontal coordinate of the cursor (the column), and `y` will hold the vertical coordinate (the row), where (0,0) is at the top left of the screen. We initialize both of them to `0`, as we want the cursor to start at the top-left of the screen. 

Two other considerations are noteworthy here. We are not adding `Position` to `Terminal`, even though you might intuitively think that if we modify the cursor position in `Terminal`, it should only be natural that we are keeping track of it there, either. However, `cursor_position` will soon describe the position of the cursor _in our current document_, and not on the screen. and is therefore different from the position of the cursor on the terminal.

This is direectly related to the other consideration: Even though we saw that in order to place the cursor on the terminal, we have to use `u16` as a type, we are using the type `usize` for the cursor position instead. As discussed before, `u16` goes up until around 65,000, which is too small for our purposes. But how big is `usize`? THe answer is: It depends on the architecture we are compiling for, either 32 bit or 64 bit. It's used in Rust for adressing memory, which is essentially what we will be doing with it, so this is the right size for us. 

Now, let's add code to `refresh_screen()` to move the cursor to the position stored in `cursor_position`. While we're at it, let's rewrite `cursor_position` to accept a `Position`.

In `terminal.rs`: 
```rust
    pub fn cursor_position(position: &Position) {
        let Position{mut x, mut y} = position;
        if x < std::u16::MAX as usize{
            x = x + 1;
        }
        if y < std::u16::MAX as usize {
            y = y + 1;
        }
        let x = x as u16;
        let y = y as u16;
        print!("{}", termion::cursor::Goto(x, y));
    }
```

In `mod.rs`:
```rust
    fn refresh_screen(&self) -> Result<(), std::io::Error>{
        Terminal::cursor_hide();      
        Terminal::cursor_position(&Position{x: 0, y: 0});

        if self.should_quit {
            Terminal::clear_screen();
            self.terminal.println_str("Goodbye!");
        } else {
            self.draw_rows();
            Terminal::cursor_position(&self.cursor_position);
        }

        Terminal::cursor_show();
        Terminal::flush()
    }
```
We are using something called [destructuring](https://doc.rust-lang.org/rust-by-example/flow_control/match/destructuring.html) here: `let Position{mut x, mut y} = position;` creates new variables x and y and binds their values to the fields of the same name in `position`. 
At this point, you could try initializing `cursor_position` with a different value, to confirm that the code works as intended so far. 

Next, we'll allow the user to move the cursor using the arrow keys.

```rust
    fn move_cursor(&mut self, key: &Key){
        let Position{mut y, mut x} = self.cursor_position;
        match key {
            Key::Up => y = y-1,
            Key::Down => y = y + 1,
            Key::Left => x = x -1,
            Key::Right => x = x + 1,
            _ => ()
        }
        self.cursor_position = Position{ x,y }
    }
    fn process_keypress(&mut self) -> Result<(), std::io::Error> {
        let pressed_key = Terminal::read_key()?;
        match pressed_key {
             Key::Ctrl('q') => self.should_quit = true,
             Key::Up |
             Key::Down |
             Key::Left |
             Key::Right => {
                self.move_cursor(&pressed_key)
             }
             _ => ()
        }
        Ok(())
    }
```
Now you should be able to move the cursor around with those keys.

## Prevent moving the cursor off screen

Currently, you can cause the `cursor_position` values to go past the right and bottom edges of the screen, and crash it by trying to go to the left or to the top. Let's prevent that by doing some bounds checking in `editor_move_cursor()`.

```rust
    fn move_cursor(&mut self, key: &Key){
        let Position{mut y, mut x} = self.cursor_position;
        let size = self.terminal.size();
        let height = size.height as usize;
        let width = size.width as usize;
        match key {
            Key::Up => {
                if y > 0 {
                    y = y-1;
                }
            }
            Key::Down => {
                if y < std::usize::MAX && y < height {
                    y = y + 1;
                }
            },
            Key::Left => {
                if x > 0 {
                    x = x -1;
                }
            }
            Key::Right => {
                if x < std::usize::MAX && x < width {
                    x = x + 1;
                }
            }
            _ => ()
        }
        self.cursor_position = Position{ x,y }
    }
```

You should be able to confirm that you can now move around the visible area, with the cursor staying within the terminal bounds. You can also place it on the last line, which still does not have a tilde, a fact that is not forgotten and will be fixed later during this tutorial.

## Navigating with <kbd>Page Up</kbd>, <kbd>Page Down</kbd> <kbd>Home</kbd> and <kbd>End</kbd>
To complete our low-level terminal code, we need to detect a few more special keypresses. We are going to map <kbd>Page Up</kbd>, <kbd>Page Down</kbd> <kbd>Home</kbd> and <kbd>End</kbd> to position our cursor at the top or bottom of the screen, or the beginning or end of the line, respectively.

```rust
    fn move_cursor(&mut self, key: &Key){
        let Position{mut y, mut x} = self.cursor_position;
        let size = self.terminal.size();
        let height = size.height as usize;
        let width = size.width as usize;
        match key {
            Key::Up => {
                if y > 0 {
                    y = y-1;
                }
            }
            Key::Down => {
                if y < std::usize::MAX && y < height {
                    y = y + 1;
                }
            }
            Key::Left => {
                if x > 0 {
                    x = x -1;
                }
            }
            Key::Right => {
                if x < std::usize::MAX && x < width {
                    x = x + 1;
                }
            }
            Key::PageUp => y = 0,
            Key::PageDown => y = height,
            Key::Home =>  x = 0,
            Key::End => x = width,
            _ => ()
        }
        self.cursor_position = Position{ x,y }
    }
    fn process_keypress(&mut self) -> Result<(), std::io::Error> {
        let pressed_key = Terminal::read_key()?;
        match pressed_key {
             Key::Ctrl('q') => self.should_quit = true,
             Key::Up |
             Key::Down |
             Key::Left |
             Key::Right |
             Key::PageUp |
             Key::PageDown |
             Key::End |
             Key::Home=> {
                self.move_cursor(&pressed_key)
             }
             _ => ()
        }
        Ok(())
    }
```

## Conclusion
I hope this chapter has given you a first feeling of pride when you saw how your text editor was taking shape. We were talking a lot about idiomatic code in the beginning, and where busy refactoring our code into separate files for quite some time, but the payoff is visible: The code is cleanly structured and therefore easy to maintain. Since we now know our way around Rust, we won't have to worry that much about refactoring in the upcoming chapters and can focus on adding functionality.
In the next chapter, we will get our program to display text files.