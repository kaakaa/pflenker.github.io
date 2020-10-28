---
layout: post
title: "Hecto: Build your own text editor in Rust"
categories: [Rust, hecto, Tutorial]
permalink: /hecto/
image: /assets/2019-11-08-hecto-chapter-7.png
date: 2019-11-08 00:00:01
last_modified_at: 2020-10-28
---
[First chapter]({% post_url 2019-11-08-rust-text-editor-chapter-1%}) - [Appendices]({% post_url 2019-11-08-rust-text-editor-appendix%}) - [Last chapter]({% post_url 2019-11-08-rust-text-editor-chapter-7%})
{: style="text-align: center"} 

This is a series of blog posts that shows you how to
build a text editor in Rust. It's a re-implementation of
[kilo](http://antirez.com/news/108) in Rust, as outlined in [this fantastic
tutorial](https://viewsourcecode.org/snaptoken/kilo/index.html). Same as the
original booklet, these blog posts guide you through all the steps to build a
basic text editor, `hecto`.

You will almost always be able to see your changes in action by applying the
changes, saving and running the program. I will explain every step along the way
as best as I can - sometimes in great detail, and often linking to other pages.
Feel free to skim over the prose and ignore the links, there is plenty to learn
just by applying the code changes and watching your text editor grow!

## Why?
I have always thought that every software engineer needs to have more than
superficial knowledge of at least two programming languages. However, I have to
admit, that in the past few years, my knowledge in pretty much everything except
JavaScript has started to fade.

That's why I started to learn Rust, and I have re-implemented `kilo` as a
learning experience. But *why*? In order to learn Rust, I wanted ro re-implement
a well-understood piece of software, so that I could focus on the language
without getting lost in the implementation details. But I did not want to
re-implement stuff I used JavaScript for, as I think that JavaScript is designed
for a different problem space than Rust. Or in other words: If you are a
plumber, you best learn how to use an axe by using it to chop down some trees,
and not to unclog a sink.

`kilo` is complex enough to pose a challenge, and when I read it, I wished it
was available for Rust - and now it is!

And *why* the name? `hecto` follows more modest goals than `kilo`. It does not
aim to be small, and it wasn't even my own idea - so it seemed appropriate to
give it a more modest name than its spiritual predecessor.

## License
- `kilo` was distributed under the [BSD-2 Clause
  License](https://opensource.org/licenses/BSD-2-Clause)
- The [original tutorial](https://viewsourcecode.org/snaptoken/kilo/) was
  distributed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/)
- `hecto` and this tutorial are licensed under [CC BY
  4.0](https://creativecommons.org/licenses/by/4.0/)

### Indication of Changes
While these blog posts are based firmly on the [original
tutorial](https://viewsourcecode.org/snaptoken/kilo/index.html), the code has
been adapted to Rust, not only by calling the closest "rust counterpart
function", but by trying to solve things "the rust way". Similarily, all
explanations have been checked and revised, and in many cases heavily rewritten,
in the context of Rust. Therefore, this tutorial should be seen as a "rust
remix" of the original `C` version.

## Feedback
I'm happy that you read my work and would love to [hear from you]({{
site.baseurl }}/about) - especially if you are either stuck or have found a
better way to solve specific things. Keep in mind that this is mostly an
exercise for me to get to know Rust - so if there's a better way to do things,
[please reach out]({{ site.baseurl }}/about)!

## Table of Contents
1. [Setup]({% post_url 2019-11-08-rust-text-editor-chapter-1%})
2. [Reading User Input]({% post_url 2019-11-08-rust-text-editor-chapter-2%})
3. [Raw User Input and Output]({% post_url
   2019-11-08-rust-text-editor-chapter-3%})
4. [A Text Viewer]({% post_url 2019-11-08-rust-text-editor-chapter-4%})
5. [A Text Editor]({% post_url 2019-11-08-rust-text-editor-chapter-5%})
6. [Search]({% post_url 2019-11-08-rust-text-editor-chapter-6%})
7. [Syntax Highlighting]({% post_url 2019-11-08-rust-text-editor-chapter-7%})

- [Appendices]({% post_url 2019-11-08-rust-text-editor-appendix%})