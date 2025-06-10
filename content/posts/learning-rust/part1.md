+++
date = '2025-06-04T10:17:26+05:30'
title = 'Learning Rust, Part 1: Memory Safety, Ownership, and Borrowing' 
+++


Rust is incredibly popular these days, with several new and exciting projects being built in the language. From a SQLite rewrite named [limbo](https://github.com/tursodatabase/limbo) to high-performance code editors like [zed](https://github.com/zed-industries/zed), and even a new TUI-based editor from Microsoft called [edit](https://github.com/microsoft/edit). The language is even making its way into the [linux kernel](https://www.zdnet.com/article/the-linux-6-15-kernel-arrives-and-its-big-a-victory-for-rust-fans/) and [Microsoft is going "all-in" on rust](https://www.youtube.com/watch?v=1VgptLwP588). With me being terminally online on X (formerly twitter) and my timeline being flooded with rust news, I decided it was time to give it a try myself.

My background is primarily in Python and Go. While I've dabbled in Rust before, this is my first systematic attempt to explore the language and document my journey. I started with  the official [rust programming book](https://doc.rust-lang.org/book/title-page.html), the recommended book along with the rust documentation and some excellent rust talks on youtube.   

To demonstrate what makes Rust special, let's look at three very similar programs in C++, Go, and Rust. Each program attempts to do something dangerous: return a reference to data that was created inside a function. This data will be destroyed when the function exits, leading to a "dangling reference" or "use-after-free" bug.

The interesting thing is how each language handles this. The C++ and Go programs will compile, but the Rust program won't. Furthermore, the Go program will appear to run "correctly," while the C++ program will produce garbage. Let's look at each one closely.


#### CPP: Undefined Behaviour

``` cpp {class="c-code-1-class" id="c-codeblock1" lineNos=inline tabWidth=2}
#include <iostream>
#include <vector>

int* getElementRef() {
    std::vector<int> v = {10, 20, 30, 40, 50};
    return &v[2]; // Returns a pointer to memory that will be freed
}

int main() {
    int* ref = getElementRef();
    // Use-after-free: v's memory has been deallocated
    std::cout << "Value: " << *ref << std::endl; // Undefined behavior! Prints garbage: e.g., 1745181257
}
```

In this C++ code, v is a local variable stored on the stack. When getElementRef finishes, v goes out of scope, and its memory is deallocated. The function returns a pointer to this now-invalid memory. When main tries to dereference this pointer, it's invoking Undefined Behavior (UB). The program might crash, or it might print a garbage value like 1745181257, which is whatever data happened to be in that memory location at the time. This is a classic and dangerous memory safety bug.

##### A little detour: Where is v allocated and deallocated?

The std::vector object itself (v) is allocated on the stack (like any local variable in a function) However, the actual elements of the vector ({10, 20, 30, 40, 50}) are stored in dynamically allocated memory (heap). The vector internally manages this heap memory. When getElementRef() returns, the vector v is destroyed (its destructor runs), and the heap memory holding its elements is freed. The function returns a pointer (&v[2]) to an element in the now-freed heap memory, leading to a dangling pointer. Dereferencing this pointer in main() (*ref) causes undefined behavior (commonly a garbage value or crash).

#### Go: Hidden Heap Allocation

```go {class="go-code-1-class" id="go-codeblock1" lineNos=inline tabWidth=2}
package main

import "fmt"

func getElementRef() *int {
	arr := [5]int{10, 20, 30, 40, 50}
	return &arr[2]
}

func main() {
	ref := getElementRef()
	fmt.Printf("Value: %d\n", *ref) // Prints: Value: 30
}
```
This Go program compiles and runs, correctly printing 30. How? The Go compiler performs what's known as **escape analysis**.

Escape analysis is a compile-time process where the compiler determines if a variable's memory address "escapes" the function it was created in. If a variable is only used within its function, it can be safely allocated on the stack, which is very fast. However, if a reference to the variable is returned or shared in a way that it could be used after the function finishes (like in our example), it's said to "escape." To ensure memory safety, the compiler moves the escaped variable to the heap. This prevents a dangling pointer but can introduce the hidden performance cost of heap allocation and garbage collection.

In our Go code, the compiler detects that a reference to arr "escapes" the getElementRef function's scope. To prevent a memory bug, it automatically allocates arr on the heap instead of the stack. You can see this happening if you build with the {{< highlight bash "hl_inline=true" >}}go build -gcflags="-m" escapeAnalysis.go{{< /highlight >}} command, which logs the following:

``` bash
# command-line-arguments
./escapeAnalysis.go:3:6: can inline getRef
./escapeAnalysis.go:8:6: can inline main
./escapeAnalysis.go:10:8: inlining call to getRef
./escapeAnalysis.go:4:2: moved to heap: arr
```

The Go compiler uses escape analysis to allocate arr on the heap rather than the stack. It's safe, but it's implicit you don’t have control or visibility. You can accidentally cause performance/memory issues if you don't understand what escapes.

#### Rust: Compile-Time Guarantees

```rust {class="my-class" id="my-codeblock" lineNos=inline tabWidth=2}
fn get_element_ref() -> &i32 {
    let v = vec![10, 20, 30, 40, 50];
    &v[2] // This is an error
}

fn main() {
    let r = get_element_ref();
    println!("Value: {}", r);
}
```

If you try to compile the above rust code you get an error. 

```bash
Compiling playground v0.0.1 (/playground)
error[E0106]: missing lifetime specifier
 --> src/main.rs:1:25
  |
1 | fn get_element_ref() -> &i32 {
  |                         ^ expected named lifetime parameter
  |
  = help: this function's return type contains a borrowed value, but there is no value for it to be borrowed from
```

The Rust compiler enforces a simple rule: references cannot outlive the data they point to.

Here, v is owned by the get_element_ref function. When the function ends, v is dropped (deallocated). The compiler sees that we are trying to return a reference (&v[2]) that would outlive its data, and it prevents the program from compiling. The error message, while a bit cryptic at first, is telling us exactly this: you're returning a borrowed value, but the original owner won't exist anymore.

### So, what is ownership?

Ownership in Rust is a memory management system where each value has a single owner that determines its lifetime, and the value is automatically dropped when the owner goes out of scope. Let's look at the 2 examples below to understand it a bit more clearly. The 3 rules that govern the ownership are

1. Each value in Rust has a variable that’s called its owner.
2. There can only be one owner at a time.
3. When the owner goes out of scope, the value is dropped.


```rust
fn main() {
    let s1 = String::from("hello");  // s1 owns the String
    let s2 = s1;                     // Ownership of the data is *moved* from s1 to s2

    // The line below would cause a compile error: borrow of moved value: `s1`
    // println!("s1: {}", s1);

    println!("s2: {}", s2);          // This is fine, s2 is the owner
}
```

### Borrowing: Access Without Ownership

If moving ownership is the only way to pass data around, it would be very restrictive. What if we want a function to use a value without taking ownership of it? This is where borrowing comes in.

```rust
// This function takes a reference to a String
fn calculate_length(s: &String) -> usize {
    s.len()
} // `s` goes out of scope here, but because it does not have ownership,
  // the String it refers to is NOT dropped.

fn main() {
    let s1 = String::from("world");

    // We pass a reference to s1 to the function.
    // s1 is "borrowed," not "moved."
    let len = calculate_length(&s1);

    // We can still use s1 here because it still owns the data.
    println!("The length of '{}' is {}.", s1, len);
}
```
By passing &s1, we are lending calculate_length temporary, read-only access to the String. main remains the owner. The compiler ensures that while s1 is borrowed, we can't do anything that might invalidate the reference (like modifying s1 if the borrow was immutable). This system of ownership and borrowing is how Rust guarantees memory safety without needing a garbage collector.

### Conclusion: 

Our small experiment of returning a reference from a function highlights a big difference in how programming languages think about safety. C++ gives full control but can lead to unsafe bugs (These bugs have been long present in c/c++ code bases and tools have been developed to identify memory safety issues but they can't guarantee 100% memory safety whereas in rust memory safety is baked in as part of the language design itself). Go adds safety with a garbage collector, but it's mostly hidden. Rust offers a middle ground: **memory safety checked at compile time**.

In the next blog, will discuss how to program with rust ownership and borrowing rules, lifetimes, mutable references and more. Thanks for reading. 