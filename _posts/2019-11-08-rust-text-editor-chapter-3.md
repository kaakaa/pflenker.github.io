---
layout: postwithdiff
title: "Hecto, Chapter 3: Raw input and output"
categories: [Rust, hecto, Tutorial]
permalink: /hecto-chapter-3/
image: /assets/2019-11-08-hecto-chapter-3.png
date: 2019-11-08 00:00:04
last_modified_at: 2020-10-29
---
[Previous chapter]({% post_url 2019-11-08-rust-text-editor-chapter-2%}) - [Overview]({% post_url 2019-11-08-rust-text-editor%}) - [Appendices]({% post_url 2019-11-08-rust-text-editor-appendix%}) - [Next Chapter]({% post_url 2019-11-08-rust-text-editor-chapter-4%}) 
{: style="text-align: center"}
In this chapter, we will tackle reading from and writing to the terminal. But
first, we need to make our code more idiomatically. Beware! The beginning of
this chapter will contain a lot of prose, which you can safely skip if you are
not interested.

## Writing idiomatic code
Whenever you deal with newer languages such as Rust, you will often hear about
_idiomatic_ code. Apparently, it is not enough to make the code solve a problem,
it should be _idiomatic_ as well. Let's discuss first why this makes sense.
Since the term _idiomatic_ comes from languages,  we start off with a linguistic
example: If I told you that with this tutorial, you could _kill two flies with
one swat_, because you learn Rust and write your very own editor at the same
time, would you understand my meaning?

If you are a German, you would probably not have any trouble, because "Killing
to flies with one swat" is a near-literal translation of a German saying. If you
are Russian, "killing two rabbits with one shot" would be more understandable
for you. But if you are not familiar with German or Russian, you would have to
try and extract the meaning of these sentences out of the context. The
_idiomatic_ way of saying this in English is, of course, "To kill two birds with
one stone". The point is: Using the correct idiom in english makes sure everyone
understands the meaning without thinking about it. If you speak unidiomatically,
you force people to think about the wording you're using, and not the arguments
you're making.

Writing idiomatic code is similar. It's easier to maintain for others, since it
sticks to certain rules and conventions, which are what the designers of the
language had in mind. Your code is typically only reviewed when it doesn't work
- either because a feature is missing and someone wants to extend it, or because
it has a bug. Making it easier for others to read and understand your code is
generally a good idea.

We saw before that the Rust compiler can give you some advise on idiomatic code
- for instance, when we prefixed a variable that we where not using with an
underscore. My advise is to not ignore compiler warnings, your final code should
always compile without warnings.

In this tutorial, though, we are sometimes adding functionality a bit ahead of
time, which will cause Rust to complain about unreachable code. This is usually
fixed a step or two later.

Let's start this chapter by making our code a bit more idiomatic.

## Read keys instead of bytes
In the previous steps, we have worked directly on bytes, which was both fun and
valuable. However, at a certain point you should ask yourself if the
functionality you are implementing could not be replaced by a library function,
as in many cases, someone else has already solved your problem, and probably
better. For me, handling with bit operations is a huge red flag that tells me
that I am probably too deep down the rabbit hole. Fortunately for us, our
dependency, `termion`, makes things already a lot easier, as it can already
group individual bytes to key presses and pass them to us. Let's implement this.

{% include hecto/use-termion-keys.html %}

<small>[See this step on
github](https://github.com/pflenker/hecto-tutorial/tree/use-termion-keys)</small>

We are now working with keys instead of bytes. With that change, we where able
to get rid of manually checking if <kbd>Ctrl</kbd> has been pressed, as all keys
are now properly handled for us. Termion provides us with values which represent
keys: `Key::Char(c)` represents single character presses, `Key::Ctrl(c)`
represents all characters pressed together with <kbd>Ctrl</kbd>, `Key::Alt(c)`
represents all characters pressed together with <kbd>Alt</kbd> and so on.

We are still mainly interested in characters and in <kbd>Ctrl-Q</kbd>.

Note the subtle difference in the inner match: `Key::Char(c)` matches _any_
Character and binds it to the variable `c`, whereas `Key::Ctrl('q')` matches
specifically <kbd>Ctrl-q</kbd>.

Before the change, we were working with bytes which we converted to characters
in order to print them out. Now, Termion hands us the characters, so in order to
print out the byte value, we use `c as u8` for the conversion.

We have also added another case to our inner `match`, and that is a special
case: The case `_` is called for every case that has not already been handled.

Matches need to be _exhaustive_, so every possibility must be handled. `_` is
the default option for everything that has not been handled before. In this
case, if anything is pressed that is neither a character nor <kbd>Ctrl-Q</kbd>,
we just print it out.

We also had to import a few things to make our code working. Similar as with
`into_raw_mode`, we need to import `TermRead` so that we can use the `keys`
method on `stdin`, but we could delete the import for `std::io::Read` in return.

## Separate the code into multiple files
It's idiomatic that the main method itself does not do much more than providing
the entry point for the app. This is the same in many programming languages, and
Rust is no exception. We want the code to be placed where it makes sense, so
that it's easier to locate and maintain code later. There are more benefits as
well, which I will explain when we encounter them.

The code we have is too low-level for now. We have to understand the whole code
to understand that, in essence, it simply echoes any pressed key to the user and
quits if <kbd>Ctrl-Q</kbd> is being pressed. Let's improve this code by creating
a code representation of our editor.

Let's start with a new file:

{% include hecto/create-struct.html %}

<small>[See this step on
github](https://github.com/pflenker/hecto-tutorial/tree/create-struct)</small>

A `struct` is a collection of variables and, eventually, functions which are
grouped together to form a meaningful entity - in our case, the Editor (It's not
very meaningful yet, but we'll get to that!). The `pub` keyword tells us that
this struct can be accessed from outside the `editor.rs`. We want to use it from
`main`, so we use `pub`. This is already the next advantage of separating our
code: We can make sure that certain functions are only called internally, while
we expose others to other parts of the system.

Now, our editor needs some functionality. Let's provide it with a `run()`
function, like this:

{% include hecto/editor-run-implementation.html %}

<small>[See this step on
github](https://github.com/pflenker/hecto-tutorial/tree/editor-run-implementation)</small>

You already know the implementation of `run` - it's copy-pasted from our `main`,
and so are the imports and `die`.

Let's focus on what's new: The `impl` block contains function definition which
can be called on the struct (We see how this works in a second). The function
gets the `pub` keyword, so we can call it from the outside. And the `run`
function accepts a parameter called `&self`, which will contain a reference to
the struct it was called upon (The `&` before `self` indicates that we are
dealing with a reference). This is equivalent to having a function outside of
the `impl` block which accepts a `&Editor` as the first parameter.

Let's see this working in practice by refactoring our `main.rs`:

{% include hecto/use-editor-struct.html %}

<small>[See this step on
github](https://github.com/pflenker/hecto-tutorial/tree/use-editor-struct)</small>


As you can see, we have removed nearly everything from the `main.rs`. We are
creating a new instance of `Editor` and we call `run()` on it. If you run the
code now, you should see that it works just fine. Now, let's make the last
remaining lines of the `main` a bit better. Structs allow us to group variables,
but for now, our struct is empty - it does not contain any variables. As soon as
we start adding things to the struct, we have to set all the fields as soon as
we create a new `Editor`. This means that for every new entry in `Editor`, we
have to go back to the `main` and change the line `let editor =
editor::Editor{};` to set the new field values. That is not great, so let's
refactor that.

Here is the change:

{% include hecto/default-editor.html %}

<small>[See this step on
github](https://github.com/pflenker/hecto-tutorial/tree/default-editor)</small>

We have now created a new function called `default`, which constructs a new
`Editor` for us. Note that the one line in `default` does not contain the
keyword `return`, and it does not end with a `;`. Rust treats the result of the
last line in a function as its output, and by omitting the `;`, we are telling
rust that we are interested in the value of that line, and not only in executing
it. Play around with that by adding the `;` and seeing what happens.

Unlike `run`, `default` is not called on an already-existing `Editor` instance.
This is indicated by the missing `&self` parameter in the function signature of
`default`. This is called a `static` method, and these are called by using `::`
as follows: `Editor::default()`.

Now, we can leave the `main.rs` alone while we focus on the `editor.rs` .

## "It looks like you're writing a program, would you like help?"
Let's conclude our detour towards idiomatic code by using another very useful
feature: Clippy. Clippy is both an annoying Windows 95 feature, and a mechanism
to point out possible improvements in our code. You can run it from the command
line by executing:

```bash
$ cargo clippy
```
Running clippy now does not produce a result - our code is good enough to pass
Clippy's default flags. However, that's not good enough for us - we want Clippy
to annoy us, so that we can learn from it. First, we run `cargo clean`, as
Clippy only creates output during compilation, and as we saw earlier, Cargo only
compiles changed files.

```bash
$ cargo clean
$ cargo clippy -- -W clippy::pedantic
```

The output is now:

```
    Compiling libc v0.2.62
    Checking numtoa v0.1.0
    Checking termion v1.5.3
    Checking hecto v0.1.0 (/home/philipp/repositories/hecto)
warning: unnecessary structure name repetition
  --> src/editor.rs:30:9
   |
30 |         Editor{}
   |         ^^^^^^ help: use the applicable keyword: `Self`
   |
   = note: `-W clippy::use-self` implied by `-W clippy::pedantic`
   = help: for further information visit https://rust-lang.github.io/rust-clippy/master/index.html#use_self

    Finished dev [unoptimized + debuginfo] target(s) in 6.16s
```

Not only does Clippy point out a weakness in our code, it also provides a link
to the documentation, so that we can read all about that error. That's great!

We can tell Clippy which flags we want to be used by default, for instance by
adding code to our `main.rs`. Let's do this, and also fix the issue it pointed
out:

{% include hecto/use-clippy.html %}

<small>[See this step on
github](https://github.com/pflenker/hecto-tutorial/tree/use-clippy)</small>

The pedantic setting is really valuable for beginners: As we don't know yet how
to write code idiomatically, we need someone at our side who points out how
things could be done better.

## Separate reading from evaluating

Let's make a function for keypress reading, and another function for mapping key
presses to editor operations. We'll also stop printing out keypresses at this
point.

{% include hecto/reading-evaluating.html %}

<small>[See this step on
github](https://github.com/pflenker/hecto-tutorial/tree/reading-evaluating)</small>

We have now added a `loop` to `run`. Loops are repeated forever until they are
explicitly interrupted. Within that loop we use another feature of Rust: `if
let`. This is a shortcut for using a `match` where we only want to handle one
case and ignore all other possible cases. Look at the code of
`process_keypress()` to see a case of `match` which could be fully replaced by
`if let`.

In `run`, we execute `self.process_keypress()` and see if the result matches
`Err`. If so, we pass the unwrapped error to `die`, if not, nothing happens.

We can see this more clearly while investigating the signature of
`process_keypress`:

```rust
   fn process_keypress(&self) -> Result<(), std::io::Error>
```
The part behind the `->` says: This function returns a `Result`. The stuff in
`<>` tells us what to expect as contents of `Ok` and `Err`, respectively: `Ok`
will be wrapping `()`, which means "Nothing", and `Err` will be wrapping
`std::io::Error`.

`process_keypress()` waits for a keypress, and then handles it. Later, it will
map various <kbd>Ctrl</kbd> key combinations and other special keys to different
editor functions, and insert any alphanumeric and other printable keys'
characters into the text that is being edited. That's why we are using `match`
instead of `if let` here.

The last line of this function is a bit difficult to understand for beginners.
Conceptually, we don't want the function to return anything. So why the
`Ok(())`? The thing is: Since an error can occur when calling `read_key`, we
want to pass that error up to the calling function. Since we don't have a
`try..catch`, we have to return something that says "Everything is OK", even
though we are not returning any value. That is precisely what `Ok(())` does: It
says "Everything is OK, and nothing has been returned".

But what if something goes wrong? Well, we can tell by the signature of
`read_key` that an error can be passed to us. If that's the case, there's no
point in continuing, we want the error to be returned as well. But in case no
error occurred, we want to continue with the unwrapped value.

That's what the question mark after `read_key` does for us: If there's an error,
return it, if not, unwrap the value and continue.

Try playing around with these concepts by removing the `?`, or the `Ok(())`, or
by changing the return value.

`read_key` also includes a loop. In that case, the loop is repeated until the
function returns, that is, as soon a valid key has been pressed. The value
returned by `io::stdin().lock().keys().next()` is very similar to the `Result`
we just discussed - It's a so-called `Option`. We will use `Option`s in depth
later. For now, it's enough to understand that an `Option` can be `None` -
meaning in this case that no key has been pressed and continuing the loop. Or it
can wrap a value with `Some`, in which case we return that unwrapped value from
`read_key`.

What makes this slightly more complicated is that the actual return value of
`io::stdin().lock().keys().next()` is a `Key` wrapped inside of a `Result`
which, in turn, is wrapped inside an `Option`. We unwrap the `Option` in
`read_key()`, and the `Result` in `process_keypress()`.

That is how the error makes its way into `run`, where it is finally handled by
`die`. Speaking of `die`, there is a new ugly wart in our code: Because we don't
know yet how to exit our code from within the program, we are `panic`king now
when the user uses <kbd>Ctrl-Q</kbd>.

We could instead call the proper method to end the program
(`std::process::exit`, in case you are interested), but similar to how we do not
want our program to crash randomly deep within our code, we also don't want it
to exit somewhere deep down, but in `run`. We solve this by adding our first
element to the `Editor` struct: a boolean which indicates if the user wants to
quit.

{% include hecto/add-should-quit.html %}

<small>[See this step on
github](https://github.com/pflenker/hecto-tutorial/tree/add-should-quit)</small>

We have to initialize `should_quit` in `default` right away, or we won't be able
to compile our code. Let's set the boolean now and quit the program when it is
`true`.

{% include hecto/exit-without-panic.html %}

<small>[See this step on
github](https://github.com/pflenker/hecto-tutorial/tree/exit-without-panic)</small>

Instead of panicking, we are now setting `should_quit`, which we check in `run`.
If it's `true`, we use the keyword `break` to end the loop. You should confirm
that exiting the program is now cleaner than it was before.

In addition to this change, we had to do a couple of other things. Since we are
_mutating_ `self` now in `process_keypress()`, we had to change `&self` to `&mut
self` in the signature. This indicates that we intend to mutate the reference
we're having. Rust is very strict about mutable references, as we will see
later.

Similarly, we had to change the signature from `run`, since we call
`process_keypress()` from within. 

Last but not least, we had to change `main`. `let editor = ...` indicates that
`editor` is a read-only reference, so we can't `run` on it, which mutates
`editor`. We could have solved this by changing it to `let mut editor`. Instead,
since we're not doing anything with `editor`, we have now removed the extra
variable and are calling `run()` now directly on the return value of
`default()`.

Now we have simplified `run()`, and we will try to keep it that way.

## Clear the screen

We're going to render the editor's user interface to the screen after each
keypress. Let's start by just clearing the screen.

{% include hecto/clear-screen.html %}

<small>[See this step on
github](https://github.com/pflenker/hecto-tutorial/tree/clear-screen)</small>

We add a new function `refresh_screen` which we are calling before exiting the
program. We move `process_keypress()` down, which means that after a user exits
the program, we still refresh the screen one more time before exiting. This will
allow us to print an exit message later.

To clear the screen, we use `print` to  write `4` bytes out to the terminal. The
first byte is `\x1b`, which is the escape character, or `27` in decimal. The
other three bytes are `[2J`.

We are writing an *escape sequence* to the terminal. Escape sequences always
start with an escape character (`27`, which, as we saw earlier, is also produced
by <kbd>Esc</kbd>) followed by a `[` character. Escape sequences instruct the
terminal to do various text formatting tasks, such as coloring text, moving the
cursor around, and clearing parts of the screen.

We are using the `J` command ([Erase In
Display](http://vt100.net/docs/vt100-ug/chapter3.html#ED)) to clear the screen.
Escape sequence commands take arguments, which come before the command. In this
case the argument is `2`, which says to clear the entire screen. `\x1b[1J` would
clear the screen up to where the cursor is, and `\x1b[0J` would clear the screen
from the cursor up to the end of the screen. Also, `0` is the default argument
for `J`, so just `\x1b[J` by itself would also clear the screen from the cursor
to the end.

In this tutorial, we will be mostly looking at
[VT100](https://en.wikipedia.org/wiki/VT100) escape sequences, which are
supported very widely by modern terminal emulators. See the [VT100 User
Guide](http://vt100.net/docs/vt100-ug/chapter3.html) for complete documentation
of each escape sequence.

After writing out to the terminal, we call `flush()`, which forces `stdout` to
print out everything it has (it might buffer some values and not print them out
directly). We are also returning the result of `flush()`, which, similar as
above, either wraps nothing or an error in case the flushing failed. This is
hard to miss: Had we added a `;` after `flush()`, we would not have returned its
result.

`termion` eliminates the need for us to write the escape sequences directly to
the terminal ourselves, so let's change our code as follows:

{% include hecto/clear-screen-with-termion.html %}

<small>[See this step on
github](https://github.com/pflenker/hecto-tutorial/tree/clear-screen-with-termion)</small>

From here on out, we will be using `termion` directly in the code instead of the
escape characters.

By the way, since we are now clearing the screen every time we run the program,
we might be missing out on valuable tips the compiler might give us. Don't
forget that you can run `cargo build` separately to take a look at the warnings.
Remember, though, that Rust does not recompile your code if it hasn't changed,
so running `cargo build` immediately after `cargo run` won't give you the same
warnings. Run `cargo clean` and then run `cargo build` to recompile the whole
project and get all the warnings.

## Reposition the cursor

You may notice that the `\x1b[2J` command left the cursor at the bottom of the
screen. Let's reposition it at the top-left corner so that we're ready to draw
the editor interface from top to bottom.

{% include hecto/reposition-cursor.html %}

<small>[See this step on
github](https://github.com/pflenker/hecto-tutorial/tree/reposition-cursor)</small>

The escape sequence behind `termion::cursor::Goto`  uses the `H` command
([Cursor Position](http://vt100.net/docs/vt100-ug/chapter3.html#CUP)) to
position the cursor. The `H` command actually takes two arguments: the row
number and the column number at which to position the cursor. So if you have an
80&times;24 size terminal and you want the cursor in the center of the screen,
you could use the command `\x1b[12;40H`. (Multiple arguments are separated by a
`;` character.) As rows and columns are numbered starting at `1`, not `0`, the
`termion` method is also 1-based.

## Clear the screen on exit

Let's clear the screen and reposition the cursor when our program crashes. If an
error occurs in the middle of rendering the screen, we don't want a bunch of
garbage left over on the screen, and we don't want the error to be printed
wherever the cursor happens to be at that point. We also take the opportunity to
print out a farewell message in case the user decides to leave `hecto`.

{% include hecto/clear-on-exit.html %}

<small>[See this step on
github](https://github.com/pflenker/hecto-tutorial/tree/clear-on-exit)</small>

## Tildes
It's time to start drawing. Let's draw a column of tildes (`~`) on the left hand
side of the screen, like [vim](http://www.vim.org/) does. In our text editor,
we'll draw a tilde at the beginning of any lines that come after the end of the
file being edited.

{% include hecto/draw-tildes.html %}

<small>[See this step on
github](https://github.com/pflenker/hecto-tutorial/tree/draw-tildes)</small>

`draw_rows()` will handle drawing each row of the buffer of text being edited.
For now it draws a tilde in each row, which means that row is not part of the
file and can't contain any text.

We don't know the size of the terminal yet, so we don't know how many rows to
draw. For now we just draw `24` rows (The `_` in `for..in` indicates that we are
not interested in any value, we just want to repeat the command a bunch of
times)

After we're done drawing, we reposition the cursor back up at the top-left
corner.

## Window size
Our next goal is to get the size of the terminal, so we know how many rows to
draw in `editor_draw_rows()`. It turns out that `termion`  provides us with a
method to get the screen size. We are going to use that in a new data structure
which represents the terminal. We place it in a new file called `terminal.rs`.

{% include hecto/add-terminal-struct.html %}

<small>[See this step on
github](https://github.com/pflenker/hecto-tutorial/tree/add-terminal-struct)</small>

Let's focus first on the contents of the new file. In it, we define `Terminal`
and a helper struct called `Size`. In `default`, we are getting termion's
`terminal_size`, convert it to a `Size` and return the new instance of
`Terminal`. To account for the potential error, we wrap it into `Ok`. 

We also don't want callers from the outside to modify the terminal size. So we
do not mark `size` as public with `pub`. Instead, we add a method called `size`
which returns a read-only reference to our internal `size`.

Let's quickly discuss the data types here. `Size`. `width` and `height` are both
`u16`s, which is an unsigned 16 bit integer and ends at around 65,000. That
makes sense for the terminal, at least for virtually every terminal I have ever
seen. 

Now that we have seen the new struct, let's investigate how it is referenced
from the editor. First, we introduce our new struct in `main.rs` the same as we
did with `editor`. Then, we say that we want to use the `Terminal` struct and
add a `pub` before that statement. What does that do?

In `editor.rs`, we can now import the terminal with `use crate::Terminal`.
Without the `pub use` statement in `main.rs`, we could not have done it like
that, instead we would have needed to use `use crate::terminal::Terminal`. In
essence, we are re-exporting the `Terminal` struct at the top level and make it
reachable via `crate::Terminal`. 

In our editor struct, we are adding a reference to our terminal while
initializing the editor in `default()`. Remember that `Terminal::default`
returns a Terminal or an Error. We unwrap the Terminal with `expect`, which does
the following: If we have a value, we return it. If we don't have a value, we
panic with the text passed to `expect`. We don't need to `die` here, since `die`
is mostly useful while we are repeatedly drawing to the screen.

There are other ways to handle that error. We could have used `expect` also
within `Terminal` - but that's not the point. The point is that Rust forces you
to think about it very early - you have to make a conscious decision on what to
do, or the program won't even compile.

Now that we have a representation of the terminal in our code, let's move a bit
of code there.

{% include hecto/more-terminal-functions.html %}

<small>[See this step on
github](https://github.com/pflenker/hecto-tutorial/tree/more-terminal-functions)</small>

What did we do? We moved all the low-level terminal stuff to `Terminal`, leaving
all the higher-level stuff in the `mod.rs`. Along the way, we have cleaned up a
few things:

- We do not need to keep track of `stdout` for the raw mode in the editor. This
  is handled internally in `Terminal` now - as long as the `Terminal` struct
  lives, `_stdout` will be present.
- We have hidden the fact that the terminal is 1-based from the caller by making
  `Terminal::cursor_position` 0-based.
- We are preventing an overflow of our `u16` in `cursor_position`. 

A few quick words on overflowing: Our types have a maximum size which they can
take. As mentioned before, this limit lies around 65,000 for `u16`. So what
happens if you add 1 to the maximum value? It becomes the smallest possible
value, so in the case of an unsigned type, 0! This is called an overflow. Why
does this happen? Let's consider a 3-bit datatype. We start at the value `000`,
which encodes 0, and whenever we want to add 1, we use the following algorithm:

- Starting at the right, if you see a `0`, flip it to a `1`.
- If you see a `1`, flip it to a `0`, move one step to the left and repeat.

This gives us the following sequence:

| Number | Binary |
|--------|--------|
| 0      | 000    |
| 1      | 001    |
| 2      | 010    |
| 3      | 011    |
| 4      | 100    |
| 5      | 101    |
| 6      | 110    |
| 7      | 111    |   


Now what happens if we try to add 1 more? All the 1s are flipped to 0, but there
is no bit left over to flip to 1. So we are back at `000`, which is 0.


By the way, the normal handling of overflows in Rust is as follows: In debug
mode (which we are using by default), the program crashes. This is what you
want: The compiler should not try to keep your program alive, but slap the bug
in your face. In production mode, an overflow occurs. This is also what you
want, because you don't want to crash your application unexpectedly in
production, instead it could make sense continuing with the overflown values. In
our case, it would mean that the cursor is not placed at the bottom or right of
the screen, but at the left, which would be annoying, but not enough to warrant
a crash. Luckily, our new code avoids this anyways: We use `saturating_add`,
which attempts to add `1`, and if that's not possible, it just returns the
maximum value.

(If you want to try out the overflow logic of Rust: you can build your
application for production with `cargo build --release`, which will place the
production executable in `target/release`).

## The last line

Maybe you noticed the last line of the screen doesn't seem to have a tilde.
That's because of a small bug in our code. When we print the final tilde, we
then print a `"\r\n"` like on any other line, but this causes the terminal to
scroll in order to make room for a new, blank line. Since we want to have a
status bar at the bottom later anyways, let's just change the range in which we
are drawing rows for now. We will revisit this later and make this more robust
against overflows/underflows, but for now let's focus on getting this working.

{% include hecto/ignore-last-line.html %}

<small>[See this step on
github](https://github.com/pflenker/hecto-tutorial/tree/ignore-last-line)</small>

## Hide the cursor when repainting

There is another possible source of the annoying flicker effect we will take
care of now. It's possible that the cursor might be displayed in the middle of
the screen somewhere for a split second while the terminal is drawing to the
screen. To make sure that doesn't happen, let's hide the cursor before
refreshing the screen, and show it again immediately after the refresh finishes.

{% include hecto/hide-cursor-on-refresh.html %}

<small>[See this step on
github](https://github.com/pflenker/hecto-tutorial/tree/hide-cursor-on-refresh)</small>

Under the hood, we use escape sequences to tell the terminal to hide and show
the cursor by writing `\x1b[25h`,  the `h` command ([Set
Mode](http://vt100.net/docs/vt100-ug/chapter3.html#SM)) and `\x1b[25l`, the `l`
command ([Reset Mode](http://vt100.net/docs/vt100-ug/chapter3.html#RM)). These
commands are used to turn on and turn off various terminal features or
["modes"](http://vt100.net/docs/vt100-ug/chapter3.html#S3.3.4). The VT100 User
Guide just linked to doesn't document argument `?25` which we use above. It
appears the cursor hiding/showing feature appeared in [later VT
models](http://vt100.net/docs/vt510-rm/DECTCEM.html).

## Clear lines one at a time

Instead of clearing the entire screen before each refresh, it seems more optimal
to clear each line as we redraw them. Let's replace `termion::clear::All` (clear
entire screen) escape sequence with `\x1b[K` sequence at the beginning of each
line we draw with `termion::clear::CurrentLine`.

{% include hecto/clear-current-line.html %}

<small>[See this step on
github](https://github.com/pflenker/hecto-tutorial/tree/clear-current-line)</small>

Note that we are now clearing the screen before displaying our goodbye message,
to avoid the effect of showing the message on top of the other lines before the
program finally terminates.

## Welcome message

Perhaps it's time to display a welcome message. Let's display the name of our
editor and a version number a third of the way down the screen.

{% include hecto/show-welcome-message.html %}

<small>[See this step on
github](https://github.com/pflenker/hecto-tutorial/tree/show-welcome-message)</small>

We have added a constant called `VERSION` to our code.  Since our `Cargo.toml`
already contains our version number, we use the `env!` macro to retrieve it. We
add it to our welcome message. 

However, we need to deal with the fact that our message might be cut off due to
the terminal size. We do that now.

{% include hecto/slice-welcome-message.html %}

<small>[See this step on
github](https://github.com/pflenker/hecto-tutorial/tree/slice-welcome-message)</small>

The `[...width]` syntax means that we want to slice the string from its
beginning until `width`. `width` has been calculated as the minimum of the
screen width or the welcome message length, which makes sure that we are never
slicing more of a string than what is already there.

Now let's center the welcome message, and while we're at it, let's move our code
to draw the welcome message to a separate function.

{% include hecto/center-welcome-message.html %}

<small>[See this step on
github](https://github.com/pflenker/hecto-tutorial/tree/center-welcome-message)</small>

To center a string, you divide the screen width by `2`, and then subtract half
of the string's length from that. In other words: `width/2 - welcome_len/2`,
which simplifies to `(width - welcome_len) / 2`. That tells you how far from the
left edge of the screen you should start printing the string. So we fill that
space with space characters, except for the first character, which should be a
tilde. `repeat` is a nice helper function which repeats the character we pass to
i, and `truncate` shortens a string to a specific width if necessary.

You should be able to confirm that it's working: If the string is too wide, it's
being truncated, otherwise the welcome string is centered.

## Move the cursor

Let's focus on input now. We want the user to be able to move the cursor around.
The first step is to keep track of the cursor's `x` and `y` position in the
editor state. We're going to add another `struct` to help us with that.

{% include hecto/add-position-struct.html %}

<small>[See this step on
github](https://github.com/pflenker/hecto-tutorial/tree/add-position-struct)</small>

`cursor_position` is a struct where `x` will hold the horizontal coordinate of
the cursor (the column), and `y` will hold the vertical coordinate (the row),
where `(0,0)` is at the top left of the screen. We initialize both of them to
`0`, as we want the cursor to start at the top-left of the screen.

Two considerations are noteworthy here. We are not adding `Position` to
`Terminal`, even though you might intuitively think that if we modify the cursor
position in `Terminal`, it should only be natural that we are keeping track of
it there, either. However, `cursor_position` will soon describe the position of
the cursor _in our current document_, and not on the screen. and is therefore
different from the position of the cursor on the terminal.

This is directly related to the other consideration: Even though we use `u16` as
our data type for the terminal dimensions, we are using the type `usize` for the
cursor position. As discussed before, `u16` goes up until around 65,000, which
is too small for our purposes - it would mean that `hecto` could not handle
documents longer than 65,000 lines. But how big is `usize`? THe answer is: It
depends on the architecture we are compiling for, either 32 bit or 64 bit. 

Now, let's add code to `refresh_screen()` to move the cursor to the position
stored in `cursor_position`. While we're at it, let's rewrite `cursor_position`
to accept a `Position`.

{% include hecto/use-cursor-position.html %}

<small>[See this step on
github](https://github.com/pflenker/hecto-tutorial/tree/use-cursor-position)</small>


We are using
[destructuring](https://doc.rust-lang.org/rust-by-example/flow_control/match/destructuring.html)
to initialitze `x` and `y` in `cursor_position`: `let Position{mut x, mut y} =
position;` creates new variables x and y and binds their values to the fields of
the same name in `position`.

At this point, you could try initializing `cursor_position` with a different
value, to confirm that the code works as intended so far.

We are also doing a conversion from the `usize` data type within `Position` to
`u16`. `u16` can not hold values big enough for `u16` to handle, in which case
the value is truncated. That is OK for now - we will add logic to make sure we
are always within the bounds of `u16` later - so we add a small directive here
to tell Clippy to not annoy us with this kind of error.

Speaking of annoying Clippy warnings, we are carrying around an old warning from
Clippy, which we are going to fix now. Next, we'll allow the user to move the
cursor using the arrow keys.


{% include hecto/navigate-with-arrow-keys.html %}

<small>[See this step on
github](https://github.com/pflenker/hecto-tutorial/tree/navigate-with-arrow-keys)</small>

Now you should be able to move the cursor around with those keys.

## Prevent moving the cursor off screen

Currently, you can cause the `cursor_position` values to go past the right and
bottom edges of the screen. Let's prevent that by doing some bounds checking in
`move_cursor()`.

{% include hecto/navigate-within-bounds.html %}

<small>[See this step on
github](https://github.com/pflenker/hecto-tutorial/tree/navigate-within-bounds)</small>

You should be able to confirm that you can now move around the visible area,
with the cursor staying within the terminal bounds. You can also place it on the
last line, which still does not have a tilde, a fact that is not forgotten and
will be fixed later during this tutorial.

## Navigating with <kbd>Page Up</kbd>, <kbd>Page Down</kbd> <kbd>Home</kbd> and <kbd>End</kbd>
To complete our low-level terminal code, we need to detect a few more special
key presses. We are going to map <kbd>Page Up</kbd>, <kbd>Page Down</kbd>
<kbd>Home</kbd> and <kbd>End</kbd> to position our cursor at the top or bottom
of the screen, or the beginning or end of the line, respectively.

{% include hecto/navigate-with-additional-keys.html %}

<small>[See this step on
github](https://github.com/pflenker/hecto-tutorial/tree/navigate-with-additional-keys)</small>

## Conclusion
I hope this chapter has given you a first feeling of pride when you saw how your
text editor was taking shape. We were talking a lot about idiomatic code in the
beginning, and where busy refactoring our code into separate files for quite
some time, but the payoff is visible: The code is cleanly structured and
therefore easy to maintain. Since we now know our way around Rust, we won't have
to worry that much about refactoring in the upcoming chapters and can focus on
adding functionality.

In the [next chapter]({% post_url 2019-11-08-rust-text-editor-chapter-4%}), we
will get our program to display text files.