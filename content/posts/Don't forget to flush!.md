+++
title = "Don't forget to flush!"
date = 2023-12-24T16:46:16Z
draft = false
+++

Little disclaimer: This article is about the `print!` macro in Rust. 

This is my second attempt at learning Rust and I'm making a lot of progress so I figure it's a good time to talk about my first ever "bug" in Rust. 

In most languages, printing a string and accepting an input on the same line is quite straightforward. Take for example the Kotlin code below which prints out the text "Enter a number:" and waits to receive a number:

```kotlin 
fun main() { 
	print("Enter a number: ")
	val number = readln().toIntOrNull()
	// do something with number
}
```

The direct Rust equivalent below though, does not have the same behavior:  
```rust
fn main() { 
	let mut number = String::new(); 
	print!("Enter a number: "); 
	stdin().lock().read_line(&mut number).unwrap(); 
}
```
All you'll see when you run the Rust code above is a blinking text cursor, not a text prompting you to enter a number. Strange. Thankfully I accidentally hit the return key after staring at this strange behavior for about 5 minutes before I got a clue as to what was going on. Pressing the return key caused the program to print the text and exit. Interesting. 

I looked into why this was happening and found a link to `print!`'s [documentation](https://doc.rust-lang.org/std/macro.print.html). Turns out the earlier behavior is technically correct; the documentation has a very important note: 

> `stdout` is frequently line-buffered by default so it may be necessary to use `io::stdout().flush()` to ensure the output is emitted immediately.

What does `stdout` being line-buffered even mean? In simpler terms, `stdout` will store all the text you write to it until it meets the newline(\n) character or you explicitly **flush** out its memory. So to get the behavior I wanted, I had to add `io::stdout().flush()` after the `print!` macro. Problem solved. Curiosity still not satisfied because why don't languages like Kotlin require me to flush the output? The answer is simply because these other languages flush `stdout` under the hood. 

Rust's decision to not autoflush, though a little controversial as is shown in the comments under this [issue](https://github.com/rust-lang/rust/issues/23818), is true to Rust's principle to refrain from implicitly doing stuff for you. David Tolnay's [comment](https://github.com/rust-lang/rust/issues/23818#issuecomment-349394249) highlights the Rust team's stance on the issue. 

In conclusion, don't forget to flush — I know I won't. 