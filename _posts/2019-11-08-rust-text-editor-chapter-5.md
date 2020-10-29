---
layout: postwithdiff
title: "Hecto, Chapter 5: A text editor"
categories: [Rust, hecto, Tutorial]
permalink: /hecto-chapter-5/
image: /assets/2019-11-08-hecto-chapter-5.png
date: 2019-11-08 00:00:06
last_modified_at: 2020-10-29
---
[Previous chapter]({% post_url 2019-11-08-rust-text-editor-chapter-4%}) - [Overview]({% post_url 2019-11-08-rust-text-editor%}) - [Appendices]({% post_url 2019-11-08-rust-text-editor-appendix%}) - [Next Chapter]({% post_url 2019-11-08-rust-text-editor-chapter-6%}) 
{: style="text-align: center"}

Now that `hecto` can read files, let's see if we can teach it to edit files as
well.

## Insert ordinary characters

Let's begin by writing a function that inserts a single character into a
`Document`, at a given position. We start by allowing to add a character into
our string at a given position.

{% include hecto/insert-characters.html %}

<small>[See this step on
github](https://github.com/pflenker/hecto-tutorial/tree/insert-characters)</small>

Let's focus first on the changes to `Row`.

Here, we handle two cases: If we happen to be at the end of our string, we push
the character onto it. This can happen if the user is at the end of a line and
keeps typing. If not, we are rebuilding our string by going through it character
by character.

We use the iterator's `take` and `skip` functions to create new iterators, one
that goes from `0` to `at` (including `at`), and another one that goes from the
element after`at` to the end. We use `collect` to combine these iterators into
strings. [`collect` is very powerful and can collect into different
collections](https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.collect).
Since there are multiple collections `collect` can create, we have to provide
the type of `result` and `remainder`, otherwise Rust wouldn't know what kind of
collection to create.

We're now also deriving `default` for `Row`. We'll use that in `Document`.

Similar to what we did in `Row`, we are handling the case where the user is
attempting to insert at the bottom of our document, for which case we create a
new row.

Now we need to call that method when a character is entered. We do that by
extending `process_keypress` in `editor`.

With that change, we can now add characters anywhere in the document. But our
cursor does not move - so we are essentially typing in our text backwards. Let's
fix that now by treating "Enter a character" as "Enter a character and go to the
right".

{% include hecto/insert-and-move.html %}

<small>[See this step on
github](https://github.com/pflenker/hecto-tutorial/tree/insert-and-move)</small>

You should now be able to confirm that putting in characters works, even at the
bottom of the file.

## Simple deletion
We now want Backspace and Delete to work.

Let's start with Delete, which should remove the character under the cursor. If
your cursor is a  line, `|`, instead of a block, "under the cursor" means "in
front of the cursor", since the cursor is a blinking line displayed on the left
side of its position. Let's start by adding a `delete` function on a `row`.

{% include hecto/simple-delete.html %}

<small>[See this step on
github](https://github.com/pflenker/hecto-tutorial/tree/simple-delete)</small>

As you can see, the code is very similar to the `insert` code we wrote before.
The difference is that in `Row`, we are not adding a character, but we are
skipping the one we want to delete when glueing together `result` and
`remainder`. And in `Document`, we do not need to handle the case where we want
to delete a row (yet), which makes that code a bit simpler than the symmertrical
`insert` code.

You should now be able to delete characters from with in a line. Let's tackle
Backspace next: Essentially, Backspace is a combination of going left and
deleting, so we adjust `process_keypress` as follows:

{% include hecto/simple-backspace.html %}

<small>[See this step on
github](https://github.com/pflenker/hecto-tutorial/tree/simple-backspace)</small>

Backspace now works within a line. We even make sure that if we are at the
beginning of the document, we are not doing a delete - otherwise, we would start
removing elements behind the cursor.

If you do a backspace at the beginning of a line, however, it moves up a line
without doing anything else. Let's fix that in the next sections.


## Complex deletion
There are two edge cases which we can't handle right now, and that is either
using Backspace at the beginning of a line, or using Delete at the end of a
line. In our case, Backspace simply goes to the left, which, at the beginning of
the line, means to go to the end of the previous line, and then attempts to
delete a character. This means that the Backspace case will be solved as soon as
we allow a delete at the end of a line.

{% include hecto/complex-delete.html %}

<small>[See this step on
github](https://github.com/pflenker/hecto-tutorial/tree/complex-delete)</small>

We start by giving `Row` the ability to append another row to it. We use this
ability in `Document`. Now, the code in `Document` looks a bit complex, and I'm
going to explain in a second why this is the case. What it does, though, is
checking if we are at the end of a line, and if a line follows after this one.
If that's the case, we remove the next line from our `vec` and append it to the
current row. If that's not the case, we simply try to delete from the current
row.

Now, why does the code look so complicated? Can't we just move the definition of
`row` up above the `if` statement to make things clearer?

This is our second big encounter with Rust's borrow checker. We can't have two
mutable references to elements within the vector at the same time, and we can't
mutate the vector while we have a mutable reference to an element in it. Why?
Because let's say we have a vector with A, B and C in it, and we have a
reference to B. A reference is like a pointer which points to where B resides in
the memory. Now we remove A, which causes B and C to move to the left. Our
reference would suddenly no longer point to B, but to C!

This means that we can't have  a reference to `row` and then delete part of the
vector. So we first read `row`'s length directly without retaining a reference.
Then we mutate the vector by removing an element from it, and then we create our
mutable reference to `row`.

Try rewriting the code if you want to, and check what the compiler tells you in
case you are interested.

## The <kbd>Enter</kbd> key
The last editor operation we have to implement is the <kbd>Enter</kbd> key. The
<kbd>Enter</kbd> key allows the user to insert new lines into the text, or split
a line into two lines. You can actually add newlines that way right now, but as
you might expect, the handling is less than optimal. This is because the
newlines are inserted as part of the row instead of resulting in the creation of
a new row.

Let's start with the easy case, adding a new row below the current one.

{% include hecto/simple-enter.html %}

<small>[See this step on
github](https://github.com/pflenker/hecto-tutorial/tree/simple-enter)</small>

We call `insert_newline` from `insert` in case a newline is handed to us. In
`insert_newline`, we are checking if enter was pressed either on the last row of
the document, or one row below it (remember that we allow navigating there). If
that is the case, we push a new row at the end of our `vec`. If that's not the
case, we insert a new row at the correct position.

Let's now handle the case where we are in the middle of a row.

{% include hecto/complex-enter.html %}

<small>[See this step on
github](https://github.com/pflenker/hecto-tutorial/tree/complex-enter)</small>

We have now added a method called `split`, which truncates the current row up
until a given index, and returns another row with everything behind that index.
On Document, we are now only pushing an empty row if we are below the last line,
in any other case, we are changing the current row with `split` and insert the
result. This works even at the end of a line, in which case the new row would
simply contain an empty string.

Great! Now we can move around our document, add whitespaces, characters, even
emojis, remove lines and so on. But editing is obviously useless without saving,
so let's handle that next.

## Save to disk
Now that we’ve finally made text editable, let’s implement saving to disk. We
start with implementing a `save` method in `Document`:

{% include hecto/save-to-disk.html %}

<small>[See this step on
github](https://github.com/pflenker/hecto-tutorial/tree/save-to-disk)</small>

We extend `Row` with a method that allows us to convert the row into a `byte`
array. In `Document`, `write_all` takes that byte array and writes it to disk.
Since our rows do not contain the newline symbol, we write it out separately.
The `b` in front of the newline string indicates that this is a byte array and
not a string.

Since writing may cause errors, our `save` function returns a `Result`, and we
are using the `?` again to pass any errors that might occur to the caller. 

In `Document`, we connect `save` to <kbd>Ctrl-S</kbd>. We check if the write was
successful by using `is_ok`, which returns `true` in case a `Result` is `Ok` and
not an `Err`, and we set the status message accordingly.

Last but not least, we change the initial status to tell our user how to write a
file.


Great, now we can open, adjust and save our files!

## Save as...
Currently, when the user runs `hecto` with no arguments, they get a blank file
to edit but have no way of saving. Let's make a `prompt()` function that
displays a prompt in the status bar, and lets the user input a line of text
after the prompt:

{% include hecto/prompt-for-filename.html %}

<small>[See this step on
github](https://github.com/pflenker/hecto-tutorial/tree/prompt-for-filename)</small>

The user's input is stored in `result`, which we initialize as an empty string.
We enter an infinite loop that repeatadly sets the status message, refreshes the
screen, and waits for a key press to handle. When the user presses enter, the
status message is cleared and the message is returned. The errors which might
occur on the way are propagated up.

Now that the user can save the file, let's handle a few more cases in our
prompt. Let's now allow the user to cancel and backspace, and let's also treat
an empty input as cancelling.

{% include hecto/enhance-prompt.html %}

<small>[See this step on
github](https://github.com/pflenker/hecto-tutorial/tree/enhance-prompt)</small>

We have changed a couple of things here, let's go through them one by one.

`prompt` no longer only contains a `Result`, but also an `Option`. The idea is
that if the prompt is successful, it can still return `None` to indicate that
the user has aborted the prompt. We have traded our `if let` for a `match` to
also handle the cases of backspace and `Esc`: In the case of `Esc`, we reset all
previously entered text before breaking the loop. In case of `Backspace`, we
reduce the input by one, removing the last character in the progress.

Then, we have created a function `save` outside of `process_keypress`. In it, we
are now aborting the save operation if the prompt has returned `None`, but also
if the prompt has returned an error. 

## Dirty flag

We'd like to keep track of whether the text loaded in our editor differs from
what's in the file. Then we can warn the user that they might lose unsaved
changes when they try to quit.

We call a `Document` "dirty" if it has been modified since opening or saving the
file. Let's add a `dirty` variable to the `Document` and initialize it with
`false`. We don't want this to be modified from the outside, so we add a
read-only `is_dirty` function to `Document`. We're also setting it to `true` on
any text change, and to `false` on `save`.

{% include hecto/dirty-flag.html %}

<small>[See this step on
github](https://github.com/pflenker/hecto-tutorial/tree/dirty-flag)</small>

Perhaps the only surprising change is that we rearranged the bounds handling in
`insert` a bit. It's now similar to the checking in `delete`.

## Quit confirmation

Now we’re ready to warn the user about unsaved changes when they try to quit. If
`document.is_dirty()`  is true, we will display a warning in the status bar, and
require the user to press Ctrl-Q three more times in order to quit without
saving.

{% include hecto/quit-confirmation.html %}

<small>[See this step on
github](https://github.com/pflenker/hecto-tutorial/tree/quit-confirmation)</small>

We have added a new constant for the additional times we require the user to
press <kbd>Ctrl-Q</kbd>, and we use it as an additional field in `Editor`. When
the document is dirty and the user attempts to quit, we count down `quit_times`
until it reaches `0` - then we finally quit. Note that we are returning within
the `match` arm for quit. That way, the code after the `match` is only called if
the user pressed another key than <kbd>Ctrl-Q</kbd>, so we can check after the
`match` if `quit_times` has been modified and reset it if necessary.

## Final touches
Congratulations, you have built a text editor! But before we move on and add
more functionality in it, let's check if we really covered our bases. Earlier in
this tutorial, we cared about overflows and `saturated_adds` and so on, but are
we really prepared to handle larger files, or will `hecto` panic? Also, since
Rust is all about performance, does our code perform well enough?

First, let's teach Clippy a few new tricks. 

{% include hecto/stricter-clippy.html %}

<small>[See this step on
github](https://github.com/pflenker/hecto-tutorial/tree/stricter-clippy)</small>

`clippy::restriction` contains a lot of warnings that might or might not
indicate errors in your code. As you can see when you run `cargo clippy` now,
the results are overwhelming!

Lucky for us, each entry comes with a link and with that link come some
explanations. Let's disable a few items for `hecto`:

{% include hecto/sensible-clippy.html %}

<small>[See this step on
github](https://github.com/pflenker/hecto-tutorial/tree/sensible-clippy)</small>

If you are interested in what these options are, check their descriptions in the
original output of clippy.

The results of Clippy are now much more manageable. There are still many entries
regarding integer arithmetic. Before we get to them, let's ask ourselves: Do we
really want to fix all of them? My opinion is: Yes, for two reasons. One is that
some of our code relies on implicit contracts behind some functions: We rely
other portions of our code to do the checking for us so that we don't have to.
But what if in the future the other part of the code changes? 

Another consideration is that if you come across code like `a+1;`, you have to
stop and investigate the surrounding code to check if this operation is valid.
You have no indication whether or not the author of this code (which could
simply be your Past Self) has paid attention to the potential overflow or not!
The easiest thing to do is to disable clippy at this line. Even this lazy
solution is an indicator for everyone reviewing the code later that you did, in
fact, take overflows into account and made a conscious decision on how to deal
with it.

Anyways, let's jump right in!

### Making Clippy happy

{% include hecto/integer-arithmetic-1.html %}

<small>[See this step on
github](https://github.com/pflenker/hecto-tutorial/tree/integer-arithmetic-1)</small>

As you can see, these are mainly small changes. I want to point out a few
things:

- We have found several potential bugs or head scratchers. For instance,
  after one of our last changes, `insert_newline` did not do some bounds
  checking on its own. From looking at `insert_newline` alone it would not be
  possible to understand that we removed it because right now, `insert_newline`
  is only called from `insert`, where we are already doing bounds checking. This
  means that there was an implicit contract to whoever calls `insert_newline` to
  make sure `at.y` is not exceeding the current document's length. We have now
  corrected this.
- We have replaced a subtraction from `len` with an addition to `at.y` in
  another place. Why? Because we can easily see in that function that `y` will
  always be smaller than `len`, so there is always room for a `+1`. It's not as
  easily visible that, or if, `len` will always be greater than 0.
- While using `saturating_sub`, we were able to get rid of some size
  comparisons, which simplifies our code.

Clippy still gives us some warnings, this time about integer division. The
problem is the following: If you divide, for example, `100/3`, the result will
be `33`, and the remainder will be removed. That is OK in our case, but the
reason to tackle this anyways is the same as before: Anybody reviewing our code
can't be sure whether or not we thought about this, or simply forgot. The least
we can do is either leave a comment or a clippy directive, which is essentially
the same as a comment saying "Trust me, this stuff is working".

{% include hecto/integer-arithmetic-2.html %}

<small>[See this step on
github](https://github.com/pflenker/hecto-tutorial/tree/integer-arithmetic-2)</small>

As you can see, we opted for the "Leave a comment" solution here. We also
re-indented some code, as the lines tend to get longer now. Let's tackle clippy's
next grievances now.

{% include hecto/indexing-panic-1.html %}

<small>[See this step on
github](https://github.com/pflenker/hecto-tutorial/tree/indexing-panic-1)</small>

This change is mainly related to accessing rows at certain positions. We used to
use the safe method `get_mut`, which does not panic, even if there was nothing
to retrieve, e.g. because the index was wrong. And we called `unwrap()` directly
on it, negating the benefits of using `get_mut` in the first place. We have
replaced it now with a direct access to `self.rows`. We have left a clippy
statement everywhere we did this, to indicate that we did, in fact, check that
we're only accessing valid indices at that time. 

If you wanted to make your code even more robust, you could replace these
occurrences with proper handling in case an index is out of bounds.

We also changed another thing: There was another implicit contract thing going
on, and that was that the length of a document always corresponds to the number
of rows in it, so we were calling `self.len()` instead of `self.rows.len()`
everywhere. Should we ever decide that documents can be longer, all our
operations on `self.row` would fail.

This is not a super important change, but it fits the spirit of our current
refactorings. All right, two more clippy warnings to go.

{% include hecto/indexing-panic-2.html %}

<small>[See this step on
github](https://github.com/pflenker/hecto-tutorial/tree/indexing-panic-2)</small>

We're taking advantage of the fact that `get`, as discussed above, only returns
a value to us if it is there, eliminating the need to perform a check before
indexing. Then, we remove `is_ok` in favor of an `if_let`, to save an `unrwap`.
Finally, we have convinced both clippy and ourselves that our code is good!

Now, there are a few things that Clippy can't detect, and we should not rely on
Clippy alone for our development. We'll deal with them next.

### Performance improvements
So far, our editor does not do much. But there are already some performance
improvements that we could make! Performance tweaks are a difficult topic, as
it's hard to draw the line between readable, maintainable code and
super-optimized code that is hard to read and maintain. We don't want `hecto` to
be the fastest editor out there, but it makes sense to take a look at a few
performance aspects. 

I want us to look out for unnecessary iterations over rows when going through
the document from top to bottom, as well as unnecessary iterations over
characters as we go through a row left to right. That's all that we are going to
focus on now - no additional caching, no fancy tricks, just looking for
redundant operations.

Let's focus on how we deal with rows. We have a common pattern that we are
repeating multiple times. For instance, here's `insert`:

```rust
pub fn insert(&mut self, at: usize, c: char) {
        if at >= self.len() {
            self.string.push(c);
        } else {
            let mut result: String = self.string[..].graphemes(true).take(at).collect();
            let remainder: String = self.string[..].graphemes(true).skip(at).collect();
            result.push(c);
            result.push_str(&remainder);
            self.string = result;
        }
        self.update_len();
    }
```
In this implementation, we iterate over our string three times: 

- Once from start to `at` to calculate `result`
- A second time from start to end (ignoring everything between start and `at`)
  to calculate `remainder`
- and once through the whole string to update `len`. 

That's not great. Let's change this.

{% include hecto/better-row-performance.html %}

<small>[See this step on
github](https://github.com/pflenker/hecto-tutorial/tree/better-row-performance)</small>

We did two things here:

- We got rid of `update_len`, as we are now manually calculating the length on
  every row operation.
- We are looping over `enumerate`, which does not only provide us with the next
  element, but also with it's index in the iterator. That way, we can easily
  calculate the length while we are moving through the row.

### Final considerations
While we undoubtedly have made `hecto` better with these changes, let's put
things a bit into perspective: How likely is it that we will ever see an
overflow happening on `usize`? Well, that depends on your system. You can check
the actual size of `usize` with the following snippet:

```rust
fn main() {
    dbg!(std::usize::MAX);        
}
```

On my machine, it outputs the following:

```
[src/main.rs:2] std::usize::MAX = 18446744073709551615
```
That means, we are near an overflow as soon as our doc is close to
18,446,744,073,709,551,615 rows, or if a row has close to
18,446,744,073,709,551,615 characters long. 

That is an insanely large number. If every row contained a byte of information,
that would be 18 EB of data. EB stands for Exabyte. Good luck finding a hard
drive that can handle these data! And even if you could, `hecto` would run into
other issues while handling this insane amount of data.

This does not mean that our work was not important. On the contrary, I believe
that thinking about these kinds of things should become a habit while you are
coding. However, you should not over-optimize your code for an edge case that
will never happen.

## Conclusion
You have now successfully built a text editor. If you are brave, you can use
`hecto` to work on `hecto`. In the [next chapter]({% post_url
2019-11-08-rust-text-editor-chapter-6%}), we will make use of `prompt()` to
implement an incremental search feature in our editor.