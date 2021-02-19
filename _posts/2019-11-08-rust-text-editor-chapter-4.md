---
layout: postwithdiff
title: "Hecto, Chapter 4: A text viewer"
categories: [Rust, hecto, Tutorial]
permalink: /hecto-chapter-4/
image: /assets/2019-11-08-hecto-chapter-4.png
date: 2019-11-08 00:00:05
last_modified_at: 2021-01-26
---
[Previous chapter]({% post_url 2019-11-08-rust-text-editor-chapter-3%}) - [Overview]({% post_url 2019-11-08-rust-text-editor%}) - [Appendices]({% post_url 2019-11-08-rust-text-editor-appendix%}) - [Next Chapter]({% post_url 2019-11-08-rust-text-editor-chapter-5%}) 
{: style="text-align: center"}

Let's see if we can turn `hecto` into a text viewer in this chapter.

## A line viewer
We need a few more data structures: A `Document` which will represent the
document the user is currently editing, as well as a `Row` in that document.

{% include hecto/add-document-rows.html %}

<small>[See this step on
github](https://github.com/pflenker/hecto-tutorial/tree/add-document-rows)</small>

In this change, we have introduced two new concepts to our code: First, we are
using a data structure called a Vector  which will hold our rows. A Vector is a
dynamic structure: It can grow and shrink on runtime, as we are adding to or
removing from it. The syntax `Vec<Row>` means that this vector will hold entries
of the type `Row`.

The other new concept is this line:

```rust
#[derive(Default)]
```

It means that the rust compiler is supposed to derive an implementation for
`default`. `default` is supposed to return a struct with its contents
initialized to a default value - which is something that the compiler can do for
us. With that directive, we do not need to implement `default` ourselves. Let's
see if we can simplify our existing code with it:

{% include hecto/use-default.html %}

<small>[See this step on
github](https://github.com/pflenker/hecto-tutorial/tree/use-default)</small>

By deriving `default` for `Position`, we have removed the duplication of
initializing the cursor position to `0`, `0`. If, in the future, we would decide
to intialize  `Position` in a different way, then we could implement `default`
ourselves without needint to touch any other code.

We can't derive `default` for our other structs - these are too complex, Rust
can not guess the default values for all struct members.

Let's fill the Document with some text now. We won't worry about reading from a
file just yet. Instead, we'll hard code a "Hello, World" string into it.

{% include hecto/hardcoded-document.html %}

<small>[See this step on
github](https://github.com/pflenker/hecto-tutorial/tree/hardcoded-document)</small>


You might be wondering about the `From<&str>` part in the `impl` block for the
row. We are now not only implementing a `from` function, but we do so by
implementing the `From` trait for `Row`. We won't need it in the scope of this
tutorial, but implementing a trait enables us to use certain functionalities in
a certain way. We will handle traits a bit later in even more detail, but if you
are interested now, check out [this part of the
docs](https://doc.rust-lang.org/rust-by-example/conversion/from_into.html) - we
get `into` for free while implementing `from`.

We will later implement a method to open a `Document` from file. At that point,
we will use `default` again when initializing our editor. But let's focus on
getting our hardcoded value displayed for now.


{% include hecto/display-row.html %}

<small>[See this step on
github](https://github.com/pflenker/hecto-tutorial/tree/display-row)</small>

Let's unravel this change starting with `Row`. We have added a method called
`render`. We call it `render`, because eventually it will be responsible for a
few more things than just returning a substring. Our `render` method is very
user friendly as it normalizes bogus input - essentially, it returns the biggest
possible substring it can generate. We're also routinely using
`unwrap_or_default`, even though it's not necessary here, as we sanitized the
start and end parameters beforehand. What happens in the last line is that we
try to create a substring from the string and either convert it or the default
value (`""`) to a string. (In Rust, there is a difference between a String and
something called a `str`. We will get to this soon.)

In `Document`, we add a method to retrieve a `Row` at a specific index. We use
Vector's `get` for this, which has the signature we need: Return `None` if the
index is out of bounds, or the `Row` if we have one.


Let's move to `Editor`. In `draw_rows`, we first rename the variable `row` to
`terminal_row` to avoid confusion with the row we are now getting from
`Document`. We are then retrieving the `row` and displaying it. The concept here
is that `Row` makes sure to return to you a substring that can be displayed,
while the `Editor` makes sure that the terminal dimensions are met.

However, our welcome message is still displayed. We don't want that when the
user is opening a file, so let's add a method `is_empty` to our `Document` and
check against it in `draw_rows`:

{% include hecto/remove-welcome-message.html %}

<small>[See this step on
github](https://github.com/pflenker/hecto-tutorial/tree/remove-welcome-message)</small>

You should be able to confirm that the message is no longer shown in the middle
of the screen.

Next, let's allow the user to open and display actual file. We start by changing
our `Document`:

{% include hecto/open-file.html %}

<small>[See this step on
github](https://github.com/pflenker/hecto-tutorial/tree/open-file)</small>

We are now using a default `Document` on start, and added a new method `open`,
which attempts to open a file and returns an error in case of a failure.

`open` reads the lines into our `Document` struct. It's not obvious from our
code, but each row in `rows` will not contain the line endings `\n` or `\r\n`,
as Rust's `line()` method will cut it away for us. That makes sense: We are
already handling new lines ourselves, so we wouldn't want to handle the ones in
the file anyways.

Let's now actually use `open` to open a file which will be passed to `hecto` by
command line:

{% include hecto/open-file-from-params.html %}

<small>[See this step on
github](https://github.com/pflenker/hecto-tutorial/tree/open-file-from-params)</small>

Try it out by running `cargo run` in contrast to `cargo run Cargo.toml`!

Here are a few things to observe. First, we can use `if..else` as a statement -
in this case, it means that the result of either block of the `if` statement is
bound to `document`. To make this work, we have to omit `;` on the last line of
each block of the `if`, but add a `;` after the last closing `}`. This ensures
that `document` is never undefined.

Since we have implemented `default` on `Document`, we can use
`unwrap_or_default` here. What this does is that in case `open` yielded an
error, a default document will be returned, the error will be discarded (we will
rectify this later though)

We are calling `Document::open()` only if we have more than one `arg`. `args` is
a vector which contains the command line parameters which have been passed to
our program. Per convention, `args[0]` is always the name of our program, so
`args[1]` contains the parameter we're after - we want to use `hecto (filename)`
to open a file.  You can pass a file name to your program by running `cargo run
(filename)` while you are developing.

Now you should see your screen fill up with lines of text when you run `cargo
run Cargo.toml`, for example.

## Scrolling

Next we want to enable the user to scroll through the whole file, instead of
just being able to see the top few lines of the file. Let's add an `offset`  to
the editor state, which will keep track of what row of the file the user is
currently scrolled to. We are reusing the `Position` struct for that.

{% include hecto/add-offset.html %}

<small>[See this step on
github](https://github.com/pflenker/hecto-tutorial/tree/add-offset)</small>

We initialize it with the default value. which means we'll be scrolled to the
top left of the file by default.

Now let's have `draw_row()` display the correct range of lines of the file
according to the value of `offset.x`, and `draw_rows` display the correct range
of rows according to the value of `offset.y`.

{% include hecto/use-offset-for-drawing.html %}

<small>[See this step on
github](https://github.com/pflenker/hecto-tutorial/tree/use-offset-for-drawing)</small>

We are adding the offset to the start and to the end, to get the slice of the
string we're after. We are also making sure that we can handle situations where
our string is not long enough to fit the screen. If the current row has ended to
the left of the current screen (which can happen if we are in a long row and
scroll to the right), the `render` method of `Row` will return an empty string.

Where do we set the value of  `offset`? Our strategy will be to check if the
cursor has moved outside of the visible window, and if so, adjust `offset` so
that the cursor is just inside the visible window. We’ll put this logic in a
function called `scroll`, and call it right after we handled the key press.

{% include hecto/set-offset-in-scroll.html %}

<small>[See this step on
github](https://github.com/pflenker/hecto-tutorial/tree/set-offset-in-scroll)</small>


To `scroll`, we need to know the width and height of the terminal and the
current position, and we want to change the values in `self.offset`. If we have
moved to the left or to the top, we want to set our offset to the new position
in the document. If we have scrolled too far to the right, we are subtracting
the current offset from the new position to calculate the new offset.

Now let's allow the cursor to advance past the bottom of the screen (but not
past the bottom of the file). We tackle scrolling to the right a bit later.

{% include hecto/scroll-down.html %}

<small>[See this step on
github](https://github.com/pflenker/hecto-tutorial/tree/scroll-down)</small>

You should be able to scroll through the entire file now, when you run `cargo
run src/editor.rs`. The handling of the last line will be a bit strange, since
we place our cursor there, but are not rendering there. This will be fixed when
we add the status bar later in this chapter. If you try to scroll back up, you
may notice the cursor isn't being positioned properly. That is because
`Position` in the state no longer refers to the position of the cursor on the
screen. It refers to the position of the cursor within the text file, but we are
still passing it to `cursor_position`. To position the cursor on the screen, we
now have to subtract the `offset` from the position within the document.

{% include hecto/scroll-up.html %}

<small>[See this step on
github](https://github.com/pflenker/hecto-tutorial/tree/scroll-up)</small>

Now let's fix the horizontal scrolling. The missing piece here is that we are
not yet allowing the cursor to scroll past the right of the screen. We fix that
symmetrical to what we did for scrolling down:

{% include hecto/scroll-right.html %}

<small>[See this step on
github](https://github.com/pflenker/hecto-tutorial/tree/scroll-right)</small>

All we had to do is changing the width used by `move_cursor`. Horizontal
scrolling does now work. In case you are wondering, it's a best practice to
implement `is_empty` as soon as you have a `len` function. We're not using it
for now, but Clippy pointed it out, and it was easy for us to implement.

Our scrolling code still contains a subtle bug, which we will fix after a few
other improvements. Can you spot it?

## Snap cursor to end of line

Now `cursor_position` refers to the cursor's position within the file, not its
position on the screen. So our goal with the next few steps is to limit the
values of `cursor_position` to only ever point to valid positions in the file,
with the exception that we allow the cursor to point one character past the end
of a line or past the end of the file, so that the user can add new characters
at the end of a line, and new lines at the end of the file can be added easily.

We are already able to prevent the user from scrolling too far to the right or
too far down. The user is still able to move the cursor past the end of a line,
however. They can do it by moving the cursor to the end of a long line, then
moving it down to the next line, which is shorter. The `cursor_position.y` value
won't change, and the cursor will be off to the right of the end of the line
it's now on.

Let's add some code to `move_cursor()` that corrects `cursor_position` if it
ends up past the end of the line it's on.

{% include hecto/snap-to-line.html %}

<small>[See this step on
github](https://github.com/pflenker/hecto-tutorial/tree/snap-to-line)</small>

We have to set `width` again, since `row` can have changed during the key
processing. We then set the new value of `x`, making sure that x does not exceed
the current row's width.

## Scrolling with <kbd>Page Up</kbd> and <kbd>Page Down</kbd>
Now that we have scrolling, let's make the <kbd>Page Up</kbd> and <kbd>Page
Down</kbd> keys scroll up or down an entire page instead of the full document.

{% include hecto/page-through-doc.html %}

<small>[See this step on
github](https://github.com/pflenker/hecto-tutorial/tree/page-through-doc)</small>

We were able to get rid of unnecessary saturating arithmetics. Why? For example,
`y` and `height` have the same type. If `y.saturating_add(terminal_height)` is
less than `height`, then `y + terminal_height` is also less than `height`.

If we try this out now, we can see that we still have some issues with the last
line. Instead of moving to the next screen on <kbd>Page Down</kbd>, our cursor
lands at the empty row at the bottom. We'll fix this at the end of this chapter
- but before we do, let's complete our cursor navigation in this file.

## Moving left at the start of a line

We want to allow the user to press <kbd>&larr;</kbd> at the beginning of the
line to move to the end of the previous line.

{% include hecto/left-at-line-start.html %}

We make sure they aren't on the very first line before we move them up a line.
We don't need to use `saturating_sub` any more, as we check explicitly if the
value we want to subtract from is bigger than `0`.

## Moving right at the end of a line

Similarly, let's allow the user to press <kbd>&rarr;</kbd> at the end of a line
to go to the beginning of the next line.

{% include hecto/right-at-line-end.html %}

<small>[See this step on
github](https://github.com/pflenker/hecto-tutorial/tree/right-at-line-end)</small>

Here we have to make sure they’re not at the end of the file before moving down
a line. We were also able to remove `saturated_add` here. `height` and `y` are
of the same type, so if `y` is smaller than `height`, then we have enough room
to add `1` to it.

## Fix scrolling

Now, let's focus on the bug that I hinted at above. To reproduce it, save the
following as a text file and open it with `hecto`:

>aaa  
>äää  
>y̆y̆y̆  
>❤❤❤ 

When you scroll around in that file, you will notice that only the first line
scrolls correctly to the right, all other lines let you scroll past the end of
the line. 

One of our first steps in this tutorial was to observe the bytes returned to us
on every key press. We observed that German Umlauts such as ä return multiple
bytes, and that's exactly what's causing the bug. 

Let's observe the behavior in more detail. Don't worry, we're not going to
revert our code to observe key presses again, instead we're going to head over
to the [Rust playground](https://play.rust-lang.org/) and paste in the following
code:

```rust
fn main() {
    dbg!("aaa".to_string().len());
    dbg!("äää".to_string().len());
    dbg!("y̆y̆y̆".to_string().len());
    dbg!("❤❤❤".to_string().len());
}
```
`dbg!` is a macro which is useful for quick and dirty debugging, it prints out
the current value of what you give in, and more. Here's what it returns for that
code:

```
[src/main.rs:2] "aaa".to_string().len() = 3
[src/main.rs:3] "äää".to_string().len() = 6
[src/main.rs:4] "y̆y̆y̆".to_string().len() = 9
[src/main.rs:5] "❤❤❤".to_string().len() = 9
```

We can see now that the length of the string can be bigger than what we thought
would be the length of the string, as some characters simply take up more than
one byte, or are a composition of more than one character. For instance, the
Female Scientist Emoji (👩‍🔬) [is a combination of the Woman Emoji (👩) and the
Microscope Emoji (🔬)](https://emojipedia.org/female-scientist/). Wrong handling
of these emojis can lead to [interesting
results](https://twitter.com/brooklynevery1/status/1167087999630434304).

So what _is_ the length of a row for us? Fundamentally, it's one element on the
screen that our mouse can move over. That is called a
[Grapheme](https://en.wikipedia.org/wiki/Grapheme), and Rust, unlike some other
languages, does not support Graphemes by default. That means that we either need
to do the coding ourselves, or we use a crate which does this for us. Since I'm
not in the mood for reinventing the wheel, let's use a crate for that.

{% include hecto/use-unicode-segmentation.html %}

<small>[See this step on
github](https://github.com/pflenker/hecto-tutorial/tree/use-unicode-segmentation)</small>

We have introduced two changes. In both cases, we are performing `graphemes()`
on the slice of the full String (indicated by [..]) and then use that iterator.
In the case of `len()`, we are calling `count()` on the iterator, which tells us
how many graphemes there are. In case of `render`, we are now starting to build
our own string instead of using the built-in methods. For that, we skip the
first graphemes (the ones to the left of the screen), and we only take
`end-start` many graphemes (the visible portion of the row). These graphemes are
then pushed into the return value.

While this works, the performance is not optimal. `count` actually goes through
the whole iterator and then returns the value. This means that for every visible
row, we are repeatedly counting the length of the full row. Let's keep track of
the length ourselves instead.

{% include hecto/manually-calculate-len.html %}

<small>[See this step on
github](https://github.com/pflenker/hecto-tutorial/tree/manually-calculate-len)</small>

Now we only have to remember to call `update_len` whenever our row changes.
Scrolling works now - or does it? Turns out we have to jump through one more
hoop to get all of it right.

### Rendering Tabs
If you try opening a file with tabs, you'll notice that the tab character takes
up a width of 8 columns or so. As you probably know, there is a long and ongoing
debate of whether or not to use tabs or spaces for indentation. Honestly, I
don't care, I have always a sufficiently advanced editor that could just roll
with any indentation type you throw at it. If I was forced to pick a side, I
would pick "spaces", though, because I find the "pros" for tabs not very
convincing - but that's a different matter. What matters, though, is that we are
simply replacing tabs with one space for the sake of this tutorial. That's
enough for our purpose, as in the Rust ecosystem, you will rarely encounter
tabs. 

Let's replace our tabs with spaces now. 

{% include hecto/render-tabs.html %}

<small>[See this step on
github](https://github.com/pflenker/hecto-tutorial/tree/render-tabs)</small>

That's it - we have finally solved all the edge cases we care about. Now let's
tackle one last issue which has been bugging us for quite some time: Let's fix
that last line.

## Status bar

The last thing we'll add before finally getting to text editing is a status bar.
This will show useful information such as the filename, how many lines are in
the file, and what line you're currently on. Later we'll add a marker that tells
you whether the file has been modified since it was last saved, and we'll also
display the filetype when we implement syntax highlighting.

First we'll simply make room for a two-line status bar at the bottom of the
screen.  We will now also fix the issues we have with the rendering of the last
line.

{% include hecto/make-room-for-status-bar.html %}

<small>[See this step on
github](https://github.com/pflenker/hecto-tutorial/tree/make-room-for-status-bar)</small>

You should now be able to confirm that two lines are cleared at the bottom and
that Page Up and Down works as expected. 

Notice how with this change, our text viewer works just fine, including
scrolling and cursor movement, and the last lines where our status bar will be
are left alone by the rest of the display code.

To make the status bar stand out, we're going to display it colored. `termion`
takes care of the corresponding escape sequences for us, so we don't have to do
this manually. The corresponding escape sequence is the `m` command ([Select
Graphic Rendition](http://vt100.net/docs/vt100-ug/chapter3.html#SGR)).  The
[VT100 User Guide](http://vt100.net/docs/vt100-ug/chapter3.html) doesn't
document color, so let's turn to the Wikipedia article on [ANSI escape
codes](https://en.wikipedia.org/wiki/ANSI_escape_code). It includes a large
table containing all the different argument codes you can use with the `m`
command on various terminals. It also includes the ANSI color table with the 8
foreground/background colors available.

We're going to use `termion`'s capability to provide RGB colors, which fall back
to simpler colors in case they are not supported by the current terminal.

{% include hecto/draw-status-bar.html %}

> Note: On some terminals, such as on Mac, the termion colors won't be displayed
> properly. For the sake of the tutorial, you could then use `termion::style::Invert`. [See this github issue for details.](https://github.com/pflenker/hecto-tutorial/issues/3)

<small>[See this step on
github](https://github.com/pflenker/hecto-tutorial/tree/draw-status-bar)</small>

We have started by extending our `terminal` with a few new functions, to set and
to reset the background color. We need to reset the colors after we use them,
otherwise the rest of the screen will also be rendered in the same color. 

We use that functionality in `editor` to draw a line of empty spaces where our
status bar will be. We have also added a function to draw the message bar below
the status bar, but we leave it empty for now.

We want to display the file name next. Let's adjust our `Document` to have an
optional file name, and set it in `open`. We're also going to prepare the
`Terminal` to set and reset the foreground color.

{% include hecto/add-file-name.html %}

<small>[See this step on
github](https://github.com/pflenker/hecto-tutorial/tree/add-file-name)</small>

Note that we have not used a `String` as a type for the file name, but an
`Option`, to indicate that we either have a filename or `None`, in case no file
name is set.

Now we’re ready to display some information in the status bar. We’ll display up
to 20 characters of the filename, followed by the number of lines in the file.
If there is no filename, we’ll display [No Name] instead.

{% include hecto/show-filename.html %}

<small>[See this step on
github](https://github.com/pflenker/hecto-tutorial/tree/show-filename)</small>

We make sure to cut the status string short in case it doesn’t fit inside the
width of the window. Notice how we still use code that draws spaces up to the
end of the screen, so that the entire status bar has a white background.

We are using a new macro here, `format!`. It's similar to `print!` and
`println!`, without actually printing something out to the screen.

Now let’s show the current line number, and align it to the right edge of the
screen.

{% include hecto/show-line-indicator.html %}

<small>[See this step on
github](https://github.com/pflenker/hecto-tutorial/tree/show-line-indicator)</small>

The current line is stored in `cursor_position.y`, which we add 1 to since the
position is 0-indexed. We are subtracting the length of the new part of the
status bar from the number of spaces we want to produce, and add it to the final
formatted string.

## Status message

We're going to add one more line below our status bar. This will be for
displaying messages to the user, and prompting the user for input when doing a
search, for example. We'll store the current message in a struct called
`StatusMessage`, which we'll put in the editor state. We'll also store a
timestamp for the message, so that we can erase it a few seconds after it's been
displayed.

{% include hecto/show-status-bar.html %}

<small>[See this step on
github](https://github.com/pflenker/hecto-tutorial/tree/show-status-bar)</small>


We initialize `status_message` to a help message with the key bindings. We also
take the opportunity and set the status message to an error if we can't open the
file, something that we silently ignored before. To do that, we have to
rearrange the code to open a document a bit, so that the correct doc is loaded
and the correct status message is set.

Now that we have a status message to display, we can modify the
`draw_message_bar()` function.

First we clear the message bar with `Terminal::clear_current_line();`. We did
not need to do that for the  status bar, since we are always overwriting the
full line on every render. Then we make sure the message will fit the width of
the screen, and then display the message, but only if the message is less than 5
seconds old.

This means that we keep the old status message around, even if we are no longer
displaying it. That is ok, since that data structure is small anyways and is not
designed to grow over time.

When you start up the program now, you should see the help message at the
bottom. It will disappear *when you press a key* after 5 seconds. Remember, we
only refresh the screen after each key press.

In the next chapter, we will turn our text viewer into a text editor, allowing
the user to insert and delete characters and save their changes to disk.

## Conclusion
In this chapter, all the refactoring of the previous chapters has paid off, as
we where able to extend our editor effortlessly

I hope that, like in the last chapter, you are looking at your new text viewer
with pride. It's coming along! Let's focus on editing text in [next chapter]({%
post_url 2019-11-08-rust-text-editor-chapter-5%}).