% std::error

This module defines an `Error` trait representing the basic expectations for an error in Rust:

```ignore
pub trait Error: Debug + Display {
    fn description(&self) -> &str;
    fn cause(&self) -> Option<&Error> {
        None
    }
}
```

At a minimum, errors must provide a description via `description()`. Additional details can be addded optionally via the `Display` trait. The error cause chain can also be provided via `cause()`.

An example:

```rust
use std::error::Error;
use std::fmt;

#[derive(Debug)]
struct SuperError {
    side: SuperErrorSideKick,
}

impl fmt::Display for SuperError {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "SuperError here!")
    }
}

impl Error for SuperError {
    fn description(&self) -> &str {
        "A SuperError."
    }
    fn cause(&self) -> Option<&Error> {
        Some(&self.side)
    }
}

#[derive(Debug)]
struct SuperErrorSideKick;

impl fmt::Display for SuperErrorSideKick {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "SuperErrorSideKick here!")
    }
}

impl Error for SuperErrorSideKick {
    fn description(&self) -> &str {
        "A SuperError side kick."
    }
}


fn generate_super_error() -> Result<(), SuperError> {
    Err(SuperError { side: SuperErrorSideKick })
}



fn main() {
    match generate_super_error() {
        Err(e) => {
            println!("Error: {}", e.description());
            println!("Caused by: {}", e.cause().unwrap());
        },
        Ok(()) => unreachable!(),
    };
}
```