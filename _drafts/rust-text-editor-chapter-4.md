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
        match io::stdout().flush() {
            Err(error) => die(error),
            _ => ()
        }
        if editor_state.quit {
                break;
        }
        match editor_process_keypress(&mut editor_state) {
            Err(error) => die(error),
            _ => ()
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
        match io::stdout().flush() {
            Err(error) => die(error),
            _ => ()
        }
        if editor_state.quit {
                break;
        }
        match editor_process_keypress(&mut editor_state) {
            Err(error) => die(error),
            _ => ()
        }

   }
}
```