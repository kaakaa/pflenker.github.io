---
layout: post
title: "Hecto, Chapter 4: A text viewer"
categories: [Rust, hecto]
---
Let's see if we can turn `hecto` into a text viewer in this chapter.

## A line viewer
Let's extend our state to hold rows of text.

```rust
/*** includes ***/
/*** structs & constants ***/
struct File {
    rows: Vec<String>,
}

struct EditorState {
    quit: bool,
    terminal_size: (u16, u16),
    cursor_position: (u16, u16),
    file: Option<File>
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
        cursor_position: (0,0),
        file: None
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

We will use `File` to represent the file we are displaying right now. For now, it only has a vector of `Strings`, but we will extend it later.

Let's fill it with some text now. We won't worry about reading from a file just yet. Instead, we'll hardcode a "Hello, World" string into it.

```rust
/*** includes ***/
/*** structs & constants ***/
/*** terminal***/
/*** file i/o **/
fn editor_open() -> File{
    File {
        rows: vec![String::from("Hello, World")]
    }
}
/*** output ***/
/*** input ***/
/*** init ***/
fn main() {
    let mut _stdout = stdout().into_raw_mode().unwrap();
    let mut editor_state = EditorState {
        quit: false,
        terminal_size: termion::terminal_size().unwrap(),
        cursor_position: (0,0),
        file: Some(editor_open())
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

`editor_open()` will eventually be for opening and reading a file from disk, so we put it in a new `/*** file i/o ***/` section. Let's display the hard coded file contents, then.

```rust
/*** includes ***/
/*** structs & constants ***/
/*** terminal***/
/*** file i/o **/
/*** output ***/
fn editor_draw_rows(terminal_size: (u16, u16), file: &Option<File>){
    let (width, height) = terminal_size;
    let mut last_file_row = 0;
    let mut rows= &Vec::new();

    if let Some(file) = file {
        last_file_row = file.rows.len() as u16;
        rows = &file.rows;
    }
    for row in 0..height {
        print!("{}", termion::clear::CurrentLine);

        if row >= last_file_row {
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
        } else {
            let current_row = &rows[row as usize];
            let line_len = std::cmp::min(width as usize, current_row.len() );
            let slice = &current_row[..line_len];
            print!("{}", slice);
        }

        if row < height-1 {
            print!("\r\n");
        }
    }
}
fn editor_refresh_screen(state: &EditorState)  {
    print!("{}{}", termion::cursor::Hide,  termion::cursor::Goto(1,1));
    if state.quit {
        print!("{}Goodbye!\r\n", termion::clear::All);

    } else {
        let (x,y) = state.cursor_position;
        editor_draw_rows(state.terminal_size, &state.file);
        print!("{}", termion::cursor::Goto(x+1, y+1));
    }

    print!("{}", termion::cursor::Show);
}
/*** input ***/
/*** init ***/
```

We wrap our previous row-drawing code in an `if` statement that checks whether we are currently drawing a row that is part of the text buffer, or a row that comes after the end of the text buffer.

To draw a row that's part of the text buffer, we simply write out the contents of the corresponding row in `File`. But first, we take care to truncate the rendered line if it would go past the end of the screen.

Next, let's allow the user to open and display actual file.

```rust
/*** includes ***/
use std::env;
use std::fs;
/*** structs & constants ***/
/*** terminal***/
/*** file i/o **/
fn editor_open(filename: &String) -> Result<File, std::io::Error>{
    let contents = fs::read_to_string(filename)?;
    let mut rows = Vec::new();
    for row in contents.lines() {
        rows.push(String::from(row));
    }
    Ok(File {
        rows
    })
}
/*** output ***/
/*** input ***/
/*** init ***/
fn main() {
    let mut _stdout = stdout().into_raw_mode().unwrap();
    let mut file = None;
    let args: Vec<String> = env::args().collect();
    if args.len() > 1 {
        let file_name = &args[1];
        if let Ok(new_file) = editor_open(file_name) {
            file = Some(new_file);
        }
    }
    let mut editor_state = EditorState {
        quit: false,
        terminal_size: termion::terminal_size().unwrap(),
        cursor_position: (0,0),
        file: file
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
We are now calling `editor_open()` only if we have more than one `arg`. Per convention, `args[0]` is always the name of our program, so `args[1]` contains the file name we're after.  You can pass a file name to your program by running `cargo run (filename)` while you are developing.

Now you should see your screen fill up with lines of text when you run `cargo run src/main.rs`, for example.

Since it is much more likely that `editor_open` will fail than, for example, `into_raw_mode`, we have to handle the error more gracefully. For now, if we can't open the file, we are letting the user edit a new one instead.

`editor_open` reads the lines into our `File` struct. It's not obvious from our code, but each row in `rows` will not contain the line endings `\n` or `\r\n`, as Rust's `line()` method will cut it away for us. That makes sense: We are already handling new lines ourselves, so we wouldn't want to handle the ones in the file anyways.

Now let’s fix a quick bug. We want the welcome message to only display when the user starts the program with no arguments, and not when they open a file, as the welcome message could get in the way of displaying the file. And while we are doing it, let's extract the logic of displaying a welcome message from `editor_draw_rows()`, to make the code more readable.

```rust
/*** includes ***/
/*** structs & constants ***/
/*** terminal***/
/*** file i/o **/
/*** output ***/
fn draw_welcome_message(width: usize) {
    let welcome_message = format!("Hecto editor -- version {}", VERSION);
    let welcome_len = std::cmp::min(width, welcome_message.len());
    let padding = (width - welcome_len)/2;
    if padding > 0 {
        print!("~");
        for _col in 0..padding - 1 {
            print!(" ");
        }
    }
    let slice = &welcome_message[..welcome_len];
    print!("{}", slice);
}
fn editor_draw_rows(terminal_size: (u16, u16), file: &Option<File>){
    let (width, height) = terminal_size;
    let mut last_file_row = 0;
    let mut rows = &Vec::new();

    if let Some(file) = file {
        last_file_row = file.rows.len() as u16;
        rows = &file.rows;
    }
    for row in 0..height {
        print!("{}", termion::clear::CurrentLine);

        if row >= last_file_row {
            if last_file_row == 0 && row == height/3 {
                draw_welcome_message(width as usize);
            } else {
                print!("~");
            }
        } else {
            let current_row = &rows[row as usize];
            let line_len = std::cmp::min(width as usize, current_row.len() );
            let slice = &current_row[..line_len];
            print!("{}", slice);
        }

        if row < height-1 {
            print!("\r\n");
        }
    }
}
/*** input ***/
/*** init ***/
```