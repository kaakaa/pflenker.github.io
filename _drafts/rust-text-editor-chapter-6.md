---
layout: postwithdiff
title: "Hecto, Chapter 6: Search"
categories: [Rust, hecto]
---
Our text editor is done - we can open, edit and save files. The upcoming two features add more functionality to it. In this chapter, we will implement a minimal search feature.

For that, we reuse `prompt()`. When the user types a search query and presses <kbd>Enter</kbd>, we'll loop through all the rows of our file, and if a row contains their query string, we'll move the cursor to match. For that, we are going to need a method which searchs a single `Row` and returns the position of the match. Let's start with that now.


{% include hecto/simple-search.html %}

<small>[See this step on github](https://github.com/pflenker/hecto-tutorial/releases/tag/simple-search)</small>

Let's go through this change starting at `Row`. We have added a function which returns an `Option`, which contains either the x position of the match or `None`. We then use the `find` method of `String` to retrieve the byte index of the match. Remember that this might not be the same as the index of the grapheme! To convert the byte index into the grapheme index, we use a slightly convoluted loop. Let's unravel that one.

`grapheme_indices()` returns an iterator over a pair of `(byte_index, grapheme)` for each grapheme in the string. `enumerate()` enumerates the iterator, so it gives us the grapheme index of each entry of the iterator. 

We take this to our advantage and use the iterator until we arrive at the grapheme with the same byte index as our match, and return the corresponding grapheme index.

In `Document`, we try to return a `Position` of the match within the full document, if there is any. We do this by iterating over all rows and performing `match` on each row, until we have a match. We then return the current row index and the index of the match as the position of the match within the document.

In `Editor`, we use a very similar logic as before around `prompt`. We retrieve the search query from the user and pass it to `find`. If we find a match, we move our cursor. If we don't, we display a message to the user.

Last but not least, we also modify our initial welcome message for the user, so that they know how to use our new search functionality.


## Incremental search

Now, let's make our search feature fancy. We want to support incremental search, meaning the file is searched after each keypress when the user is typing in their search query. 

To implement this, we're going to get `prompt` to take a callback function as an argument. We'll have to call this function after each keypress, passing the current search query inputted by the user and the last key they pressed. I'll explain the concept in a bit.

{% include hecto/incremental-search.html %}

<small>[See this step on github](https://github.com/pflenker/hecto-tutorial/releases/tag/incremental-search)</small>

We are using a new concept here called [closures](https://doc.rust-lang.org/rust-by-example/fn/closures.html). It comes with a new bit of syntax, which makes this code a bit hard to read.

Let's start with the concept. What we want is to pass a _function_ to `prompt`, and we want this function to be called every time the user has modified his query. We want this so that we can perform a search on the partial query while the user is typing.

How do we do that? First, we need to modify the signature for `prompt`. It now looks like this:

```rust
 fn prompt<C>(&mut self, prompt: &str, callback: C) -> Result<Option<String>, std::io::Error> where C: Fn(&mut Self, Key, &String) {
     //...
 }
```                        

The initial `<C>` comes from the concept of Traits. We won't be implementing any complex traits in the scope of this tutorial, so you won't be seeing this syntax anywhere else. Essentially, the `C` is a placeholder for the callback, and you can see it repeated as the type for the paramter `callback`. Another new thing is the `where` at the end of the method definition, and this is where we define what kind of function `C` should be. It should essentially take three parameters: The `Editor`, the `Key` which has been pressed (we ignore it for now, but we are going to use it later), and a `&String`, which will be the (partial) search query. We can see `callback` being called with these parameters after the user has pressed a key.

How do we define a callback? Well, we can see the easiest case in the `prompt` for `save`: `|_, _, _| {}`.  This is a closure which takes three parameters, ignores all of them and does nothing.

A more meaningful example is the addition to our Search-Prompt: Here, the closure takes three parameters (ignoring the middle parameter) and actually does something. As it's called with every partial query, we are performing  a search and setting the cursor with every key press.

Maybe you are familiar with closures in other languages, and you might ask yourself: Can't we just access `self` directly within the closure, instead of passing it to `prompt` and then to the callback?

Well, yes and no. Yes, because a major feature of closures is that they allow you to access variables from outside of the closure definition, but no, because that does not work in this case.

The reason is that `prompt` takes a mutable reference to `self`, as it sets the status message, and it would also pass a mutable reference to `callback`. That is not allowed.

That’s all there is to it. We now have incremental search.

## Restore cursor position when cancelling search
If the user cancels his search, we want to reset the cursor to the old position. For that, we are going to save the position before we start the `prompt`, and we check the return value. If it's `None´, the user has cancelled his query, and we want to restore his old cursor position and scroll to it.

{% include hecto/reset-position.html %}

<small>[See this step on github](https://github.com/pflenker/hecto-tutorial/releases/tag/reset-position)</small>

To clone the old position, we derive the Clone trait. This allows us to clone all values of the `Position` struct by calling `clone`.

## Search forward and backward

The last feature we'd like to add is to allow th euser to advance to the next or previous match in the file using the arrow keys. 
The <kbd>&uarr;</kbd> and <kbd>&larr;</kbd> keys will go to the previous match, and the <kbd>&darr;</kbd> and <kbd>&rarr;</kbd> keys will go to the next match.

{% include hecto/navigate-right.html %}

<small>[See this step on github](https://github.com/pflenker/hecto-tutorial/releases/tag/navigate-right)</small>

We start by accepting a `start` position in our `find` methods, indicating that we want to search the next match after that position. For `row`, this means that we skip everything in the string up until `after` before performing `find`. We need to add `after` to the grapheme index of the match, since the grapheme index indicates the position in the substring without the first part of the string.

In `Document`, we are now skipping the first `y` rows when performing the search. We then try to find a match in the current row with the current `x` position. If we can't find anything, we proceed to the next line, but we set x to 0 there, to make sure we start looking from the start of the next row.


We have adjusted our error message and are calling `find` now with the cursor position. We are also now using the formerly unused `key` parameter of our closure and match against the user input. If the user presses down or right, we are moving the cursor to the right as well before continuing the search with the updated parameter. Why? Because if our cursor is already at the position of a search match, we don't want the current position to be returned to us, so we start one step to the left of the current position.  
In case our find did not return a match, we want the cursor to stay on its previous position. So we track if we have moved the cursor in `moved` and move it back in case there is no result.

## Searching backwards

Now let's take a look at searching backwards. 

{% include hecto/navigate-left.html %}

<small>[See this step on github](https://github.com/pflenker/hecto-tutorial/releases/tag/navigate-left)</small>

That is quite a big change for something that should just be the opposite of what we just implemented!

Let's unravel the change step by step, starting at `Row`. 

We have renamed `after` to `at`, since we are not necessarily looking at something after, but also before the given index. We are then calculating the size of our substring which we want to search based on the search direction: Either we want to search from the beginning to our current position, or from our current position to the end of the row. Once we have the right length, we use an iterator to collect our actual substring into a string. We allow integer arithmetics here since we have made sure that end is always bigger than or equal to start, and always smaller than or equal to the length of the current row.

You can see an enum in action: `SearchDirection`. We will see its definition in a bit. For now, we use it to determine whether or not we should find the first occurence in a given string, with `find`, or the last, with `rfind`. This ensures that we are finding matches in the correct direction.

In `Document`, we are building our own iterator, since depending on the search direction we either need to search forward or backward through the document. 

We determine the range of rows through which we will be iterating based on the search direction, and then we perform the search. Same as before: If we have a match, then we can return the position we are currently on. If not, we have to set `y` to the next or previous row, and have to set x either to 0, in case we are searching forward, or to the length of the previous line, in case we are searching backwards.

In `editor`, you can see the definition of our new struct, `SearchDirection`. It derives `Copy` and `Clone`, which allows it to be passed around (the structure is small enough that copying it instead of passing a reference does not really matter), and `PartialEq` ensures that we can actually compare two search directions to one another.

Then, we are defining a new variable, `direction`, which we intend to set from within the `prompt` closure. We set the search direction to `backward` in case the back or left key have been pressed, and to `forward` on right and down. Then we pass the search direction to the adjusted `find` method.
 
Note that we also set the direction to forward again on any other keypress. If we wouldn't do that, the search behaviour would be weird: Let's say the cursor is on `baz` of `foo bar baz`, the user has entered `b` and the Search Direction is backward. If you typed `a`, the query would be `ba` and the user would jump back to `bar` instead of staying on `baz`, even though `baz` also matches the query.

We have also removed the final search and the "Not found" message in case the user cancels the search. We are already searching within the closure, so an additional search at the end does not bring us any benefit. 

Last but not least, we needed to update the signature for our callback (and thus, for `prompt`), so that `callback` is able to modify variables accessed within it. That's being done by changing `callback` to `FnMut`.

Congratulation, our search feature works now!

## Conclusion
Since our functionality is now complete, we used this chapter again to focus a bit more on Rust topics and brushed the topic of Closures and Traits. Our editor is almost complete - in the next chapter, we are going to implement Syntax Highlighting and File Type Detection, to complete our text editor.