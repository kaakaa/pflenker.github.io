---
layout: post
title: "Hecto, Chapter 7: Syntax Highlighting"
categories: [Rust, hecto]
---
We are almost done with our text editor - we're only missing some syntax highlighting.

## Colorful Digits
Let's start by just getting some color on the screen, as simply as possible. We'll attempt to highlight numbers by coloring each digit character red. 

```rust
    pub fn to_string_range(&self, start: usize, end: usize) -> String {
        self.render_with_renderer(start, end, |c| c.to_string())
    }
 
    pub fn render(&self, start: usize, end: usize) -> String {
         self.render_with_renderer(start, end, |c| {
            if c == '\t' {
               ' '.to_string()
            } else if let Some(_) = c.to_digit(10) {
                format!("{}{}{}",termion::color::Fg(color::Rgb(220,163,163)),c,color::Fg(color::Reset))
            } else {
                c.to_string()
            }
         })
    }
    fn render_with_renderer<R>(&self,start: usize, end: usize, renderer: R) -> String  where R: Fn(char)->String {
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
            result = format!("{}{}", result, renderer(c));
        }
        result
    }
```

We have refactored the callback of `render_with_renderer` to return a `String` instead of a character, to allow any transformation during rendering where we can add invisible characters, such as coloring, to the output. We have also adjusted `render` to check if the current character is a digit, and we are coloring said digit if it is.

## Refactor syntax highlighting
Now we know how to color text, but we’re going to have to do a lot more work to actually highlight entire strings, keywords, comments, and so on. We can’t just decide what color to use based on the class of each character, like we’re doing with digits currently. What we want to do is figure out the highlighting for each row of text before we display it, and then rehighlight a line whenever it gets changed. What makes things more complicated is that the highlighting depends on characters currently out of view - for instance, if a `String` starts to the left of the currently visible portion of the row, we want to treat a `"` on screen as the end of a string, and not as the start of one. Our current strategy to look at each visible character is therefore not sufficient.

Instead, we are going to store the highlighting of each character of a row in a vector. Let's start by adding an enum which will hold our different highlighting types as well as a vector to hold them.

```rust
enum HighlightingType {
    None,
    Number
}

pub struct Row {
    string: String,
    highlighting: Vec<HighlightingType>,
    len: usize
}

impl Row {
    pub fn new() -> Row {
        Row {
            string: String::new(),
            highlighting: Vec::new(),
            len: 0
        }
    }
    pub fn from(slice: &str) -> Row{
        let string = String::from(slice);
        Row {
            len: string.chars().count(),
            highlighting: Vec::new(),
            string
        }
    }
    //...
}
```
For now, we’ll focus on highlighting numbers only. So we want every character that’s part of a number to have a corresponding `HighlightingType::Number` value in the `highlighting` vector, and we want every other value in `highlighting` to be `HighlightingType::None`.

Let's create a new `highlight` function in our row. This function will go through the characters of the `string` and highlight them by setting each value in the `highlighting` vector.

```rust
    pub fn from(slice: &str) -> Row{
        let string = String::from(slice);
        let mut row = Row {
            len: string.chars().count(),
            highlighting: Vec::new(),
            string
        };
        row.highlight();
        row
    }
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
        self.highlight();
    }
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
        self.highlight();
    }
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
        self.highlight();
    }
      pub fn append(&mut self, string: &String) {
        self.string = format!("{}{}", self.string, string);
        self.len = self.string.chars().count();
        self.highlight();
    }
     fn highlight(&mut self) {
        let mut highlighting = Vec::new();
        for c in self.string.chars() {
            if let Some(_) = c.to_digit(10) {
                highlighting.push(HighlightingType::Number);
            } else {
                highlighting.push(HighlightingType::None);
            }
        }
        self.highlighting = highlighting;
    }
```
The code of `highlight` is straightforward: If a character is a digit, we push `HighlightingType::Number`, otherwise, we push `HighlightingType::None`. We now have an array which has the same length as `self.string.chars()`. We call `highlight` on every row construction and modification, to make sure our highlighting stays in synch with the changes.

Now we want to have a function which returns the color we want to use for the actual highlighting for an enum. We can implement that directly on our enum:

```rust
impl HighlightingType {
    fn to_color(&self) -> impl color::Color {
        match self {
            HighlightingType::Number => color::Rgb(220,163,163),
            _ => color::Rgb(255,255,255),
        }
    }
}
```
We are returning red now for numbers and white for all other cases. Now let's finally draw the highlighted text to the screen!

```rust
pub fn to_string_range(&self, start: usize, end: usize) -> String {
        self.render_with_renderer(start, end, |c,_| c.to_string())
    }
 
    pub fn render(&self, start: usize, end: usize) -> String {
         self.render_with_renderer(start, end, |c, index| {
             let  highlighting =  self.highlighting.get(index).unwrap_or(&HighlightingType::None);                
            format!("{}{}{}",termion::color::Fg(highlighting.to_color()),c,color::Fg(color::Reset))
         })
    }
    fn render_with_renderer<R>(&self,start: usize, end: usize, renderer: R) -> String  where R: Fn(char, usize)->String {
        let mut end = end;
        let mut start = start;
        if end > self.string.len() {
            end = self.string.len();
        }
        if start > end {
            start = end;
        }
        let mut result = String::new();
        let mut index = start;
        for c in self.string.chars().skip(start).take(end - start) {
            result = format!("{}{}", result, renderer(c, index));
            index +=1;
        }
        result
    }
```
First, we are changing `render_with_renderer` to return the current index to the closure. We use that in `render` to get the current highlight and apply it. Even though we are pretty sure that `get` will always return the right value to us, we are using `unwrap_or`, which returns the value in case there is one, or a default value, in this case `HighlightingType::None`, if not. 

This works, but do we really have to write out an escape sequence before every single character? In practice, most characters are going to be the same color as the previous character, so most of the escape sequences are redundant. Let’s keep track of the current text color as we loop through the characters, and only print out an escape sequence when the color changes.

```rust
    pub fn render(&self, start: usize, end: usize) -> String {
        let mut current_highlighting = &HighlightingType::None;
        let rendered_string = self.render_with_renderer(start, end, |c, index| {
            let highlighting =  self.highlighting.get(index).unwrap_or(&HighlightingType::None); 
            let mut start ;
            if highlighting != current_highlighting {
                current_highlighting = highlighting;
                start = color::Fg(highlighting.to_color()).to_string();
            } else {
                start = String::new();
            }
            format!("{}{}",start,c)
         });
         
         format!("{}{}", rendered_string, color::Fg(color::Reset))
    }

```
We use `current_highlighting` to keep track of what we are currently rendering. We needed to change the signature of `render_with_renderer` to take an `FnMut` callback to be able to capture `current_highlighting` in the closure.  We are also resetting the foreground color after rendering, to make sure we are not messing up the screen for everything that comes after us.

## Colorful search results

Before we start highlighting strings and keywords and all that, let's use our highlighting system to highlight search results. We'll start by adding `Match` to the `HighlightingType` enum, and mapping it to the color blue in `to_color`. 

```rust
#[derive(PartialEq)]
enum HighlightingType {
    None,
    Number,
    Match
}

impl HighlightingType {
    fn to_color(&self) -> impl color::Color {
        match self {
            HighlightingType::Number => color::Rgb(220,163,163),
            HighlightingType::Match => color::Rgb(38,139,210),
            _ => color::Rgb(255,255,255),
        }
    }
}
```

Next, we want to add a method called `highlight_string` to our row, which accepts an optional word. If no word is given, the original highlighting is restored.

```rust
    pub fn highlight_word(&mut self, word: Option<&String>) {
        self.highlight();
        if let Some(word) = word {
            let length = word.chars().count();
            if length == 0 {
                return;
            }
            let mut search_index = 0;
            loop {
                if let Some(next_match) = self.find(word, search_index, SearchDirection::Forward){
                    for i in next_match..next_match + length {
                        if i < self.highlighting.len() {
                            self.highlighting[i] = HighlightingType::Match;
                        }
                    }  
                    search_index = next_match + 1;
                    if search_index > self.len() {
                        break;
                    }
                } else {
                    break;
                }


            }
        } 

    }
```

We start by resetting the highlighting, to avoid effects where the previous highlighting stays visible (for example, when the user is changing his search query). Then, we are getting the length of the word. If it is an empty string, there is nothing to do. Otherwise, we use `self.find` to find all occurences of the word in the current row. If we have a match, we set the whole word to `HighlightingType::Match` by using the position of the match as the starting point, and the position of the end of the match, calculated by `next_match + length`, as the end. Then we advance the cursor of our search by one and continue the search. We break the loop if either we have no match left, or our search index grows too large.

Now let's call this code from the 'Document' and the 'Editor'.

```rust
    pub fn highlight_word(&mut self, word: Option<&String>) {
        for row in self.rows.iter_mut() {
            row.highlight_word(word);
        }
    }
```

```rust

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
                     editor.document.highlight_word(Some(query)); 
                   
                 })?;
                 if query == None {
                     self.cursor_position = old_position;
                     self.scroll();
                 }
                 self.document.highlight_word(None);
                 
                 
             }
```

The `document` simply calls `highlight_word` on all its rows. In the `Editor`, we highlight the word within the searrch closure, and we reset the highlighting after the search has been done.

Try it out, and you will see that all search results light up in your terminal.


