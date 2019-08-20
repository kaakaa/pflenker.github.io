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
                         self.scroll();
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


## Incremental search

Now, let's make our search feature fancy. We want to support incremental search, meaning the file is searched after each keypress when the user is typing in their search query. 

To implement this, we're going to get `prompt` to take a callback function as an argument. We'll have to call this function after each keypress, passing the current search query inputted by the user and the last key they pressed.

```rust
    fn prompt<R>(&mut self, prompt: &str, callback: R) -> Result<Option<String>, std::io::Error>
        where R: Fn(Key, &String) {
        
        let mut result = String::new();
        loop {
            self.status_message = StatusMessage::from(format!("{}{}", prompt, result));
            self.refresh_screen()?;
            let key = Terminal::read_key()?;
            match key {
                Key::Backspace => {
                    if result.len() > 0 {
                        result.truncate(result.len()-1);
                    } else {
                        break;
                    }
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
            callback(key, &result)
            
        }
        self.status_message = StatusMessage::from(String::new());
        if result.len() == 0 {
            return Ok(None);
        }
        Ok(Some(result))
    }

    //
    let new_name = self.prompt("Save as: ", |_,_|{})?;
    //
    if let Some(query) = self.prompt("Search: ", |_,_|{})? {                  

```

The syntax of creating a function with a callback argument should be familiar from the previous chapter. We have also added a call to `callback` everytime we register a key press. We have also adjusted our previous calls to `prompt`. It looks a bit strange now, but essentially, `|_,_|{}` means: A closure with two parameters, which we both ignore and in which we do nothing. Let's start doing something during search now!

```rust
             Key::Ctrl('f') => {
                 self.prompt("Search: ", |_ ,query|{
                     if let Some(position) = self.document.find(&query) {
                         self.cursor_position = position;
                         self..scroll();
                     }
                 })?;
                 
                 
             }
```

However, if we try to do it like this, we run into an error. The compiler complains about multiple things, but it all boils down to one thing: We can't mutate `self` in the closure. Even if we had a mutable reference to it (which we haven't), we couldn't! 

The reason is Rusts's borrowing mechanism, which only allows one mutable reference to something at a time. Essentially, this prevents bugs like this: You have reference A and reference B. You delete an object using reference A. Reference B is now invalid, interacting with it would cause an error.

In this case, however, it is not easy to see where those two mutable references come from, because both are hidden. One comes from the Closure: A Closure "captures" the variables referenced within, so by mutating `self` in the closure, we make need a mutable reference. The other reference is hidden in `self.prompt`. As we saw earlier, `self.prompt` is roughly equivalent to calling a function with a signature like `fn prompt(editor: &Editor ...)`. And when we look at the signature of our `prompt`, we can actually see that it requires `self` to be mutable.

We can fix this easily by passing the reference to `self` to the closure.

```rust
    fn prompt<R>(&mut self, prompt: &str, callback: R) -> Result<Option<String>, std::io::Error>
        where R: Fn(&mut Editor, Key, &String) {
        
        let mut result = String::new();
        loop {
            self.status_message = StatusMessage::from(format!("{}{}", prompt, result));
            self.refresh_screen()?;
            let key = Terminal::read_key()?;
            match key {
                Key::Backspace => {
                    if result.len() > 0 {
                        result.truncate(result.len()-1);
                    } else {
                        break;
                    }
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
            callback(self, key, &result);
            
        }
        self.status_message = StatusMessage::from(String::new());
        if result.len() == 0 {
            return Ok(None);
        }
        Ok(Some(result))
    }

    //...

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
                    let new_name = self.prompt("Save as: ", |_,_,_|{})?;
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
                 self.prompt("Search: ", |editor, _ ,query|{
                     if let Some(position) = editor.document.find(&query) {
                         editor.cursor_position = position;
                         editor.scroll();
                     }
                 })?;
                 
                 
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

That’s all there is to it. We now have incremental search.

## Restore cursor position when cancelling search
If the user cancels his search, we want to reset the cursor to the old position. For that, we are going to save the position before we start the `prompt`, and we check the return value. If it's `None´, the user has cancelled his query, and we want to restore his old cursor position and scroll to it.

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
                    let new_name = self.prompt("Save as: ", |_,_,_|{})?;
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
                 let old_position = Position {
                     ..self.cursor_position
                 };
                 let query = self.prompt("Search: ", |editor, _ ,query|{
                     if let Some(position) = editor.document.find(&query) {
                         editor.cursor_position = position;
                         editor.scroll();
                     }
                 })?;
                 if query == None {
                     self.cursor_position = old_position;
                     self.scroll();
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
        self.scroll();
        if self.quit_times < QUIT_TIMES {
            self.quit_times = QUIT_TIMES;
            self.status_message = StatusMessage::from(String::new());
        }
        
        Ok(())
    }
```

We use the [Struct Update Syntax](https://doc.rust-lang.org/book/ch05-01-defining-structs.html#creating-instances-from-other-instances-with-struct-update-syntax) to create `old_position` from the values of `cursor_position`. 

## Search forward and backward

The last feature we'd like to add is to allow th euser to advance to the next or previous match in the file using the arrow keys. 
The <kbd>&uarr;</kbd> and <kbd>&larr;</kbd> keys will go to the previous match, and the <kbd>&darr;</kbd> and <kbd>&rarr;</kbd> keys will go to the next match.

We start by accepting a `start` position in our `find` methods, indicating that we want to search the next match after that position.

```rust
    pub fn find(&self, query: &String, after: usize) -> Option<usize> {
       if after >= self.len() {
           return None;
       }
       let byte_index = self.to_string_range(after, self.len()).find(&query[..]);
       if let Some(byte_index) = byte_index {
        let mut index = 0;
        for (i, _) in self.string.char_indices() {
            if i == byte_index {
                return Some(index + after);
            }
            index += 1;
        }
       }
       None

    }
```
We are returning `index + after` instead of `after` on the success case, because `index` is the index relative to the shortened string. `find`, however, should return the absolute position of the index in the string.

Next, we implement the functionality in `Document`:
```rust
    pub fn find(&self, query: &String, after: &Position) -> Option<Position> {
        let mut position = Position{x: after.x, y: after.y};
        for row in self.rows.iter().skip(position.y) {
            if let Some(x) = row.find(&query, position.x) {
                position.x = x;
                return Some(position);
            } 
            position.x = 0;
            position.y +=1;
        }
        None
    }
```
After we have processed the first row in our `for..in` statement without a mathc, we are resetting `x` to `0` to ensure we start searching the next row on the left. Now, let's look at the changes to `Editor`. 

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
                    let new_name = self.prompt("Save as: ", |_,_,_|{})?;
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
                 let old_position = Position {
                     ..self.cursor_position
                 };
                 let query = self.prompt("Search (ESC to cancel, Arrows to navigate): ", |editor, key, query|{
                     let mut moved = false;
                     match  key {
                         Key::Right | Key::Down  => {
                             editor.move_cursor(&Key::Right);
                             moved = true;
                         }
                         _ => ()
                     }
                     if let Some(position) = editor.document.find(&query, &editor.cursor_position) {
                        editor.cursor_position = position;
                        editor.scroll();
                     }  else if moved == true {
                         editor.move_cursor(&Key::Left);
                     } 
                   
                 })?;
                 if query == None {
                     self.cursor_position = old_position;
                     self.scroll();
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
        self.scroll();
        if self.quit_times < QUIT_TIMES {
            self.quit_times = QUIT_TIMES;
            self.status_message = StatusMessage::from(String::new());
        }
        
        Ok(())
    }
```

We have adjusted our error message and are calling `find` now with the cursor position. We are also now using the formerly unused `key` parameter of our closure and match against the user input. If the user presses down or right, we are moving the cursor to the right as well before continuing the search with the updated parameter. Why? Because if our cursor is already at the position of a search match, we don't want the current position to be returned to us, so we start one step to the left of the current position.  
In case our find did not return a match, we want the cursor to stay on its previous position. So we track if we have moved the cursor in `moved` and move it back in case there is no result.

Now let's take a look at searching backwards. To do that, we are going to create an Enum which tells us in which direction we want to search. We place that on top of `editor`:

```rust
#[derive(PartialEq, Copy, Clone)]
pub enum SearchDirection {
   Forward,
   Backward
}
```
Here, we are using something called a Trait Annotation. On very simple structures, like this enum, certain traits can be derived automatically. We are interested in `Copy`, so that we can pass our enum as a parameter without needing to worry about manually copying it or passing references around. To use `Copy`, we also have to use `Clone`, which is used under the hood. Last but not least, we are interested in comparing `enum` entries, so we derive `PartialEq`. Without that trait, something like `if foo == SearchDirection::Forward` would not be possible.

Let's start using this enum in our rows first.

```rust
 pub fn find(&self, query: &String, at: usize, direction: SearchDirection) -> Option<usize> {
       let mut start = 0;
       let mut end = self.len();
       if direction == SearchDirection::Forward {
           start = at;
       } else {
           end = at;
       }
       
       let range = self.to_string_range(start, end);       
       let byte_index;
       if direction == SearchDirection::Forward {
           byte_index = range.find(&query[..]);
       } else {
           byte_index = range.rfind(&query[..]);
       }

       if let Some(byte_index) = byte_index {
        let mut index = 0;
        for (i, _) in self.string.char_indices() {
            if i == byte_index {
                 return Some(index + start);
            }
            index += 1;
        }
       }
       None

    }
```

We are using `direction` to determine whether we are looking at `0..at` (for backward searches) or at `at..len` (for forward searches). We use it also to determine whether or not to use `find` or `rfind`. `rfind` is searching from the back, so if we have a string `foo bar baz` and we search for `ba`, we want the index of `baz` and not of `bar` to be returned to us.

Let's call this logic from our document.
```rust
    pub fn find(&self, query: &String, at: &Position, direction: SearchDirection) -> Option<Position> {
        let mut position = Position{x: at.x, y: at.y};
        let mut start = 0;
        let mut end = self.len();
        if direction == SearchDirection::Forward {
            start = position.y;
        } else {
            end = position.y;
        }
        for _ in start..end {
            let row = &self.rows[position.y];
            if let Some(x) = row.find(&query, position.x, direction) {
                position.x = x;
                return Some(position);
            } 
            if direction == SearchDirection::Forward{
                position.y +=1; 
                position.x = 0;
            } else {
                position.y -=1;
                position.x = self.rows[position.y].len();
            } 
    
            
        }
        None
    }
```

We have now built our own iterator instead of using `iter`. Similar to what we did in `row`, we use the direction to set whether we are looking at `0..current row` or at `current row..end of file`  Then we get the current row, search in it and return the position if we have a match. If not, depending on our direction we either advance one row and set our search cursor, `x`, to the left, or we go one row up and set our search cursor to the far right.

Now let's call it from the `editor`.

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
                    let new_name = self.prompt("Save as: ", |_,_,_|{})?;
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
                 let old_position = Position {
                     ..self.cursor_position
                 };
                 let mut direction = SearchDirection::Forward;
                 let query = self.prompt("Search (ESC to cancel, Arrows to navigate): ", |editor, key, query|{
                     let mut moved = false;
                     match  key {
                         Key::Right | Key::Down  => {
                             direction = SearchDirection::Forward;
                             editor.move_cursor(&Key::Right);
                             moved = true;
                         },
                         Key::Left | Key::Up => direction = SearchDirection::Backward,
                         _ => direction = SearchDirection::Forward
                     }
                     if let Some(position) = editor.document.find(&query, &editor.cursor_position, direction) {
                        editor.cursor_position = position;
                        editor.scroll();
                     }  else if moved == true {
                         editor.move_cursor(&Key::Left);
                     } 
                   
                 })?;
                 if query == None {
                     self.cursor_position = old_position;
                     self.scroll();
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
        self.scroll();
        if self.quit_times < QUIT_TIMES {
            self.quit_times = QUIT_TIMES;
            self.status_message = StatusMessage::from(String::new());
        }
        
        Ok(())
    }
```

Note here that we had to change the signature of our callback to `FnMut` to allow mutating the search direction from within the closure. Then we use the keys Left and Up to change the search direction to backward. Note that we set the direction to forward again on any other keypress. If we wouldn't do that, the search behaviour would be weird: Let's say the cursor is on `baz` of `foo bar baz`, the user has entered `b` and the Search Direction is backward. If you typed `a`, the query would be `ba` and the user would jump back to `bar` instead of staying on `baz`, even though `baz` also matches the query.

Congratulation, our search feature works now!

## Conclusion
Since our functionality is now complete, we used this chapter again to focus a bit more on Rust topics and brushed the topic of Closures and Traits. Our editor is almost complete - in the next chapter, we are going to implement Syntax Highlighting and File Type Detection, to complete our text editor.