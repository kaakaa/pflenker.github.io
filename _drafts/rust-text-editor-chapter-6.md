---
layout: post
title: "Hecto, Chapter 6: Search"
categories: [Rust, hecto]
---
Our text editor is done - we can open, edit and save files. The upcoming two features add more functionality to it. In this chapter, we will implement a minimal search feature.

For that, we use `prompt`. When the user types a search query and presses <kbd>Enter</kbd>, we'll loop through all the rows of our file, and if a row contains their query string, we'll move the cursor to match. For that, we are going to need a method which searchs a single `Row` and returns the position of the match. Let's start with that now.

```rust
   pub fn find(&self, query: String) -> Option<usize> {
        self.string.find(&query[..])
    }
```

Since `find` takes a `&str` as an argument, we convert `query` to a `&str` by slicing it. Omitting the start and the end index of `query` slices the whole string. 

But wait - did you notice the mistake we just made? We are returning the _byte index_ of the match. As we saw earlier, this does not necessarily correspond to the position of the first character, so we might be returning a wrong position here. We need to convert our byte index to the character index.

```rust
    pub fn find(&self, query: String) -> Option<usize> {
       let byte_index = self.string.find(&query[..]);
       if let Some(byte_index) = byte_index {

        let mut index = 0;
        for (i, _) in self.string.char_indices() {
            if i == byte_index {
                return Some(index);
            }
            index += 1;
        }
       }
       None
    }
```


Let's call that method from `Document`:

```rust
    pub fn find(&self, query: String) -> Option<Position> {
        let mut position = Position{ x: 0, y: 0};
        for (i, row) in self.rows.iter().enumerate() {
            position.y = i;
            if let Some(x) = row.find(&query) {
                position.x = x;
                return Some(position);
            }
        }
        None
    }
```

We are building up our `Position` by setting `y` to the value of the row we are currently investigating. If we find a match, we return the position of the match as `x` and the current row as `y`. If there is no match at all, we return `None`.

Let's add that to the `Editor`, and adjust our help message along the way:

```rust
        let mut initial_status = String::from("HELP:  Ctrl-F = find | Ctrl-S = save | Ctrl-Q = quit");
        //
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
             Key::Ctrl('f') => {
                 if let Some(query) = self.prompt("Search: ")? { 
                     if let Some(position) = self.document.find(query) {
                         self.cursor_position = position;
                     }
                 }
                 
             }
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
```
This code is familiar from our implementation of the save dialog.
