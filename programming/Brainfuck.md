An esoteric programming language using only the characters:
- `+`
	- Increment the current cell value
- `-`
	- Decrement the current cell value
- `<`
	- Decrement the current cell index
- `>`
	- Increment the current cell index
- `.`
	- Print the current cell value
- `,`
	- Accept one byte of input
- `[`
	- If the byte at the data pointer is zero, then instead of moving the instruction pointer forward to the next command, jump it forward to the command after the matching `]` command.
- `]`
	- If the byte at the data pointer is nonzero, then instead of moving the instruction pointer forward to the next command, jump it back to the command after the matching `[` command.

Programs typically have a fixed length array of 30,000 byte/char/u8 cells but implementations can vary.

### Simple Hello World

This program is an example how how you could output "Hello World":

```
++++++++[>++++[>++>+++>+++>+<<<<-]>+>+>->>+[<]<-]>>.>---.+++++++..+++.>>.<-.<.+++.------.--------.>>+.>++.
```

### Complex Hello World

A more complex version of the same program (will often cause bugs in interpreters):

```
>++++++++[-<+++++++++>]<.>>+>-[+]++>++>+++[>[->+++<<+++>]<<]>-----.>->
+++..+++.>-.<<+[>[+>+]>>]<--------------.>>.+++.------.--------.>+.>+.
```

## Building the Interpreter

### Initial Implementations

I started off by just building a straight port of a python implementation I found [here](https://github.com/Camto/Shorterpreters/blob/master/Brainfuck/brainfuck.py).
This script was demoed in a [youtube video](https://youtu.be/4uNM73pfJn0) explaining the process of creating an interpeter for this language.
This lead me to build the following functions:

```rust
/// Function to find all pairs of brackets, denoting loop structures
fn find_loops(prog: &str) -> HashMap<usize, usize> {
    let mut loop_stack = Vec::with_capacity(100);
    let mut loops = HashMap::new();
    for (ip, instruction) in prog.char_indices() {
        match instruction {
            '[' => loop_stack.push(ip),
            ']' => {
                let loop_start = loop_stack.pop().unwrap();
                loops.insert(loop_start, ip);
                loops.insert(ip, loop_start);
            }
            _ => {}
        }
    }
    loops
}

/// Function to execute a brainf*** program and return its output as a String
pub fn interpret(prog: &str) -> String {
    let loop_table = find_loops(prog);
    let prog = prog.chars().collect::<Vec<_>>();
    let mut user_input: Vec<char> = Vec::new();
    let mut tape: Vec<Cell> = Vec::from([0]);
    let mut ip = 0;
    let mut cell_index = 0;
    let mut output = String::new();
    while ip < prog.len() {
        let instruction = prog[ip] as char;
        let cell_val = tape.get_mut(cell_index).unwrap();
        match instruction {
            '+' => *cell_val = cell_val.wrapping_add(1),
            '-' => *cell_val = cell_val.wrapping_sub(1),
            '<' => cell_index -= 1,
            '>' => {
                cell_index += 1;
                if cell_index == tape.len() {
                    tape.push(0);
                }
            }
            '.' => output.push(tape[cell_index] as char),
            ',' => {
                if user_input.is_empty() {
					// Ignore the implementation details for `get_input`
                    user_input = get_input();
                }
                *cell_val = user_input.remove(0) as Cell
            }
            '[' if *cell_val == 0 => ip = loop_table[&ip],
            ']' if *cell_val != 0 => ip = loop_table[&ip],
            _ => {}
        }
        ip += 1
    }
    output
}
```

In my opinion, this code is simple but it's a bit cluttered. 
This interpret function is doing way too much, making it difficult to reuse portions of the code. 
Another problem with the `interpret` function is that it will panic when executing programs that make use of integer underflow when modifying the current cell index.
This happens because the implementation of this function uses an expanding vector/list of integers rather than a fixed-length structure like an array or a preallocated vector.
To fix this, I copied and pasted the function and changed the implementation to use a [Hashmap](https://doc.rust-lang.org/std/collections/struct.HashMap.html) allowing for arbitrary cell indices without requiring a preallocated block of memory.
The implementation looked like this:
```rust
pub fn interpret_with_wrapping(prog: &str) -> String {
    let loop_table = find_loops(prog);
    let prog = prog.chars().collect::<Vec<_>>();
    let mut user_input: Vec<char> = Vec::new();
    let mut tape: HashMap<usize, Cell> = HashMap::from_iter([(0, 0)]);
    let mut ip = 0;
    let mut cell_index = 0;
    let mut output = String::new();
    while ip < prog.len() {
        let instruction = prog[ip] as char;
        tape.entry(cell_index).or_insert(0);
        let cell_val = tape.get_mut(&cell_index).unwrap();
        match instruction {
            '+' => *cell_val = cell_val.wrapping_add(1),
            '-' => *cell_val = cell_val.wrapping_sub(1),
            '<' => cell_index = cell_index.wrapping_sub(1),
            '>' => {
                cell_index = cell_index.wrapping_add(1);
                if !tape.contains_key(&cell_index) {
                    tape.insert(cell_index, 0);
                }
            }
            '.' => output.push(tape[&cell_index] as char),
            ',' => {
                if user_input.is_empty() {
                    user_input = get_input();
                }
                *cell_val = user_input.remove(0) as Cell
            }
            '[' if tape[&cell_index] == 0 => ip = loop_table[&ip],
            ']' if tape[&cell_index] != 0 => ip = loop_table[&ip],
            _ => {}
        }
        ip += 1
    }
    output
}
```

Following this, I applied some [clippy](https://github.com/rust-lang/rust-clippy) linting suggestions, added benchmarks, and then restructured the code, none of which I'll show here since it's not particularly interesting.

### Error Handling

The next major change I made to implementations was the addition of error handling.
In its current form, the interpreters will panic anywhere the index `[]` operator is used as well as anywhere `.unwrap()` is called.

To start off, I added the [miette crate](https://crates.io/crates/miette) for its ability to have a nice output format for errors that includes the location of the error and its root cause.

Example:
![](Pasted%20image%2020230218164143.png)

I also added the [thiserror crate](https://crates.io/crates/thiserror)  to define the errors that the interpreters will produce.
The error types I eventually ended up with after fiddling around with the code are as follows:

```rust
use miette::{Diagnostic, Result, SourceSpan};
use thiserror::Error;

#[derive(Error, Diagnostic, Debug)]
pub enum BrainfuckError {
    #[error("Could not parse program")]
    ParseError {
        #[source_code]
        src: String,
        #[label("It might be this one")]
        location: SourceSpan,
        #[diagnostic_source]
        err_type: LoopErrorType,
        char_index: usize,
    },
    #[error("An error occurred while executing the program")]
    ExecutionError {
        #[source_code]
        src: String,
        #[label("Occurred at instruction: {instruction_ptr}, cycle: {cycle}")]
        location: SourceSpan,
        instruction_ptr: usize,
        cycle: usize,
        #[diagnostic_source]
        err_type: ExecutionErrorType,
    },
}

#[derive(Error, Diagnostic, Debug)]
pub enum ExecutionErrorType {
    #[error("Negative unsigned integer overflow ocurred in cell index")]
    CellIndexUnderflow,
}
  
#[derive(Error, Diagnostic, Debug)]
pub enum LoopErrorType {
    #[error("Unclosed bracket")]
    UnclosedBracket,
    #[error("Unexpected closing bracket")]
    UnexpectedClosingBracket,
}
```

As a quick summary, `BrainfuckError` is the main error type that will be returned by any interpreter. 
It has the variants `ParseError` and `ExecutionError`.
A brainfuck program can contain any characters but only specific ones are mapped to instruction while the rest act as comments.
Thus, a parsing error only occurs if the program contains invalid loop constructs (unbalanced brackets).

Examples of programs that would fail to parse (each line is a new program):

```
+>><+[[]--.

+]>
```

With our new error types, the function for finding all the loop constructs looks like this now:

```rust
// I also renamed the function
fn verify_loops(prog: &str) -> Result<HashMap<usize, usize>, BrainfuckError> {
    let mut stack = Vec::with_capacity(100);
    let mut loops = HashMap::new();
    for (ip, instruction) in prog.char_indices() {
        match instruction {
            '[' => stack.push((ip, instruction)),
            ']' => {
                let (loop_start, _instruction) = match stack.pop() {
                    Some(x) => Ok(x),
                    None => Err(BrainfuckError::ParseError {
                        location: (ip, 0).into(),
                        src: prog.into(),
                        err_type: LoopErrorType::UnexpectedClosingBracket,
                        char_index: ip,
                    }),
                }?;
                loops.insert(loop_start, ip);
                loops.insert(ip, loop_start);
            }
            _ => {}
        }
    }
    match stack.pop() {
        Some((index, instruction)) => Err(match instruction {
            '[' => BrainfuckError::ParseError {
                location: (index, 0).into(),
                src: prog.into(),
                err_type: LoopErrorType::UnclosedBracket,
                char_index: index,
            },
            _ => panic!(),
        }),
        None => Ok(loops),
    }
}
```

Here's a diff view to better see what changes were made:

![](error_handling_diff.png)

I went through a few iterations with the interpreters to figure out the way I wanted to handle errors which lead me to update the error type and interpreter function as follows:

![](Pasted%20image%2020230218172605.png)

![](Pasted%20image%2020230218173619.png)