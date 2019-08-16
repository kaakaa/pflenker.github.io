---
layout: post
title: "Hecto, Chapter 4: A text viewer"
categories: [Rust, hecto]
---
Let's see if we can turn `hecto` into a text viewer in this chapter.

## A line viewer
We need one more data structures: A `Document`, which will represent the, you guessed it, document the user is currently editing. We create a new folder called `document` for that.

We start with the Row first, so we create a file called `row.rs` in `document` with the following content:

```rust
pub struct Row {
    string: String
}
```
Then we create a `mod.rs` alongside it - this will be the acutual `Document` module. It has the following contents:

```rust
mod row;
pub use row::Row;

pub struct Document {
    rows: Vec<Row>
}

impl Document {
    pub fn new() -> Document{
        Document{
            rows: Vec::new()
        }
    }
}

```

Now we have two `mod.rs` files. Going forward, this tutorial will be referring to the modules by name - for example, when talking about `editor`, we mean the file `editor/mod.rs`.

We are also using a new keyword here: `pub use`. With `pub use`, we make sure that others can reference a row with `document::Document::Row` instead of having to use `document::Document::row::Row`. 

Let's use this all together in `editor`:
```rust
mod document;
use document::Document;
//...
    pub fn new() -> Result<Editor, std::io::Error> {
        Ok(Editor{
            should_quit: false,
            terminal: Terminal::new()?,
            document: Document::new(),
            cursor_position: Position {
                x: 0,
                y: 0
            }
        })
    }
```


We will use `Document` to represent the file we are displaying right now. For now, it only has a vector of `Row`s, but we will extend it later. A Vector is a data structure similar to an array, with the main difference that arrays have fixed lenghts, whereas vectors can grow dynamically. Since we eventually want the user to edit rows, we choose a Vector over an array.

Let's fill it with some text now. We won't worry about reading from a file just yet. Instead, we'll hardcode a "Hello, World" string into it.

Let's prepare `row`:

```rust
impl Row {
    fn from(slice: &str) -> Row{
        Row {
            string: String::from(slice)
        }
    }
}
```
Then let's use this in `document`:

```rust
impl Document {
    pub fn new() -> Document{
        let mut rows =  Vec::new();
        rows.push(Row::from("Hello, World!"));
        Document{
            rows
        }
    }
}



```

We will later implement a method to open a `Document` from file. When we do it, `new` will return an empty document again. But let's focus on getting our hardcoded value displayed for now. Same as before, we start with exposing methods in `Row`, which we will then use in `Document` and in `Editor`. 

First in `row`:
```rust
    pub fn render(&self, start: usize, end: usize) -> String {
        let mut end = end;
        let mut start = start;
        if end > self.string.len() {
            end = self.string.len();
        }
        if start > end {
            start = end;
        }
        String::from(&self.string[start..end]);
    }
```

We call this method `render`, beause eventually it will be responsible for a few more things than just returning a substring. Our `render` method is very user friendly as it normalizes bogus input. Essentially, it returns the biggest possible sub string it can generate.

Now let's create a method to access a `Row` on a `Document`.

```rust
    pub fn row(&self, index: usize) -> Option<&Row> {
        if index >= self.rows.len() {
            return None;
        }
        Some(&self.rows[index])
    }
```
Let's tie this together in the `Editor`.

```rust
mod document;
use document::Row;
//...
    pub fn draw_row(&self, row: &Row) {
        let start = 0;
        let end = self.terminal.size().width as usize;
        let row = row.render(start,end);
        print!("{}\r\n", row)
    }

    fn draw_rows(&self) {
        let height = self.terminal.size().height;
        for terminal_row in 0..height-1 {
            Terminal::clear_current_line();
            if let Some(row) = self.document.row(terminal_row as usize) {
                self.draw_row(row);
            } else if terminal_row == height / 3 {
                self.draw_welcome_message()
            } else {
                print!("~\r\n");
            }
        }
    }
```
We have added a new method to `Document` which returns a row to us if we have one, and `None` otherwise. Next, in `draw_rows`, we first rename the variable `row` to `terminal_row` to avoid confusion with the row we are now getting from `Document`. We are then retrieving the `row` and displaying it. The concept here is that `Row` makes sure to return to you a substring that can be displayed, while the `Editor` makes sure that the terminal dimensions are met.

However, our editor name is still displayed. We don't want that when the user is opening a file, so let's add a method `is_empty` to our `Document`:

```rust
    pub fn is_empty(&self) -> bool {
        self.rows.is_empty()
    }
```
Now we check against that in `draw_rows`:

```rust
    fn draw_rows(&self) {
        let height = self.terminal.size().height;
        for terminal_row in 0..height-1 {
            Terminal::clear_current_line();
            if let Some(row) = self.document.row(terminal_row as usize) {
                self.draw_row(row);
            } else if self.document.is_empty() && terminal_row == height / 3 {
                self.draw_welcome_message()
            } else {
                print!("~\r\n");
            }
        }
    }
```

You should be able to confirm that the message is no longer shown in the middle of the screen. Next, let's allow the user to open and display actual file. We start by changing our `Document`:

```rust
use std::fs;
//...

  pub fn new() -> Document{
        Document{
            rows: Vec::new()
        }
    }
    pub fn open(filename: &String ) -> Result<Document, std::io::Error> {
        let contents = fs::read_to_string(filename)?;
        let mut rows = Vec::new();
        for value in contents.lines() {
            rows.push(Row::from(value));
        }
        Ok(Document{
            rows
        })
    }
```
We have now restored the original functionality of `Document::new`, and added a new method `open`, which attempts to open a file and returns an error in case of a failure.


`open` reads the lines into our `Document` struct. It's not obvious from our code, but each row in `rows` will not contain the line endings `\n` or `\r\n`, as Rust's `line()` method will cut it away for us. That makes sense: We are already handling new lines ourselves, so we wouldn't want to handle the ones in the file anyways.

Let's use this code to initialize our document in the `Editor`. 

```rust
    pub fn new() -> Result<Editor, std::io::Error> {
        let args: Vec<String> = env::args().collect();
        let mut file = None;
        if args.len() > 1 {
             let file_name = &args[1];
            if let Ok(new_file) = Document::open( &file_name) {
                file = Some(new_file);
            } 
        } 

        let mut document;
        if let Some(file) = file {
            document = file;
        } else {
            document = Document::new()
        }
        
        Ok(Editor{
            should_quit: false,
            terminal: Terminal::new()?,
            document,
            cursor_position: Position {
                x: 0,
                y: 0
            }
        })
    }

```

Here are a few things to observe. First, note how we go at great lengths to ensure that no variable is ever undefined, and that a definite `Document` is finally returned with `Editor`. It's one of the strengths of Rust that it just doesn't compile with code that might be undefined. You can also see that the Rust compiler is smart enough to figure out that `document` will never be undefined, as it is defined in the next `if..else` statement.


We are calling `Document::open()` only if we have more than one `arg`. `args` is a vector which contains the command line parameters which have been passed to our program. Per convention, `args[0]` is always the name of our program, so `args[1]` contains the parameter we're after - we want to use `hecto (filename)` to open a file.  You can pass a file name to your program by running `cargo run (filename)` while you are developing.

Now you should see your screen fill up with lines of text when you run `cargo run src/mod.rs`, for example.

Since it is much more likely that `editor_open` will fail than, for example, `into_raw_mode`, we have to handle the error more gracefully and should not simply `panic` if a file can't be found. For now, if we can't open the file, we are letting the user edit a new one instead. 

## Scrolling

Next we want to enable the user to scroll through the whole file, instead of just being able to see the top few lines of the file. Let's add an `offset`  to the editor state, which will keep track of what row of the file the user is currently scrolled to. We are reusing the `Position` struct for that.

```rust
pub struct Editor {
    should_quit: bool,
    terminal: Terminal,
    cursor_position: Position,
    offset: Position,
    document: Document
}

//...
    pub fn new() -> Result<Editor, std::io::Error> {
        let args: Vec<String> = env::args().collect();
        let mut file = None;
        if args.len() > 1 {
            let file_name = &args[1];
            if let Ok(new_file) = Document::open( &file_name) {
                file = Some(new_file);
            } 
        } 

        let mut document;
        if let Some(file) = file {
            document = file;
        } else {
            document = Document::new()
        }
        
        Ok(Editor{
            should_quit: false,
            terminal: Terminal::new()?,
            document,
            cursor_position: Position {
                x: 0,
                y: 0
            },
            offset: Position {
                x: 0,
                y: 0
            }
        })
    }

```

We initialize it to `0`, which means we'll be scrolled to the top of the file by default.

Now let's have `draw_row()` display the correct range of lines of the file according to the value of `offset.x`.

```rust
    pub fn draw_row(&self, row: &Row) {
        let width = self.terminal.size().width as usize;
        let start = self.offset.x as usize;
        let end = self.offset.x + width;
        let row = row.render(start,end);
        print!("{}\r\n", row)
    }
```

We are adding the offset to the start and to the end, to get the slice of the string we're after. We are also making sure that we can handle situations where our string is not long enough to fit the screen, or has ended to the left of the current screen (which can happen if we are in a long row and scroll to the right).

Similarily, let's make sure we only draw rows which are currently visible.

```rust
    fn draw_rows(&self) {
        let height = self.terminal.size().height;
        for terminal_row in 0..height-1 {
            Terminal::clear_current_line();
            if let Some(row) = self.document.row(terminal_row as usize + self.offset.y) {
                self.draw_row(row);
            } else if self.document.is_empty() && terminal_row == height / 3 {
                self.draw_welcome_message()
            } else {
                print!("~\r\n");
            }
        }
    }
```

Where do we set the value of  `offset`? Our strategy will be to check if the cursor has moved outside of the visible window, and if so, adjust `offset` so that the cursor is just inside the visible window. We’ll put this logic in a function called `scroll`, and call it right after we handled the key press.


```rust
    fn scroll(&mut self) {
        let Position{x,y} = self.cursor_position;
        let width = self.terminal.size().width as usize;
        let height = self.terminal.size().height as usize;
        let mut offset = &mut self.offset;
        if y < offset.y {
            offset.y = y;
        } else  if y >= offset.y + height {
            offset.y = y - height + 1;
        }
        if x < offset.x {
            offset.x = x;
        } else if x >= offset.x + width {
            offset.x = x - width  + 1;
        }    
    }

    //...
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
             Key::Home=>  self.move_cursor(&pressed_key),
             _ => ()
        }
        self.scroll();
        Ok(())
    }
```
To `scroll`, we need to know the width and height of the terminal and the current position, and we want to change the values in `self.offset`. If we have moved to the left or to the top, we want to set our offset to the new position in the document. If we have scrolled too far to the right, we are subtracting the current offset from the new position to calculate the new offset. 

To be able to do that, we need to add a method that returns the length of our document.
```rust
    pub fn len(&self) -> usize {
        self.rows.len()
    }
```

Now let's allow the cursor to advance past the bottom of the screen (but not past the bottom of the file). We tackle scrolling to the right a bit later.

```rust
    fn move_cursor(&mut self, key: &Key){
        let size = self.terminal.size();
        let height = self.document.len();
        let width = size.width as usize;
        let mut position = &mut self.cursor_position;
        match key {
            Key::Up => {
                if position.y > 0 {
                    position.y = position.y-1;
                }
            }
            Key::Down => {
                if position.y < std::usize::MAX && position.y < height {
                    position.y = position.y + 1;
                } 
            }
            Key::Left => {
                if position.x > 0 {
                    position.x = position.x -1;
                }
            }
            Key::Right => {
                if position.x < std::usize::MAX && position.x < width {
                    position.x = position.x + 1;
                }
            }
            Key::PageUp => position.y = 0,
            Key::PageDown => position.y = height,
            Key::Home =>  position.x = 0,
            Key::End => position.x = width,
            _ => ()
        }
        
    }
```

You should be able to scroll through the entire file now, when you run `cargo run src/main.rs`. The handling of the last line will be a bit strange, since we place our cursor there, but are not rendering there. This will be fixed when we add the status bar later in this chapter.  
If you try to scroll back up, you may notice the cursor isn't being positioned properly. That is because `Position` in the state no longer refers to the position of the cursor on the screen. It refers to the position of the cursor within the text file. To position the cursor on the screen, we now have to subtract the `offset` from the  position.

```rust 
 fn refresh_screen(&self) -> Result<(), std::io::Error>{
        Terminal::cursor_hide();      
        Terminal::cursor_position(&Position{x: 0, y: 0});

        if self.should_quit {
            Terminal::clear_screen();
            print!("Goodbye!\r\n");
        } else {
            self.draw_rows();
            let terminal_position = Position{
                x: self.cursor_position.x - self.offset.x,
                y: self.cursor_position.y - self.offset.y
            };
            Terminal::cursor_position(&terminal_position);
        }

        Terminal::cursor_show();
        Terminal::flush()
    }
```


Now let's fix the horizontal scrolling. The only missing piece here is that we are not yet allowing the cursor to scroll past the right of the screen. First, we need to add a `len` method to `Row`.

```rust
    pub fn len(&self) -> usize {
        self.string.len()
    }
```

Then, we use it in the `Editor`. Let's do that now.

```rust

    fn move_cursor(&mut self, key: &Key){
        let height = self.document.len();
        let mut width = 0;
        let mut position = &mut self.cursor_position;

        if let Some(row) = self.document.row(position.y) {
            width = row.len() as usize;
        }
        match key {
            Key::Up => {
                if position.y > 0 {
                    position.y = position.y-1;
                }
            }
            Key::Down => {
                if position.y < std::usize::MAX && position.y < height {
                    position.y = position.y + 1;
                } 
            }
            Key::Left => {
                if position.x > 0 {
                    position.x = position.x -1;
                }
            }
            Key::Right => {
                if position.x < std::usize::MAX && position.x < width {
                    position.x = position.x + 1;
                }
            }
            Key::PageUp => position.y = 0,
            Key::PageDown => position.y = height,
            Key::Home =>  position.x = 0,
            Key::End => position.x = width,
            _ => ()
        }
        
    }
```
All we had to do is changing the width used by `move_cursor`. Horizontral scrolling does now work.

## Snap cursor to end of line

Now `cursor_position` refers to the cursor's position within the file, not its position on the screen. So our goal with the next few steps is to limit the values of `cursor_position` ` to only ever point to valid positions in the file, with the exception that we allow the cursor to point one character past the end of a line or past the end of the file, so that the user can add new characters at the end of a line, and \ new lines at the end of the file can be added easily.

We are already able to prevent the user from scrolling too far to the right or too far down. The user is still able to move the cursor past the end of a line, however. They can do it by moving the cursor to the end of a long line, then moving it down to the next line, which is shorter. The `cursor_position.y` value won't change, and the cursor will be off to the right of the end of the line it's now on.

Let's add some code to `move_cursor()` that corrects `cursor_position` if it ends up past the end of the line it's on.

```rust
    fn move_cursor(&mut self, key: &Key){
        let height = self.document.len();
        let mut width = 0;
        let mut position = &mut self.cursor_position;

        if let Some(row) = self.document.row(position.y) {
            width = row.len() as usize;
        }
        match key {
            Key::Up => {
                if position.y > 0 {
                    position.y = position.y-1;
                }
            }
            Key::Down => {
                if position.y < std::usize::MAX && position.y < height {
                    position.y = position.y + 1;
                } 
            }
            Key::Left => {
                if position.x > 0 {
                    position.x = position.x -1;
                }
            }
            Key::Right => {
                if position.x < std::usize::MAX && position.x < width {
                    position.x = position.x + 1;
                }
            }
            Key::PageUp => position.y = 0,
            Key::PageDown => position.y = height,
            Key::Home =>  position.x = 0,
            Key::End => position.x = width,
            _ => ()
        }
        width = 0;
        if  position.y < height {
            if let Some(row) = self.document.row(position.y) {
                width = row.len();
            }

        }
        if position.x > width {
            position.x = width;
        }
            
    }
```

We have to set `width` again, since `row` could point to a different line than it did before. We then set `position.y` to the end of that line if `position.y`, which will be the new cursor position, is to the right of the end of that line. Also note that we consider non-existing lines to be of length `0`, which works for our purposes here. As a side note, we could have saved the `if..let` since we are explicitly checking if `position.y` contains a value that matches an existing row, but it's better to explicitly do the check than to use `unwrap` here. The code above might change, so our assumption might not always be true.


## Scrolling with <kbd>Page Up</kbd> and <kbd>Page Down</kbd>
Now that we have scrolling, let's make the <kbd>Page Up</kbd> and <kbd>Page Down</kbd> keys scroll up or down an entire page.

```rust
   fn move_cursor(&mut self, key: &Key){
        let terminal_height = self.terminal.size().height as usize;
        let height = self.document.len();
        let mut width = 0;
        let mut position = &mut self.cursor_position;

        if let Some(row) = self.document.row(position.y) {
            width = row.len() as usize;
        }
        match key {
            Key::Up => {
                if position.y > 0 {
                    position.y -= 1;
                }
            }
            Key::Down => {
                if position.y < std::usize::MAX && position.y < height {
                    position.y += 1;
                } 
            }
            Key::Left => {
                if position.x > 0 {
                    position.x -= 1;
                }
            }
            Key::Right => {
                if position.x < std::usize::MAX && position.x < width {
                    position.x += 1;
                }
            }
            Key::PageUp =>{
                if position.y > terminal_height {
                    position.y -= terminal_height;
                } else {
                    position.y = 0;
                }
            }
            Key::PageDown =>{
                if position.y + terminal_height < height {
                    position.y += terminal_height;
                } else {
                    position.y = height;  
                }
            } 
            Key::Home =>  position.x = 0,
            Key::End => position.x = width,
            _ => ()
        }
        width = 0;
        if  position.y < height {
            if let Some(row) = self.document.row(position.y) {
                width = row.len();
            }

        }
        if position.x > width {
            position.x = width;
        }
            
    }
```
If we try this out now, we can see that we still have some issues with the last line. Instead of moving to the next screen on <kbd>Page Down</kbd>, our cursor lands at the empty row at the bottom. Let's fix this - but before we do, let's complete our cursor navigation in this file.

## Moving left at the start of a line

We want to allow the user to press <kbd>&larr;</kbd> at the beginning of the line to move to the end of the previous line.
```rust
fn move_cursor(&mut self, key: &Key){
        let terminal_height = self.terminal.size().height as usize;
        let height = self.document.len();
        let mut width = 0;
        let mut position = &mut self.cursor_position;

        if let Some(row) = self.document.row(position.y) {
            width = row.len() as usize;
        }
        match key {
            Key::Up => {
                if position.y > 0 {
                    position.y -= 1;
                }
            }
            Key::Down => {
                if position.y < std::usize::MAX && position.y < height {
                    position.y += 1;
                } 
            }
            Key::Left => {
                if position.x > 0 {
                    position.x -= 1;
                } else if position.y > 0 {
                    position.y = position.y -1;
                    if let Some(row) = self.document.row(position.y) {
                        position.x = row.len();
                    } else {
                        position.x = 0;
                    }
                }
            }
            Key::Right => {
                if position.x < std::usize::MAX && position.x < width {
                    position.x += 1;
                }
            }
            Key::PageUp =>{
                if position.y > terminal_height {
                    position.y -= terminal_height;
                } else {
                    position.y = 0;
                }
            }
            Key::PageDown =>{
                if position.y + terminal_height < height {
                    position.y += terminal_height;
                } else {
                    position.y = height;  
                }
            } 
            Key::Home =>  position.x = 0,
            Key::End => position.x = width,
            _ => ()
        }
        width = 0;
        if  position.y < height {
            if let Some(row) = self.document.row(position.y) {
                width = row.len();
            }

        }
        if position.x > width {
            position.x = width;
        }
            
    }
```

We make sure they aren't on the very first line before we move them up a line.

## Moving right at the end of a line

Similarly, let's allow the user to press <kbd>&rarr;</kbd> at the end of a line to go to the beginning of the next line.

```rust
fn move_cursor(&mut self, key: &Key){
        let terminal_height = self.terminal.size().height as usize;
        let height = self.document.len();
        let mut width = 0;
        let mut position = &mut self.cursor_position;

        if let Some(row) = self.document.row(position.y) {
            width = row.len() as usize;
        }
        match key {
            Key::Up => {
                if position.y > 0 {
                    position.y -= 1;
                }
            }
            Key::Down => {
                if position.y < std::usize::MAX && position.y < height {
                    position.y += 1;
                } 
            }
            Key::Left => {
                if position.x > 0 {
                    position.x -= 1;
                } else if position.y > 0 {
                    position.y = position.y -1;
                    if let Some(row) = self.document.row(position.y) {
                        position.x = row.len();
                    } else {
                        position.x = 0;
                    }
                }
            }
            Key::Right => {
                if position.x < std::usize::MAX && position.x < width {
                    position.x += 1;
                } else if position.y < height {
                    position.y += 1;
                    position.x = 0;
                }
            }
            Key::PageUp =>{
                if position.y > terminal_height {
                    position.y -= terminal_height;
                } else {
                    position.y = 0;
                }
            }
            Key::PageDown =>{
                if position.y + terminal_height < height {
                    position.y += terminal_height;
                } else {
                    position.y = height;  
                }
            } 
            Key::Home =>  position.x = 0,
            Key::End => position.x = width,
            _ => ()
        }
        width = 0;
        if  position.y < height {
            if let Some(row) = self.document.row(position.y) {
                width = row.len();
            }

        }
        if position.x > width {
            position.x = width;
        }
            
    }
```
Here we have to make sure they’re not at the end of the file before moving down a line.  We have added a lot of functionality to `move_cursor`, but didn't think about checking for over- or underflows. However, by looking at our code it's obvious that we are only incrementing values if we have checked before that they are either smaller than another value of the same type, such as `height`, or we have compared the value against `usize::max`. Likewise, in all cases where we decrement a value, we have made sure that it is not `0`. It's good to make it a habit to keep this in mind directly when writing code, as it is much more difficult to figure stuff like this out when the code has grown a lot.

## Rendering Tabs
If you try opening a file with tabs, you'll notice that the tab character takes up a width of 8 colums or so. As you probably know, there is a long and ongoing debate of whether or not to use tabs or spaces for indendation. Honestly, I don't care, I have always a sufficiently advanced editor that could just roll with any indentation type you throw at it.. If I was forced to pick a side, I would pick "spaces", though, because I find the "pros" for tabs not very convincing - but that's a different matter. What matters, though, is that we are simply replacing tabs with one space for the sake of this tutorial. That's enough for our purpose, as in the Rust ecosystem, you will rarely encounter tabs. Feel free to extend your solution to replace tabs with multiple spaces. Be aware, though, that this means that you need to update the cursor position accordingly, since you don't want the cursor to be positioned within those whitespaces.

Let's replace our tabs with spaces now. This is where our `render` method on `Row` becomes useful.

```rust
    pub fn render(&self, start: usize, end: usize) -> String {
        let mut end = end;
        let mut start = start;
        if end > self.string.len() {
            end = self.string.len();
        }
        if start > end {
            start = end;
        }
        let mut result = String::new();
        for c in self.string.chars().skip(start).take(end - start) {
            if c == '\t' {
                result.push(' ');
            } else {
                result.push(c);
            }
        }
        result
    }
```
Wow, that looks complex! Why aren't we simply calling `replace` on our string and that's it?
The reason is that under the hood, `replace` would create a new string one by one based on the old. THat is also the same what happens when we create a slice from the old string. So if we would simply extend our old solution by `replace`ing the result, we would go through our string multiple times. So we have now built a solution where we iterate over the string ourselves.  
We accomplish this by using a few facets of an `Iterator`. As we have seen in an earlier chapter, an iterator allows us to use `for..in` to process its entries. `string.chars()` returns such an iterator. Calling `start(n)` on an iterator also returns an iterator, but only with the first `n` elements skipped. And calling `take(n)` on an iterator returns an iterator as well, but only with `n` elements. The end result is what we want: An iterator going from `start` to `end`.

While we are discussing how hard strings are, let's fix another issue. You can either skip to the next diff, or follow along to learn about the background of the bug that we are seeing. Either create a small throwaway project, or use the [Rust playground](https://play.rust-lang.org/) with the following code:

```rust
fn main() {
    let mut vec = String::new();
    vec.push('a');
    dbg!(vec.len());
    vec.push('ä');
    dbg!(vec.len());
    vec.push('❤');
    dbg!(vec.len());
}
```

`dbg!` is a macro which is useful for quick and dirty debugging, it prints out the current value of what you give in, and some more. Here's what it returns for that code:

```
[src/main.rs:4] vec.len() = 1
[src/main.rs:6] vec.len() = 3
[src/main.rs:8] vec.len() = 6
```
After adding the first character, the length of the string is 1. After adding an umlaut, we see that the lenght has increased by two, and not one. This means that the lenght of the string does not match the number of characters in that string, which is what we are after. You can reproduce that behaviour by creating a `happy.txt` with a long line containing of nothing but ❤, and opening it with `hecto`. You will see that you are allowed to scroll much further to the right than allowed. Let's fix that by changing the output of `len` on our `Row`:

```rust
    pub fn new() -> Row {
        Row {
            string: String::new(),
            len: 0
        }
    }
    pub fn from(slice: &str) -> Row{
        let string = String::from(slice);
        Row {
            len: string.chars().count(),
            string
        }
    }
    pub fn len(&self) -> usize {
        self.len
    }
```
Instead of going through the whole string whenever someone calls `len`, we keep track of the actual length of our row in `len`. We just have to remember to set `len` manually if we are ever going to change `Row`.

That's about all the string handling we are going to do with `hecto` in the scope of this tutorial. THere are other cases that need considering - for instance, `y̆` is a composition of two characters, which show up as one character on the screen. If you need to handle this kind of data, search [crates.io](http://www.crates.io) for crates to handle strings for you.


## Status bar

The last thing we'll add before finally getting to text editing is a status bar. This will show useful information such as the filename, how many lines are in the file, and what line you're currently on. Later we'll add a marker that tells you whether the file has been modified since it was last saved, and we'll also display the filetype when we implement syntax highlighting.

First we'll simply make room for a two-line status bar at the bottom of the screen.

In `terminal.rs`:
```rust
   pub fn new() -> Result<Terminal, std::io::Error>{
        let size =  termion::terminal_size()?;
        let stdout = stdout().into_raw_mode()?;
        Ok(Terminal {
            size: Size {
                width: size.0, 
                height: size.1 - 2,
            },
            _stdout: stdout
        })
    }
```    
 We will now also fix the issues we have with the rendering of the last line.

In `Editor`:
```rust
    fn draw_rows(&self) {
        let height = self.terminal.size().height;
        for terminal_row in 0..height {
            Terminal::clear_current_line();
            if let Some(row) = self.document.row(terminal_row as usize + self.offset.y) {
                self.draw_row(row);
            } else if self.document.is_empty() && terminal_row == height / 3 {
                self.draw_welcome_message()
            } else {
                print!("~\r\n");
            }
        }
    }
```
You should now be able to confirm that two lines are cleared at the bottom and that Page Up and Down works as expected. Note that the meaning of "terminal.size" has now slightly changed from "The size of the terminal" to "The size of the canvas". We will discuss this and other design weaknesses later as an opportunity to improve on the editor.

Notice how with this change, our text viewer works just fine, including scrolling and cursor movement, and the last lines where our status bar will be are left alone by the rest of the display code.

To make the status bar stand out, we're going to display it colored. `termion` takes care of the corresponding escape sequences for us, so we don't have to do this manually. The corresponding escape sequence is the `m` command ([Select Graphic Rendition](http://vt100.net/docs/vt100-ug/chapter3.html#SGR)).  The [VT100 User Guide](http://vt100.net/docs/vt100-ug/chapter3.html) doesn't document color, so let's turn to the Wikipedia article on [ANSI escape codes](https://en.wikipedia.org/wiki/ANSI_escape_code). It includes a large table containing all the different argument codes you can use with the `m` command on various terminals. It also includes the ANSI color table with the 8 foreground/background colors available.

We start by extending our Terminal with a few functions.

In `terminal.rs`:

```rust
use termion::color;
//...
    pub fn set_bg_color(color: color::Rgb) {
        print!("{}",color::Bg(color));
    }
    pub fn reset_bg_color() {
         print!("{}", color::Bg(color::Reset));
    }
```
This is similar to what we have done before: We are simply abstracting away the printing of the chars. With that, our `Editor` looks like this:

```rust 
use termion::color;
const STATUS_BG_COLOR: color::Rgb = color::Rgb(239,239,239);
//...
    fn draw_status_bar(&self) {
        let spaces = " ".repeat(self.terminal.size().width as usize);
        Terminal::set_bg_color(STATUS_BG_COLOR);
        print!("{}\r\n",spaces);
        Terminal::reset_bg_color();
    }
    fn refresh_screen(&self) -> Result<(), std::io::Error>{
        Terminal::cursor_hide();      
        Terminal::cursor_position(&Position{x: 0, y: 0});

        if self.should_quit {
            Terminal::clear_screen();
            print!("Goodbye!\r\n");
        } else {
            self.draw_rows();
            self.draw_status_bar();
            let terminal_position = Position{
                x: self.cursor_position.x - self.offset.x,
                y: self.cursor_position.y - self.offset.y
            };
            Terminal::cursor_position(&terminal_position);
        }

        Terminal::cursor_show();
        Terminal::flush()
    }
```

It's worth noting that we need to reset the colors after we use them, otherwise the rest of the screen will also be rendered in the same color. Try playing around wiht it by removing the `reset_bg_color` call from `draw_status_bar` and see when and how the screen changes.

We want to display the file name next. Let's adjust our `Document` to have an optional file name, and set it in `open`. 

```rust
pub struct Document {
    rows: Vec<String>,
    file_name: Option<String>
}
//...

    pub fn new() -> Document{
        Document{
            rows: Vec::new(),
            pub file_name: None
        }
    }
    pub fn open(file_name: &String ) -> Result<Document, std::io::Error> {
        let contents = fs::read_to_string(file_name)?;
        let mut rows = Vec::new();
        for value in contents.lines() {
            rows.push(String::from(value));
        }
        Ok(Document{
            rows,
            file_name: Some(file_name.clone())
        })
    }
```

Now let's prepare the `Terminal` to set and reset the foreground color:

```rust
    pub fn set_fg_color(color: color::Rgb) {
        print!("{}",color::Fg(color));
    }
    pub fn reset_fg_color() {
         print!("{}", color::Fg(color::Reset));
    }
```


Now we’re ready to display some information in the status bar. We’ll display up to 20 characters of the filename, followed by the number of lines in the file. If there is no filename, we’ll display [No Name] instead.

```rust 
const STATUS_FG_COLOR: color::Rgb = color::Rgb(63,63,63);

//...

 fn draw_status_bar(&self) {
        let mut status;
        let width = self.terminal.size().width as usize;
        if let Some(name) = &self.document.file_name {
            let mut filename_len = name.len();
            if filename_len > 20 {
                filename_len = 20;
            }
            status = format!("{} - {} lines", &name[..filename_len], self.document.len());
        } else {
            status = String::from("[No Name]");
        }
        let mut num_spaces = 0;
        if width > status.len() {
            num_spaces = width - status.len();
        }
        for _i in 0..num_spaces {
            status.push_str(" ");
        }
        status.truncate(width);
        Terminal::set_bg_color(STATUS_BG_COLOR);
        Terminal::set_fg_color(STATUS_FG_COLOR);
        print!("{}\r\n",status);
        Terminal::reset_fg_color();
        Terminal::reset_bg_color();
    }

```

We make sure to cut the status string short in case it doesn’t fit inside the width of the window. Notice how we still use code that draws spaces up to the end of the screen, so that the entire status bar has a white background.

Now let’s show the current line number, and align it to the right edge of the screen.

```rust
    fn draw_status_bar(&self) {
        let mut status;
        let width = self.terminal.size().width as usize;
        if let Some(name) = &self.document.file_name {
            let mut filename_len = name.len();
            if filename_len > 20 {
                filename_len = 20;
            }
            status = format!("{} - {} lines", &name[..filename_len], self.document.len());
        } else {
            status = String::from("[No Name]");
        }
        let line_indicator = format!("{}/{}", self.cursor_position.y+1, self.document.len());
        let mut num_spaces = 0;
        let len = status.len() + line_indicator.len();
        if width > len {
            num_spaces = width - len;
        }
        for _i in 0..num_spaces {
            status.push_str(" ");
        }
        status = format!("{}{}", status, line_indicator);
        status.truncate(width);
        Terminal::set_bg_color(STATUS_BG_COLOR);
        Terminal::set_fg_color(STATUS_FG_COLOR);
        print!("{}\r\n",status);
        Terminal::reset_fg_color();
        Terminal::reset_bg_color();
    }
```
The current line is stored in `cursor_position.y`, which we add 1 to since the position is 0-indexed. We are subtracting the length of the new part of the status bar from the number of spaces we want to produce, and add it to the final formatted string.

## Status message

We're going to add one more line below our status bar. This will be for displaying messages to the user, and prompting the user for input when doing a search, for example. We'll store the current message in a struct called `StatusMessage`, which we'll put in the editor state. We'll also store a timestamp for the message, so that we can erase it a few seconds after it's been displayed.

```rust
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

//...


    pub fn new() -> Result<Editor, std::io::Error> {
        let args: Vec<String> = env::args().collect();
        let mut file = None;
        let mut initial_status = String::from("HELP: Ctrl-Q = quit");
        let mut document;

        if args.len() > 1 {
            let file_name = &args[1];
            if let Ok(new_file) = Document::open( &file_name) {
                file = Some(new_file);
            } else {
                initial_status = format!("ERR: Could not open file:{}", file_name);
            }
        } 

        if let Some(file) = file {
            document = file;
        } else {
            document = Document::new()
        }
        
        Ok(Editor{
            should_quit: false,
            terminal: Terminal::new()?,
            document,
            cursor_position: Position {
                x: 0,
                y: 0
            },
            offset: Position {
                x: 0,
                y: 0
            },
            status_message: StatusMessage::from(initial_status)
        })
    }

```
We initialize `status_message` to a help message with the key bindings. We also take the opportunity and set the status message to an error if we can't open the file, something that we silently ignored before.

Now that we have a status message to display, let’s draw the message bar in a new `editor_draw_message_bar()` function.

```rust
use std::time::Duration;
//...

 fn draw_message_bar(&self) {
        Terminal::clear_current_line();
        let message = &self.status_message ;
        if Instant::now() - message.time < Duration::new(5,0) {
          let mut text = message.text.clone();
          text.truncate(self.terminal.size().width as usize);
          print!("{}", text);
        }
    }
    fn refresh_screen(&self) -> Result<(), std::io::Error>{
        Terminal::cursor_hide();      
        Terminal::cursor_position(&Position{x: 0, y: 0});

        if self.should_quit {
            Terminal::clear_screen();
            print!("Goodbye!\r\n");
        } else {
            self.draw_rows();
            self.draw_status_bar();
            self.draw_message_bar();
            let terminal_position = Position{
                x: self.cursor_position.x - self.offset.x,
                y: self.cursor_position.y - self.offset.y
            };
            Terminal::cursor_position(&terminal_position);
        }

        Terminal::cursor_show();
        Terminal::flush()
    }
```
First we clear the message bar with `Terminal::clear_current_line();`. We did not need to do that for the  status bar, since we are always overwriting the full line on every render. Then we make sure the message will fit the width of the screen, and then display the message, but only if the message is less than 5 seconds old.

This means that we keep the old status message around, even if we are no longer displaying it. That is ok, since that data structure is small anyways and is not designed to grow over time.

When you start up the program now, you should see the help message at the bottom. It will disappear *when you press a key* after 5 seconds. Remember, we only refresh the screen after each keypress.

In the next chapter, we will turn our text viewer into a text editor, allowing the user to insert and delete characters and save their changes to disk.

## Conclusion
In this chapter, all the refactoring of the previous chapters has paid off, as we where able to extend our editor effortlessly. However, as hinted here and there, the current architecture is still far from perfect. However, a different architecture would mean a lot of refactoring during the course of the tutorial, so I leave the final refactoring to you after our tutorial is finished.

I hope that, like in the last chapter, you are looking at your new text viewer with pride. It's coming along! Let's focus on editing text in the next chapter.