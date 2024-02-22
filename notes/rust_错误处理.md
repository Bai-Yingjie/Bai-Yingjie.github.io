Beginner's guide to Error Handling in Rust
==========================================

- [Beginner's guide to Error Handling in Rust](#beginners-guide-to-error-handling-in-rust)
  - [Ignore the error](#ignore-the-error)
  - [Terminate the program](#terminate-the-program)
  - [Use a fallback value](#use-a-fallback-value)
  - [Bubble up the error](#bubble-up-the-error)
  - [Bubble up multiple errors](#bubble-up-multiple-errors)
  - [Match boxed errors](#match-boxed-errors)
  - [Applications vs Libraries](#applications-vs-libraries)
    - [Application](#application)
    - [Library](#library)
  - [Create custom errors](#create-custom-errors)
  - [Bubble up custom errors](#bubble-up-custom-errors)
  - [Match custom errors](#match-custom-errors)
  - [Conclusion](#conclusion)


Error handling in Rust is very different if you're coming from other languages. In languages like Java, JS, Python etc, you usually `throw` exceptions and `return` successful values. In Rust, you return something called a `Result`.

The `Result<T, E>` type is an [enum](https://doc.rust-lang.org/book/ch06-01-defining-an-enum.html) that has two variants - `Ok(T)` for successful value or `Err(E)` for error value:

```rust
enum Result<T, E> {
   Ok(T),
   Err(E),
}
```

Returning errors instead of throwing them is a paradigm shift in error handling. If you're new to Rust, there will be some friction initially as it requires you to reason about errors in a different way.

In this post, I'll go through some common error handling patterns so you gradually become familiar with how things are done in Rust:

-   Ignore the error
-   Terminate the program
-   Use a fallback value
-   Bubble up the error
-   Bubble up multiple errors
-   Match boxed errors
-   Libraries vs Applications
-   Create custom errors
-   Bubble up custom errors
-   Match custom errors

Ignore the error
----------------

Let's start with the simplest scenario where we just ignore the error. This sounds careless but has a couple of legitimate use cases:

-   We're prototyping our code and don't want to spend time on error handling.
-   We're confident that the error won't occur.

Let's say that we're reading a file which we're pretty sure would be present:

```rust
use std::fs;

fn main() {
  let content = fs::read_to_string("./Cargo.toml").unwrap();
  println!("{}", content)
}
```

Even though we know that the file would be present, the compiler has no way of knowing that. So we use `unwrap` to tell the compiler to trust us and return the value inside. If the `read_to_string` function returns an `Ok()` value, `unwrap` will get the contents of `Ok` and assign it to the `content` variable. If it returns an error, it will "panic". Panic either terminates the program or exits the current thread.

Note that `unwrap` is used in quite a lot of Rust examples to skip error handling. This is mostly done for convenience and shouldn't be used in real code as it is.

Terminate the program
---------------------

Some errors cannot be handled or recovered from. In these cases, it's better to *fail fast* by terminating the program.

Let's use the same example as above - we're reading a file which we're sure to be present. Let's imagine that, for this program, that file is absolutely important without which it won't work properly. If for some reason, this file is absent, it's better to terminate the program.

We can use `unwrap` as before or use `expect` \- it's same as `unwrap` but lets us add extra error message.

```rust
use std::fs;

fn main() {
  let content = fs::read_to_string("./Cargo.toml").expect("Can't read Cargo.toml");
  println!("{}", content)
}
```

See also: [`panic!`](https://doc.rust-lang.org/std/macro.panic.html)

Use a fallback value
--------------------

In some cases, you can handle the error by falling back to a default value.

For example, let's say we're writing a server and the port it listens to can be configured using an environment variable. If the environment variable is not set, accessing that value would result in an error. But we can easily handle that by falling back to a default value.

```rust
use std::env;

fn main() {
  let port = env::var("PORT").unwrap_or("3000".to_string());
  println!("{}", port);
}
```

Here, we've used a variation of `unwrap` called `unwrap_or` which lets us supply default values.

See also: [`unwrap_or_else`](https://doc.rust-lang.org/std/result/enum.Result.html#method.unwrap_or_else), [`unwrap_or_default`](https://doc.rust-lang.org/std/result/enum.Result.html#method.unwrap_or_default)

Bubble up the error
-------------------

When you don't have enough context to handle the error, you can bubble up (propagate) the error to the caller function.

Here's a contrived example which uses a webservice to get the current year:

```rust
use std::collections::HashMap;

fn main() {
  match get_current_date() {
    Ok(date) => println!("We've time travelled to {}!!", date),
    Err(e) => eprintln!("Oh noes, we don't know which era we're in! :( \n  {}", e),
  }
}

fn get_current_date() -> Result<String, reqwest::Error> {
  let url = "https://postman-echo.com/time/object";
  let result = reqwest::blocking::get(url);

  let response = match result {
    Ok(res) => res,
    Err(err) => return Err(err),
  };

  let body = response.json::<HashMap<String, i32>>();

  let json = match body {
    Ok(json) => json,
    Err(err) => return Err(err),
  };

  let date = json["years"].to_string();

  Ok(date)
}
```

There are two function calls inside the `get_current_date` function (`get` and `json`) that return `Result` values. Since `get_current_date` doesn't have context of what to do when they return errors, it uses pattern matching to propagate the errors to `main`.

Using pattern matching to handle multiple or nested errors can make your code "noisy". Instead, we can rewrite the above code using the [`?` operator](https://doc.rust-lang.org/book/ch09-02-recoverable-errors-with-result.html#a-shortcut-for-propagating-errors-the--operator):

```rust
use std::collections::HashMap;

fn main() {
  match get_current_date() {
    Ok(date) => println!("We've time travelled to {}!!", date),
    Err(e) => eprintln!("Oh noes, we don't know which era we're in! :( \n  {}", e),
  }
}

fn get_current_date() -> Result<String, reqwest::Error> {
  let url = "https://postman-echo.com/time/object";
  let res = reqwest::blocking::get(url)?.json::<HashMap<String, i32>>()?;
  let date = res["years"].to_string();

  Ok(date)
}
```

This looks much cleaner!

The `?` operator is similar to `unwrap` but instead of panicking, it propagates the error to the calling function. One thing to keep in mind is that we can use the `?` operator only for functions that return a `Result` or `Option` type.

Bubble up multiple errors
-------------------------

In the previous example, the `get` and `json` functions return a `reqwest::Error` error which we've propagated using the `?` operator. But what if we've another function call that returned a different error value?

Let's extend the previous example by returning a formatted date instead of the year:

```rust
+ use chrono::NaiveDate;
  use std::collections::HashMap;

  fn main() {
    match get_current_date() {
      Ok(date) => println!("We've time travelled to {}!!", date),
      Err(e) => eprintln!("Oh noes, we don't know which era we're in! :( \n  {}", e),
    }
  }

  fn get_current_date() -> Result<String, reqwest::Error> {
    let url = "https://postman-echo.com/time/object";
    let res = reqwest::blocking::get(url)?.json::<HashMap<String, i32>>()?;
-   let date = res["years"].to_string();
+   let formatted_date = format!("{}-{}-{}", res["years"], res["months"] + 1, res["date"]);
+   let parsed_date = NaiveDate::parse_from_str(formatted_date.as_str(), "%Y-%m-%d")?;
+   let date = parsed_date.format("%Y %B %d").to_string();

    Ok(date)
  }
```

The above code won't compile as `parse_from_str` returns a `chrono::format::ParseError` error and not `reqwest::Error`.

We can fix this by `Box`ing the errors:

```rust
  use chrono::NaiveDate;
  use std::collections::HashMap;

  fn main() {
    match get_current_date() {
      Ok(date) => println!("We've time travelled to {}!!", date),
      Err(e) => eprintln!("Oh noes, we don't know which era we're in! :( \n  {}", e),
    }
  }

- fn get_current_date() -> Result<String, reqwest::Error> {
+ fn get_current_date() -> Result<String, Box<dyn std::error::Error>> {
    let url = "https://postman-echo.com/time/object";
    let res = reqwest::blocking::get(url)?.json::<HashMap<String, i32>>()?;

    let formatted_date = format!("{}-{}-{}", res["years"], res["months"] + 1, res["date"]);
    let parsed_date = NaiveDate::parse_from_str(formatted_date.as_str(), "%Y-%m-%d")?;
    let date = parsed_date.format("%Y %B %d").to_string();

    Ok(date)
  }
```

Returning a trait object `Box<dyn std::error::Error>` is very convenient when we want to return multiple errors!

See also: [`anyhow`](https://github.com/dtolnay/anyhow), [`eyre`](https://github.com/yaahc/eyre)

Match boxed errors
------------------

So far, we've only printed the errors in the `main` function but not handled them. If we want to handle and recover from boxed errors, we need to "downcast" them:

```rust
  use chrono::NaiveDate;
  use std::collections::HashMap;

  fn main() {
    match get_current_date() {
      Ok(date) => println!("We've time travelled to {}!!", date),
-     Err(e) => eprintln!("Oh noes, we don't know which era we're in! :( \n  {}", e),
+     Err(e) => {
+       eprintln!("Oh noes, we don't know which era we're in! :(");
+       if let Some(err) = e.downcast_ref::<reqwest::Error>() {
+         eprintln!("Request Error: {}", err)
+       } else if let Some(err) = e.downcast_ref::<chrono::format::ParseError>() {
+         eprintln!("Parse Error: {}", err)
+       }
+     }
    }
  }

  fn get_current_date() -> Result<String, Box<dyn std::error::Error>> {
    let url = "https://postman-echo.com/time/object";
    let res = reqwest::blocking::get(url)?.json::<HashMap<String, i32>>()?;

    let formatted_date = format!("{}-{}-{}", res["years"], res["months"] + 1, res["date"]);
    let parsed_date = NaiveDate::parse_from_str(formatted_date.as_str(), "%Y-%m-%d")?;
    let date = parsed_date.format("%Y %B %d").to_string();

    Ok(date)
  }
```

Notice how we need to be aware of the implementation details (different errors inside) of `get_current_date` to be able to downcast them inside `main`.

See also: [`downcast`](https://doc.rust-lang.org/std/error/trait.Error.html#method.downcast), [`downcast_mut`](https://doc.rust-lang.org/std/error/trait.Error.html#method.downcast_mut)

Applications vs Libraries
-------------------------

As mentioned previously, the downside to boxed errors is that if we want to handle the underlying errors, we need to be aware of the implementation details. When we return something as `Box<dyn std::error::Error>`, the concrete type information is erased. To handle the different errors in different ways, we need to downcast them to concrete types and this casting can fail at runtime.

However, saying something is a "downside" is not very useful without context. A good rule of thumb is to question whether the code you're writing is an "application" or a "library":

### Application

-   The code you're writing would be used by end users.
-   Most errors generated by application code won't be handled but instead logged or reported to the user.
-   It's okay to use boxed errors.

### Library

-   The code you're writing would be consumed by other code. A "library" can be open source crates, internal libraries etc
-   Errors are part of your library's API, so your consumers know what errors to expect and recover from.
-   Errors from your library are often handled by your consumers so they need to be structured and easy to perform [exhaustive match](https://doc.rust-lang.org/1.30.0/book/2018-edition/ch06-02-match.html#matches-are-exhaustive) on.
-   If you return boxed errors, then your consumers need to be aware of the errors created by your code, your dependencies, and so on!
-   Instead of boxed errors, we can return custom errors.

Create custom errors
--------------------

For library code, we can convert all the errors to our own custom error and propagate them instead of boxed errors. In our example, we currently have two errors - `reqwest::Error` and `chrono::format::ParseError`. We can convert them to `MyCustomError::HttpError` and `MyCustomError::ParseError` respectively.

Let's start by creating an enum to hold our two error variants:

```rust
// error.rs

pub enum MyCustomError {
  HttpError,
  ParseError,
}
```

The [`Error`](https://doc.rust-lang.org/std/error/trait.Error.html) trait requires us to implement the [`Debug`](https://doc.rust-lang.org/std/fmt/trait.Debug.html) and [`Display`](https://doc.rust-lang.org/std/fmt/trait.Display.html) traits:

```rust
// error.rs

use std::fmt;

#[derive(Debug)]
pub enum MyCustomError {
  HttpError,
  ParseError,
}

impl std::error::Error for MyCustomError {}

impl fmt::Display for MyCustomError {
  fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
    match self {
      MyCustomError::HttpError => write!(f, "HTTP Error"),
      MyCustomError::ParseError => write!(f, "Parse Error"),
    }
  }
}
```

We've created our own custom error!

This is obviously a simple example as the error variants don't contain much information about the error. But this should be sufficient as a starting point for creating more complex and realistic custom errors. Here are some real life examples: [ripgrep](https://github.com/BurntSushi/ripgrep/blob/12.1.1/crates/regex/src/error.rs), [reqwest](https://github.com/seanmonstar/reqwest/blob/v0.10.7/src/error.rs), [csv](https://github.com/BurntSushi/rust-csv/blob/master/src/error.rs) and [serde\_json](https://github.com/serde-rs/json/blob/master/src/error.rs)

See also: [`thiserror`](https://github.com/dtolnay/thiserror), [`snafu`](https://github.com/shepmaster/snafu)

Bubble up custom errors
-----------------------

Let's update our code to return the custom errors we just created:

```rust
  // main.rs

+ mod error;

  use chrono::NaiveDate;
+ use error::MyCustomError;
  use std::collections::HashMap;

  fn main() {
    // skipped, will get back later
  }

- fn get_current_date() -> Result<String, Box<dyn std::error::Error>> {
+ fn get_current_date() -> Result<String, MyCustomError> {
    let url = "https://postman-echo.com/time/object";
-   let res = reqwest::blocking::get(url)?.json::<HashMap<String, i32>>()?;
+   let res = reqwest::blocking::get(url)
+     .map_err(|_| MyCustomError::HttpError)?
+     .json::<HashMap<String, i32>>()
+     .map_err(|_| MyCustomError::HttpError)?;

    let formatted_date = format!("{}-{}-{}", res["years"], res["months"] + 1, res["date"]);
-   let parsed_date = NaiveDate::parse_from_str(formatted_date.as_str(), "%Y-%m-%d")?;
+   let parsed_date = NaiveDate::parse_from_str(formatted_date.as_str(), "%Y-%m-%d")
+     .map_err(|_| MyCustomError::ParseError)?;
    let date = parsed_date.format("%Y %B %d").to_string();

    Ok(date)
  }
```

Notice how we're using `map_err` to convert the error from one type to another type.

But things got verbose as a result - our function is littered with these `map_err` calls. We can implement the [`From`](https://doc.rust-lang.org/std/convert/trait.From.html) trait to automatically coerce the error types when we use the `?` operator:

```rust
  // error.rs

  use std::fmt;

  #[derive(Debug)]
  pub enum MyCustomError {
    HttpError,
    ParseError,
  }

  impl std::error::Error for MyCustomError {}

  impl fmt::Display for MyCustomError {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
      match self {
        MyCustomError::HttpError => write!(f, "HTTP Error"),
        MyCustomError::ParseError => write!(f, "Parse Error"),
      }
    }
  }

+ impl From<reqwest::Error> for MyCustomError {
+   fn from(_: reqwest::Error) -> Self {
+     MyCustomError::HttpError
+   }
+ }

+ impl From<chrono::format::ParseError> for MyCustomError {
+   fn from(_: chrono::format::ParseError) -> Self {
+     MyCustomError::ParseError
+   }
+ }
```
```rust
  // main.rs

  mod error;

  use chrono::NaiveDate;
  use error::MyCustomError;
  use std::collections::HashMap;

  fn main() {
    // skipped, will get back later
  }

  fn get_current_date() -> Result<String, MyCustomError> {
    let url = "https://postman-echo.com/time/object";
-   let res = reqwest::blocking::get(url)
-     .map_err(|_| MyCustomError::HttpError)?
-     .json::<HashMap<String, i32>>()
-     .map_err(|_| MyCustomError::HttpError)?;
+   let res = reqwest::blocking::get(url)?.json::<HashMap<String, i32>>()?;

    let formatted_date = format!("{}-{}-{}", res["years"], res["months"] + 1, res["date"]);
-   let parsed_date = NaiveDate::parse_from_str(formatted_date.as_str(), "%Y-%m-%d")
-     .map_err(|_| MyCustomError::ParseError)?;
+   let parsed_date = NaiveDate::parse_from_str(formatted_date.as_str(), "%Y-%m-%d")?;
    let date = parsed_date.format("%Y %B %d").to_string();

    Ok(date)
  }
```

We've removed `map_err` and the code looks much cleaner!

However, `From` trait is not magic and there are times when we need to use `map_err`. In the above example, we've moved the type conversion from inside the `get_current_date` function to the `From<X> for MyCustomError` implementation. This works well when the information needed to convert from one error to `MyCustomError` can be obtained from the original error object. If not, we need to use `map_err` inside `get_current_date`.

Match custom errors
-------------------

We've ignored the changes in `main` until now, here's how we can handle the custom errors:

```rust
  // main.rs

  mod error;

  use chrono::NaiveDate;
  use error::MyCustomError;
  use std::collections::HashMap;

  fn main() {
    match get_current_date() {
      Ok(date) => println!("We've time travelled to {}!!", date),
      Err(e) => {
        eprintln!("Oh noes, we don't know which era we're in! :(");
-       if let Some(err) = e.downcast_ref::<reqwest::Error>() {
-         eprintln!("Request Error: {}", err)
-       } else if let Some(err) = e.downcast_ref::<chrono::format::ParseError>() {
-         eprintln!("Parse Error: {}", err)
-       }
+       match e {
+         MyCustomError::HttpError => eprintln!("Request Error: {}", e),
+         MyCustomError::ParseError => eprintln!("Parse Error: {}", e),
+       }
      }
    }
  }

  fn get_current_date() -> Result<String, MyCustomError> {
    let url = "https://postman-echo.com/time/object";
    let res = reqwest::blocking::get(url)?.json::<HashMap<String, i32>>()?;

    let formatted_date = format!("{}-{}-{}", res["years"], res["months"] + 1, res["date"]);
    let parsed_date = NaiveDate::parse_from_str(formatted_date.as_str(), "%Y-%m-%d")?;
    let date = parsed_date.format("%Y %B %d").to_string();

    Ok(date)
  }
```

Notice how unlike boxed errors, we can actually match on the variants inside `MyCustomError` enum.

Conclusion
----------

Thanks for reading! I hope this post was helpful in introducing the basics of error handling in Rust. I've added the examples to a [repo in GitHub](https://github.com/sheshbabu/rust-error-handling-examples) which you can use for practice. If you've more questions, please contact me at sheshbabu \[at\] gmail.com. Feel free to follow me in [Twitter](https://twitter.com/sheshbabu) for more posts like this :)

Tagged under: [Rust](https://www.sheshbabu.com/tags/Rust/), [Rust Beginners](https://www.sheshbabu.com/tags/Rust-Beginners/)
Published on: 02 Aug 2020

From: https://www.sheshbabu.com/posts/rust-error-handling/