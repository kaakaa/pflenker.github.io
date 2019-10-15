---
layout: postwithdiff
title: "Hecto, Chapter 7: Syntax Highlighting"
categories: [Rust, hecto]
---
We are almost done with our text editor - we're only missing some syntax highlighting.

## Colorful Digits
Let's start by just getting some color on the screen, as simply as possible. We'll attempt to highlight numbers by coloring each digit character red.

{% include hecto/simple-number-highlighting.html %}

<small>[See this step on github](https://github.com/pflenker/hecto-tutorial/releases/tag/simple-number-highlighting)</small>

We have now converted our grapheme into a character, so that we can use `is_ascii_digit` - which determines whether or not a character is a digit. If it is, we change the foreground color to [a shade of red](https://rgb.to/rgb/220,163,205) and reset it immediately afterwards.

## Refactor syntax highlighting
Now we know how to color text, but we’re going to have to do a lot more work to actually highlight entire strings, keywords, comments, and so on. We can’t just decide what color to use based on the class of each character, like we’re doing with digits currently. What we want to do is figure out the highlighting for each row of text before we display it, and then rehighlight a line whenever it gets changed. What makes things more complicated is that the highlighting depends on characters currently out of view - for instance, if a `String` starts to the left of the currently visible portion of the row, we want to treat a `"` on screen as the end of a string, and not as the start of one. Our current strategy to look at each visible character is therefore not sufficient.

Instead, we are going to store the highlighting of each character of a row in a vector. Let's start by adding an enum which will hold our different highlighting types as well as a vector to hold them.

{% include hecto/highlighting-type.html %}

<small>[See this step on github](https://github.com/pflenker/hecto-tutorial/releases/tag/highlighting-type)</small>

Highlighting will be controlled by the `Document`, the rows do not "highlight themselves". The reason will become clearer as we add more code - essentially, to highlight a row, more information than what is present in the `Row` is needed. This means that any operation on the `Row` from the outside can potentially render the highlighting invalid. When developing functionality for `Row`, we are going to pay special attention so that the worst that can happen is a wrong highlighting (as opposed to a crash).

This is why we are not setting the highlight for example in `split`.

For now, we’ll focus on highlighting numbers only. So we want every character that’s part of a number to have a corresponding `Type::Number` value in the `highlighting` vector, and we want every other value in `highlighting` to be `Type::None`.

Let's create a new `highlight` function in our row. This function will go through the characters of the `string` and highlight them by setting each value in the `highlighting` vector.

{% include hecto/add-highlighting-to-row.html %}

<small>[See this step on github](https://github.com/pflenker/hecto-tutorial/releases/tag/add-highlighting-to-row)</small>

The code of `highlight` is straightforward: If a character is a digit, we push `Type::Number`, otherwise, we push `Type::None`.

Now we want to have a function which maps the `Type` to a color. Let's implement that as a method of `Type`:

{% include hecto/color-mapping.html %}

<small>[See this step on github](https://github.com/pflenker/hecto-tutorial/releases/tag/color-mapping)</small>

We are returning red now for numbers and white for all other cases. Now let's finally draw the highlighted text to the screen!

{% include hecto/apply-highlighting.html %}

<small>[See this step on github](https://github.com/pflenker/hecto-tutorial/releases/tag/apply-highlighting)</small>

First, we call `highlight` everywhere in `Document` where we modify a row. Then, we refactor our rendering: We get the correct highlighting for the current index (which we obtain by using `enumerate`). We change the color, append to the string which we return, and change the color back again.

This works, but do we really have to write out an escape sequence before every single character? In practice, most characters are going to be the same color as the previous character, so most of the escape sequences are redundant. Let’s keep track of the current text color as we loop through the characters, and only print out an escape sequence when the color changes.

{% include hecto/improve-highlighting.html %}

<small>[See this step on github](https://github.com/pflenker/hecto-tutorial/releases/tag/improve-highlighting)</small>

We use `current_highlighting` to keep track of what we are currently rendering. When it changes, we add the color change to the render string. We have also moved the ending of the highlighting outside of the loop, so that we reset the color at the end of each line. To allow comparison between `Type`s, we derive `PartialEq` again.

## Colorful search results

Before we start highlighting strings and keywords and all that, let's use our highlighting system to highlight search results. We'll start by adding `Match` to the `HighlightingType` enum, and mapping it to the color blue in `to_color`.
{% include hecto/add-match-type.html %}

<small>[See this step on github](https://github.com/pflenker/hecto-tutorial/releases/tag/add-match-type)</small>


Next, we want to change `highlight` so that it accepts an optional word. If no word is given, no match is highlighted.

{% include hecto/highlight-matches.html %}

<small>[See this step on github](https://github.com/pflenker/hecto-tutorial/releases/tag/highlight-matches)</small>

Before we investigate the changes in `highlight`, let's first focus on the other changes: We allow an optional parameter to `highlight`, which we provide as `None` everywhere we call `highlight`. We then add a method called `highlight` to the document, which triggers a highlighting of all rows in the doc.

We use that method during our search by passing the current query in, and `None` as soon as the search has been aborted.

Now, what about `highlight`?

First, we collect all the matches of the word we have in the current row. That is more efficient than performing the search on every character.

We are using two new concepts while doing so. One is `while..let`, which is similar to `if..let`, but it loops as long as the condition is satisifed. We use that to loop through our current row, advancing the starting point for our search directly behind the last match on every turn.

We are also using a new method to add `1`: `checked_add`. This function returns an `Option` with the result if no overflow occured, or `None` otherwise. Why can't we use `saturating_add` here? 
Well, let's assume our match happens to be the very last part of the row, and we are also at the end of `usize`. Then we can't advance `next_index` any further with `saturading_add`, it would return the same result over and over, the `while` condition would never be false and we would loop indefinitely.

Speaking of loops, we also changed our loop for processing characters. This allows us to consume multiple characters at a time. We use this to push multiple highlighting types at once while we are at the index of a match which we want to highlight.

Try it out, and you will see that all search results light up in your editor.


## Colorful numbers
Alright, let's start working on highlighting numbers properly.

Right now, nubers are highlighted even if they're part of an identifier, such as the 32 in `u32`. To fix that, we'll require that numbers are preceded by a separator character, which includes whitespace or punctuation characters. We can use `is_ascii_punctuation` and `is_ascii_whitespace` for that.

{% include hecto/highlight-numbers-after-separator.html %}

<small>[See this step on github](https://github.com/pflenker/hecto-tutorial/releases/tag/highlight-numbers-after-separator)</small>

We're adding a new variable `prev_separator` to check if the last character we saw was a valid separator, after which we want to  highlight numbers properly, or any other character, after which we don't want to highlight numbers.

We are also accessing the previous highlighting so that even if we are not behind a separator, we continue highlighting numbers as numbers - which allows us to highlight numbers with more than one digit.

At the end of the loop, we set `prev_is_separator` to `true` if the current character is either an ascii punctiation or a whitespace, otherwise it's set to `false`. Then we increment `index` to consume the character.

Now let's support highlighting numbers that contain decimal points.

{% include hecto/highlight-dot-in-numbers.html %}

<small>[See this step on github](https://github.com/pflenker/hecto-tutorial/releases/tag/highlight-dot-in-numbers)</small>

A `.` character that comes after a character that we just highlighted as a number will now be considered part of the number.

## Detect file type
Before we go on to highlight other things, we're going to add filetype detection to our editor. This will allow us to have different rules for how to highlight different types of files. For example, text files shouldn't have any highlighting, and Rust files should highlight numbers, strings, chars, comments and many keywords specific to Rust.

Let's create a struct `FileType` which will hold our Filetype information for now.

```rust
#[derive(Default)]
struct HighlightingOptions {
    numbers: bool
}

struct FileType {
    name: String,
    hl_opts: HighlightingOptions
}

impl Default for FileType {
    fn default() -> Self {
        Self{
            name: String::from("No filetype"),
            hl_opts: HighlightingOptions::default()
        }
    }
}

pub struct Document {
    rows: Vec<Row>,
    pub file_name: Option<String>,
    dirty: bool,
    file_type: FileType
}



impl Document {
    pub fn new() -> Document{
        Document{
            rows: Vec::new(),
            file_name: None,
            dirty: false,
            file_type: FileType::default()
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
            file_type: FileType::default()
        })
    }
}

```

`HighlightingOptions` will hold a couple of booleans which determine whether or not a certain type should be highlighted. For now, we only add `numbers` to determine whether or not numbers should be highlighted. We use `#[derive(Default)]`  for this struct so that `HighlightOptions::default` returns `HighlightOptions` initialized with default values. Since `HighlightOptions` will only contain `bool`s, and the default for bools is `false`, this suits us well and means that when we add a highlighting option, we only need to change it where we need it, for everything else it will just be unused.

We implement the same trait for `FileType`, this time, we set the string to `"No filetype"` and a default `HighlightingOptions` object. For now, we only use the default file type, even when opening files. Let's get the filetype displayed in the editor.

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
        let line_indicator = format!("{} | {}/{}",self.document.file_type.name, self.cursor_position.y+1, self.document.len());
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
Now we need a way to detect the file type and set the correct `Highlighting_Options`.

```rust
impl FileType {
    fn from(file_name: &String) -> Self {
        if file_name.ends_with(".rs") {
            return Self{
                name: String::from("Rust"),
                hl_opts: HighlightingOptions{
                    numbers: true
                }
            }
        }
        Self::default()
    }
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
            file_type: FileType::from(file_name)
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
            self.file_type = FileType::from(file_name);
        } else {
            return Err(Error::new(ErrorKind::Other, "Can't save file!"))
        }
        Ok(())
    }
```
We add a method to determine the file type from its name. If we can't, we simply return the `default` value. We set the file type on `open` and on `save`. You are now able to open a file, verify it displays the correct file type, and confirm that the file type changes when you change the file ending on save. Very satisfying!

Now, let's actually highlight the files. For that, our rows need to know the highlighting type, which needs to be changed if the user saves, to account for the updated highlighting.

```rust
use crate::editor::document::HighlightingOptions;

pub struct Row {
    string: String,
    highlighting: Vec<HighlightingType>,
    len: usize,
    hl_opts: Option<HighlightingOptions>
}

impl Row {
    pub fn new() -> Row {
        Row {
            string: String::new(),
            highlighting: Vec::new(),
            len: 0,
            hl_opts: None
        }
    }
    pub fn from(slice: &str) -> Row{
        let string = String::from(slice);
        let mut row = Row {
            len: string.chars().count(),
            highlighting: Vec::new(),
            string,
            hl_opts: None
        };
        row.highlight();
        row
    }
    pub fn set_highlight(&mut self, hl_opts: HighlightingOptions) {
        self.hl_opts = Some(hl_opts);
        self.highlight();
    }
}
```
We set and update the highlighting whenever the filetype changes in `document`.

```rust
pub fn open(file_name: &String ) -> Result<Document, Error> {
        let contents = fs::read_to_string(file_name)?;
        let file_type = FileType::from(file_name);
        let mut rows = Vec::new();
        for value in contents.lines() {
            let mut row = Row::from(value);
            row.set_highlight(file_type.hl_opts);
            rows.push(row);
        }
        Ok(Document{
            rows,
            file_name: Some(file_name.clone()),
            dirty: false,
            file_type
        })
    }
    pub fn save(&mut self) -> Result<(), Error> {
        if let Some(file_name) = &self.file_name {
            let mut file = fs::File::create(file_name)?;
            self.file_type = FileType::from(file_name);
            for row in self.rows.iter_mut() {
                row.set_highlight(self.file_type.hl_opts);
                file.write_all(row.to_string().as_bytes())?;
                file.write_all(b"\n")?;
            }
            self.dirty = false;

        } else {
            return Err(Error::new(ErrorKind::Other, "Can't save file!"))
        }
        Ok(())
    }
    fn insert_newline(&mut self, at: &Position) {
        if at.y > self.len() {
            return;
        }
        let mut new_row = Row::new();
        new_row.set_highlight(self.file_type.hl_opts);
        if at.y < self.len() {
            let current_row = self.rows.get_mut(at.y).unwrap();
            new_row.append(&current_row.to_string_range(at.x, current_row.len()));
            current_row.truncate(at.x);
        }

        if  at.y == self.len() ||  at.y + 1 == self.len() {
            self.rows.push(new_row);
        } else if at.y < self.len()-1 {
            self.rows.insert(at.y+1, new_row)
        }
    }
```

Now, let's use the highlighting options to finally highlight numbers in Rust files:

```rust
 fn highlight(&mut self) {
        let mut highlighting = Vec::new();
        let chars: Vec<char> = self.string.chars().collect();
        let mut index = 0;
        let mut prev_is_separator = true;
        loop {
            if index == self.len() {
                break;
            }
            let opts = self. hl_opts.unwrap_or_default();
            let previous_highlight;
            if index > 0 {
                previous_highlight = highlighting.get(index-1).unwrap_or(&HighlightingType::None);
            } else {
                previous_highlight = &HighlightingType::None;
            }

            let c = chars[index];
            if opts.numbers {
                if (c.is_ascii_digit() && (prev_is_separator || previous_highlight == &HighlightingType::Number)) ||
                    (c == '.' && previous_highlight == &HighlightingType::Number) {
                        highlighting.push(HighlightingType::Number);
                } else {
                        highlighting.push(HighlightingType::None);
                }
            } else {
                highlighting.push(HighlightingType::None);
            }

            prev_is_separator = c.is_ascii_punctuation() || c.is_ascii_whitespace();
            index +=1;
        }
        self.highlighting = highlighting;
    }

```

## Colorful strings and characters
With all that out of the way, we can finally get to highlighting more things! Let's start with strings and characters. We do both at the same time, as they are very similar.

```rust
#[derive(PartialEq)]
enum HighlightingType {
    None,
    Number,
    Match,
    String,
    Character
}

impl HighlightingType {
    fn to_color(&self) -> impl color::Color {
        match self {
            HighlightingType::Number => color::Rgb(220,163,163),
            HighlightingType::Match => color::Rgb(38,139,210),
            HighlightingType::String => color::Rgb(211,54,130),
            HighlightingType::Character => color::Rgb(108,113,196),
            _ => color::Rgb(255,255,255),
        }
    }
}

```
and in Document:

```rust

#[derive(Default, Clone, Copy)]
pub struct HighlightingOptions {
    numbers: bool,
    strings: bool,
    characters: bool
}
impl FileType {
    fn from(file_name: &String) -> Self {
        if file_name.ends_with(".rs") {
            return Self{
                name: String::from("Rust"),
                hl_opts: HighlightingOptions{
                    numbers: true,
                    strings: true,
                    characters: true
                }
            }
        }
        Self::default()
    }
}
```
Now for the actual highlighting code. We will use an `in_string` and `in_character` variable to keep track of whether we are currently inside a string or character. If we are, then we'll keep highlighting the current character as a string until we hit the closing quote.

```rust
 fn highlight(&mut self) {
        let mut highlighting = Vec::new();
        let chars: Vec<char> = self.string.chars().collect();
        let mut index = 0;
        let mut prev_is_separator = true;
        let mut in_string = false;
        let mut in_character = false;
        loop {
            if index == self.len() {
                break;
            }
            let opts = self. hl_opts.unwrap_or_default();
            let previous_highlight;
            if index > 0 {
                previous_highlight = highlighting.get(index-1).unwrap_or(&HighlightingType::None);
            } else {
                previous_highlight = &HighlightingType::None;
            }

            let c = chars[index];


            if opts.strings {
                if in_string {
                    highlighting.push(HighlightingType::String);
                    if c == '"' {
                        in_string = false;
                        prev_is_separator = true;
                    } else {
                        prev_is_separator = false;
                    }
                    index +=1;
                    continue;
                } else if prev_is_separator && c == '"' {
                    highlighting.push(HighlightingType::String);
                    in_string = true;
                    prev_is_separator = true;
                    index +=1;
                    continue;
                }

            }
            if opts.characters {
                if in_character {
                    highlighting.push(HighlightingType::Character);
                    if c == '\'' {
                        in_character = false;
                        prev_is_separator = true;
                    } else {
                        prev_is_separator = false;
                    }
                    index +=1;
                    continue;
                } else if prev_is_separator && c == '\'' {
                    highlighting.push(HighlightingType::Character);
                    in_character = true;
                    prev_is_separator = true;
                    index +=1;
                    continue;
                }

            }

            if opts.numbers {
                if (c.is_ascii_digit() && (prev_is_separator || previous_highlight == &HighlightingType::Number)) ||
                    (c == '.' && previous_highlight == &HighlightingType::Number && !prev_is_separator) {
                        highlighting.push(HighlightingType::Number);
                        prev_is_separator = true;
                        index +=1;
                        continue;
                }
            }



            highlighting.push(HighlightingType::None);


            prev_is_separator = c.is_ascii_punctuation() || c.is_ascii_whitespace();
            index +=1;
        }
        self.highlighting = highlighting;
    }

```

The code for both cases is nearly identical (we will add an exception soon). Going through it from top to bottom: If `in_string` is set, then we know that the current character can be highlighted as a string. Then we check if the current character is the closing quote, and if so, we reset `in_string` to `0`. Then, since we highlighted the current character, we have to consume it by incrementing `index` and `continue`ing out of the current loop iteration. We also set `prev_is_separator` to `true` so that if we're done highlighting the string, the closing quote is considered a separator.

IF we're not currently in a string, then we have to check if we're at the beginning of one by checking for the starting quote. If we are, we set `in_string` to `true`, highlight the quote and consume it.

Except for the difference in variable names and the quotes, we do the same for characters.

We should probably take escaped quotes into account when highlighting strings and characters. If the sequence `\"` occurs in a string, then the escaped quote doesn't close the string in the vast majority of languages.

```rust
   fn highlight(&mut self) {
        let mut highlighting = Vec::new();
        let chars: Vec<char> = self.string.chars().collect();
        let mut index = 0;
        let mut prev_is_separator = true;
        let mut in_string = false;
        let mut in_character = false;
        loop {
            if index >= chars.len() {
                break;
            }
            let opts = self. hl_opts.unwrap_or_default();
            let previous_highlight;
            if index > 0 {
                previous_highlight = highlighting.get(index-1).unwrap_or(&HighlightingType::None);
            } else {
                previous_highlight = &HighlightingType::None;
            }

            let c = chars[index];


            if opts.strings {
                if in_string {
                    highlighting.push(HighlightingType::String);
                    if c == '\\' && index + 1 < self.len() {
                        highlighting.push(HighlightingType::String);
                        index +=2;
                        continue;
                    }
                    if c == '"' {
                        in_string = false;
                        prev_is_separator = true;
                    } else {
                        prev_is_separator = false;
                    }
                    index +=1;
                    continue;
                } else if prev_is_separator && c == '"' {
                    highlighting.push(HighlightingType::String);
                    in_string = true;
                    prev_is_separator = true;
                    index +=1;
                    continue;
                }

            }
            if opts.characters {
                if in_character {
                    highlighting.push(HighlightingType::Character);
                    if c == '\\' && index + 1 < chars.len() {
                        highlighting.push(HighlightingType::Character);
                        index +=2;
                        continue;
                    }
                    if c == '\'' {
                        in_character = false;
                        prev_is_separator = true;
                    } else {
                        prev_is_separator = false;
                    }
                    index +=1;
                    continue;
                } else if prev_is_separator && c == '\'' {
                    highlighting.push(HighlightingType::Character);
                    in_character = true;
                    prev_is_separator = true;
                    index +=1;
                    continue;
                }

            }

            if opts.numbers {
                if (c.is_ascii_digit() && (prev_is_separator || previous_highlight == &HighlightingType::Number)) ||
                    (c == '.' && previous_highlight == &HighlightingType::Number && !prev_is_separator) {
                        highlighting.push(HighlightingType::Number);
                        prev_is_separator = true;
                        index +=1;
                        continue;
                }
            }



            highlighting.push(HighlightingType::None);


            prev_is_separator = c.is_ascii_punctuation() || c.is_ascii_whitespace();
            index +=1;
        }
        self.highlighting = highlighting;
    }
```
If we're in a string and the current character is a `\`, _and_ there is at least one more character in that line that comes after the backslash, then we highligh the character that comes after the backslash with `HighlightingType::String` and consume it. We increment `index` by `2` to consume both characters at once.

In Rust, our character highlighting still has a bug. Rust has a concept called [Lifetimes](https://doc.rust-lang.org/1.9.0/book/lifetimes.html), and a lifetime can be indicated with a single `'`. Our highlighting does not account for this. Our highlighting is a bit overeager for characters anyways, since it allows the same content between two single quotes as strings. Let's make character detection a bit more sophisticated.

```rust
        if opts.characters {
                if c == '\'' &&  index + 2 < chars.len()  {
                    let mut last_index = index + 2;
                    let next_char = chars[index+1];
                    if next_char == '\\' && index + 3 < chars.len() {
                        last_index +=1;
                    }
                    let last_char = chars[last_index];
                    if last_char == '\'' {
                        for _ in index..last_index + 1 {
                            highlighting.push(HighlightingType::Character);
                            index +=1;
                            prev_is_separator = true;

                        }
                        continue;
                    }

                }
            }
```

We got rid of `in_character` again. If we encounter a `'`, we now look at the next two characters as well. Then we check if the next character is a `\`. If it is, we also look at another character. Then we check the last character. If it is a closing `'`, we highlight and consume all the characters in between and end the current iteration of the loop. This means that in our syntax highlighting, we only allow one character between two single quotes, unless there is an escaped character.

## Colorful single-line comments
Next, let's highlight single-line comments. (We'll leave multi-line comments until the end, because they're complicated).

```rust

#[derive(Default, Clone, Copy)]
pub struct HighlightingOptions {
    numbers: bool,
    strings: bool,
    characters: bool,
    comments: bool
}
impl FileType {
    fn from(file_name: &String) -> Self {
        if file_name.ends_with(".rs") {
            return Self{
                name: String::from("Rust"),
                hl_opts: HighlightingOptions{
                    numbers: true,
                    strings: true,
                    characters: true,
                    comments: true
                }
            }
        }
        Self::default()
    }
}

```

```rust
#[derive(PartialEq)]
enum HighlightingType {
    None,
    Number,
    Match,
    String,
    Character,
    Comments
}

impl HighlightingType {
    fn to_color(&self) -> impl color::Color {
        match self {
            HighlightingType::Number => color::Rgb(220,163,163),
            HighlightingType::Match => color::Rgb(38,139,210),
            HighlightingType::String => color::Rgb(211,54,130),
            HighlightingType::Character => color::Rgb(108,113,196),
            HighlightingType::Comments => color::Rgb(133,153,0),
            _ => color::Rgb(255,255,255),
        }
    }
}
    fn highlight(&mut self) {
        let mut highlighting = Vec::new();
        let chars: Vec<char> = self.string.chars().collect();
        let mut index = 0;
        let mut prev_is_separator = true;
        let mut in_string = false;
        loop {
            if index >= chars.len() {
                break;
            }
            let opts = self. hl_opts.unwrap_or_default();
            let previous_highlight;
            if index > 0 {
                previous_highlight = highlighting.get(index-1).unwrap_or(&HighlightingType::None);
            } else {
                previous_highlight = &HighlightingType::None;
            }

            let c = chars[index];


            if opts.strings {
                if in_string {
                    highlighting.push(HighlightingType::String);
                    if c == '\\' && index + 1 < chars.len() {
                        highlighting.push(HighlightingType::String);
                        index +=2;
                        continue;
                    }
                    if c == '"' {
                        in_string = false;
                        prev_is_separator = true;
                    }

                    index +=1;
                    continue;
                } else if prev_is_separator && c == '"' {
                    highlighting.push(HighlightingType::String);
                    in_string = true;
                    index +=1;
                    continue;
                }

            }
            if opts.characters {
                if c == '\'' &&  index + 2 < chars.len()  {
                    let mut last_index = index + 2;
                    let next_char = chars[index+1];
                    if next_char == '\\' && index + 3 < chars.len() {
                        last_index +=1;
                    }
                    let last_char = chars[last_index];
                    if last_char == '\'' {
                        for _ in index..last_index + 1 {
                            highlighting.push(HighlightingType::Character);
                            index +=1;
                            prev_is_separator = true;

                        }
                        continue;
                    }

                }
            }
            if opts.comments {
                if c == '/' && self.find(&"//".to_string(), index, SearchDirection::Forward).unwrap_or(index+1) == index {
                    for _ in index..chars.len() {
                        highlighting.push(HighlightingType::Comments);

                    }
                    break;
                }
            }

            if opts.numbers {
                if (c.is_ascii_digit() && (prev_is_separator || previous_highlight == &HighlightingType::Number)) ||
                    (c == '.' && previous_highlight == &HighlightingType::Number && !prev_is_separator) {
                        highlighting.push(HighlightingType::Number);
                        prev_is_separator = true;
                        index +=1;
                        continue;
                }
            }



            highlighting.push(HighlightingType::None);


            prev_is_separator = c.is_ascii_punctuation() || c.is_ascii_whitespace();
            index +=1;
        }
        self.highlighting = highlighting;
    }

```
If we are encountering an `/`, we are checking if we can find `//` at our current position. If yes, we highlight the rest of the row as a comment and end the highlihgting of the current row. We are doing comment highlihgting after checking for strings, so that comment starts within strings are not highlighted.

## Colorful keywords
Now let's turn to highlighting keywords. We're going to allow languages to specify two types of keywords that will be highlighted in different colors. (IN Rust, we'll highlight actual keywords in one color and common type names in the other color).

```rust
#[derive(PartialEq)]
enum HighlightingType {
    None,
    Number,
    Match,
    String,
    Character,
    Comments,
    PrimaryKeywords,
    SecondaryKeywords
}

impl HighlightingType {
    fn to_color(&self) -> impl color::Color {
        match self {
            HighlightingType::Number => color::Rgb(220,163,163),
            HighlightingType::Match => color::Rgb(38,139,210),
            HighlightingType::String => color::Rgb(211,54,130),
            HighlightingType::Character => color::Rgb(108,113,196),
            HighlightingType::Comments => color::Rgb(133,153,0),
            HighlightingType::PrimaryKeywords => color::Rgb(181,137,0),
            HighlightingType::SecondaryKeywords => color::Rgb(42,161,152),
            _ => color::Rgb(255,255,255),
        }
    }
}

```
Let's add two `Vector`s to the `HighlightingOptions`, one for primary, one for secondary keywords.

```rust
use crate::editor::{Position, SearchDirection};
use std::fs;
mod row;
pub use row::Row;
use std::io::{Error,Write, ErrorKind};

#[derive(Default, Clone)]
pub struct HighlightingOptions {
    numbers: bool,
    strings: bool,
    characters: bool,
    comments: bool,
    primary_keywords: Vec<String>,
    secondary_keywords: Vec<String>,
}


pub struct FileType {
    pub name: String,
    hl_opts: HighlightingOptions
}

impl Default for FileType {
    fn default() -> Self {
        Self{
            name: String::from("No filetype"),
            hl_opts: HighlightingOptions::default()
        }
    }
}

impl FileType {
    fn from(file_name: &String) -> Self {
        if file_name.ends_with(".rs") {
            Self{
                name: String::from("Rust"),
                hl_opts: HighlightingOptions{
                    numbers: true,
                    strings: true,
                    characters: true,
                    comments: true,
                    primary_keywords: vec![
                        "as".to_string(),
                        "break".to_string(),
                        "const".to_string(),
                        "continue".to_string(),
                        "crate".to_string(),
                        "else".to_string(),
                        "enum".to_string(),
                        "extern".to_string(),
                        "false".to_string(),
                        "fn".to_string(),
                        "for".to_string(),
                        "if".to_string(),
                        "impl".to_string(),
                        "in".to_string(),
                        "let".to_string(),
                        "loop".to_string(),
                        "match".to_string(),
                        "mod".to_string(),
                        "move".to_string(),
                        "mut".to_string(),
                        "pub".to_string(),
                        "ref".to_string(),
                        "return".to_string(),
                        "self".to_string(),
                        "Self".to_string(),
                        "static".to_string(),
                        "struct".to_string(),
                        "super".to_string(),
                        "trait".to_string(),
                        "true".to_string(),
                        "type".to_string(),
                        "unsafe".to_string(),
                        "use".to_string(),
                        "where".to_string(),
                        "while".to_string(),
                        "dyn".to_string(),
                        "abstract".to_string(),
                        "become".to_string(),
                        "box".to_string(),
                        "do".to_string(),
                        "final".to_string(),
                        "macro".to_string(),
                        "override".to_string(),
                        "priv".to_string(),
                        "typeof".to_string(),
                        "unsized".to_string(),
                        "virtual".to_string(),
                        "yield".to_string(),
                        "async".to_string(),
                        "await".to_string(),
                        "try".to_string(),
                        ],
                    secondary_keywords: vec![
                        "bool".to_string(),
                        "char".to_string(),
                        "i8".to_string(),
                        "i16".to_string(),
                        "i32".to_string(),
                        "i64".to_string(),
                        "isize".to_string(),
                        "u8".to_string(),
                        "u16".to_string(),
                        "u32".to_string(),
                        "u64".to_string(),
                        "usize".to_string(),
                        "f32".to_string(),
                        "f64".to_string(),
                        ],
                }
            }
        } else {
            Self::default()
        }

    }
}
```

We can't automatically derive the `Copy` trait any more, so we need to adjust the rest of our code a bit:

```rust
  pub fn open(file_name: &String ) -> Result<Document, Error> {
        let contents = fs::read_to_string(file_name)?;
        let file_type = FileType::from(file_name);
        let mut rows = Vec::new();
        for value in contents.lines() {
            let mut row = Row::from(value);
            row.set_highlight(file_type.hl_opts.clone());
            rows.push(row);
        }
        Ok(Document{
            rows,
            file_name: Some(file_name.clone()),
            dirty: false,
            file_type
        })
    }
    pub fn save(&mut self) -> Result<(), Error> {
        if let Some(file_name) = &self.file_name {
            let mut file = fs::File::create(file_name)?;
            self.file_type = FileType::from(file_name);
            for row in self.rows.iter_mut() {
                row.set_highlight(self.file_type.hl_opts.clone());
                file.write_all(row.to_string().as_bytes())?;
                file.write_all(b"\n")?;
            }
            self.dirty = false;

        } else {
            return Err(Error::new(ErrorKind::Other, "Can't save file!"))
        }
        Ok(())
    }
     fn insert_newline(&mut self, at: &Position) {
        if at.y > self.len() {
            return;
        }
        let mut new_row = Row::new();
        new_row.set_highlight(self.file_type.hl_opts.clone());
        if at.y < self.len() {
            let current_row = self.rows.get_mut(at.y).unwrap();
            new_row.append(&current_row.to_string_range(at.x, current_row.len()));
            current_row.truncate(at.x);
        }

        if  at.y == self.len() ||  at.y + 1 == self.len() {
            self.rows.push(new_row);
        } else if at.y < self.len()-1 {
            self.rows.insert(at.y+1, new_row)
        }
    }
```
Now that we have the keywords available, let's highlight them. We start with the primary keywords first.

```rust
 fn highlight(&mut self) {
        let mut highlighting = Vec::new();
        let chars: Vec<char> = self.string.chars().collect();
        let mut index = 0;
        let mut prev_is_separator = true;
        let mut in_string = false;
        'outer: loop {
            if index >= chars.len() {
                break;
            }
            let mut opts = &HighlightingOptions::default();
            if !self.hl_opts.is_none() {
               opts = self.hl_opts.as_ref().unwrap()
            }
            let previous_highlight;
            if index > 0 {
                previous_highlight = highlighting.get(index-1).unwrap_or(&HighlightingType::None);
            } else {
                previous_highlight = &HighlightingType::None;
            }

            let c = chars[index];


            if opts.strings {
                if in_string {
                    highlighting.push(HighlightingType::String);
                    if c == '\\' && index + 1 < chars.len() {
                        highlighting.push(HighlightingType::String);
                        index +=2;
                        continue;
                    }
                    if c == '"' {
                        in_string = false;
                        prev_is_separator = true;
                    }

                    index +=1;
                    continue;
                } else if prev_is_separator && c == '"' {
                    highlighting.push(HighlightingType::String);
                    in_string = true;
                    index +=1;
                    continue;
                }

            }
            if opts.characters {
                if c == '\'' &&  index + 2 < chars.len()  {
                    let mut last_index = index + 2;
                    let next_char = chars[index+1];
                    if next_char == '\\' && index + 3 < chars.len() {
                        last_index +=1;
                    }
                    let last_char = chars[last_index];
                    if last_char == '\'' {
                        for _ in index..last_index + 1 {
                            highlighting.push(HighlightingType::Character);
                            index +=1;
                            prev_is_separator = true;

                        }
                        continue;
                    }

                }
            }
            if opts.comments {
                if c == '/' && self.find(&"//".to_string(), index, SearchDirection::Forward).unwrap_or(index+1) == index {
                    for _ in index..chars.len() {
                        highlighting.push(HighlightingType::Comments);

                    }
                    break;
                }
            }

          if  prev_is_separator {
                for keyword in opts.primary_keywords.iter() {
                    let length = keyword.chars().count();
                    if index + length >= self.len() {
                        continue;
                    }
                    if self.string[index..index+keyword.len()].find(keyword).is_none() {
                        continue;
                    }
                    for _ in index..index+length {
                         highlighting.push(HighlightingType::PrimaryKeywords);
                    }
                    index += length;
                    continue 'outer;


              }

            }


            if opts.numbers {
                if (c.is_ascii_digit() && (prev_is_separator || previous_highlight == &HighlightingType::Number)) ||
                    (c == '.' && previous_highlight == &HighlightingType::Number && !prev_is_separator) {
                        highlighting.push(HighlightingType::Number);
                        prev_is_separator = true;
                        index +=1;
                        continue;
                }
            }



            highlighting.push(HighlightingType::None);


            prev_is_separator = c.is_ascii_punctuation() || c.is_ascii_whitespace();
            index +=1;
        }

        self.highlighting = highlighting;
    }
```
Let's go through this change step by step. We have positioned our highlighting after the highlighting of comments and strings, to make sure that keywords within a comment or a string are not highlighted. Then, we require that the character before a keyword is a separator, to make sure we do not highlight the `as` in `whereas`. Then we go through all the keywords and check for each keyword if it would fit into the remainder of the current row. If not, we contiunue with the next keyword. Then, we do a few things at once: we slice the row starting with the current index and ending with the length of the keyword, before we check if that remaining slice contains the keyword itself. If not, we continue. If so, we have found the keyword we where looking for, so we highlight the keyword and consume it before continuing.

We are using nested loops here: We are within a `for` loop and want to continue with the outermost loop. In order to do that, we have [labelled](https://doc.rust-lang.org/rust-by-example/flow_control/loop/nested.html) the outer loop to be able to cleanly continue with it from within the inner `for` loop.

Now let's fix a quick bug: We require a keyword to be followed by a separator, so that we do not highlight the `do` in `document`. And we need to make sure that our keyword is not treated as a separator for the following characters.

```rust

      if  prev_is_separator {
                for keyword in opts.primary_keywords.iter() {
                    let length = keyword.chars().count();
                    if index + length >= chars.len() {
                        continue;
                    }
                    if self.string[index..index+keyword.len()].find(keyword).is_none() {
                        continue;
                    }
                    if index + length < chars.len() {
                        let next_char =  chars[index+length];
                        if !next_char.is_ascii_punctuation() && !next_char.is_ascii_whitespace() {
                            continue;
                        }
                    }

                    for _ in index..index+length {
                         highlighting.push(HighlightingType::PrimaryKeywords);
                         index +=1;
                    }
                    prev_is_separator = false;
                    continue 'outer;


              }

            }
```

Now, let's try and highlight secondary keywords as well.


```rust
        if  prev_is_separator {
              for (keywords, highlighting_type) in [(opts.primary_keywords.iter(), HighlightingType::PrimaryKeywords), (opts.secondary_keywords.iter(), HighlightingType::SecondaryKeywords)].iter_mut(){
                for keyword in keywords {
                                let length = keyword.chars().count();
                                if index + length >= chars.len() {
                                    continue;
                                }
                                if self.string[index..index+keyword.len()].find(keyword).is_none() {
                                    continue;
                                }
                                if index + length < chars.len() {
                                    let next_char =  chars[index+length];
                                    if !next_char.is_ascii_punctuation() && !next_char.is_ascii_whitespace() {
                                        continue;
                                    }
                                }

                                for _ in index..index+length {
                                    highlighting.push(*highlighting_type);
                                    index +=1;
                                }
                                prev_is_separator = false;
                                continue 'outer;


                 }
              }


            }
```

We have now added another loop around the inner loop, which loops over an array with two tuples. Tuples are collection of different types, and constructed with `()`. Our tuple contains both the iterator and the highlighting type for said iterator. More on tuples [can be found in the documentation.](https://doc.rust-lang.org/rust-by-example/primitives/tuples.html)

WHen you open your editor now, you should be able to see primary and secondary keywords highlighted.

## Colorful multi-line comments
Okay, we have one last feature to implement: multi-line comment highlighting.