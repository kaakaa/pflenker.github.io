---
layout: post
title: "Hecto, Chapter 5: A text editor"
categories: [Rust, hecto]
---
Now that `hecto` can read files, let's see if we can teach it to edit files as well.

## Insert ordinary characters

Let's begin by writing a function that inserts a single character into a `Row`, at a given position.

```rust
mpl Row {
    fn insert(&mut self, at: u16, c: char) {
        let at = at as usize;
        if at >= self.string.len() {
            self.string.push(c);
        } else {
            self.string.insert(at, c)
        }
    }
    fn len(&self) -> u16{
        self.string.len() as u16
    }
    fn len_render(&self) -> u16{
        self.render.len() as u16
    }
    fn slice(&self, start: u16, end: u16) -> &str{
        let start = start as usize;
        let end = end as usize;
        &self.string[start.. end]
    }
    fn slice_render(&self, start: u16, end: u16) -> &str{
        let start = start as usize;
        let end = end as usize;
        &self.render[start.. end]
    }
    fn from(line: &str) -> Row{
        let string = String::from(line);
        let tab = " ".repeat(TAB_STOP);
        let render = string.replace("\t", &tab);
        Row{
            string,
            render
        }
    }
}
```

First we validate `at`, which is the index we want to insert the character into. Notice that `at` is allowed to go past the end of the string, in which case the character should be inserted at the end of the string.

Now we need to call that method on the appropriate row when a character is entered.

```rust
impl  File {
    fn len (&self) -> u16{
        self.rows.len() as u16
    }
    fn row(&self, index: u16) -> Option <&Row>{
        self.rows.get(index as usize)
    }
    fn row_mut(&mut self, index: u16) -> Option <&mut Row>{
        self.rows.get_mut(index as usize)
    }
    fn new() -> File {
        File{
            filename: None,
            rows: Vec::new()
        }
    }
}

fn editor_process_keypress(mut state: &mut EditorState) -> Result<(), std::io::Error>{
    let pressed_key = editor_read_key()?;
    match pressed_key {
        Key::Ctrl('q') => state.quit = true,
        Key::Char(c) => {
            if let Some(row) = &mut state.file.row_mut(state.cursor_position.y) {
                row.insert(state.cursor_position.x, c);
            }


        }
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

```
Note that we need a new method `get_mut` which hands us a mutable row. After this change, we observe ...nothing. We are only updating `string`, but not `render`. Let's fix that.

```rust
   fn insert(&mut self, at: u16, c: char) {
        let at = at as usize;
        if at >= self.string.len() {
            self.string.push(c);
        } else {
            self.string.insert(at, c);
        }
        let render_index = at;
        self.render = self.string.replace("\t", &" ".repeat(TAB_STOP));
    }
```

We are following the lazy approach here and completely overwrite `render` instead of, for example, getting the correct index to update within `render` and use `render.insert()`.

Now we need to set the cursor accordingly. That is easy if we treat "Entering a character" as "Enter a character and go to the right".
```rust
fn editor_process_keypress(mut state: &mut EditorState) -> Result<(), std::io::Error>{
    let pressed_key = editor_read_key()?;
    match pressed_key {
        Key::Ctrl('q') => state.quit = true,
        Key::Char(c) => {
            if let Some(row) = &mut state.file.row_mut(state.cursor_position.y) {
                row.insert(state.cursor_position.x, c);
                editor_move_cursor(&mut state, &Key::Right);
            }
        }
        Key::Up |
        Key::Down |
        Key::Left |
        Key::Right |
        Key::PageUp |
        Key::PageDown |
        Key::End |
        Key::Home
        =>  editor_move_cursor(&mut state, &pressed_key),
        _ => ()
    }
    editor_scroll(&mut state);
    Ok(())
}
```

Entering characters now works in existing documents. However, we can't work on new documents, nor can we add a new line at the bottom. To accomplish that, we have to add a new `Row` to `File` when we are at the bottom.

```rust
n editor_process_keypress(mut state: &mut EditorState) -> Result<(), std::io::Error>{
    let pressed_key = editor_read_key()?;
    match pressed_key {
        Key::Ctrl('q') => state.quit = true,
        Key::Char(c) => {
            if (state.cursor_position.y == state.file.len()) {
                state.file.append_row();
            }
            if let Some(row) = &mut state.file.row_mut(state.cursor_position.y) {
                row.insert(state.cursor_position.x, c);
                 editor_move_cursor(&mut state, &Key::Right);
            }


        }
        Key::Up |
        Key::Down |
        Key::Left |
        Key::Right |
        Key::PageUp |
        Key::PageDown |
        Key::End |
        Key::Home
        =>  editor_move_cursor(&mut state, &pressed_key),
        _ => ()
    }
    editor_scroll(&mut state);
    Ok(())
}
impl  File {
    fn len (&self) -> usize{
        self.rows.len() as usize
    }
    fn row(&self, index: usize) -> Option <&Row>{
        self.rows.get(index as usize)
    }
    fn row_mut(&mut self, index: usize) -> Option <&mut Row>{
        self.rows.get_mut(index as usize)
    }
    fn append_row(&mut self) {
        self.rows.push(Row::from(""));
    }

    fn new() -> File {
        File{
            filename: None,
            rows: Vec::new()
        }
    }
}
```
