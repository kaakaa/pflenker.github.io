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
To display each row at the column offset, we'll use `editor_state.offset.col` as the starting index to the array of each `row` we display, and `state.offset.col ` added to `width`as the end index. Also, we make sure we are not going past the boundaries of our current row.

```rust
/*** includes ***/
/*** structs & constants ***/
/*** terminal***/
/*** file i/o ***/
/*** output ***/
fn editor_draw_rows(state: &EditorState ){
    
    let Size{width, height} = state.terminal_size;
    let mut last_file_row = 0;
    let mut rows = &Vec::new();

    if let Some(file) = &state.file {
        last_file_row = file.len();
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
            let current_row_len = current_row.len() as u16;
            let mut line_end = state.offset.col + width;
            let mut line_start = state.offset.col;
            if line_end > current_row_len {
                line_end = current_row_len;
            }
            if line_start > current_row_len {
                line_start = current_row_len;
            }
            let line_start = line_start as usize;
            let line_end = line_end as usize;
            let slice = &current_row[line_start..line_end];
            print!("{}", slice);
        }

        if terminal_row < height-1 {
            print!("\r\n");
        }
    }
}
/*** input ***/
/*** init ***/

```
Note that we have to handle the case where we scrolled horizontally past the end of the line. In that case, we set `line_len` to `0` so that nothing is displayed on that
line.

Now let's update `editor_scroll()` to handle horizontal scrolling.

```rust
/*** includes ***/
/*** structs & constants ***/
/*** terminal***/
/*** file i/o ***/
/*** output ***/
/*** input ***/

fn editor_scroll(state: &mut EditorState) {
    let Position{row,col} = state.cursor_position;
    let terminal_height = state.terminal_size.height;
    let terminal_width = state.terminal_size.width;
    let mut offset = &mut state.offset;
    if row < offset.row {
        offset.row = row;
    } else  if row >= offset.row + terminal_height {
        offset.row = row - terminal_height + 1;
    }
     if col < offset.col {
        offset.col = col;
     } else if col >= offset.col + terminal_width {
        offset.col = col - terminal_width + 1;
    }

}
/*** init ***/
```
As you can see, it is exactly parallel to the vertical scrolling code. We just replace `row` with `col`, `offset.row` with `offset.cols`, and `height` with `width`.

Now let's allow the user to scroll past the right edge of the screen.

```rust
/*** includes ***/
/*** structs & constants ***/
/*** terminal***/
/*** file i/o ***/
/*** output ***/
/*** input ***/
fn editor_move_cursor(state: &mut EditorState, key: &Key){
    let Position{mut row, mut col} = state.cursor_position;    
    let Size{mut height,  mut width} = state.terminal_size;
    if let Some(file) = &state.file {
               height = file.len();
               if  row <height {
                   width = file.rows[row as usize].len() as u16;
               } else {
                   width = 0;
               }
    }
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
            if col > 0 {
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
/*** init ***/
```

You should be able to confirm that horizontal scrolling now works. 
Next, let's fix the cursor positioning, just like we did with vertical scrolling.

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
        print!("{}", termion::cursor::Goto(col-state.offset.col+1, row-state.offset.row+1));
    }

    print!("{}", termion::cursor::Show);
}
/*** init ***/
```

## Snap cursor to end of line

Now `cursor_position` refers to the cursor's position within the file, not its position on the screen. So our goal with the next few steps is to limit the values of `cursor_position` ` to only ever point to valid positions in the file.  
Otherwise, the user could move the cursor way off to the right of a line and start inserting text there, which wouldn't make much sense. (The only exceptions to this rule are that `cursor_position.col` can point one character past the end of a line so that characters can be inserted at the end of the line, and `cursor_position.row` can point one line past the end of the file so that new lines at the end of the file can be added easily.)

We are already able to prevent the user from scrolling too far to the right or too far down. The user is still able to move the cursor past the end of a line, however. They can do it by moving the cursor to the end of a long line, then moving it down to the next line, which is shorter. The `cursor_position.col` value won't change, and the cursor will be off to the right of the end of the line it's now on.

Let's add some code to `editor_move_cursor()` that corrects `cursor_position` if it ends up past the end of the line it's on.

```rust
/*** includes ***/
/*** structs & constants ***/
/*** terminal***/
/*** file i/o ***/
/*** output ***/
/*** input ***/

fn editor_move_cursor(state: &mut EditorState, key: &Key){
    let Position{mut row, mut col} = state.cursor_position;    
    let Size{mut height,  mut width} = state.terminal_size;
    if let Some(file) = &state.file {
               height = file.len();
               if  row < height {
                   width = file.rows[row as usize].len() as u16;
               } else {
                   width = 0;
               }
    }
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
            if col > 0 {
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
    if  row < height {
        if let Some(file) = &state.file {
            width = file.rows[row as usize].len() as u16;       
        }
    } else {
            width = 0;
    }
    
    if col > width {
        col = width;
    }
    state.cursor_position = Position{row,col}
}
/*** init ***/
```

We have to set `width` again, since `row` could point to a different line than it did before. We then set `col` to the end of that line if `col`, which will be the new cursor position, is to the right of the end of that line. Also note that we consider non-existing lines to be of length `0`, which works for our purposes here.

## Moving left at the start of a line

Let's allow the user to press <kbd>&larr;</kbd> at the beginning of the line to move to the end of the previous line.
```rust
fn editor_move_cursor(state: &mut EditorState, key: &Key){
    let Position{mut row, mut col} = state.cursor_position;    
    let Size{mut height,  mut width} = state.terminal_size;
    if let Some(file) = &state.file {
               height = file.len();
               if  row < height {
                   width = file.row(row).len();
               } else {
                   width = 0;
               }
    }
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
            if col > 0 {
               col = col-1;
            } else if row > 0  {
                row = row - 1;
                if let Some(file) = &state.file {
                    col = file.row(row).len();
                }
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
    if  row < height {
        if let Some(file) = &state.file {
            width = file.row(row).len();
        }
    } else {
            width = 0;
    }

    if col > width {
        col = width;
    }
    state.cursor_position = Position{row,col}
}
```

We make sure they aren't on the very first line before we move them up a line.

## Moving right at the end of a line

Similarly, let's allow the user to press <kbd>&rarr;</kbd> at the end of a line
to go to the beginning of the next line.

```rust
fn editor_move_cursor(state: &mut EditorState, key: &Key){
    let Position{mut row, mut col} = state.cursor_position;    
    let Size{mut height,  mut width} = state.terminal_size;
    if let Some(file) = &state.file {
               height = file.len();
               if  row < height {
                   width = file.row(row).len();
               } else {
                   width = 0;
               }
    }
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
            if col > 0 {
               col = col-1;
            } else if row > 0  {
                row = row - 1;
                if let Some(file) = &state.file {
                    col = file.row(row).len();
                }
            }
        },
        Key::Right =>{
            if col < width {
                col = col+1;
            } else if row < height {
                row = row +1;
                col = 0;
            }
        },
        Key::PageUp => row = 0,
        Key::PageDown => row = height,
        Key::Home =>  col = 0,
        Key::End => col = width,
        _ => ()
    }
    if  row < height {
        if let Some(file) = &state.file {
            width = file.row(row).len();
        }
    } else {
            width = 0;
    }

    if col > width {
        col = width;
    }
    state.cursor_position = Position{row,col}
}
```
Here we have to make sure they’re not at the end of the file before moving down a line.

## Rendering tabs
If you try opening a file with tabs, you'll notice that the tab character takes up a width of 8 colums or so. The length of a tab is up to the terminal being used and its settings. We want to *know* the length of each tab, and we also want control over how to render tabs, so we're going to add a second string to the `FileRow` struct called `render`, which will contain the actual characters to draw on the screen for that row of text. We'll only use `render` for tabs for now, but in the future it could be used to render nonprintable control characters as a `^` character followed by another character, such as `^A` for the <kbd>Ctrl-A</kbd> character (this is a common way to display control characters in the terminal).

So, let's start by adding `render` and `render_len` (which returns the size of the ccontents of `render`) to the `FileRow` struct, and initializing them correctly. We won’t worry about how to render tabs just yet, instead, we let `string` and `render` hold the same values.

```rust
struct FileRow {
    string: String,
    render: String
}

impl FileRow {
    fn len(&self) -> u16{
        self.string.len() as u16
    }
    fn render_len(&self) -> u16{
        self.render.len() as u16
    }    
    fn slice(&self, start: u16, end: u16) -> &str{
        let start = start as usize;
        let end = end as usize;
        &self.string[start.. end ]
    }
    fn from(line: &str) -> FileRow{
        let string = String::from(line);
        let render = String::from(line);
        FileRow{
            string,
            render
        }
    }
}
```

Now let's change the implementation of `slice` to use `render`, and rename it to `slice_render` to make it clear for the caller (We need no `slice` method on the string itself yet, so there is no need to implement it). We also need to update `editor_draw_rows` to use `len_render`.

```rust
impl FileRow {
    fn len(&self) -> u16{
        self.string.len() as u16
    }
    fn len_render(&self) -> u16{
        self.render.len() as u16
    }    
    fn slice_render(&self, start: u16, end: u16) -> &str{
        let start = start as usize;
        let end = end as usize;
        &self.render[start.. end]
    }
    fn from(line: &str) -> FileRow{
        let string = String::from(line);
        let render = String::from(line);
        FileRow{
            string,
            render
        }
    }
}

/*** output ***/

fn editor_draw_rows(state: &EditorState ){
    
    let Size{width, height} = state.terminal_size;
    let mut last_file_row = 0;
    if let Some(file) = &state.file {
        last_file_row = file.len();
    }
    
   for terminal_row in 0..height {
        print!("{}", termion::clear::CurrentLine);
        let file_row = terminal_row + state.offset.y;
        if file_row >= last_file_row {
            if last_file_row == 0 && terminal_row == height/3 {
                draw_welcome_message(width as usize);
            } else {
                print!("~");
            }
        } else {
             if let Some(file) = &state.file {
                let current_row = file.row(file_row);

                let current_row_len = current_row.len_render();
                let mut line_end = state.offset.x + width;
                let mut line_start = state.offset.x;
                if line_end > current_row_len {
                    line_end = current_row_len;
                }
                if line_start > current_row_len {
                    line_start = current_row_len;
                }
                let slice = &current_row.slice_render(line_start, line_end);
                print!("{}", slice);
            }
         
        }

        if terminal_row < height-1 {
            print!("\r\n");
        }
    }
}
```
Now the text viewer is displaying the characters in `render`. Let's add code that renders tabs as multiple space characters.

```rust
const TAB_STOP: usize = 4;

impl FileRow {
    fn len(&self) -> u16{
        self.string.len() as u16
    }
    fn len_render(&self) -> u16{
        self.render.len() as u16
    }    
    fn slice_render(&self, start: u16, end: u16) -> &str{
        let start = start as usize;
        let end = end as usize;
        &self.render[start.. end]
    }
    fn from(line: &str) -> FileRow{
        let string = String::from(line);
        let tab = " ".repeat(TAB_STOP);
        let render = string.replace("\t", &tab);
        FileRow{
            string,
            render
        }
    }
}

```

We make our code a bit cleaner by moving the number 4 to the top, and we use `repeat` to repeat the space as often as we want to.

## Tabs and the cursor

The cursor doesn't currently interact with tabs very well. When we position the cursor on the screen, we're still assuming each character takes up only one column on the screen. To fix this, let's introduce a new  coordinate variable, `render_position`. While `cursor_position.x` is an index into the `string` field of an `FileRow`, the `render_position.x` variable will be an index into the `render` field. If there are no tabs on the current line, then `render_position.x` will be the same as `cursor_position.x`. If there are tabs, then `render_position.x` will be greater than `cursor_position.x` by however many extra spaces those tabs take up when rendered.

We won't be using `cursor_position.y` for now, but in the future we might to render a file row over multiple actual rows (For example for soft wraps), and then we would also need a reference where the cursor is in the rendered document.

Start by adding `render_position` to the global state struct, and initializing it.

```rust

/*** structs & constants ***/
struct EditorState {
    quit: bool,
    terminal_size: Size,
    cursor_position: Position,
    render_position: Position,
    offset: Position,
    file: Option<File>,
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
            x: 0,
            y: 0
        },
        offset: Position {
            x: 0,
            y: 0
        },
        render_position: Position {
            x: 0,
            y: 0
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
We'll set the value of `render_position` at the top of `editor_scroll()`. For now we'll just set it to be the same as `cursor_position`. Then we'll replace  `cursor_position` with `render_position` in `editor_scroll()`, because scrolling should take into account the characters that are actually rendered to the screen, and the rendered position of the cursor.

```rust

fn editor_scroll(state: &mut EditorState) {
    state.render_position = Position {
        ..state.cursor_position
    };
    let Position{x,y} = state.render_position;
    let terminal_height = state.terminal_size.height;
    let terminal_width = state.terminal_size.width;
    let mut offset = &mut state.offset;
    if y < offset.y {
        offset.y = y;
    } else  if y >= offset.y + terminal_height {
        offset.y = y - terminal_height + 1;
    }
     if x < offset.x {
        offset.x = x;
     } else if x >= offset.x + terminal_width {
        offset.x = x - terminal_width + 1;
    }

}

```
Now change `cursor_position` to `render_position` in `editor_refresh_screen()` where we set the cursor
position for the terminal.

```rust
fn editor_refresh_screen(state: &EditorState)  {
    print!("{}{}", termion::cursor::Hide,  termion::cursor::Goto(1,1));
    if state.quit {
        print!("{}Goodbye!\r\n", termion::clear::All);

    } else {
        let Position{x, y} = state.render_position;
        editor_draw_rows(&state);
        print!("{}", termion::cursor::Goto(x-state.offset.x+1, y-state.offset.y+1));
    }

    print!("{}", termion::cursor::Show);
}
```
All that's left to do is calculate the value of `render_position` properly in `editor_scroll()`. For that, we need to find out how many tabs are to the left of the `cursor_position` and multiply it with `TAB_STOP` - 1.

```rust
/*** input ***/
fn editor_set_render_pos(state: &mut EditorState) {
    let y = state.cursor_position.y;
    let mut x = state.cursor_position.x;
    if let Some(file) = &state.file {
        if y < file.len(){
            let row = file.row(y);
            let left_string = row.slice(0,x);
            let left_tabs = left_string.matches("\t").count() *  (TAB_STOP - 1);
            x = x + left_tabs as u16;
        }
    } 
    state.render_position = Position{x,y};
}

fn editor_scroll(state: &mut EditorState) {
    
    editor_set_render_pos(state);
    
    let Position{x,y} = state.render_position;
    let terminal_height = state.terminal_size.height;
    let terminal_width = state.terminal_size.width;
    let mut offset = &mut state.offset;
    if y < offset.y {
        offset.y = y;
    } else  if y >= offset.y + terminal_height {
        offset.y = y - terminal_height + 1;
    }
     if x < offset.x {
        offset.x = x;
     } else if x >= offset.x + terminal_width {
        offset.x = x - terminal_width + 1;
    }

}
```
We use the built-in method `matches` to count the occurences of the tab in our string. We multiply it with `TAB_STOP -1`, because the tab character already counts as one.

You should now be able to confirm that the cursor moves properly within lines that contain tabs.

## Scrolling with <kbd>Page Up</kbd> and <kbd>Page Down</kbd>
Now that we have scrolling, let's make the <kbd>Page Up</kbd> and <kbd>Page Down</kbd> keys scroll up or down an entire page.

```rust
fn editor_move_cursor(state: &mut EditorState, key: &Key){
    let Position{mut y, mut x} = state.cursor_position;    
    let terminal_height = state.terminal_size.height;
    let mut height = 0;
    let mut width = 0;
    if let Some(file) = &state.file {
               height = file.len();
               if  y < height {
                   width = file.row(y).len();
               } else {
                   width = 0;
               }
    } 
    match key {
        Key::Up => {
            if y > 0 {
               y = y-1;
            }
        },
        Key::Down => {
            if y < height {
                y = y+1;
            }
        }
        Key::Left => {
            if x > 0 {
               x = x-1;
            } else if y > 0  {
                y = y - 1;
                if let Some(file) = &state.file {
                    x = file.row(y).len();
                }
            }
        },
        Key::Right =>{
            if x < width {
                x = x+1;
            } else if y < height {
                y = y +1;
                x = 0;
            }
        },
        Key::PageUp => {
            if y > terminal_height {
                y = y - terminal_height + 1;
            } else {
                y = 0
            }
            
        },
        Key::PageDown => {
            if height > y + terminal_height {
                y = y + terminal_height - 1;
            } else {
                y = height;
            }
            
        },
        Key::Home =>  x = 0,
        Key::End => x = width,
        _ => ()
    }
    if  y < height {
        if let Some(file) = &state.file {
            width = file.row(y).len();
        }
    } else {
            width = 0;
    }

    if x > width {
        x = width;
    }
    state.cursor_position = Position{x,y}
}
```
To scroll up a page, we have to check if we are already at a position below `terminal_size`. If so, we go up by subtracting the terminal height from our current position. We add one again because we do not want to end up at the end of the previous page, but at the beginning of the current page.

To scrolll down, we have to check if the file is longer than the terminal. If it is, we add the height of the terminal to the current position and subtract one again. Symmetrical to the behaviour of <kbd>Page Up</kbd>, <kbd>Page Down</kbd> should bring us to the end of the current page, and not to the top of the new page.

## Status bar

The last thing we'll add before finally getting to text editing is a status bar. This will show useful information such as the filename, how many lines are in the file, and what line you're currently on. Later we'll add a marker that tells you whether the file has been modified since it was last saved, and we'll also display the filetype when we implement syntax highlighting.

First we'll simply make room for a one-line status bar at the bottom of the screen.

```rust
fn editor_initialize() -> Result<EditorState, std::io::Error> {
    let mut file = None;
    let args: Vec<String> = env::args().collect();
    let (width, height) =  termion::terminal_size()?;
    let height = height -1;
    if args.len() > 1 {
        let file_name = &args[1];
        if let Ok(new_file) = editor_open(file_name) {
            file = Some(new_file);
        }
    }
   Ok(EditorState {
        quit: false,
        terminal_size: Size {
            width,
            height
        },
        cursor_position: Position{
            x: 0,
            y: 0
        },
        offset: Position {
            x: 0,
            y: 0
        },
        render_position: Position {
            x: 0,
            y: 0
        },
        file: file
    })
}
```
We decrement the terminal_size by one, assuming that the user did not scale his terminal down to one line only (it's harder than you think - apparently many terminals have a minimum height of 2, which makes sense - since you need to have at least one row for the prompt and one for output.)

Notice how with this change, our text viewer works just fine, including scrolling and cursor movement, and the last line where our status bar will be is left alone by the rest of the display code.

To make the status bar stand out, we're going to display it colored. `termion` takes care of the corresponding escape sequences for us, so we don't have to worry about that.

Let's draw a blank status bar of space characters.

```rust
use termion::color;


const STATUS_BG_COLOR: color::Rgb = color::Rgb(63,63,63);


fn editor_draw_status_bar(state: &EditorState) {
    let spaces = " ".repeat(state.terminal_size.width as usize);
    print!("{}{}{}{}{}",color::Bg(STATUS_BG_COLOR), color::Fg(STATUS_BG_COLOR), spaces, color::Bg(color::Reset),  color::Fg(color::Reset));
}
fn editor_draw_rows(state: &EditorState ){
    
    let Size{width, height} = state.terminal_size;
    let mut last_file_row = 0;
    if let Some(file) = &state.file {
        last_file_row = file.len();
    }
    
   for terminal_row in 0..height {
        print!("{}", termion::clear::CurrentLine);
        let file_row = terminal_row + state.offset.y;
        if file_row >= last_file_row {
            if last_file_row == 0 && terminal_row == height/3 {
                draw_welcome_message(width as usize);
            } else {
                print!("~");
            }
        } else {
             if let Some(file) = &state.file {
                let current_row = file.row(file_row);

                let current_row_len = current_row.len_render();
                let mut line_end = state.offset.x + width;
                let mut line_start = state.offset.x;
                if line_end > current_row_len {
                    line_end = current_row_len;
                }
                if line_start > current_row_len {
                    line_start = current_row_len;
                }
                let slice = &current_row.slice_render(line_start, line_end);
                print!("{}", slice);
            }
         
        }
        print!("\r\n");
    }
}

fn editor_refresh_screen(state: &EditorState)  {
    print!("{}{}", termion::cursor::Hide,  termion::cursor::Goto(1,1));
    if state.quit {
        print!("{}Goodbye!\r\n", termion::clear::All);

    } else {
        let Position{x, y} = state.render_position;
        editor_draw_rows(&state);
        editor_draw_status_bar(&state);
        print!("{}", termion::cursor::Goto(x-state.offset.x+1, y-state.offset.y+1));
    }

    print!("{}", termion::cursor::Show);
}
```
There are a few things to note here.
- Remember the bug we once had where we printed an extra `\r\n` at the bottom? Well, we need that now back into the code, otherwise our status bar would come after the last line on screen.
- We are setting the background and the foreground color to the same value, otherwise our spaces would be rendered in the foreground color.
- We need to reset the background and the foreground color again at the end, otherwise our editor will have athe colors all over the place

Our `print!` statement is getting a bit out of hand: We have a couple of placeholders with more to come, which all need to be filled in the correct order. Let's refactor that bit to make it a bit easier to maintain.

```rust
fn editor_draw_status_bar(state: &EditorState) {
    let spaces = " ".repeat(state.terminal_size.width as usize);
    print!("{bg}{string}bgreset}",
    bg = color::Fg(STATUS_BG_COLOR), 
    string = spaces, 
    bgreset = color::Bg(color::Reset),  
}

```

Since we want to display the filename in the status bar, let's add a `filename` string to the `File` struct, and save a copy of the filename there when a file is opened.

```rust
struct File {
    rows: Vec<FileRow>,
    filename: String
}



/*** file i/o ***/
fn editor_open(filename: &String) -> Result<File, std::io::Error>{
    let contents = fs::read_to_string(filename)?;
    let mut rows = Vec::new();
    for value in contents.lines() {
        rows.push(FileRow::from(value));
    }
    Ok(File {
        rows,
        filename: filename.clone()
    })
}
```

`clone()` makes a copy of the string for us to use. It does not make a huge difference for now if we keep our own copy of the file name or not - but it will later, as we might want to change the file name.

Now we’re ready to display some information in the status bar. We’ll display up to 20 characters of the filename, followed by the number of lines in the file. If there is no filename, we’ll display [No Name] instead.

```rust 
const STATUS_FG_COLOR: color::Rgb = color::Rgb(63,63,63);


fn editor_draw_status_bar(state: &EditorState) {
    let mut status;
    let width = state.terminal_size.width;
    if let Some(file) = &state.file {
        let mut filename_len = file.filename.len();
        if filename_len > 20 {
            filename_len = 20;
        }
        status = format!("{} - {} lines", &file.filename[..filename_len], file.len());
        
    } else {
        status = String::from("[No Name]");
    }
    let mut num_spaces = 0;
    let len = status.len() as u16;
    if width > len {
       num_spaces = width - len;
    }
    for _i in 0..num_spaces {        
        status.push_str(" ");
    }
    status.truncate(width as usize);
    print!("{bg}{fg}{string}{fgreset}{bgreset}",
    fg = color::Fg(STATUS_FG_COLOR), 
    bg = color::Bg(STATUS_BG_COLOR), 
    string = status, 
    bgreset = color::Bg(color::Reset),  
    fgreset = color::Fg(color::Reset));
}

```

We make sure to cut the status string short in case it doesn’t fit inside the width of the window. Notice how we still use code that draws spaces up to the end of the screen, so that the entire status bar has a white background.

Now let’s show the current line number, and align it to the right edge of the screen.

```rust
fn editor_draw_status_bar(state: &EditorState) {
    let mut filename_status;
    let width = state.terminal_size.width;
    let mut file_len = 0;
    if let Some(file) = &state.file {
        let mut filename_len = file.filename.len();
        file_len = file.len();
        if filename_len > 20 {
            filename_len = 20;
        }
        filename_status = format!("{} - {} lines", &file.filename[..filename_len], file.len());
        
    } else {
        filename_status = String::from("[No Name]");
    }
    let line_indicator = format!("{}/{}", state.cursor_position.y+1, file_len);

    let mut spaces = String::from("");
    let mut num_spaces = 0;
    let len = filename_status.len()  + line_indicator.len();
    let len = len as u16;
    if width > len  {
       num_spaces = width - len;
    }
    for _i in 0..num_spaces {        
        spaces.push_str(" ");
    }

    let mut status = format!("{}{}{}", filename_status, spaces,line_indicator);
    status.truncate(width as usize);
    print!("{bg}{fg}{status}{fgreset}{bgreset}",
    fg = color::Fg(STATUS_FG_COLOR), 
    bg = color::Bg(STATUS_BG_COLOR), 
    status = status, 
    bgreset = color::Bg(color::Reset),  
    fgreset = color::Fg(color::Reset));
}
```
The current line is stored in `cursor_position.y`, which we add 1 to since the position is 0-indexed. We are subtracting the length of the new part of the status bar from the number of spaces we want to produce, and add it to the final formatted string.

## Status message

We're going to add one more line below our status bar. This will be for displaying messages to the user, and prompting the user for input when doing a search, for example. We'll store the current message in a struct called `StatusMEssage`, which we'll put in the editor state. We'll also store a timestamp for the message, so that we can erase it a few seconds after it's been displayed.

```rust
use std::time::Instant;

struct StatusMessage {
    text: String,
    time: Instant
}
impl StatusMessage {
    fn from(message: String) -> StatusMessage {
        StatusMessage {
            time: Instant::now(),
            text: message
        }
    }
}

struct EditorState {
    quit: bool,
    terminal_size: Size,
    cursor_position: Position,
    render_position: Position,
    offset: Position,
    file: Option<File>,
    status_message: StatusMessage
}

/*** init ***/

fn editor_initialize() -> Result<EditorState, std::io::Error> {
   let (width, height) =  termion::terminal_size()?;
   let height = height -2;
   let help_message = String::from("HELP: Ctrl-Q = quit");
   let mut state = EditorState {
        quit: false,
        terminal_size: Size {
            width,
            height
        },
        cursor_position: Position{
            x: 0,
            y: 0
        },
        offset: Position {
            x: 0,
            y: 0
        },
        render_position: Position {
            x: 0,
            y: 0
        },
        file: None,
        status_message: StatusMessage::from(help_message),
    };
    let args: Vec<String> = env::args().collect();
    if args.len() > 1 {
        let file_name = &args[1];
        if let Ok(new_file) = editor_open(file_name) {
            state.file = Some(new_file);
        } else {
            state.status_message = StatusMessage::from(format!("ERR: Could not open file:{}", file_name));
        }
    }
    
   Ok(state)
}

```
We initialize `status_message` to  a help message with the key bindings. We also take the opportunity and set the status message to an error if we can't open the file, something that we silently ignored before.

Now that we have a status message to display, let’s make room for a second line beneath our status bar where we’ll display the message. 

```rust
fn editor_draw_status_bar(state: &EditorState) {
    let mut filename_status;
    let width = state.terminal_size.width;
    let mut file_len = 0;
    if let Some(file) = &state.file {
        let mut filename_len = file.filename.len();
        file_len = file.len();
        if filename_len > 20 {
            filename_len = 20;
        }
        filename_status = format!("{} - {} lines", &file.filename[..filename_len], file.len());
        
    } else {
        filename_status = String::from("[No Name]");
    }
    let line_indicator = format!("{}/{}", state.cursor_position.y+1, file_len);

    let mut spaces = String::from("");
    let mut num_spaces = 0;
    let len = filename_status.len()  + line_indicator.len();
    let len = len as u16;
    if width > len  {
       num_spaces = width - len;
    }
    for _i in 0..num_spaces {        
        spaces.push_str(" ");
    }

    let mut status = format!("{}{}{}", filename_status, spaces,line_indicator);
    status.truncate(width as usize);
    print!("{bg}{fg}{status}{fgreset}{bgreset}\r\n",
    fg = color::Fg(STATUS_FG_COLOR), 
    bg = color::Bg(STATUS_BG_COLOR), 
    status = status, 
    bgreset = color::Bg(color::Reset),  
    fgreset = color::Fg(color::Reset));
}
/*** init ***/
fn editor_initialize() -> Result<EditorState, std::io::Error> {
   let (width, height) =  termion::terminal_size()?;
   let height = height -2;
   let mut state = EditorState {
        quit: false,
        terminal_size: Size {
            width,
            height
        },
        cursor_position: Position{
            x: 0,
            y: 0
        },
        offset: Position {
            x: 0,
            y: 0
        },
        render_position: Position {
            x: 0,
            y: 0
        },
        file: None,
        status_message: None
    };
    editor_set_status_message(&mut state, String::from("HELP: Ctrl-Q = quit"));
    let args: Vec<String> = env::args().collect();
    if args.len() > 1 {
        let file_name = &args[1];
        if let Ok(new_file) = editor_open(file_name) {
            state.file = Some(new_file);
        } else {
            editor_set_status_message(&mut state, format!("ERR: Could not open file:{}", file_name));
        }
    }
    
   Ok(state)
}
```
We decrement `height` again, and print a newline after the first status bar. We now have a blank final line once again.

Let’s draw the message bar in a new `editor_draw_message_bar()` function.

```rust
fn editor_draw_message_bar(state: &EditorState) {
    print!("{}", termion::clear::CurrentLine);
    let message = &state.status_message ;
    if Instant::now() - message.time < Duration::new(5,0) {
          let mut text = message.text.clone();
          text.truncate(state.terminal_size.width as usize);
          print!("{}", text);
    }
    
}


fn editor_refresh_screen(state: &EditorState)  {
    print!("{}{}", termion::cursor::Hide,  termion::cursor::Goto(1,1));
    if state.quit {
        print!("{}Goodbye!\r\n", termion::clear::All);

    } else {
        let Position{x, y} = state.render_position;
        editor_draw_rows(&state);
        editor_draw_status_bar(&state);
        editor_draw_message_bar(&state);
        print!("{}", termion::cursor::Goto(x-state.offset.x+1, y-state.offset.y+1));
    }

    print!("{}", termion::cursor::Show);
}
```
First we clear the message bar with ` termion::cursor::Hide`. We did not need to do that for the other status bar, since we are always overwriting the full line on every render. THen we make sure the message will fit the width of the screen, and then display the message, but only if the message is less than 5 seconds old.

This means that we keep the old status message around, even if we are no longer displaying it. That is ok, since that data structure is small anyways and is not designed to grow over time.

When you start up the program now, you should see the help message at the bottom. It will disappear *when you press a key* after 5 seconds. Remember, we only refresh the screen after each keypress.

In the next chapter, we will turn our text viewer into a text editor, allowing the user to insert and delete characters and save their changes to disk.
