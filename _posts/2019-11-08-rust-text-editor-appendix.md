---
layout: postwithdiff
title: "Hecto: Appendices"
categories: [Rust, hecto, Tutorial]
permalink: /hecto-appendices/
date: 2019-11-08 00:00:09
last_modified_at: 2020-10-29
---
[First chapter]({% post_url 2019-11-08-rust-text-editor-chapter-1%}) - [Overview]({% post_url 2019-11-08-rust-text-editor%}) - [Last chapter]({% post_url 2019-11-08-rust-text-editor-chapter-7%})
{: style="text-align: center"}
## How the diffs work
Most steps in this tutorial are presented as a diff. A diff shows you the
changes you need to make to the previous step’s code to get to the current step.
Here’s a sample diff:

{% include hecto/read-keys.html %}

<small>[See this step on
github](https://github.com/pflenker/hecto-tutorial/tree/read-keys)</small>

Each diff contains one block per modified file. Each block starts with a header
that contains the filename of the file you need to edit ("src/main.rs"). After
the header, the contents of the file are shown. Additions are highlighted in
green, deletions in red. The old line numbers are printed at the left, the new
line numbers on the right. 

In this example, you need to add a new line on top of the file, leave the next
line alone, delete the line that starts with "println" and add a couple of more
lines in its place.

At the bottom of each diff, you find a link to github. If you follow that link,
you can see the project on github with the code changes applied.

## What to do if you are stuck

Some of the code in this tutorial is very tricky to type in exactly, especially
if you’re not used to Rust. It’s  specially easy to make a mistake when you’re
making a change to a line, and you think you’re done changing that line, but you
missed one little change to another part of that same line. It’s important to
take your time, and compare the changed parts of the diff character-by-character
with your code to make sure they’re the same.

If you suspect you made an error, but don't know where it is, you should do a
diff between your version and the tutorial's version. The easiest way to do so
is to use an online diffing tool like
[DiffChecker](https://www.diffchecker.com/). You paste the code from the
tutorial to the left (you can obtain it from github by following the "See this
change on github" link), and your code to the right and check the differences.

## Where to get help

If you are having trouble, feel free to [email me]({{ site.baseurl }}/about).

## Ideas for improvements and features

If you want to extend hecto on your own, I suggest trying to actually use hecto
as your text editor for a while. You will very quickly become painfully aware of
all sorts of features you’re used to having in a text editor, but are missing in
hecto. Those are the features you should try to add. And you should use hecto
when you work on hecto.

If you're still looking for ideas, here's a small list.
- Support more filetypes
- Make highlighting markers configurable. For instance, in some languages, a
  single `#` instead of two slashes indicate a single line comment. 
- Allow alternatives. For instance, in many languages, characters are not
  highlighted separately, but instead, strings can be delimited with single or
  double quotes.
- Line numbers: Display the line number to the left of each line of the file.
- Auto indent: When starting a new line, indent it to the same level as the
  previous line.+
- Hard-wrap lines: Insert a newline in the text when the user is about to type
  past the end of the screen. Try not to insert the newline where it would split
  up a word.
- Soft-wrap lines: When a line is longer than the screen width, use multiple
  lines on the screen to display it instead of horizontal scrolling.
- Better handling of indices: We have been a bit indicisive about when to use
  `saturating_add` and similar functions and when to do pointer arithmetic. We
  also have not done a good job at safely and consistently accessing entries of
  a `Vec`. 
- Copy and paste: Give the user a way to select text, and then copy the selected
  text when they press Ctrl-C, and let them paste the copied text when they
  press Ctrl-V.
- Multiple buffers: Allow having multiple files open at once, and have some way
  of switching between them.


## Credits
As written in the beginning, this tutorial has been heavily inspired by this
[Build your own text
editor](https://viewsourcecode.org/snaptoken/kilo/index.html) tutorial. The
nifty diffs have been done with
[diff2html](https://github.com/rtfpessoa/diff2html).

## License
- `kilo` was distributed under the [BSD-2 Clause
  License](https://opensource.org/licenses/BSD-2-Clause)
- The [original tutorial](https://viewsourcecode.org/snaptoken/kilo/) was
  distributed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/)
- `hecto` and this tutorial are licensed under [CC BY
  4.0](https://creativecommons.org/licenses/by/4.0/)