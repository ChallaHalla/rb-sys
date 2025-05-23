{{#title Ruby on Rust: Error Handling}}

# Error Handling in Rust Ruby Extensions

Proper error handling is critical for robust Ruby extensions. This guide covers how to handle errors in Rust and map
them to appropriate Ruby exceptions.

<div class="warning">

Improper error handling can lead to crashes that take down the entire Ruby VM. This chapter shows you how to properly
raise and handle exceptions in your Rust extensions.

</div>

## Overview of Error Handling Approaches

When building Ruby extensions with Rust, you'll typically use one of these error handling patterns:

1. **Result-based error handling**: Using Rust's `Result<T, E>` type to return errors
2. **Ruby exception raising**: Converting Rust errors into Ruby exceptions
3. **Panic catching**: Handling unexpected Rust panics and converting them to Ruby exceptions

<div class="note">

The most common pattern in rb-sys extensions is to use Rust's `Result<T, magnus::Error>` type, where the `Error` type
represents a Ruby exception that can be raised.

</div>

## The Result Type and Magnus::Error

Magnus uses `Result<T, Error>` as the standard way to handle errors. The `Error` type represents a Ruby exception that
can be raised:

```rust
use magnus::{Error, Ruby};

fn might_fail(ruby: &Ruby, value: i64) -> Result<i64, Error> {
    if value < 0 {
        return Err(Error::new(
            ruby.exception_arg_error(),
            "Value must be positive"
        ));
    }
    Ok(value * 2)
}
```

The `Error` type:

- Contains a reference to a Ruby exception class
- Includes an error message
- Can be created from an existing Ruby exception

## Mapping Rust Errors to Ruby Exceptions

### Standard Ruby Exception Types

Magnus provides access to all standard Ruby exception types:

```rust
use magnus::{Error, Ruby};

fn divide(ruby: &Ruby, a: f64, b: f64) -> Result<f64, Error> {
    if b == 0.0 {
        return Err(Error::new(
            ruby.exception_zero_div_error(),
            "Division by zero"
        ));
    }
    Ok(a / b)
}

fn process_array(ruby: &Ruby, index: isize, array: RArray) -> Result<Value, Error> {
    if index < 0 || index >= array.len() as isize {
        return Err(Error::new(
            ruby.exception_index_error(),
            format!("Index {} out of bounds (0..{})", index, array.len() - 1)
        ));
    }
    array.get(index as usize)
}

fn parse_number(ruby: &Ruby, input: &str) -> Result<i64, Error> {
    match input.parse::<i64>() {
        Ok(num) => Ok(num),
        Err(_) => Err(Error::new(
            ruby.exception_arg_error(),
            format!("Cannot parse '{}' as a number", input)
        )),
    }
}
```

Common Ruby exception types available through the Ruby API:

| Method                             | Exception Class       | Typical Use Case                 |
| ---------------------------------- | --------------------- | -------------------------------- |
| `ruby.exception_arg_error()`       | `ArgumentError`       | Invalid argument value or type   |
| `ruby.exception_index_error()`     | `IndexError`          | Array/string index out of bounds |
| `ruby.exception_key_error()`       | `KeyError`            | Hash key not found               |
| `ruby.exception_name_error()`      | `NameError`           | Reference to undefined name      |
| `ruby.exception_no_memory_error()` | `NoMemoryError`       | Memory allocation failure        |
| `ruby.exception_not_imp_error()`   | `NotImplementedError` | Feature not implemented          |
| `ruby.exception_range_error()`     | `RangeError`          | Value outside valid range        |
| `ruby.exception_regexp_error()`    | `RegexpError`         | Invalid regular expression       |
| `ruby.exception_runtime_error()`   | `RuntimeError`        | General runtime error            |
| `ruby.exception_script_error()`    | `ScriptError`         | Problem in script execution      |
| `ruby.exception_syntax_error()`    | `SyntaxError`         | Invalid syntax                   |
| `ruby.exception_type_error()`      | `TypeError`           | Type mismatch                    |
| `ruby.exception_zero_div_error()`  | `ZeroDivisionError`   | Division by zero                 |

### Creating Custom Exception Classes

You can define custom exception classes for your extension:

```rust
use magnus::{class, Error, Ruby};

#[magnus::init]
fn init(ruby: &Ruby) -> Result<(), Error> {
    let module = ruby.define_module("MyExtension")?;

    // Create custom exception classes
    let std_error = ruby.exception_standard_error();
    let custom_error = module.define_class("CustomError", std_error)?;
    let validation_error = module.define_class("ValidationError", custom_error)?;

    // Register them as constants for easier access
    ruby.define_global_const("MY_CUSTOM_ERROR", custom_error)?;

    Ok(())
}

// Using the custom exception
fn validate(ruby: &Ruby, value: &str) -> Result<(), Error> {
    if value.is_empty() {
        return Err(Error::new(
            ruby.class_path_to_value("MyExtension::ValidationError"),
            "Validation failed: value cannot be empty"
        ));
    }
    Ok(())
}
```

### Passing and Re-raising Ruby Exceptions

You can pass along existing Ruby exceptions:

```rust
use magnus::{Error, Ruby, Value};

fn process_data(ruby: &Ruby, input: Value) -> Result<Value, Error> {
    // Call a method that might raise
    let result = match input.funcall(ruby, "process", ()) {
        Ok(val) => val,
        Err(err) => return Err(err), // Pass along the original error
    };

    // Or with the ? operator
    let result = input.funcall(ruby, "process", ())?;

    Ok(result)
}
```

For wrapping and adding context to errors:

```rust
fn compute_with_context(ruby: &Ruby, input: Value) -> Result<Value, Error> {
    match complex_operation(ruby, input) {
        Ok(result) => Ok(result),
        Err(err) => {
            // Create a new error with additional context
            Err(Error::new(
                ruby.exception_runtime_error(),
                format!("Computation failed: {}", err.message(ruby).unwrap_or_default())
            ))
        }
    }
}
```

## Handling Rust Panics

Rust panics should be caught and converted to Ruby exceptions to prevent crashing the Ruby VM:

```rust
use magnus::{Error, Ruby};
use std::panic::{self, catch_unwind};

fn dangerous_operation(ruby: &Ruby, input: i64) -> Result<i64, Error> {
    // Catch any potential panics
    let result = catch_unwind(|| {
        // Code that might panic
        if input == 0 {
            panic!("Unexpected zero value");
        }
        input * 2
    });

    match result {
        Ok(value) => Ok(value),
        Err(_) => Err(Error::new(
            ruby.exception_runtime_error(),
            "Internal error: Rust panic occurred"
        )),
    }
}
```

## Error Handling Patterns

### The Question Mark Operator

The `?` operator simplifies error handling by automatically propagating errors:

```rust
fn multi_step_operation(ruby: &Ruby, value: i64) -> Result<i64, Error> {
    // Each operation can fail, ? will return early on error
    let step1 = validate_input(ruby, value)?;
    let step2 = transform_data(ruby, step1)?;
    let step3 = final_calculation(ruby, step2)?;

    Ok(step3)
}
```

### Pattern Matching on Errors

For more sophisticated error handling, pattern match on error types:

```rust
fn handle_specific_errors(ruby: &Ruby, value: Value) -> Result<Value, Error> {
    let result = value.funcall(ruby, "some_method", ());

    match result {
        Ok(val) => Ok(val),
        Err(err) if err.is_kind_of(ruby, ruby.exception_zero_div_error()) => {
            // Handle division by zero specially
            Ok(ruby.integer_from_i64(0))
        },
        Err(err) if err.is_kind_of(ruby, ruby.exception_arg_error()) => {
            // Convert ArgumentError to a different type
            Err(Error::new(
                ruby.exception_runtime_error(),
                format!("Invalid argument: {}", err.message(ruby).unwrap_or_default())
            ))
        },
        Err(err) => Err(err), // Pass through other errors
    }
}
```

### Context Managers / RAII Pattern

Use Rust's RAII (Resource Acquisition Is Initialization) pattern for cleanup operations:

```rust
use magnus::{Error, Ruby};
use std::fs::File;
use std::io::{self, Read};

struct TempResource {
    data: Vec<u8>,
}

impl TempResource {
    fn new() -> Self {
        // Allocate resource
        TempResource { data: Vec::new() }
    }
}

impl Drop for TempResource {
    fn drop(&mut self) {
        // Clean up will happen automatically, even if an error occurs
        println!("Cleaning up resource");
    }
}

fn process_with_resource(ruby: &Ruby) -> Result<Value, Error> {
    // Resource is created
    let mut resource = TempResource::new();

    // If an error occurs here, resource will still be cleaned up
    let file_result = File::open("data.txt");
    let mut file = match file_result {
        Ok(f) => f,
        Err(e) => return Err(Error::new(
            ruby.exception_io_error(),
            format!("Could not open file: {}", e)
        )),
    };

    // Resource will be dropped at the end of this scope
    Ok(ruby.ary_new_from_ary(&[1, 2, 3]))
}
```

## Best Practices for Error Handling

<div class="tip">

Following these practices will make your extensions more robust and provide a better experience for users.

</div>

### 1. Be Specific with Exception Types

Choose the most appropriate Ruby exception type:

```rust,hidelines=#
# use magnus::{Error, Ruby};
#
# fn example(ruby: &Ruby, index: usize, array: &[i32]) -> Result<i32, Error> {
// ✅ GOOD: Specific exception type
if index >= array.len() {
    return Err(Error::new(
        ruby.exception_index_error(),
        format!("Index {} out of bounds (0..{})", index, array.len() - 1)
    ));
}
#
# // Example with bad practice commented out
# /*
# // ❌ BAD: Generic RuntimeError for specific issue
# if index >= array.len() {
#     return Err(Error::new(
#         ruby.exception_runtime_error(),
#         "Invalid index"
#     ));
# }
# */
#     Ok(array[index])
# }
```

<div class="note">

Ruby has a rich hierarchy of exception types. Using the specific exception type helps users handle errors properly in
their Ruby code.

</div>

### 2. Provide Clear Error Messages

Include relevant details in error messages:

```rust
// ✅ GOOD: Descriptive error with context
let err_msg = format!(
    "Cannot parse '{}' as a number in range {}-{}",
    input, min, max
);

// ❌ BAD: Vague error message
let err_msg = "Invalid input";
```

### 3. Maintain Ruby Error Hierarchies

Respect Ruby's exception hierarchy:

```rust
// ✅ GOOD: Proper exception hierarchy
let io_error = ruby.exception_io_error();
let file_error = module.define_class("FileError", io_error)?;
let format_error = module.define_class("FormatError", file_error)?;

// ❌ BAD: Improper exception hierarchy
let format_error = module.define_class("FormatError", ruby.class_object())?; // Not inheriting from StandardError
```

### 4. Avoid Panicking

Use `Result` instead of panic:

```rust
// ✅ GOOD: Return Result for expected error conditions
fn process(value: i64) -> Result<i64, Error> {
    if value < 0 {
        return Err(Error::new(
            ruby.exception_arg_error(),
            "Value must be positive"
        ));
    }
    Ok(value * 2)
}

// ❌ BAD: Panicking on expected error condition
fn process(value: i64) -> i64 {
    if value < 0 {
        panic!("Value must be positive"); // Will crash the Ruby VM!
    }
    value * 2
}
```

### 5. Catch All Ruby Exceptions

When calling Ruby methods, always handle exceptions:

```rust
// ✅ GOOD: Catch exceptions from Ruby method calls
let result = match obj.funcall(ruby, "some_method", ()) {
    Ok(val) => val,
    Err(err) => {
        // Handle or re-raise the error
        return Err(err);
    }
};

// ❌ BAD: Not handling potential Ruby exceptions
let result = obj.funcall(ruby, "some_method", ()).unwrap(); // May panic!
```

## Error Handling with RefCell

When using `RefCell` for interior mutability, handle borrow errors gracefully:

```rust
use std::cell::RefCell;
use magnus::{Error, Ruby};

#[magnus::wrap(class = "Counter")]
struct MutCounter(RefCell<u64>);

impl MutCounter {
    fn new() -> Self {
        MutCounter(RefCell::new(0))
    }

    fn increment(ruby: &Ruby, self_: &Self) -> Result<u64, Error> {
        match self_.0.try_borrow_mut() {
            Ok(mut value) => {
                *value += 1;
                Ok(*value)
            },
            Err(_) => Err(Error::new(
                ruby.exception_runtime_error(),
                "Cannot modify counter: already borrowed"
            )),
        }
    }

    // Better approach: complete borrows before starting new ones
    fn safe_increment(&self) -> u64 {
        let mut value = self.0.borrow_mut();
        *value += 1;
        *value
    }
}
```

## Conclusion

<div class="warning">

Never use `unwrap()` or `expect()` in production code for your Ruby extensions. These can cause panics that will crash
the Ruby VM. Always use proper error handling with `Result` and `Error` types.

</div>

Effective error handling makes your Ruby extensions more robust and user-friendly. By using the right exception types
and providing clear error messages, you create a better experience for users of your extension.

Remember these key points:

- Use `Result<T, Error>` for functions that can fail
- Choose appropriate Ruby exception types
- Provide clear, detailed error messages
- Handle Rust panics to prevent VM crashes
- Respect Ruby's exception hierarchy

<div class="tip">

After you've handled errors in your Rust code, try to test your extension with invalid inputs to ensure it fails
gracefully with appropriate Ruby exceptions rather than crashing.

</div>
