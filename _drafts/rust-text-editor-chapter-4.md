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


## Vertical scrolling

Next we want to enable the user to scroll through the whole file, instead of just being able to see the top few lines of the file. Let's add a `row_offset`  variable to the global editor state, which will keep track of what row of the file the user is currently scrolled to.

```rust
/*** includes ***/
/*** structs & constants ***/
struct EditorState {
    quit: bool,
    terminal_size: (u16, u16),
    cursor_position: (u16, u16),
    file: Option<File>,
    row_offset: u32
}
/*** terminal***/
/*** file i/o **/
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
        row_offset:0,
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

We initialize it to `0`, which means we'll be scrolled to the top of the file by default.

Now let's have `editor_draw_rows()` display the correct range of lines of the file according to the value of `row_offset`.

```rust
/*** includes ***/
/*** structs & constants ***/
/*** terminal***/
/*** file i/o **/
fn editor_draw_rows(state: &EditorState ){
    
    let (width, height) = state.terminal_size;
    let mut last_file_row = 0;
    let mut rows = &Vec::new();

    if let Some(file) = &state.file {
        last_file_row = file.rows.len() as u16;
        rows = &file.rows ;
    }
    for terminal_row in 0..height {
        print!("{}", termion::clear::CurrentLine);
        let file_row = terminal_row  + state.row_offset;
        if file_row >= last_file_row {
            if last_file_row == 0 && terminal_row == height/3 {
                draw_welcome_message(width as usize);
            } else {
                print!("~");
            }
        } else {
            let current_row = &rows[file_row as usize];
            let line_len = std::cmp::min(width as usize, current_row.len() );
            let slice = &current_row[..line_len];
            print!("{}", slice);
        }

        if terminal_row < height-1 {
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
        editor_draw_rows(&state);
        print!("{}", termion::cursor::Goto(x+1, y+1));
    }

    print!("{}", termion::cursor::Show);
}
/*** input ***/
/*** init ***/
```
We have actually done some refactoring here. First, `editor_draw_rows` now accepts the whole `EditorState` instead of individual parameters. There will be hardly anything in the state which we *don't* want `editor_draw_rows` to know about, so instead of adding individual parameters, passing the whole state makes a lot of sense.

Second, we now have two concepts of rows, and to not confuse them, we have renamed `row` to `terminal_row`, which describes the row which is drawn to the terminal, and have introduced `file_row`, which holds where we are in the current file. We add `row_offset` to `terminal_row` to arrive at `file_row` . Note also that `row_offset` is also of type `u16` - which means that our editor will not be able to deal with files larger than about 65,000 lines. This is acceptable for now since larger files would cause other problems anyway (we load the files completely into memory, for starters)

Where do we set the value of `row_offset`? Our strategy will be to check if the cursor has moved outside of the visible window, and if so, adjust `row_offset` so that the cursor is just inside the visible window. We’ll put this logic in a function called `editor_scroll`, and call it right after we handled the key press.


```rust
/*** includes ***/
/*** structs & constants ***/
struct Position {
    row: u16,
    col: u16
}

struct Size {
    width: u16,
    height: u16
}
struct EditorState {
    quit: bool,
    terminal_size: Size,
    cursor_position: Position,
    file: Option<File>,
    row_offset: u16
}
/*** terminal***/
/*** file i/o **/
/*** output ***/
fn editor_draw_rows(state: &EditorState ){
    
    let Size{width, height} = state.terminal_size;
    let mut last_file_row = 0;
    let mut rows = &Vec::new();

    if let Some(file) = &state.file {
        last_file_row = file.rows.len() as u16;
        rows = &file.rows ;
    }
    for terminal_row in 0..height {
        print!("{}", termion::clear::CurrentLine);
        let file_row = terminal_row + state.row_offset;
        if file_row >= last_file_row {
            if last_file_row == 0 && terminal_row == height/3 {
                draw_welcome_message(width as usize);
            } else {
                print!("~");
            }
        } else {
            let current_row = &rows[file_row as usize];
            let line_len = std::cmp::min(width as usize, current_row.len() );
            let slice = &current_row[..line_len];
            print!("{}", slice);
        }

        if terminal_row < height-1 {
            print!("\r\n");
        }
    }
}

fn editor_refresh_screen(state: &EditorState)  {
    print!("{}{}", termion::cursor::Hide,  termion::cursor::Goto(1,1));
    if state.quit {
        print!("{}Goodbye!\r\n", termion::clear::All);

    } else {
        let Position{row, col} = state.cursor_position;
        editor_draw_rows(&state);
        print!("{}", termion::cursor::Goto(col+1, row+1));
    }

    print!("{}", termion::cursor::Show);
}
/*** input ***/

fn editor_scroll(state: &mut EditorState) {
    let Position{row,col:_} = state.cursor_position;
    let terminal_height = state.terminal_size.height;
    let mut new_offset = state.row_offset;
    if row < state.row_offset {
        new_offset = row;
    }
    if row >= state.row_offset + terminal_height {
        new_offset = row - terminal_height + 1;
    }
    state.row_offset = new_offset;
    
}


fn editor_move_cursor(old_position: &Position, terminal_size: &Size, key: &Key) -> Position{
    let Position{mut row, mut col} = old_position;
    let Size{height, width} = terminal_size;
    let height = *height;
    let width = *width;
    match key {
        Key::Up => {
            if row > 0 {
               row = row-1;
            }
        },
        Key::Down => {
            if row < height {
                row = row+1;
            }
        }
        Key::Left => {
            if col != 0 {
               col = col-1;
            }
        },
        Key::Right =>{
            if col < width {
                col = col+1;
            }
        },
        Key::PageUp => row = 0,
        Key::PageDown => row = height,
        Key::Home =>  col = 0,
        Key::End => col = width,
        _ => ()
    }
    Position{row,col}
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
            editor_move_cursor(&mut state, &pressed_key);
            editor_scroll(&mut state);    
        }
        _ => ()
    }
    Ok(())
}

/*** init ***/
fn main() {
    let mut _stdout = stdout().into_raw_mode().unwrap();
    let mut file = None;
    let args: Vec<String> = env::args().collect();
    let (width, height) =  termion::terminal_size().unwrap();
    if args.len() > 1 {
        let file_name = &args[1];
        if let Ok(new_file) = editor_open(file_name) {
            file = Some(new_file);
        }
    }
    let mut editor_state = EditorState {
        quit: false,
        terminal_size: Size {
            width,
            height
        },
        cursor_position: Position{
            row: 0,
            col: 0
        },
        row_offset:0,
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
Now that we are using the size of the terminal and the position of the cursor in multiple places, another refactoring is in order, so that our code is more expressive. We are now representing the position of the cursor as a `Position`, and the size of the screen as the `Size`.

With that, it's easier to implement the `editor_scroll` method.  
The first `if` statement checks if the cursor is above the visible window, and if so, scrolls up to where the cursor is. The second `if` statement checks if the cursor is past the bottom of the visible window, and contains slightly more complicated arithmetic because `E.rowoff` refers to what's at the *top* of the screen, and we have to get `E.screenrows` involved to talk about what's at the *bottom* of the screen.

Now let's allow the cursor to advance past the bottom of the screen (but not past the bottom of the file).

```rust
/*** includes ***/
/*** structs & constants ***/
/*** terminal***/
/*** file i/o ***/
/*** output ***/
/*** input ***/

fn editor_move_cursor(state: &mut EditorState, key: &Key){
    let Size{mut height,  width} = state.terminal_size;
    if let Some(file) = &state.file {
               height = file.rows.len() as u16 - 1;
    }
    let Position{mut row, mut col} = state.cursor_position;    
    match key {
        Key::Up => {
            if row > 0 {
               row = row-1;
            }
        },
        Key::Down => {
            if row < height {
                row = row+1;
            }
        }
        Key::Left => {
            if col != 0 {
               col = col-1;
            }
        },
        Key::Right =>{
            if col < width {
                col = col+1;
            }
        },
        Key::PageUp => row = 0,
        Key::PageDown => row = height,
        Key::Home =>  col = 0,
        Key::End => col = width,
        _ => ()
    }
    state.cursor_position = Position{row,col}
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
            let Size{width, mut height} = state.terminal_size;
            if let Some(file) = &state.file {
                height = file.rows.len() as u16 - 1;
            }
            let size = Size{width, height};
                            
            state.cursor_position = editor_move_cursor(&state.cursor_position, &size, &pressed_key);
            state.row_offset = editor_scroll(&state);
        }
        _ => ()
    }
    Ok(())
}
/*** init ***/
```

You should be able to scroll through the entire file now, when you run `cargo run src/main.rs`. If you try to scroll back up, you may notice the cursor isn't being positioned properly. That is because `Position` in the state no longer refers to the position of the cursor on the screen. It refers to the position of the cursor within the text file. To position the cursor on the screen, we now have to subtract `editor_state.row_offset` from the row position.

```rust
/*** includes ***/
/*** structs & constants ***/
/*** terminal***/
/*** file i/o ***/
/*** output ***/
/*** input ***/
fn editor_refresh_screen(state: &EditorState)  {
    print!("{}{}", termion::cursor::Hide,  termion::cursor::Goto(1,1));
    if state.quit {
        print!("{}Goodbye!\r\n", termion::clear::All);

    } else {
        let Position{row, col} = state.cursor_position;
        editor_draw_rows(&state);
        print!("{}", termion::cursor::Goto(col+1, row-state.row_offset+1));
    }

    print!("{}", termion::cursor::Show);
}
/*** input ***/
/*** init ***/
```

## Horizontal scrolling

Now let's work on horizontal scrolling. We'll implement it in just about the same way we implemented vertical scrolling. We will start by adding a column offset to the global editor state. For that, we define a struct which holds both offsets. But wait, we already have a struct which defines a position! We will reuse that one to hold our offsets.

```rust
/*** includes ***/
/*** structs & constants ***/
struct EditorState {
    quit: bool,
    terminal_size: Size,
    cursor_position: Position,
    offset: Position,
    file: Option<File>,
}
/*** terminal***/
/*** file i/o ***/
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
fn editor_draw_rows(state: &EditorState ){
    
    let Size{width, height} = state.terminal_size;
    let mut last_file_row = 0;
    let mut rows = &Vec::new();

    if let Some(file) = &state.file {
        last_file_row = file.rows.len() as u16;
        rows = &file.rows ;
    }
   for terminal_row in 0..height {
        print!("{}", termion::clear::CurrentLine);
        let file_row = terminal_row + state.offset.row;
        if file_row >= last_file_row {
            if last_file_row == 0 && terminal_row == height/3 {
                draw_welcome_message(width as usize);
            } else {
                print!("~");
            }
        } else {
            let current_row = &rows[file_row as usize];
            let line_len = std::cmp::min(width as usize, current_row.len() );
            let slice = &current_row[..line_len];
            print!("{}", slice);
        }

        if terminal_row < height-1 {
            print!("\r\n");
        }
    }
}

fn editor_refresh_screen(state: &EditorState)  {
    print!("{}{}", termion::cursor::Hide,  termion::cursor::Goto(1,1));
    if state.quit {
        print!("{}Goodbye!\r\n", termion::clear::All);

    } else {
        let Position{row, col} = state.cursor_position;
        editor_draw_rows(&state);
        print!("{}", termion::cursor::Goto(col+1, row-state.offset.row+1));
    }

    print!("{}", termion::cursor::Show);
}
/*** input ***/
fn editor_scroll(state: &mut EditorState) {
    let Position{row,col:_} = state.cursor_position;
    let terminal_height = state.terminal_size.height;
    let mut new_offset = state.offset.row;
    if row < state.offset.row {
        new_offset = row;
    }
    if row >= state.offset.row + terminal_height {
        new_offset = row - terminal_height + 1;
    }
    state.offset.row = new_offset;
    
}
/*** init ***/
fn main() {
    let mut _stdout = stdout().into_raw_mode().unwrap();
    let mut file = None;
    let args: Vec<String> = env::args().collect();
    let (width, height) =  termion::terminal_size().unwrap();
    if args.len() > 1 {
        let file_name = &args[1];
        if let Ok(new_file) = editor_open(file_name) {
            file = Some(new_file);
        }
    }
    let mut editor_state = EditorState {
        quit: false,
        terminal_size: Size {
            width,
            height
        },
        cursor_position: Position{
            row: 0,
            col: 0
        },
        offset: Position {
            row: 0,
            col: 0
        },
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
To display each row at the column offset, we'll use `editor_state.offset.col` as an index to the array of each `row` we display, and subtract the number of characters that are to the left of the offset from the length of the row.
