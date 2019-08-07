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