---
layout: post
title: "Hecto, Chapter 3: Raw input and output"
categories: [Rust, hecto]
---
## Press <kbd>Ctrl-Q</kbd> to quit

Last chapter we saw that the <kbd>Ctrl</kbd> key combined with the alphabetic keys seemed to map to bytes 1&ndash;26. We can use this to detect <kbd>Ctrl</kbd> key combinations and map them to different operations in our editor. We'll start by mapping <kbd>Ctrl-Q</kbd> to the quit operation.

```rust
/*** includes ***/
/*** helpers ***/
fn to_ctrl_byte(c: char) -> u8{
    let byte = c as u8;
    byte & 0b00011111
}

/*** terminal***/
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
                if byte ==  to_ctrl_byte('q') {
                    break;
                }
            },
            Err(err) => die(err)
        }
    }
}
```

The `to_ctrl_byte` function bitwise-ANDs a character with the value `00011111`, in binary. If you are not familiar with bitwise operations, here are some useful explanations:
- You can use `println!("{:#b}", x);` to print out the binary representation of an `u8`.  Try this to see the actual bytes which are read into our program.
- When you compare the output for <kbd>Ctrl-Key</kbd> with the output of the key without <kbd>Ctrl</kbd>, you will notice that Ctrl sets the upper 3 bits to `0`
- If we now remember how bitwise and works, we can see that `to_ctrl_byte` does just the same.

The ASCII character set seems to be designed this way on purpose.  (It is also similarly designed so that you can set and clear a bit to switch between lowercase and uppercase. If you are interested, find out which byte it is and what the impact is on combinations such as  <kbd>Ctrl-A</kbd> in contrast to  <kbd>Ctrl-a</kbd>).

## Refactor keyboard input
### Read keys instead of bytes
In the previous steps, we have worked directly on bytes, which was both fun and valuable. However, at a certain point you should ask yourself if the funtionality you are implementing could not be replaced by a library function, as in many cases, someone else has already solved your problem, and probably better.
For me, handling with bit operations is a huge red flag that tells me that I am probably too deep down the rabbit hole.
Fortunately for us, our dependency, `termion`, makes things already a lot easier, as it can already group individual bytes to keypresses and pass them to us. Let's implement this.

```rust
/*** imports ***/
use termion::raw::{IntoRawMode, RawTerminal};
use termion::input::TermRead;
use termion::event::Key;

/*** terminal***/
/*** init ***/
fn main() {
    let mut _stdout = None;
    match  enable_raw_mode() {
        Ok(s) => _stdout = Some(s),
        Err(err) => die(err)
    }

    for key in io::stdin().lock().keys() {
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
```

With that change, we where able to get rid of manually checking if <kbd>Ctrl</kbd> has been pressed, as all keys are now properly handled for us.

### Separate reading from evaluating

Let's make a function for low-level keypress reading, and another function for mapping keypresses to editor operations. We'll also stop printing out keypresses at this point.

```rust
/*** includes ***/
/*** helpers ***/

/*** terminal***/


fn editor_read_key() -> Result<Key, std::io::Error> {
    loop {
        let key = io::stdin().lock().keys().next();
        match key {
            Some(key) => return key,
            _ => ()
        }
    }
}

/*** input ***/
fn editor_process_keypress() -> Result<bool, std::io::Error>{
    let pressed_key = editor_read_key();
    let mut should_process_next = true;
    match pressed_key? {
        Key::Ctrl('q') => should_process_next = false,
        _ => ()
    }
    Ok(should_process_next)
}

/*** init ***/
fn main() {
    let mut _stdout = None;
    match  enable_raw_mode() {
        Ok(s) => _stdout = Some(s),
        Err(err) => die(err)
    }

    loop {
        match editor_process_keypress() {
           Ok(should_process_next) => {
            if !should_process_next {
                break;
            }
           },
           Err(error) => die(error),
       }
   }
}
```

`editor_read_key()`'s job is to wait for one keypress, and return it. Note that we can be passed `None` by the system, on which we wait for the next "real" keypress, discarding the current one. This expresses our desire to read an actual key from the user, and not `None`.
Later, we'll expand this function to handle escape sequences, which involves reading multiple bytes that represent a single keypress, as is the case with the arrow keys. Since we are now explicitly using a `loop` in `main()`, we do not need to use `for..in` here.

`editor_process_keypress()` waits for a keypress, and then handles it. Later, it will map various <kbd>Ctrl</kbd> key combinations and other special keys to different editor functions, and insert any alphanumeric and other printable keys' characters into the text that is being edited.

`editor_process_keypress()` returns a boolean which indicates to the `main` whether or not it should listen for the next key press. We could have exited the program from within `editor_process_keypress` by calling `std::process::exit`, but that would prevent Rust from doing cleanup and leave the terminal in an undefined state.

Note that `editor_read_key()` belongs in the `/*** terminal ***/` section because it deals with low-level terminal input, whereas `editor_process_keypress()` belongs in the new `/*** input ***/` section because it deals with mapping keys to editor functions at a much higher level.

Now we have simplified `main()`, and we will try to keep it that way.

## Clear the screen

We're going to render the editor's user interface to the screen after each keypress. Let's start by just clearing the screen.

```rust
/*** includes ***/
use std::io::{self,  stdout, Write, Stdout};
//...

/*** helpers ***/
/*** terminal***/
/*** output ***/
fn editor_refresh_screen() -> Result<(), std::io::Error> {
    print!("\x1b[2J");
    io::stdout().flush()
}
/*** input ***/
/*** init ***/
fn main() {
    let mut _stdout = None;
    match  enable_raw_mode() {
        Ok(s) => _stdout = Some(s),
        Err(err) => die(err)
    }

    loop {
        match editor_refresh_screen() {
            Err(error) => die(error),
            _ => ()
        }
        match editor_process_keypress() {
           Ok(should_process_next) => {
            if !should_process_next {
                break;
            }
           },
           Err(error) => die(error),
       }
   }
}
```
`print` is a macro which writes its parameters to `stdout` - we have used this before. However, since `stdout` is buffered, the results of `print` are not always written to the screen. So we call `flush()` manually to force Rust to write everything in its buffer to the terminal.

To clear the screen, we are writing `4` bytes out to the terminal. The first byte is `\x1b`, which is the escape character, or `27` in decimal. (Try and remember `\x1b`, we will be using it a lot.) The other three bytes are `[2J`.

We are writing an *escape sequence* to the terminal. Escape sequences always start with an escape character (`27`, which, as we saw earlier, is also produced by <kbd>Esc</kbd>) followed by a `[` character. Escape sequences instruct the terminal to do various text formatting tasks, such as coloring text, moving the cursor around, and clearing parts of the screen.

We are using the `J` command ([Erase In Display](http://vt100.net/docs/vt100-ug/chapter3.html#ED)) to clear the screen. Escape sequence commands take arguments, which come before the command. In this case the argument is `2`, which says to clear the entire screen. `<esc>[1J` would clear the screen up to where the cursor is, and `<esc>[0J` would clear the screen from the cursor up to the end of the screen.
Also, `0` is the default argument for `J`, so just `<esc>[J` by itself would also clear the screen from the cursor to the end.

In this tutorial, we will be mostly looking at [VT100](https://en.wikipedia.org/wiki/VT100) escape sequences, which are supported very widely by modern terminal emulators. See the [VT100 User Guide](http://vt100.net/docs/vt100-ug/chapter3.html) for complete documentation of each escape sequence.

Similar with how we have investigated the byte-wise output for every keypress first until we had a firm grip on the concepts, and then replaced it with library functions, we will also use `termion` to write the escape characters for us, which will make our code more readable.

```rust
/*** includes ***/
/*** terminal***/
/*** output ***/
fn editor_refresh_screen() -> Result<(), std::io::Error> {
    println!("{}", termion::clear::All);
    io::stdout().flush()
}
/*** input ***/
/*** init ***/
```

From here on out, we will be using `termion` directly in the code instead of the escape characters.

## Reposition the cursor

You may notice that the `<esc>[2J` command left the cursor at the bottom of the screen. Let's reposition it at the top-left corner so that we're ready to draw the editor interface from top to bottom.

``` rust
/*** includes ***/
/*** helpers ***/
/*** terminal***/
/*** output ***/
fn editor_refresh_screen() -> Result<(), std::io::Error> {
    println!("{}{}", termion::clear::All, termion::cursor::Goto(1, 1));
    io::stdout().flush()
}
/*** input ***/

/*** init ***/

```

The escape sequence behind `termion::clear::All`  uses the `H` command ([Cursor Position](http://vt100.net/docs/vt100-ug/chapter3.html#CUP)) to position the cursor. The `H` command actually takes two arguments: the row number and the column number at which to position the cursor. So if you have an 80&times;24 size terminal and you want the cursor in the center of the screen, you could use the command `<esc>[12;40H`. (Multiple arguments are separated by a `;` character.) As rows and columns are numbered starting at `1`, not `0`, the `termion` method is also 1-based.

## Clear the screen on exit

Let's clear the screen and reposition the cursor when our program exits. If an error occurs in the middle of rendering the screen, we don't want a bunch of garbage left over on the screen, and we don't want the error to be printed wherever the cursor happens to be at that point.

```rust
/*** includes ***/
/*** helpers ***/
/*** terminal***/
fn die(e: std::io::Error) {
    println!("{}{}", termion::clear::All, termion::cursor::Goto(1, 1));
    panic!(e);
}
/*** output ***/

fn clear_screen() {
   print!("\x1b[2J");
   print!("\x1b[H");
}

fn editor_refresh_screen() -> Result<(), std::io::Error> {
    clear_screen();
    io::stdout().flush()
}

/*** input ***/
/*** init ***/
fn main() {
    let mut _stdout = None;
    match  enable_raw_mode() {
        Ok(s) => _stdout = Some(s),
        Err(err) => die(err)
    }

    loop {
        match editor_refresh_screen() {
            Err(error) => die(error),
            _ => ()
        }
        match editor_process_keypress() {
           Ok(should_process_next) => {
            if !should_process_next {
                println!("{}{}", termion::clear::All, termion::cursor::Goto(1, 1));
                break;
            }
           },
           Err(error) => die(error),
       }
   }
}
```

We have two exit points where we want to clear the screen at: When the user presses <kbd>Ctrl-Q</kbd> to quit, or when an error occurs. Luckily, we thought early about error handling, so all we have to do is adding `clear_screen` to `die()`.

## Tildes
It's time to start drawing. Let's draw a column of tildes (`~`) on the left hand side of the screen, like [vim](http://www.vim.org/) does. In our text editor, we'll draw a tilde at the beginning of any lines that come after the end of the file being edited.

```rust
/*** includes ***/
/*** helpers ***/
/*** terminal***/
/*** output ***/
fn editor_draw_rows(){
    for _y in 0..24 {
        print!("~\r\n");
    }
}
fn editor_refresh_screen() -> Result<(), std::io::Error> {
    clear_screen();
    editor_draw_rows();
    print!("\x1b[H");
    io::stdout().flush()
}
/*** input ***/
/*** init ***/
```

`editor_draw_rows()` will handle drawing each row of the buffer of text being edited. For now it draws a tilde in each row, which means that row is not part of the file and can't contain any text.

We don't know the size of the terminal yet, so we don't know how many rows to draw. For now we just draw `24` rows.

After we're done drawing, we do another `<esc>[H` escape sequence to reposition the cursor back up at the top-left corner.

## Window size
Our next goal is to get the size of the terminal, so we know how many rows to draw in `editor_draw_rows()`. Getting the terminal size is in most cases done by adding `libc` as a dependency, and then calling `ioctl()` on it. But it turns out that we don't need to add another dependency to our editor, as `termion` already provides us with a method to get the screen size. Let's use it.


```rust
/*** includes ***/
/*** helpers ***/
/*** output ***/
fn editor_draw_rows() ->  Result<(), std::io::Error> {
    let (_, height) = termion::terminal_size()?;
    for _y in 0..height {
        print!("~\r\n");
    }
    Ok(())
}
fn editor_refresh_screen() -> Result<(), std::io::Error> {
    clear_screen();
    editor_draw_rows()?;
    print!("\x1b[H");
    io::stdout().flush()
}
/*** input ***/
/*** init ***/
```
