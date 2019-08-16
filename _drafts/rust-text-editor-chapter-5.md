---
layout: post
title: "Hecto, Chapter 5: A text editor"
categories: [Rust, hecto]
---
Now that `hecto` can read files, let's see if we can teach it to edit files as well.

## Insert ordinary characters

Let's begin by writing a function that inserts a single character into a `Document`, at a given position. We start by allowing to add a character into our string at a given position.

In `Row`:

```rust
    pub fn insert(&mut self, at: usize, c: char  ) {
        if at == self.len() {
            self.string.push(c);
            self.len += 1;
        } else {
            let mut result = String::new();
            let mut index = 0;
            for (i, character) in self.string.chars().enumerate() {
                index += 1;
                if i == at {
                    result.push(c);
                }
                result.push(character);
            }
            self.string = result;
            self.len = index;
        }
    }
```
Here, we handle two cases: If we happen to be at the end of our string, we push the charater onto it. This can happen if the user is at the end of a line and keeps typing. If not, we are rebuilding our string by going through it character by character. As we saw before, this is not the same as going through the String index by index! `enumerate` is a useful function which transforms our iterator in a way that it does not only return each element, but also each index.

Let's use that function in our `Document`. 

```rust
    pub fn insert(&mut self, at: &Position, c: char){
        if at.y == self.len() {
            let mut row = Row::new();
            row.insert(0,c);
            self.rows.push(row);
        } else if at.y < self.len() {
            let row = self.rows.get_mut(at.y).unwrap();
            row.insert(at.x, c);
        }
    }
```

Symmetrical to our code in `Row`, we are handling the case where the user is attempting to insert at the bottom of our document, for which case we create a new row.

T
Did you notice that we are using `unwrap` here? We use it to get the value returned from `get_mut`. It's safe to do that here, since we know from the if statement that we have a row at that position. We could either have been more cautious, returning on `if let None`, or we could have been more bold, directly accessing the row with `self.rows[at.y]` (which would `panic` if the index is out of bounds). I prefer `unwrap` over the direct access here, even though the result is the same (`panic` if the index is out of bounds). `unwrap` expresses my conscious choice at this point to anyone who maintains the code 

Now we need to call that method when a character is entered.

```rust
    fn process_keypress(&mut self) -> Result<(), std::io::Error> {
        let pressed_key = Terminal::read_key()?;
        match pressed_key {
             Key::Ctrl('q') => self.should_quit = true,
             Key::Char(c) => self.document.insert(&self.cursor_position, c),
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
We can now add characters anywhere in the document. But our cursor does not move - so we are essentially typing in our text backwards. Let's fix that now by treating "Enter a character" as "Enter a character and go to the right". 

```rust

    fn process_keypress(&mut self) -> Result<(), std::io::Error> {
        let pressed_key = Terminal::read_key()?;
        match pressed_key {
             Key::Ctrl('q') => self.should_quit = true,
             Key::Char(c) => {
                 self.document.insert(&self.cursor_position, c);
                 self.move_cursor(&Key::Right);
             },
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

You should now be able to confirm that putting in characters works, even at the bottom of the file.

## Simple deletion
We now want Backspace and Delete to work Let's start with Delete, which should remove the character in front of the cursor. In reality, "in front of the cursor" means "on the cursor", since the cursor is a blinking line displayed on the left side of its position. Let's start by adding a `delete` function on a `row`.

```rust
    pub fn delete(&mut self, at: usize) {
        if at > self.len {
            return;
        }
        let mut result = String::new();
        let mut index = 0;
        for (i, character) in self.string.chars().enumerate() {
            index += 1;
            if i != at {
                result.push(character);
            }
        }
        self.string = result;
        self.len = index;
    }
```
This is quite similar to `insert` from above. Let's continue with `Document`.

```rust
    pub fn delete(&mut self, at: &Position) {
        if at.y >= self.len() {
            return;
        }
        let row = self.rows.get_mut(at.y).unwrap();
        row.delete(at.x);
    }
```

Now, we add handling for `Delete` in our `Editor`. 

```rust
    fn process_keypress(&mut self) -> Result<(), std::io::Error> {
        let pressed_key = Terminal::read_key()?;
        match pressed_key {
             Key::Ctrl('q') => self.should_quit = true,
             Key::Char(c) => {
                self.document.insert(&self.cursor_position, c);
                self.move_cursor(&Key::Right);
             },
             Key::Delete => self.document.delete(&self.cursor_position),
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

You should now be able to delete characters from with in a line. Let's tackle Backspace next: Essentially, Backspace is a combination of going left and deleting, so we adjust `process_keypress` as follows: 

```rust
    fn process_keypress(&mut self) -> Result<(), std::io::Error> {
        let pressed_key = Terminal::read_key()?;
        match pressed_key {
             Key::Ctrl('q') => self.should_quit = true,
             Key::Char(c) => {
                self.document.insert(&self.cursor_position, c);
                self.move_cursor(&Key::Right);
             },
             Key::Delete => self.document.delete(&self.cursor_position),
             Key::Backspace => {
                 self.move_cursor(&Key::Left);
                 self.document.delete(&self.cursor_position);
             }
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
Backspace now works within a line. If you do it at the beginning of a line, however, it moves up a line without doing anything else. Let's fix that in the next sections.


## Complex deletion
There are two edge cases which we can't handle right now, and that is either using Backspace at the beginning of a line, or using Delete at the end of a line. Since in our case Backspace _is_ a delete operateion preceded by a cursor move, it's technically only one edge case that we should handle, and it involves copying the contents of one row into another.

We start by giving `Row` the ability to return a copy of its own string and append another string:

```rust
    pub fn to_string(&self) -> String {
        self.string.clone()
    }
    pub fn append(&mut self, string: &String) {
        self.string = format!("{}{}", self.string, string);
        self.len = self.string.chars().count();
    }
```

Note that since Rust does not abstract the difficulty with Strings away from us, it's plain to see for us how often we are actually cloning or copying our strings around. `to_string` returns a _copy_ of our string, and append_string creates a copy of the former internal string and the input string. After this tutorial is done, you will find several opportunities of improving these matters in the code. 

Now we call these methods from `delete`.

```rust
   pub fn delete(&mut self, at: &Position) {
        let len = self.len();
        if at.y >= len {
            return;
        }
        let row = self.rows.get(at.y).unwrap();
        if at.x == row.len() && at.y < len -1 {
            let next_string = self.rows.get(at.y + 1).unwrap().to_string();
            let row = self.rows.get_mut(at.y).unwrap();
            row.append_string(&next_string);
            self.rows.remove(at.y+1);
            
        } else {
            let row = self.rows.get_mut(at.y).unwrap();
            row.delete(at.x); 
        }
        
    }
```
We make sure that we are at the end of the row, and not on the last line before performing the delete. We get the string of the next row, append it to the current row and then remove the next row.

## The <kbd>Enter</kbd> key
The last editor operation we have to implement is the <kbd>Enter</kbd> key. The <kbd>Enter</kbd> key allows the user to insert new lines into the text, or split a line into two lines. The first thing to note is that unlike Delete and Backspace, <kbd>Enter</kbd> is passed to us as a char. You can actually add newlines that way right now, but as you might expect, the handling is less than optimal. This is because the newlines are inserted as part of the row instead of resulting in the creation of a new row.

Let's start with the easy case, adding a new row below the current one. We do this by adding the logic to the `Document` first.

```rust
    pub fn insert_newline(&mut self, at: &Position) {
        let new_row = Row::new();
        if at.y == self.len() -1  || at.y == self.len(){
            self.rows.push(new_row);
        } else if at.y < self.len()-1 {
            self.rows.insert(at.y+1, new_row)
        }
    }
  pub fn insert(&mut self, at: &Position, c: char){
        if c == '\n' {
           self.insert_newline(at);
           return;
        } 
        if at.y == self.len() {
            let mut row = Row::new();
            row.insert(0,c);
            self.rows.push(row);
        } else if at.y < self.len() {
            let row = self.rows.get_mut(at.y).unwrap();
            row.insert(at.x, c);
        }
    }

```
To make the code clearer, we have created a function called `insert_newline` which we are calling based on whether or not the character is a `\n`. This method is symmetrical to `insert`: We check if we are at the edge of the document or not, and are adding our new row at the appropriate place. Unlike in `insert`, we also account for the fact that the user can either push a new row when he is on the last line of the document, or one line below that, since we allow the user to start typing below the document.

We don't need to adjust the code in `Editor`. You can now add new lines, and if you happen to be at the end of the line, your cursor will also positioned correctly.

Let's now handle the case where we are in the middle of a row. We start by giving our row the possibility to return only parts of its string, and also the possibiltity to truncate the row to a certain width.

```rust
    pub fn to_string_range(&self, start: usize, end: usize) -> String {
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
                result.push(c);
        }
        result
    }
```

But wait - isn't that almost just a copy-paste of `render`? Let's try to combine these methods with the following method:

```rust
    fn render_with_renderer<R>(&self,start: usize, end: usize, renderer: R) -> String where R: Fn(char)->char {
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
            result.push(renderer(c));
        }
        result
    }
```
`render_with_renderer` is a function which accepts a function which is called a `renderer`. This function needs to accept a `char` and return a `char`, which is what we are defining after the `where` keyword. I recommend reading [the chapter on Closures](https://doc.rust-lang.org/1.1.0/book/closures.html) from the official docs if you are interested in why the function signature needs to look like this. Be warned, though, that to fully understand closures, you also need to understand [traits](https://doc.rust-lang.org/1.1.0/book/trait-objects.html), a concept that we haven't used in our code so far.

The idea of `render_with_renderer` is that there might be different representations of the internal string, and `render_with_renderer` is responsible for building the string without caring about the actual representation.

Let's rewrite our two existing methods then:

```rust
    pub fn to_string_range(&self, start: usize, end: usize) -> String {
        self.render_with_renderer(start, end, |c| c)
    }
 
    pub fn render(&self, start: usize, end: usize) -> String {
         self.render_with_renderer(start, end, |c| {
            if c == '\t' {
               ' '
            } else {
                c
            }
         })
    }
```
You can recognize closures by the pipe symbols. `|c| c` is similar to 

```rust
fn myClosure(c: char) -> char {
    c
}
```

As we will see later, it is not exactly a shorthand, since Closures enable us to do much more. For now, with this knowledge, you can see that `to_string_range` calls `render_with_renderer` with a renderer that doesn't do anything - it is returning the character handed to it. `render`, on the other hand, provides a renderer which returns a whitespace for a tab, or the character otherwise.

Now that we can return parts of the row, we also need a way to shorten a row to a certain value. We do that by implementing a `truncate` method:

```rust
    pub fn truncate(&mut self, width: usize) {
        if width > self.len {
            return;
        }
        let mut result = String::new();
        for character in self.string.chars().take(width) {
            result.push(character);
        }
        self.string = result;
        self.len = width;
    }
```

Now let's use that code from `document` by extending `insert_newline`.

```rust
    pub fn insert_newline(&mut self, at: &Position) {
        let mut new_row = Row::new();

        if at.y < self.len() {
            let current_row = self.rows.get_mut(at.y).unwrap();
            new_row.append(&current_row.to_string_range(at.x, current_row.len()));
            current_row.truncate(at.x);
        }  

        if at.y == self.len() -1  || at.y == self.len(){
            self.rows.push(new_row);
        } else if at.y < self.len()-1 {
            self.rows.insert(at.y+1, new_row)
        }
    }
```

Great! Now we can move around our document, add whitespaces, characters, even emojis, remove lines and so on. But editing is obviously useless without saving, so let's handle that next.

## Save to disk
Now that we’ve finally made text editable, let’s implement saving to disk. We stat with implementing a `save` method in `Document`:

```rust
    pub fn save(&self) -> Result<(), Error> {
        
        if let Some(file_name) = &self.file_name {
            let mut file = fs::File::create(file_name)?;
            for row in self.rows.iter() {
                file.write_all(row.to_string().as_bytes())?;
                file.write_all(b"\n")?;

            }
        } 
        Ok(())
    }
```
`write_all` takes a byte array, which in case of our rows we provide with `as_bytes`, and in case of the newline, we indicate by prefixing our String literal with `b`.

Now let's add that to our `Editor`:

```rust

    pub fn new() -> Result<Editor, std::io::Error> {
        let args: Vec<String> = env::args().collect();
        let mut file = None;
        let mut initial_status = String::from("HELP:  Ctrl-S = save | Ctrl-Q = quit");
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

//...
    fn process_keypress(&mut self) -> Result<(), std::io::Error> {
        let pressed_key = Terminal::read_key()?;
        match pressed_key {
             Key::Ctrl('q') => self.should_quit = true,
             Key::Ctrl('s') => {
                if let Ok(_) = self.document.save() {
                    self.status_message = StatusMessage::from("File saved successfully.".to_string());
                } else {
                    
                    self.status_message = StatusMessage::from("Error writing file!".to_string());
                }
             },
             Key::Char(c) => {
                self.document.insert(&self.cursor_position, c);
                self.move_cursor(&Key::Right);
             },
             Key::Delete => self.document.delete(&self.cursor_position),
             Key::Backspace => {
                 self.move_cursor(&Key::Left);
                 self.document.delete(&self.cursor_position);
             }
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

We also adjust our initial status message, to show the user which key combination to use to save. Great, now we can open, adjust and save our files!

## Save as...
Currently, when the user runs `hecto` with no arguments, they get a blank file to edit but have no way of saving. Let's make a `prompt()` function that displays a prompt in the status bar, and lets the user input a line of text after the prompt:

```rust
    fn prompt(&mut self, prompt: &str) -> Result<String, std::io::Error> {
        
        let mut result = String::new();
        loop {
            self.status_message = StatusMessage::from(format!("{}{}", prompt, result));
            self.refresh_screen()?;
            if let Key::Char(c) = Terminal::read_key()? {
                if c == '\n' {
                    self.status_message = StatusMessage::from(String::new());
                    break;
                }
                if !c.is_control() {
                    result.push(c);
                }
            }  
        }
        Ok(result)
    }
```

The user's input is stored in `result`, which we initialize as an empty string. We enter an infinite loop that repeatadly sets the status message, refreshes the screen, and waits for a keypress to handle. When the user presses enter, the status message is cleared and the message is returned. The errors which might occur on the way are propagated up. 
Now let's prompt the user for a filename before calling `save`, when the filename is `None`. 

```rust
    fn process_keypress(&mut self) -> Result<(), std::io::Error> {
        let pressed_key = Terminal::read_key()?;
        match pressed_key {
             Key::Ctrl('q') => self.should_quit = true,
             Key::Ctrl('s') => {
                if self.document.file_name == None {
                    self.document.file_name = Some(self.prompt("Save as: ")?);
                }
                if let Ok(_) = self.document.save() {
                    self.status_message = StatusMessage::from("File saved successfully.".to_string());
                } else {
                    self.status_message = StatusMessage::from("Error writing file!".to_string());
                }
             },
             Key::Char(c) => {
                self.document.insert(&self.cursor_position, c);
                self.move_cursor(&Key::Right);
             },
             Key::Delete => self.document.delete(&self.cursor_position),
             Key::Backspace => {
                 self.move_cursor(&Key::Left);
                 self.document.delete(&self.cursor_position);
             }
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

Great, now the user can save the file! Let's handle a few more cases in our prompt. Let's now allow the user to cancel and backspace, and let's also treat an empty input as cancelling.

```rust
    fn prompt(&mut self, prompt: &str) -> Result<Option<String>, std::io::Error> {
        
        let mut result = String::new();
        loop {
            self.status_message = StatusMessage::from(format!("{}{}", prompt, result));
            self.refresh_screen()?;
            match Terminal::read_key()?{
                Key::Backspace => {
                    if result.len() > 0 {
                        result.truncate(result.len()-1);
                    } else {
                        break;
                }
                Key::Char(c) => {
                    if c == '\n' {
                        break;
                    }
                    if !c.is_control() {
                        result.push(c);
                    }
                },
                Key::Esc =>{
                    result.truncate(0);
                    break;
                }
                _ => ()
            }
            
        }
        self.status_message = StatusMessage::from(String::new());
        if result.len() == 0 {
            return Ok(None);
        }
        Ok(Some(result))
    }
```
The biggest change to the function is the return type: It is no longer only a `Result` with a `String`, but a `Result` with an `Option` with a `String`, to indicate 1) that something could go wrong in this function, and 2) the result could be undefined.

We have moved setting an emtpy status message to outside of the loop, and we now return `None` if the result is emtpy - which can happen if the user has not entered anything. In case he presses `Esc`, we are also setting the result to `0`, thus also returning `None`. In case of backspace, we are either shortening the result by one character, or, if we had no result, we are also returning `None`, cancelling the prompt.

Now let's adjust our call to `prompt` in `process_keypress`:

```rust
    fn process_keypress(&mut self) -> Result<(), std::io::Error> {
        let pressed_key = Terminal::read_key()?;
        match pressed_key {
             Key::Ctrl('q') => self.should_quit = true,
             Key::Ctrl('s') => {
                if self.document.file_name == None {
                    let new_name = self.prompt("Save as: ")?;
                    let new_name = self.prompt("Save as: ")?;
                    if let None = new_name{
                        self.status_message = StatusMessage::from("Save aborted.".to_string());    
                        return Ok(())
                    } 
                    self.document.file_name = new_name;
                    
                }
                if let Ok(_) = self.document.save() {
                    self.status_message = StatusMessage::from("File saved successfully.".to_string());
                } else {
                    self.status_message = StatusMessage::from("Error writing file!".to_string());
                }
             },
             Key::Char(c) => {
                self.document.insert(&self.cursor_position, c);
                self.move_cursor(&Key::Right);
             },
             Key::Delete => self.document.delete(&self.cursor_position),
             Key::Backspace => {
                 self.move_cursor(&Key::Left);
                 self.document.delete(&self.cursor_position);
             }
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

We are first unwrapping the `Result` with the `?`, passing any potential error up. Then we are looking at `new_name`. If it's `None`, we are displaying a message and returning. If not, we are assigning it to `file_name` and continue. `save` will then use that new file name to try and save the file.

## Dirty flag

We'd like to keep track of whether the text loaded in our editor differs from what's in the file. Then we can warn the user that they might lose unsaved changes when they try to quit.

We call a `Document` "dirty" if it has been modified since opening or saving the file. Let's add a `dirty` variable to the `Document` and initialize it with `false`. We don't want this to be modified from the outside, so we add a read-only `is_dirty` function to `Document`. We're also setting it to `true` on any text change, and to `false` on `save`.

```rust
  pub fn new() -> Document{
        Document{
            rows: Vec::new(),
            file_name: None,
            dirty: false
        }
    }
    pub fn is_dirty(&self) -> bool {
        self.dirty
    }
    pub fn open(file_name: &String ) -> Result<Document, Error> {
        let contents = fs::read_to_string(file_name)?;
        let mut rows = Vec::new();
        for value in contents.lines() {
            rows.push(Row::from(value));
        }
        Ok(Document{
            rows,
            file_name: Some(file_name.clone()),
            dirty: false,
        })
    }
    pub fn save(&mut self) -> Result<(), Error> {
        if let Some(file_name) = &self.file_name {
            let mut file = fs::File::create(file_name)?;
            for row in self.rows.iter() {
                file.write_all(row.to_string().as_bytes())?;
                file.write_all(b"\n")?;
            }
            self.dirty = false;
        } else {
            return Err(Error::new(ErrorKind::Other, "Can't save file!"))
        }
        Ok(())
    }
     pub fn insert(&mut self, at: &Position, c: char){
        self.dirty = true;
        if c == '\n' {
           self.insert_newline(at);
           return;
        } 

        if at.y == self.len() {
            let mut row = Row::new();
            row.insert(0,c);
            self.rows.push(row);
        } else if at.y < self.len() {
            let row = self.rows.get_mut(at.y).unwrap();
            row.insert(at.x, c);
        }
    }
    pub fn delete(&mut self, at: &Position) {
        let len = self.len();
        if at.y >= len {
            return;
        }
        self.dirty = true;
        let row = self.rows.get(at.y).unwrap();
        if at.x == row.len() && at.y < len -1 {
            let next_row = self.rows.get(at.y + 1).unwrap();
            let next_string = next_row.to_string();
            let row = self.rows.get_mut(at.y).unwrap();
            row.append(&next_string);
            self.rows.remove(at.y+1);
            
        } else {
            let row = self.rows.get_mut(at.y).unwrap();
            row.delete(at.x); 
        }
        
    }
```


Now that everything is in place, let's display it.

```rust
 fn draw_status_bar(&self) {
        let mut status;
        let width = self.terminal.size().width as usize;
        let mut modified_indicator = "";
        if self.document.is_dirty()  == true {
            modified_indicator = " (modified)";
        }
        if let Some(name) = &self.document.file_name {
            let mut filename_len = name.len();
            if filename_len > 20 {
                filename_len = 20;
            }
            status = format!("{} - {} lines{}", &name[..filename_len], self.document.len(), modified_indicator);
        } else {
            status = format!("[No Name]{}", modified_indicator);
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

You should now be able to confirm that as soon as you edit a file, `(modified)` is displayed next to the file name. It disappears on successful saving of the file.

## Quit confirmation

Now we’re ready to warn the user about unsaved changes when they try to quit. If `document.is_dirty()`  is true, we will display a warning in the status bar, and require the user to press Ctrl-Q three more times in order to quit without saving.

```rust
    fn process_keypress(&mut self) -> Result<(), std::io::Error> {
        let pressed_key = Terminal::read_key()?;
        match pressed_key {
             Key::Ctrl('q') => {
                 if self.quit_times > 0 && self.document.is_dirty(){
                    self.status_message = StatusMessage::from(format!("WARNING! File has unsaved changes. Press Ctrl-Q {} more times to quit.", self.quit_times));;
                    self.quit_times -=1;
                     return Ok(());
                 }
                 self.should_quit = true
             },
             Key::Ctrl('s') => {
                if self.document.file_name == None {
                    let new_name = self.prompt("Save as: ")?;
                    if let None = new_name{
                        self.status_message = StatusMessage::from("Save aborted.".to_string());    
                        return Ok(())
                    } 
                    self.document.file_name = new_name;
                    
                }
                if let Ok(_) = self.document.save() {
                    self.status_message = StatusMessage::from("File saved successfully.".to_string());
                } else {
                    self.status_message = StatusMessage::from("Error writing file!".to_string());
                }
             },
             Key::Char(c) => {
                self.document.insert(&self.cursor_position, c);
                self.move_cursor(&Key::Right);
             },
             Key::Delete => self.document.delete(&self.cursor_position),
             Key::Backspace => {
                 self.move_cursor(&Key::Left);
                 self.document.delete(&self.cursor_position);
             }
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
        if self.quit_times < QUIT_TIMES {
            self.quit_times = QUIT_TIMES;
            self.status_message = StatusMessage::from(String::new());
        }
        
        Ok(())
    }
```

## Conclusion
You have now successfully built a text editor. If you are brave, you can use `hecto` to work on `hecto`. In the next chapter, we will make use of `prompt()` to implement an incremental search feature in our editor.