---
layout: post
title: "Hecto, Chapter 3: Raw input and output"
categories: [Rust, hecto]
---
In this chapter, we will tackle reading from and writing to the terminal.

## Press <kbd>Ctrl-Q</kbd> to quit

Last chapter we saw that the <kbd>Ctrl</kbd> key combined with the alphabetic keys seemed to map to bytes 1&ndash;26. We can use this to detect <kbd>Ctrl</kbd> key combinations and map them to different operations in our editor. We'll use that to map <kbd>Ctrl-Q</kbd> to the quit operation.

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
    let mut _stdout = stdout().into_raw_mode().unwrap();
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

The ASCII character set seems to be designed this way on purpose.  (It is also similarly designed so that you can set and clear a bit to switch between lowercase and uppercase. If you are interested, find out which byte it is and what the impact is on combinations such as  <kbd>Ctrl</kbd>-<kbd>a</kbd> in contrast to <kbd>Ctrl</kbd>-<kbd>Shift</kbd>-<kbd>a</kbd>  .

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
    let mut _stdout = enable_raw_mode().unwrap();

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
        if let Some(key) = io::stdin().lock().keys().next(){
            return key;
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
    let mut _stdout = stdout().into_raw_mode().unwrap();
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

`editor_read_key()`'s job is to wait for one keypress, and return it.
Later, we'll expand this function to handle escape sequences, which involves reading multiple bytes that represent a single keypress, as is the case with the arrow keys. Since we are now explicitly using a `loop` in `main()`, we do not need to use `for..in` here.

`editor_process_keypress()` waits for a keypress, and then handles it. Later, it will map various <kbd>Ctrl</kbd> key combinations and other special keys to different editor functions, and insert any alphanumeric and other printable keys' characters into the text that is being edited. That's why we are using `match` instead of `if let` here.

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
    let mut _stdout = stdout().into_raw_mode().unwrap();
    loop {
        if let Error(err) = editor_refresh_screen() {
            die(error);
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

Similar with how we have investigated the byte-wise output for every keypress first until we had a firm grip on the concepts, and then replaced it with library functions, we will now use `termion` to write the escape characters for us, which will make our code more readable.

```rust
/*** includes ***/
/*** terminal***/
/*** output ***/
fn editor_refresh_screen() -> Result<(), std::io::Error> {
    print!("{}", termion::clear::All);
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
    print!("{}{}", termion::clear::All, termion::cursor::Goto(1, 1));
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
    print!("{}{}", termion::clear::All, termion::cursor::Goto(1, 1));
    panic!(e);
}
/*** output ***/
fn editor_refresh_screen() -> Result<(), std::io::Error> {
    print!("{}{}", termion::clear::All, termion::cursor::Goto(1, 1));
    io::stdout().flush()
}

/*** input ***/
/*** init ***/
fn main() {
    let mut _stdout = stdout().into_raw_mode().unwrap();
    loop {
        if let Error(err) = editor_refresh_screen() {
            die(error);
        }
        match editor_process_keypress() {
           Ok(should_process_next) => {
            if !should_process_next {
                print!("{}{}", termion::clear::All, termion::cursor::Goto(1, 1));
                break;
            }
           },
           Err(error) => die(error),
       }
   }
}
```

We have two exit points where we want to clear the screen at: When the user presses <kbd>Ctrl-Q</kbd> to quit, or when an error occurs. Luckily, we thought early about error handling, so all we have to do is extending `die()`.

## Handling state and writes
Before we continue, we have to think about a few general concepts of our program. You might have already noticed that if we continue flushing in `editor_refresh_screen()`, we need to `flush` everywhere else where we write to the screen as well. This has three drawbacks:
- We might forget flushing
- There are performance drawbacks (drawing to the screen is costly!)
- It goes against the rule that [you should not repeat yourself](https://en.wikipedia.org/wiki/Don%27t_repeat_yourself).

This mistake snuck in because we where only thinking about the next problem at hand - clearing the screen and positioning the cursor - that we got lost of the big picture. We will fix that in a second, but before we do, let's first consider what we want our program to do on a high level:
- Read input from user
- Evaluate the input
- Draw the screen based on the input from the user

To accomplish this, we will need a way to handle the input from the user and access it later. We also need to keep track of previous interactions from the user. In other words, , we need to handle the state of the editor: The user input modifies the state, then we react on that input, then we write to the terminal and wait for the next input. We already see the first bit of state which we have passed around: `should_process_next`. Let's refactor this to a proper state.
```rust
/*** includes ***/
/*** structs & constants ***/
struct EditorState {
    quit: bool
}
/*** terminal***/
/*** output ***/
fn editor_refresh_screen(state: &EditorState)  {
    print!("{}{}", termion::clear::All, termion::cursor::Goto(1, 1));
    if state.quit {
        print!("Goodbye!\r\n");
    }
}
/*** input ***/
fn editor_process_keypress(state: &mut EditorState) -> Result<(), std::io::Error>{
    let pressed_key = editor_read_key();
    match pressed_key? {
        Key::Ctrl('q') => state.quit = true,
        _ => ()
    }
    Ok(())
}
/*** init ***/
fn main() {
    let mut _stdout = stdout().into_raw_mode().unwrap();
    let mut editor_state = EditorState {
        quit: false
    };

    loop {
        editor_refresh_screen(&editor_state);
        if let Err(error) = io::stdout().flush() {
           die(error);
        }
        if editor_state.quit {
                break;
        }
       if let Err(error) = editor_process_keypress(&mut editor_state) {
            die(error);
        }

   }
}
```

We have created a new struct which will hold the editor state. For now, it only holds `quit`. `editor_process_keypress()` modifies the state. We have also moved the `break` condition, so that we break *after* refreshing the screen, but *before* getting the next keyboard input.

We are also passing the state to `editor_refresh_screen()` and display a nice message when the user quits. We also moved the flushing to `main`, and handle the potential error there.

## Tildes
It's time to start drawing. Let's draw a column of tildes (`~`) on the left hand side of the screen, like [vim](http://www.vim.org/) does. In our text editor, we'll draw a tilde at the beginning of any lines that come after the end of the file being edited.

```rust
/*** includes ***/
/*** structs & constants ***/
/*** helpers ***/
/*** terminal***/
/*** output ***/
fn editor_draw_rows(){
    for _y in 0..24 {
        print!("~\r\n");
    }
}
fn editor_refresh_screen(state: &EditorState)  {
    print!("{}{}", termion::clear::All, termion::cursor::Goto(1, 1));
    if state.quit {
        print!("Goodbye!\r\n");
        return;
    }
    editor_draw_rows();
    print!("{}", termion::cursor::Goto(1, 1));
}
/*** input ***/
/*** init ***/
```

`editor_draw_rows()` will handle drawing each row of the buffer of text being edited. For now it draws a tilde in each row, which means that row is not part of the file and can't contain any text.

We don't know the size of the terminal yet, so we don't know how many rows to draw. For now we just draw `24` rows.

After we're done drawing, we reposition the cursor back up at the top-left corner.

## Window size
Our next goal is to get the size of the terminal, so we know how many rows to draw in `editor_draw_rows()`. It turns out that `termion`  provides us with a method to get the screen size. We need the screen size in many places later on, so it makes sense to add it to the state in the beginning.


```rust
/*** includes ***/
/*** structs & constants ***/
/*** helpers ***/
/*** output ***/
fn editor_draw_rows(height: u16){
    for _y in 0..height {
        print!("~\r\n");
    }
}
fn editor_refresh_screen(state: &EditorState)  {
    print!("{}{}", termion::clear::All, termion::cursor::Goto(1, 1));
    if state.quit {
        print!("Goodbye!\r\n");
        return;
    }
    editor_draw_rows(state.terminal_size.1);
    print!("{}", termion::cursor::Goto(1, 1));
}
/*** input ***/
/*** init ***/
fn main() {
    let mut _stdout = stdout().into_raw_mode().unwrap();
    let mut editor_state = EditorState {
        quit: false,
        terminal_size: termion::terminal_size().unwrap()
    };

    loop {
        editor_refresh_screen(&editor_state);
        if let Err(error) = io::stdout().flush() {
           die(error);
        }
        if editor_state.quit {
                break;
        }
        if let Err(error) = editor_process_keypress(&mut editor_state) {
            die(error);
        }

   }
}
```

Note that same as before, we do not care about the error when trying to get the terminal size.

Maybe you noticed the last line of the screen doesn't seem to have a tilde. That's because of a small bug in our code. When we print the final tilde, we then print a `"\r\n"` like on any other line, but this causes the terminal to scroll in order to make room for a new, blank line. Let's make the last line an exception when we print our `"\r\n"`'s.

```rust
/*** includes ***/
/*** structs & constants ***/
/*** helpers ***/
/*** output ***/
fn editor_draw_rows(height: u16){
    for row in 0..height {
        print!("~");
        if row < height-1 {
            print!("\r\n");
        }
    }
}
/*** input ***/
/*** init ***
```

## Hide the cursor when repainting

There is another possible source of the annoying flicker effect we will take care of now. It's possible that the cursor might be displayed in the middle of the screen somewhere for a split second while the terminal is drawing to the screen. To make sure that doesn't happen, let's hide the cursor before refreshing the screen, and show it again immediately after the refresh finishes.

```rust
/*** includes ***/
/*** structs & constants ***/
/*** helpers ***/
/*** output ***/
fn editor_refresh_screen(state: &EditorState)  {
    print!("{}{}{}", termion::cursor::Hide, termion::clear::All, termion::cursor::Goto(1, 1));
    if state.quit {
        print!("Goodbye!\r\n");
    } else {
        editor_draw_rows(state.terminal_size.1);
        print!("{}", termion::cursor::Goto(1, 1));
    }

    print!("{}", termion::cursor::Show);
}
/*** input ***/
/*** init ***

```

Under the hood, we use escape sequences to tell the terminal to hide and show the cursor by writing `\x1b[?25h`,  the `h` command ([Set Mode](http://vt100.net/docs/vt100-ug/chapter3.html#SM)) and `\x1b[?25l`, the `l` command ([Reset Mode](http://vt100.net/docs/vt100-ug/chapter3.html#RM)). These commands are used to turn on and turn off various terminal features or ["modes"](http://vt100.net/docs/vt100-ug/chapter3.html#S3.3.4). The VT100 User Guide just linked to doesn't document argument `?25` which we use above. It appears the cursor hiding/showing feature appeared in [later VT models](http://vt100.net/docs/vt510-rm/DECTCEM.html).

## Clear lines one at a time

Instead of clearing the entire screen before each refresh, it seems more optimal to clear each line as we redraw them. Let's remove `termion::clear::All` (clear entire screen) escape sequence, and instead put a `<esc>[K` sequence at the beginning of each line we draw with `termion::clear::CurrentLine`.

```rust
/*** includes ***/
/*** structs & constants ***/
/*** helpers ***/
/*** output ***/
fn editor_draw_rows(height: u16){
    for row in 0..height {
        print!("{}~", termion::clear::CurrentLine);
        if row < height-1 {
            print!("\r\n");
        }
    }
}
fn editor_refresh_screen(state: &EditorState)  {
    print!("{}{}", termion::cursor::Hide, termion::cursor::Goto(1, 1));
    if state.quit {
        print!("{}Goodbye!\r\n", termion::clear::All);
    } else {
        editor_draw_rows(state.terminal_size.1);
        print!("{}", termion::cursor::Goto(1, 1));
    }

    print!("{}", termion::cursor::Show);
}
/*** input ***/
/*** init ***
```
Note that we are now clearing the screen before displaying our goodbye message, to avoid the effect of showing the message on top of the other lines before the program finally terminates.

## Welcome message

Perhaps it's time to display a welcome message. Let's display the name of our editor and a version number a third of the way down the screen.

```rust
/*** includes ***/
/*** structs & constants ***/
const VERSION: &'static str = env!("CARGO_PKG_VERSION");

/*** terminal***/
/*** output ***/
fn editor_draw_rows(width: u16, height: u16){
    for row in 0..height {

        print!("{}", termion::clear::CurrentLine);
        if row == height/3 {
            let welcome_message = format!("Hecto editor -- version {}", VERSION);
            let welcome_len = std::cmp::min(width as usize, welcome_message.len());
            let slice = &welcome_message[..welcome_len];
            print!("{}", slice);
        } else {
            print!("~");
        }
        if row < height-1 {
            print!("\r\n");
        }
    }
}
fn editor_refresh_screen(state: &EditorState)  {
    print!("{}{}", termion::cursor::Hide, termion::cursor::Goto(1, 1));
    if state.quit {
        print!("{}Goodbye!\r\n", termion::clear::All);
    } else {
        editor_draw_rows(state.terminal_size.0, state.terminal_size.1);
        print!("{}", termion::cursor::Goto(1, 1));
    }

    print!("{}", termion::cursor::Show);
}
/*** input ***/
/*** init ***/
```

Since our `Cargo.toml` already contains our version number, we use the `env!` macro to get it. We add it to our welcome message. We also truncate the length of the string in case the terminal is too tiny to fit our welcome messae - which is why we are now passing the full dimensions to `editor_draw_rows`.

Now let's center it.
```rust
/*** includes ***/
/*** structs & constants ***/
/*** terminal***/
/*** output ***/
fn editor_draw_rows(width: u16, height: u16){
    for row in 0..height {

        print!("{}", termion::clear::CurrentLine);
        if row == height/3 {
            let width = width as usize;
            let welcome_message = format!("Hecto editor -- version {}", VERSION);
            let welcome_len = std::cmp::min(width, welcome_message.len());
            let padding = (width - welcome_len)/2;
            if padding > 0 {
                print!("~");
                for _i in 0..padding - 1 {
                    print!(" ");
                }
            }
            let slice = &welcome_message[..welcome_len];
            print!("{}", slice);
        } else {
            print!("~");
        }
        if row < height-1 {
            print!("\r\n");
        }
    }
}

/*** input ***/
/*** init ***/

```
To center a string, you divide the screen width by `2`, and then subtract half of the string's length from that. In other words: `width/2 - welcome_len/2`, which simplifies to `(width - welcome_len) / 2`. That tells you how far from the left edge of the screen you should start printing the string. So we fill that space with space characters, except for the first character, which should be a tilde.

## Move the cursor

Let's focus on input now. We want the user to be able to move the cursor around. The first step is to keep track of the cursor's `x` and `y` position in the editor state.

```rust
/*** includes ***/
/*** structs & constants ***/
struct EditorState {
    quit: bool,
    terminal_size: (u16, u16),
    cursor_position: (u16, u16)
}
/*** terminal***/
/*** output ***/
/*** input ***/
/*** init ***/
fn main() {
    let mut _stdout = stdout().into_raw_mode().unwrap();
    let mut editor_state = EditorState {
        quit: false,
        terminal_size: termion::terminal_size().unwrap(),
        cursor_position: (0,0)
    };

    loop {
        editor_refresh_screen(&editor_state);
        if let Err(error) = io::stdout().flush() {
           die(error);
        }
        if editor_state.quit {
                break;
        }
        if let Err(error) = editor_process_keypress(&mut editor_state) {
            die(error);
        }

   }
}
```

`cursor_position` is a tuple where the first value will hold the horiozontal coordinate of the cursor (the column), and the second value will hold the vertical coordinate (the row). We initialize both of them to `0`, as we want the cursor to start at the top-left of the screen. (Even though the terminal coordinates are `1`-based, we stick to the standards of virtually every programming language out there by starting with `0`.)

Now, let's add code to `editor_refresh_screen()` to move the cursor to the position stored in `cursor_position`.

```rust
/*** includes ***/
/*** structs & constants ***/
/*** terminal***/
/*** output ***/
fn editor_refresh_screen(state: &EditorState)  {
    print!("{}{}", termion::cursor::Hide,  termion::cursor::Goto(1,1));
    if state.quit {
        print!("{}Goodbye!\r\n", termion::clear::All);

    } else {
        let (x,y) = state.cursor_position;
        editor_draw_rows(state.terminal_size.0, state.terminal_size.1);
        print!("{}", termion::cursor::Goto(x+1, y+1));
    }

    print!("{}", termion::cursor::Show);
}
/*** input ***/
/*** init ***/
```
We add `1` to `x` and `y` from `cursor_position()` to convert from `0`-indexed values to the `1`-indexed values that the terminal uses.

At this point, you could try initializing `cursor_position` with a different value, to confirm that the code works as intended so far.

Next, we'll allow the user to move the cursor using the arrow keys.

```rust
/*** includes ***/
/*** structs & constants ***/
/*** terminal***/
/*** output ***/
/*** input ***/
fn editor_move_cursor(old_position: (u16,u16), key: &Key) -> (u16, u16) {
    let (x,y) = old_position;
    match key {
        Key::Up => return (x, y-1),
        Key::Down => return (x, y+1),
        Key::Left => return (x-1, y),
        Key::Right => return (x+1, y),
        _ => ()
    }
}

fn editor_process_keypress(state: &mut EditorState) -> Result<(), std::io::Error>{
    let pressed_key = editor_read_key()?;
    match pressed_key {
        Key::Ctrl('q') => state.quit = true,
        Key::Up |
        Key::Down |
        Key::Left |
        Key::Right => {
            state.cursor_position = editor_move_cursor(state.cursor_position, &pressed_key)
        }
        _ => ()
    }
    Ok(())
}
/*** init ***/
```
Now you should be able to move the cursor around with those keys.

We should take a moment to appreciate what `termion` does for us. You might remember from earlier steps that Arrow Keys are emitting an Escape Charater, followed by `[` and then a character. We would have to do the handling of these multiple bytes manually, listening to the next bytes until we either know what we are dealing with or treating the input as an unknown key. With `termion`, we can simply use the `Key` type.

## Prevent moving the cursor off screen

Currently, you can cause the `cursor_position` values to go into the negatives, or go past the right and bottom edges of the screen. Let's prevent that by doing some bounds checking in `editor_move_cursor()`.

```rust
/*** includes ***/
/*** structs & constants ***/
/*** terminal***/
/*** output ***/
/*** input ***/
fn editor_move_cursor(old_position: (u16,u16), terminal_size: (u16, u16), key: &Key) -> (u16, u16){
    let (mut row, mut col) = old_position;
    let (max_row, max_col) = terminal_size;
    match key {
        Key::Up => {
            if col > 0 {
               col = col-1;
            }
        },
        Key::Down => {
            if col < max_col {
                col = col+1;
            }
        }
        Key::Left => {
            if row != 0 {
               row = row-1;
            }
        },
        Key::Right =>{
            if row < max_row {
                row = row+1;
            }
        },
        _ => ()
    }
    (row,col)
}

fn editor_process_keypress(state: &mut EditorState) -> Result<(), std::io::Error>{
    let pressed_key = editor_read_key()?;
    match pressed_key {
        Key::Ctrl('q') => state.quit = true,
        Key::Up |
        Key::Down |
        Key::Left |
        Key::Right => {
            state.cursor_position = editor_move_cursor(state.cursor_position, state.terminal_size, &pressed_key)
        }
        _ => ()
    }
    Ok(())
}
/*** init ***/
```

## Navigating with <kbd>Page Up</kbd>, <kbd>Page Down</kbd> <kbd>Home</kbd> and <kbd>End</kbd>
To complete our low-level terminal code, we need to detect a few more special keypresses that use escape sequences, like the arrow keys did. We are going to map <kbd>Page Up</kbd>, <kbd>Page Down</kbd> <kbd>Home</kbd> and <kbd>End</kbd> to position our cursor at the top or bottom of the screen, or the beginning or end of the line, respectively.

```rust
/*** includes ***/
/*** structs & constants ***/
/*** terminal***/
/*** output ***/
/*** input ***/
fn editor_move_cursor(old_position: (u16,u16), terminal_size: (u16, u16), key: &Key) -> (u16, u16){
    let (mut row, mut col) = old_position;
    let (max_row, max_col) = terminal_size;
    match key {
        Key::Up => {
            if col > 0 {
               col = col-1;
            }
        },
        Key::Down => {
            if col < max_col {
                col = col+1;
            }
        }
        Key::Left => {
            if row != 0 {
               row = row-1;
            }
        },
        Key::Right =>{
            if row < max_row {
                row = row+1;
            }
        },
        Key::PageUp => col = 0,
        Key::PageDown => col = max_col,
        Key::Home =>  row = 0,
        Key::End => row = max_row,
        _ => ()
    }
    (row,col)
}

fn editor_process_keypress(state: &mut EditorState) -> Result<(), std::io::Error>{
    let pressed_key = editor_read_key()?;
    match pressed_key {
        Key::Ctrl('q') => state.quit = true,
        Key::Up |
        Key::Down |
        Key::Left |
        Key::Right |
        Key::PageUp |
        Key::PageDown |
        Key::End |
        Key::Home
        => {
            state.cursor_position = editor_move_cursor(state.cursor_position, state.terminal_size, &pressed_key)
        }
        _ => ()
    }
    Ok(())
}
/*** init ***/
```

In the next chapter, we will get our program to display text files.