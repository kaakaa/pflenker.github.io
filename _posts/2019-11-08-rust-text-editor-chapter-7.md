---
layout: postwithdiff
title: "Hecto, Chapter 7: Syntax Highlighting"
categories: [Rust, hecto, Tutorial]
permalink: /hecto-chapter-7/
image: /assets/2019-11-08-hecto-chapter-7.png
date: 2019-11-08 00:00:08
last_modified_at: 2020-10-28
---
[Previous chapter]({% post_url 2019-11-08-rust-text-editor-chapter-6%}) - [Overview]({% post_url 2019-11-08-rust-text-editor%}) - [Appendices]({% post_url 2019-11-08-rust-text-editor-appendix%})
{: style="text-align: center"} 

We are
almost done with our text editor - we're only missing some syntax highlighting.

## Colorful Digits
Let's start by just getting some color on the screen, as simply as possible.
We'll attempt to highlight numbers by coloring each digit character red.

{% include hecto/simple-number-highlighting.html %}

<small>[See this step on
github](https://github.com/pflenker/hecto-tutorial/tree/simple-number-highlighting)</small>

We have now converted our grapheme into a character, so that we can use
`is_ascii_digit` - which determines whether or not a character is a digit. If it
is, we change the foreground color to [a shade of
red](https://rgb.to/rgb/220,163,205) and reset it immediately afterwards.

## Refactor syntax highlighting
Now we know how to color text, but we’re going to have to do a lot more work to
actually highlight entire strings, keywords, comments, and so on. We can’t just
decide what color to use based on the class of each character, like we’re doing
with digits currently. What we want to do is figure out the highlighting for
each row of text before we display it, and then re-highlight a line whenever it
gets changed. What makes things more complicated is that the highlighting
depends on characters currently out of view - for instance, if a `String` starts
to the left of the currently visible portion of the row, we want to treat a `"`
on screen as the end of a string, and not as the start of one. Our current
strategy to look at each visible character is therefore not sufficient.

Instead, we are going to store the highlighting of each character of a row in a
vector. Let's start by adding an enum which will hold our different highlighting
types as well as a vector to hold them.

{% include hecto/highlighting-type.html %}

<small>[See this step on
github](https://github.com/pflenker/hecto-tutorial/tree/highlighting-type)</small>

Highlighting will be controlled by the `Document`, the rows do not "highlight
themselves". The reason will become clearer as we add more code - essentially,
to highlight a row, more information than what is present in the `Row` is
needed. This means that any operation on the `Row` from the outside can
potentially render the highlighting invalid. When developing functionality for
`Row`, we are going to pay special attention so that the worst that can happen
is a wrong highlighting (as opposed to a crash).

This is why we are not setting the highlight for example in `split`.

For now, we’ll focus on highlighting numbers only. So we want every character
that’s part of a number to have a corresponding `Type::Number` value in the
`highlighting` vector, and we want every other value in `highlighting` to be
`Type::None`.

Let's create a new `highlight` function in our row. This function will go
through the characters of the `string` and highlight them by setting each value
in the `highlighting` vector.

{% include hecto/add-highlighting-to-row.html %}

<small>[See this step on
github](https://github.com/pflenker/hecto-tutorial/tree/add-highlighting-to-row)</small>

The code of `highlight` is straightforward: If a character is a digit, we push
`Type::Number`, otherwise, we push `Type::None`.

Now we want to have a function which maps the `Type` to a color. Let's implement
that as a method of `Type`:

{% include hecto/color-mapping.html %}

<small>[See this step on
github](https://github.com/pflenker/hecto-tutorial/tree/color-mapping)</small>

We are returning red now for numbers and white for all other cases. Now let's
finally draw the highlighted text to the screen!

{% include hecto/apply-highlighting.html %}

<small>[See this step on
github](https://github.com/pflenker/hecto-tutorial/tree/apply-highlighting)</small>

First, we call `highlight` everywhere in `Document` where we modify a row. Then,
we refactor our rendering: We get the correct highlighting for the current index
(which we obtain by using `enumerate`). We change the color, append to the
string which we return, and change the color back again.

This works, but do we really have to write out an escape sequence before every
single character? In practice, most characters are going to be the same color as
the previous character, so most of the escape sequences are redundant. Let’s
keep track of the current text color as we loop through the characters, and only
print out an escape sequence when the color changes.

{% include hecto/improve-highlighting.html %}

<small>[See this step on
github](https://github.com/pflenker/hecto-tutorial/tree/improve-highlighting)</small>

We use `current_highlighting` to keep track of what we are currently rendering.
When it changes, we add the color change to the render string. We have also
moved the ending of the highlighting outside of the loop, so that we reset the
color at the end of each line. To allow comparison between `Type`s, we derive
`PartialEq` again.

## Colorful search results

Before we start highlighting strings and keywords and all that, let's use our
highlighting system to highlight search results. We'll start by adding `Match`
to the `HighlightingType` enum, and mapping it to the color blue in `to_color`.
{% include hecto/add-match-type.html %}

<small>[See this step on
github](https://github.com/pflenker/hecto-tutorial/tree/add-match-type)</small>


Next, we want to change `highlight` so that it accepts an optional word. If no
word is given, no match is highlighted.

{% include hecto/highlight-matches.html %}

<small>[See this step on
github](https://github.com/pflenker/hecto-tutorial/tree/highlight-matches)</small>

Before we investigate the changes in `highlight`, let's first focus on the other
changes: We allow an optional parameter to `highlight`, which we provide as
`None` everywhere we call `highlight`. We then add a method called `highlight`
to the document, which triggers a highlighting of all rows in the doc.

We use that method during our search by passing the current query in, and `None`
as soon as the search has been aborted.

Now, what about `highlight`?

First, we collect all the matches of the word we have in the current row. That
is more efficient than performing the search on every character.

We are using two new concepts while doing so. One is `while..let`, which is
similar to `if..let`, but it loops as long as the condition is satisifed. We use
that to loop through our current row, advancing the starting point for our
search directly behind the last match on every turn.

We are also using a new method to add `1`: `checked_add`. This function returns
an `Option` with the result if no overflow ocurred, or `None` otherwise. Why
can't we use `saturating_add` here? Well, let's assume our match happens to be
the very last part of the row, and we are also at the end of `usize`. Then we
can't advance `next_index` any further with `saturating_add`, it would return
the same result over and over, the `while` condition would never be false and we
would loop indefinitely.

Speaking of loops, we also changed our loop for processing characters. This
allows us to consume multiple characters at a time. We use this to push multiple
highlighting types at once while we are at the index of a match which we want to
highlight.

Try it out, and you will see that all search results light up in your editor.


## Colorful numbers
Alright, let's start working on highlighting numbers properly.

Right now, numbers are highlighted even if they're part of an identifier, such as
the 32 in `u32`. To fix that, we'll require that numbers are preceded by a
separator character, which includes whitespace or punctuation characters. We can
use `is_ascii_punctuation` and `is_ascii_whitespace` for that.

{% include hecto/highlight-numbers-after-separator.html %}

<small>[See this step on
github](https://github.com/pflenker/hecto-tutorial/tree/highlight-numbers-after-separator)</small>

We're adding a new variable `prev_separator` to check if the last character we
saw was a valid separator, after which we want to  highlight numbers properly,
or any other character, after which we don't want to highlight numbers.

We are also accessing the previous highlighting so that even if we are not
behind a separator, we continue highlighting numbers as numbers - which allows
us to highlight numbers with more than one digit.

At the end of the loop, we set `prev_is_separator` to `true` if the current
character is either an ascii punctuation or a whitespace, otherwise it's set to
`false`. Then we increment `index` to consume the character.

Now let's support highlighting numbers that contain decimal points.

{% include hecto/highlight-dot-in-numbers.html %}

<small>[See this step on
github](https://github.com/pflenker/hecto-tutorial/tree/highlight-dot-in-numbers)</small>

A `.` character that comes after a character that we just highlighted as a
number will now be considered part of the number.

## Detect file type
Before we go on to highlight other things, we're going to add filetype detection
to our editor. This will allow us to have different rules for how to highlight
different types of files. For example, text files shouldn't have any
highlighting, and Rust files should highlight numbers, strings, chars, comments
and many keywords specific to Rust.

Let's create a struct `FileType` which will hold our Filetype information for
now.

{% include hecto/introduce-filetype.html %}

<small>[See this step on
github](https://github.com/pflenker/hecto-tutorial/tree/introduce-filetype)</small>

`HighlightingOptions` will hold a couple of booleans which determine whether or
not a certain type should be highlighted. Since it's closely related to the file
type, we keep both in the same file. For now, we only add `numbers` to determine
whether or not numbers should be highlighted. We use `#[derive(Default)]`  for
this struct so that `HighlightOptions::default` returns `HighlightOptions`
initialized with default values. Since `HighlightOptions` will only contain
`bool`s, and the default for bools is `false`, this suits us well and means that
when we add a highlighting option, we only need to change it where we need it,
for everything else it will just be unused.

We implement `default` for `FileType`, this time, we set the string to `"No
filetype"` and a default `HighlightingOptions` object. For now, we only use the
default file type, even when opening files. Let's get the filetype displayed in
the editor.

{% include hecto/show-filetype.html %}

<small>[See this step on
github](https://github.com/pflenker/hecto-tutorial/tree/show-filetype)</small>

We have decided against making the name property of `FileType` directly editable
by `Document`, and we have further decided to hide away the fact that `FileType`
is a `struct` from `Editor`. As far as the editor is concerned, `FileType` is
simply a string.

Now we need a way to detect the file type and set the correct
`Highlighting_Options`.

{% include hecto/detect-filetype.html %}

<small>[See this step on
github](https://github.com/pflenker/hecto-tutorial/tree/detect-filetype)</small>

We add `from` to `FileType` to determine the file type from its name. If we
don't know the type, we simply return the `default` value. We set the file type
on `open` and on `save`. You are now able to open a file, verify it displays the
correct file type, and confirm that the file type changes when you change the
file ending on save. Very satisfying!

Now, let's actually highlight the files. For that, we need to actually do
something with the `HighlightingOptions`.

{% include hecto/highlight-with-filetype.html %}

<small>[See this step on
github](https://github.com/pflenker/hecto-tutorial/tree/highlight-with-filetype)</small>

The bulk of this change deals with passing around the highlighting options, so
that `Row` has it available when its being highlighted. Then, within
`highlight`, we simply wrap the highlighting for numbers in another `if`
statement which checks whether or not `numbers` is enabled.

You can now see that Rust files are highlighted correctly, and non-rust files
are not highlighted. But what is that? When you save a new file as a rust file,
the highlighting starts acting all weird. Very unsatisfying!

While we're at it, let's also fix another minor nit-pick: We have taken great
effort to ensure no one can write the file type and the highlighting options,
but we keep the members of `HighlightingOptions` open for anyone to edit.

{% include hecto/fix-filetype.html %}

<small>[See this step on
github](https://github.com/pflenker/hecto-tutorial/tree/fix-filetype)</small>

We are now re-highlighting each row as we save the file, and our `number`
property is now finally read-only.

A quick side note: If you're eagle-eyed, you will have noticed that the function
`numbers`  accepts `self` instead of `&self` (Notice the missing `&`). The
distinction is not important in the scope of this tutorial, and as our struct
grows, we will be reverting back to `&` soon. In a nutshell, for small structs,
it's more efficient to work directly with the value (without the `&`) than with
the reference. Try adding the `&` and check the clippy output in case you're
interested.

## Colorful strings
With all that out of the way, we can finally get to highlighting more things!
Let's start with highlighting strings.

{% include hecto/add-string-options.html %}

<small>[See this step on
github](https://github.com/pflenker/hecto-tutorial/tree/add-string-options)</small>

Now for the actual highlighting code. We will use an `in_string` variable to
keep track of whether we are currently inside a string. If we are, then we'll
keep highlighting the current character as a string until we hit the closing
quote.

{% include hecto/highlight-strings.html %}

<small>[See this step on
github](https://github.com/pflenker/hecto-tutorial/tree/highlight-strings)</small>


Let'sa go through this change from top to bottom: If `in_string` is set, then we
know that the current character can be highlighted as a string. Then we check if
the current character is the closing quote, and if so, we reset `in_string` to
`false`. Then, since we highlighted the current character, we have to consume it
by incrementing `index` and `continue`ing out of the current loop iteration. We
also set `prev_is_separator` to `true` so that if we're done highlighting the
string, the closing quote is considered a separator.

IF we're not currently in a string, then we have to check if we're at the
beginning of one by checking for the starting quote. If we are, we set
`in_string` to `true`, highlight the quote and consume it.

We have also introduced a new concept here: The [dereferencing operator,
`*`.](https://doc.rust-lang.org/book/ch04-02-references-and-borrowing.html). The
point is that `c` in this code is a reference to a character, and we check if it
is the desired character by comparing it with a reference to another character,
like `c == &'"'`. Alternatively, we can _dereference_ c and directly compare it
to the desired character. This makes the code a bit easier to read. Let's apply
this in other parts as well.

{% include hecto/use-deref.html %}

<small>[See this step on
github](https://github.com/pflenker/hecto-tutorial/tree/use-deref)</small>

We should probably take escaped quotes into account when highlighting strings
and characters. If the sequence `\"` occurs in a string, then the escaped quote
doesn't close the string in the vast majority of languages.

{% include hecto/allow-escaped-strings.html %}

<small>[See this step on
github](https://github.com/pflenker/hecto-tutorial/tree/allow-escaped-strings)</small>

If we're in a string and the current character is a `\`, _and_ there is at least
one more character in that line that comes after the backslash, then we highligh
the character that comes after the backslash with `HighlightingType::String` and
consume it. We increment `index` by `2` to consume both characters at once.

## Colorful characters

Now that strings work, let's focus on characters next. We start with the basics
first.

{% include hecto/add-character-options.html %}

<small>[See this step on
github](https://github.com/pflenker/hecto-tutorial/tree/add-character-options)</small>

You might want to do character highlighting the same way as we highlight
strings. That would create two problems, though: First, we would over-eagerly
highlight nonsense like `'this is definitely no character'`. Second, it would
not work with  [Lifetimes](https://doc.rust-lang.org/1.9.0/book/lifetimes.html):
A lifetime can be indicated with a single `'`. Our highlighting would not take
this into account and highlight everything after the opening `'` as a character.

When we highlight strings, we are also not looking for a closing character and
simply end the highlighting of a string when the line ends. That's fine, since
an unclosed string is probably a typo anyways, and that typo is easier to spot
if string-highlighting goes from the opening quotes to the end than with no
highlighting at all.

Let's see how we can implement character highlighting:

{% include hecto/highlight-characters.html %}

<small>[See this step on
github](https://github.com/pflenker/hecto-tutorial/tree/highlight-characters)</small>

We are handling character highlighting before string highlighting, but only if
we are not currently within a string. That way, we can make sure that on the one
hand side, string opening quotes are not handled within a character, and on the
other hand side, character opening quotes are not handled within a string.

If we are on a character opening quote, our plan is to handle three different
cases:

- We are on a `'` and the next character is anything other than `\`. Then we
  look at the character after the next. If it's a `'`, we highlight all three
  characters. In other words, we are handling `'*'` here.
- We are on a `'` and the next character is a `\`. If it is, we look one
  character further. If it's a `'`, we highlight all four characters. In other
  words, we are handling escaped characters, `\*'`, here.
- In all other cases, we are not highlighting the current `'` as a character and
  advance the index by one, consuming the `'`.

Looking at the code, we start by setting `prev_is_separator` to `true`: In all
three cases, when we are done, we want the last consumed character to be treated
as a separator, as it will always be a `'`.

Next, we are looking at the next character to determine where we expect the `'`
to match the opening quote. If we are looking at a `\` as the next character, we
expect the closing character being the 3rd character after the current
character. Otherwise, we expect it to be the 2nd.

We are then looking at the character we expect to be the closing quote. If it is
the closing quote, we are consuming all characters up and including the closing
quote. (The `=` in the `for..in` loop means that we are going to include the
last index of the provided range)

If we did not find the closing quote where we expect it to be, we are simply
highlighting the current `'` as `None` and consume it.

## Colorful single-line comments
Next, let's highlight single-line comments. (We'll leave multi-line comments
until the end, because they're complicated).

{% include hecto/add-comment-options.html %}

<small>[See this step on
github](https://github.com/pflenker/hecto-tutorial/tree/add-comment-options)</small>

Single line comments are easy to detect: If we encounter a `/` outside of a
string, all we need to do is check if it is followed by another `/`. If so,
treat the whole rest of the line as a comment.


{% include hecto/highlight-comments.html %}

<small>[See this step on
github](https://github.com/pflenker/hecto-tutorial/tree/highlight-comments)</small>

You should now be able to confirm that highlighting single line comments works.
Before we move to other things, however, we need to improve our code.

## Improve code quality
If you are following along using Clippy, you might have noticed that Clippy now
highlights the entirety of our highlight function. The grievance that it wants
to point out is that our function is getting too long. Adding to this function
is easy as we go along, but if you take a step back and try to take the whole
thing in, then you will realize just how difficult it is to follow along. In one
of the early chapters of the tutorial, this was the precise reason why we
started taking stuff out of `main`: To maintain readability. We are going to do
the same thing now.

Our strategy will be to split our big `highlight` function into separate
independent functions. Each function can either return `false`, meaning that
they did not highlight the current character, or `true`, indicating that they
handled the character. We are also going to allow these functions to modify the
pointer to the current character, to allow each function to consume more than
one character at a time.

{% include hecto/refactor-highlighting.html %}

<small>[See this step on
github](https://github.com/pflenker/hecto-tutorial/tree/refactor-highlighting)</small>

If you have trouble following this diff alone, don't forget that [the direct
link below the diff lets you see the full file after the
change.](https://github.com/pflenker/hecto-tutorial/tree/refactor-highlighting)

Let's go through this change by looking at `highlight` first. That is the
function we wanted to simplify, and it has gotten so much smaller. We have
removed any highlighting logic except for the highlighting of `None`, in case no
other highlighting has matched. By the way, if-statements are evaluated from
left to right, so, for instance, if `highlight_comment` returns `true`, none of
the other highlight functions is being called.

We will now investigate all the changes to all five highlighters we have written
so far.

`highlight_char` and `highlight_comment` are largely unchanged, they have been
moved to separate functions and changed to return `true` in case they do some
highlighting, and `false` otherwise.

`highlight_string` has changed quite a lot. Instead of having a variable which
keeps track of whether or not we are in a string, we are now checking if we are
at an opening quote. If so, we consume the remainder of the row until we meed
the end of it or a closing quote and highlight it as a string.

`highlight_number`, with which we started the whole highlighting process, has
also changed slightly. If invoked, it checks if the previous character was a
separator (instead of having a variable keeping track of it). If so, the
highlighting is done now the same as in other functions: We consume the whole
number, including the dot, in one go before returning from this function.

We have now placed `highlight_match` below the loop and have changed the logic a
bit. Previously, we were storing the indices of the matches to be used later
during highlighting. Now, we are updating `self.highlighting` directly from
within `highlight_match`. We need to handle this case separately as we want our
match not to interfere with other highlights. For instance, if we search for
`con`, we want the first three letters of `const` to be highlighted as a search
result, and the remaining two letters as a keyword.


### Side note on refactoring
We have now done quite a big refactoring, and I want to say a word or two about
the refacotring process itself. Many tutorials present you with a polished final
solution which you can happily implement. But that is not how code evolves in
practice. How we did it in this tutorial is much more realistic: You start with
a simple problem, in this case, highlighting a single digit, and then you expand
it over time. As the features grow, your code grows, too, and the assumptions
you made at the beginning on what the code should look like will soon be
outdated or plain wrong.

Being able to judge when to refactor is a skill that needs training, it doesn't
come by itself. Luckily, there are tools to help you build that skill. One is
clippy. We could have ignored or silenced this warning, but ultimately, clippy
was right in pointing out that our code was getting too complex! A refactoring
was in order, if not overdue.

Is the code now perfect? Definitely not. There are at least the following two
things wrong with it:
- The loop in `highlight` implicitly relies on the highlight functions to
  advance `index`. If any of these functions returns `true`, but does not modify
  `index`, we run into an infinite loop. This is not obvious in the code and
  therefore not ideal.
- The highlighting will not work around the borders of `usize`. We are pushing
  things into the `highlighting` array without checking if it is safe, and we
  are advancing `index` in many cases without any kind of check. It is not easy
  to see and understand how our code will behave in this case. Will it crash?
  Will it enter an infinite loop?

We are not going to solve these issues in this tutorial, but I invite you to
take a stab at them, they are not terribly hard to solve.

If you are now disappointed that even after our refactoring our code still has
flaws, then consider this: These flaws have been in the code before, but now
they are much, much easier to see. Also, we wouldn't have detected them if we
hadn't done the refactoring in the first place.

In a real world scenario, I would consider the weakness around large files as
not relevant - we saw earlier why we would run in to all kinds of other problems
first before these flaws would really become important, at least on modern
systems.

I would also either refactor or at least document the other issue, to make sure
that everyone who extends the highlighting in the future (could be my own future
me) knows what to do.


## Colorful keywords
Now let's turn to highlighting keywords. We're going to allow languages to
specify two types of keywords that will be highlighted in different colors. (In
Rust, we'll highlight actual keywords in one color and common type names in the
other color).

Let's add two `Vector`s to the `HighlightingOptions`, one for primary, one for
secondary keywords.

{% include hecto/add-keyword-options.html %}

<small>[See this step on
github](https://github.com/pflenker/hecto-tutorial/tree/add-keyword-options)</small>

We had to jump through a surprisingly large number of loops to get this to work.
So what happened?

When we added a more complex data structure like `Vec<String>` to
`HighlightingOptions`, it's no longer sensible to copy `HighlightingOptions`
around all the time, which was the case before. Also, we can no longer let the
compiler derive the `Copy` trait for us, so we needed to change a few things in
our code to make sure that from now on, only a reference to
`HighlightingOptions` is being passed around.

Now that we have the keywords available, let's highlight them. We start with the
primary keywords first.

{% include hecto/highlight-primary-keywords.html %}

<small>[See this step on
github](https://github.com/pflenker/hecto-tutorial/tree/highlight-primary-keywords)</small>


To make keyword highlighting work, we are building a new function called
`highlight_str`, which is responsible for highlighting a substring with a given
type. It does this by comparing every character after the current character with
the given string. If the whole string is matched, `self.highlighting` is updated
with the right highlighting type. If one character is off, the whole
highlighting for this string is aborted.

This means that we need to pass around `highlighting::Type`, but instead of
worrying about references and borrowing, wee are simply deriving `Copy` and
`Clone` for that type and simply copy it around. Since we have no plans for
`Type` to grow, this is a sensible decision for now.

If you start `hecto` now, you will see that primary keywords are highlighted.
However, there is still a bug: We don't want the `in` of `String` to be
highlighted, keywords need to be preceded and followed by a separator. Let's fix
that

{% include hecto/fix-keyword-highlighting.html %}

<small>[See this step on
github](https://github.com/pflenker/hecto-tutorial/tree/fix-keyword-highlighting)</small>

Now, let's try and highlight secondary keywords as well.

{% include hecto/highlight-secondary-keywords.html %}

<small>[See this step on
github](https://github.com/pflenker/hecto-tutorial/tree/highlight-secondary-keywords)</small>

We have now extracted the core of `highlight_primary_keywords_ into a more
generic function which highlights a given word with a given type if it's
surrounded by separators.

When you open your editor now, you should be able to see primary and secondary
keywords highlighted.

## Colorful multi-line comments
Okay, we have one last feature to implement: multi-line comment highlighting.
Let's start by adding the appropriate entry to `HighlightingOptions` and `Type`.

{% include hecto/add-ml-comment-options.html %}

<small>[See this step on
github](https://github.com/pflenker/hecto-tutorial/tree/add-ml-comment-options)</small>

We'll highlight multi-line comments to be the same color as single-line
comments. Now let's do the highlighting. We won't worry about multiple lines
just yet.

{% include hecto/single-ml-comments.html %}

<small>[See this step on
github](https://github.com/pflenker/hecto-tutorial/tree/single-ml-comments)</small>

This is essentially a combination of highlighting strings and highlighting
single-line comments. If we are on a `/*`, we use `find` to get the index of the
closing `*/` . Since theoretically we could have multiple comments in the same
line, we only search from the current index position forward in the string.

We highlight the whole comment, plus the 4 characters needed by the opening and
closing characters of the comment.

Now, how do we highlight multiple rows? The strategy we will be using is that we
pass the information if we ended on an unclosed comment back to `document`, so
that `document` can highlight the next `row` differently, if needed.

{% include hecto/multiline-comments.html %}

<small>[See this step on
github](https://github.com/pflenker/hecto-tutorial/tree/multiline-comments)</small>

We have now changed the signature of `highlight`: It receives a new boolean, and
it returns a boolean. The boolean it receives is `true` when the highlighting
should start within a multiline comment. If that is the case, we look for the
closing index before we enter the highlighting loop and highlight everything
until then as a mutli line comment. Similar to what we are doing in
`highlight_multiline_comment`, we go up until the end of the line if we can't
find the closing `*/`.

During highlighting, we are now keeping track if the last thing we highlighted
was a multiline comment. If we end the loop on a multiline comment, we check if
the last thing we saw was the ending of a multiline comment, as we want to
return `false` if we are out of a multi line comment, and `true` if the multi
line comment we are in has not been closed yet.

You can try saving a file with multi line comments now, or you can enter a multi
line comment and then trigger a search to see the correct highlighting. Let's
now make sure that the highlighting is also correct in all other cases.

{% include hecto/full-multiline-highlighting.html %}

<small>[See this step on
github](https://github.com/pflenker/hecto-tutorial/tree/full-multiline-highlighting)</small>

We have now restructured the code a bit so that the whole document is
re-highlighted on any update or delete operation. If you check it now, you
should notice two things: First, it works! Very satisfying! Second, the
performance is abysmal. Not satisfying!

The reason is, of course, that we are re-highlighting the whole document all the
time, on every change, to make sure the highlighting is correct everywhere and
in every case. Let's try to improve the performance by highlighting the document
less often.

{% include hecto/better-ml-highlighting.html %}

<small>[See this step on
github](https://github.com/pflenker/hecto-tutorial/tree/better-ml-highlighting)</small>

We have now removed all highlighting directly within `Row`, as we are now doing
the highlighting controlled by the `editor`. To do that, we have added a new
parameter to `highlight` in `row`: `until`, which denotes the index of the last
line for which the highlighting should be calculated. We know that the
highlighting of everything on screen always depends on the earlier rows in the
document, but not the rows below.

We are setting the highlighting in the editor's `refresh_screen` method. This
also means that we have to change how search results are being displayed, as
`refresh_screen` is always called when we conduct a search, which means that we
would overwrite the highlighting of a word during `refresh_screen`, rendering
our highlighting of the match useless. We solve this by storing the currently
highlighted word as part of the `editor` struct, setting and resetting this
during the search.

You should now be able to verify that the performance is now much, much better.
The higher up the user is in the file, the better the performance will be, as we
will be only highlighting rows up until the end of the current screen. But if
the user is at the bottom of the file, we are still doing a lot of highlighting
of all the rows above.

It's true that we need to take the rows above into account when highlighting to
understand whether or not we are in a multi-line comment or not. However, there
is no need to actually re-do the highlighting for all these lines.

Let's make sure that only the line which has been edited is being highlighted,
as well as all following lines (to make sure multi line comments still work).
Together with the recent change, this means that we are highlighting only the
lines between the current line and the last line on screen.

{% include hecto/final-tweaks.html %}

<small>[See this step on
github](https://github.com/pflenker/hecto-tutorial/tree/final-tweaks)</small>

We solve this by storing on a row whether or not it is currently properly
highlighted. If `highlight` is called on a row which is already highlighted, we
only check if this row ends with an unclosed multiline comment, to determine the
return value of `highlight`. If it's not highlighted, we do the highlighting as
usual and then set `is_highlighted` to `true`.

We also make sure that `is_highlighted` is `false` whenever a row is modified or
created.

In our `document`, we are then setting `is_highlighted` to `false` for every row
after and including the row which has been currently edited.

In combination with the previous change, this means that all rows after the
current row are marked for re-highlighting whenever we change something, but
re-highlighting is only done up until the last row on screen. In combination,
this means that when you are typing, at most all the rows currently on the
screen are being re-highlighted, which can be done at a much better performance.

To make sure our search results are still displayed, we re-highlight a row even
if it has been highlighted previously in case a word is provided to `highlight`.

## You're done!
That’s it! Our text editor is finished. In the [appendices]({% post_url
2019-11-08-rust-text-editor-appendix%}), you’ll find some ideas for features you
might want to extend the editor with on your own.