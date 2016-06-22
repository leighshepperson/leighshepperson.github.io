---
layout: post
title:  "Let's Build a Bank Account in Rust"
date:   2016-06-16 23:21:31 +0100
categories: jekyll update
---
### Introduction

In this post, we'll build a simple bank account in Rust. We'll cover the material quickly, so if you need more details, please see [this](https://doc.rust-lang.org/book/README.html).

### Install Rust

Make sure you've installed Rust - go to the [getting started guide](https://doc.rust-lang.org/book/getting-started.html) if you haven't done so already.

### Create a new project

We'll use cargo to create a new project. Let's call the project `simple_bank` and create it by running the command

{% highlight bash %}
cargo new simple_bank --bin
{% endhighlight %}

Now navigate to the root of the project and exectue the command `cargo run`. You should see the words "Hello, world!" in the console. Now we're ready to build our simple bank.

### Structs

We're going to model a bank account using a struct. In Rust, structs are just complex data types with named fields. So let's start by trying to define an `Account` struct:

{% highlight rust %}

struct Account {
    name: str,
    balance: f32
}

{% endhighlight %}

Now run the project using `cargo run`. Whoops -- you should see the following error:

{% highlight bash %}
"2:14 error: the trait `core::marker::Sized` is not implemented for the type `str` [E0277]"
{% endhighlight %}

You get this error because `str` is an unsized type - in particular, it's an immutable UTF-8 sequence of arbitrary length held somewhere in memory and it is accessed using a `&str` reference. (Here, `&str` is called a string slice and, in Rust, a slice is a view into a block of memory). Since `&str` is a sized type, let's try this instead:

{% highlight rust %}

struct Account {
    name: &str,
    balance: f32
}

{% endhighlight %}

Now run the project using `cargo run`. Double whoops -- you should see the following error:

{% highlight bash %}
"2:15 error: missing lifetime specifier [E0106]"
{% endhighlight %}

To understand this error, suppose you initialise a resource called `account_name` and then create an `Account` struct that borrows a reference to `account_name`:

{% highlight rust %}
let account_name = "John Smith";
let account = Account { name: account_name, balance: 100.0 };
{% endhighlight %}

Now suppose the `account_name` resource gets deallocated while the `Account` struct still holds a reference to it. If we try to access `account_name` using the `Account` struct, then the program will error. To stop this from happening, the `Account` struct needs to have the **same** lifetime as the `account_name` resource. We do this using the `'a` lifetime:

{% highlight rust %}
struct Account<'a> {
    name: &'a str,
    balance: f32
}
{% endhighlight %}

In Rust, lifetimes are a pretty complicated subject, but they also form the core of the language; you can find out more about them here:

[https://doc.rust-lang.org/book/lifetimes.html](https://doc.rust-lang.org/book/lifetimes.html)

[http://rustbyexample.com/scope/lifetime/explicit.html](http://rustbyexample.com/scope/lifetime/explicit.html)

Now let's create an `Account` and print a message to the console:

{% highlight rust %}
fn main() {
    let account = Account { name: "John Smith", balance: 200.0 };

    println!("The account name is {} and the current balance is {} pounds.", account.name, account.balance);
}
{% endhighlight %}

If you run `cargo run`, then you should see this:

{% highlight bash %}
"The account name is John Smith and the current balance is 200 pounds."
{% endhighlight %}

### Functions

Now let's write a function to wrap up the print statement we saw at the end of the last section:

{% highlight rust %}
fn info(account: &Account) -> String {
    format!("The account name is {} and the current balance is {} pounds.", account.name, account.balance)
}

fn main() {
    let account = Account { name: "John Smith", balance: 100.0 };

    println!("{:?}", info(&account));
}

{% endhighlight %}

If you run `cargo run`, then you should see this:

{% highlight bash %}
"The account name is John Smith and the current balance is 100 pounds."
{% endhighlight %}

Let's talk about the implementation:

First, notice that the function's parameter type is `&Account`. This means it accepts a reference to an `Account` struct. So to use it, we need to call the `info` method like this: `info(&account)`. 

The `format!` macro replaces the empty brackets with the values `account.name` and `account.balance` respectively. The return type of the `format!` macro is `String`. This is different from `str` in the basic sense that `str` is immutable and `String` is not. 

Notice that there is no `return` statement in our function. Essentially, when we want to return from a function or method, then we *don't* use a semicolon. If a function or method does not return anything, then we *use* a semicolon. (Note, the `return` keyword does exist, and it's useful for early returns.)

### Implementations

Given the `info` function in the previous section, it makes more sense to associate it with the `Account` struct. The way to do this is by defining an implementation of `Account`:

{% highlight rust %}
impl<'a> Account<'a>{
    fn info(&self) -> String {
        format!("The account name is {} and the current balance is {} pounds.", self.name, self.balance)
    }
}
{% endhighlight %}

Note, since we are using the `'a` lifetime, you need to make sure this matches up with the `impl` definition. 

You use the `info` method via the dot notation like this:

{% highlight rust %}
fn main() {
    let account_name = "John Smith";
    let account = Account { name: account_name, balance: 100.0 };

    println!("{:?}", account.info());
}
{% endhighlight %}

Now let's define a deposit method by extending the implementation of the `Account` struct: 

{% highlight rust %}
fn deposit(&mut self, amount: f32) {
    self.balance += amount;
}
{% endhighlight %}

In the deposit method, we get access to the `Account` struct's fields by using the `self` keyword. Since we want to modify its data, we need to use the `mut` keyword. By default, everything in Rust is immutable. So to make an object mutable, you need to explicitly state that fact. 

You can use the deposit method like this:

{% highlight rust %}
fn main() {
    let mut account = Account { name: "John Smith", balance: 200.0 };
    account.deposit(32.0);

    println!("{:?}", account.info());
}
{% endhighlight %}

and it should print out

{% highlight bash %}
"The account name is John Smith and the current balance is 232 pounds."
{% endhighlight %}

Now for the withdraw method. A simple implementation is to just do the reverse of the deposit method. But our simple bank does not allow overdrafts! So we want to tell the user that they are not allowed to withdraw more money than their current balance. To do this, we are going to use Rust's `Result` type. The `Result` type is an enum with two variants `Ok` and `Err`. In `Rust`, enums can have data, so we can give the `Err` variant a string message as its data. However, we don't want to return anything specific in the `Ok` variant, so we just return `Ok(())`. Let's define the withdraw method:

{% highlight rust %}
fn withdraw(&mut self, amount: f32) -> Result<(), String> {
    let is_sufficient_funds = self.balance - amount >= 0.0;

    match is_sufficient_funds {
        true => {
            self.balance -= amount;
            Ok(())
        },
        false => Err(format!("Unable to withdraw. {} has insufficient funds!", self.name))
    }
}
{% endhighlight %}

In the withdraw function, we've used pattern matching on the boolean value `is_sufficient_funds`. If this matches `true`, then we return the success variant `Ok`. Otherwise we just return the `Err` variant with an error message.

To use the method, we'll use pattern matching to print out the error message if it occurs: 

{% highlight rust %}
fn main() {
    let mut account = Account { name: "John Smith", balance: 200.0 };

    match account.withdraw(499.0) {
        Err(message) => println!("{:?}", message),
        Ok(_) => ()
    }

    match account.withdraw(10.0) {
        Err(message) => println!("{:?}", message),
        Ok(_) => ()
    }

     println!("{:?}", account.info());
}
{% endhighlight %}

The output of this should look like this:

{% highlight bash %}
"Unable to withdraw. John Smith has insufficient funds!"
"The account name is John Smith and the current balance is 190 pounds."
{% endhighlight %}

Finally, let's define a transfer method. This method will simply transfer money from one account to another. We can implement this by reusing the withdraw and deposit methods defined earlier. We do this as follows:

{% highlight rust %}
fn transfer(&mut self, recipient: &mut Account, amount: f32) -> Result<(), String> {
    try!(self.withdraw(amount));
    recipient.deposit(amount);
    Ok(())    
}
{% endhighlight %}

In the transfer method, we are using the `try!` macro. Error handling from Result types can be tedious in Rust, especially if we want the program to just continue if everything is OK. Essentially, if the line `try!(self.withdraw(amount))` returns an `Err` variant, then the variant is returned from the `transfer` function. Otherwise, it continues and finally returns `Ok`.

So let's finish up by putting everything together:

{% highlight rust %}

struct Account<'a> {
    name: &'a str,
    balance: f32
}

impl<'a> Account<'a>{
    fn info(&self) -> String {
        format!("The account name is {} and the current balance is {} pounds.", self.name, self.balance)
    }

    fn deposit(&mut self, amount: f32) {
        self.balance += amount;
    }

    fn withdraw(&mut self, amount: f32) -> Result<(), String> {
        let is_sufficient_funds = self.balance - amount >= 0.0;

        match is_sufficient_funds {
            true => {
                self.balance -= amount;
                Ok(())
            },
            false => Err(format!("Unable to withdraw. {} has insufficient funds!", self.name))
        }
    }

    fn transfer(&mut self, recipient: &mut Account, amount: f32) -> Result<(), String> {
        try!(self.withdraw(amount));
        recipient.deposit(amount);
        Ok(())
    }
}

fn main() {
    let mut johns_account = Account { name: "John Smith", balance: 200.0 };
    let mut janes_account = Account { name: "Jane Doe", balance: 300.0 };

    println!("{:?}", "Let's look at John's and Jane's current accounts:" );

    println!("{:?}", johns_account.info());
    println!("{:?}", janes_account.info());

    println!("{:?}", "Now let's try to transfer 300 pounds from John to Jane:");

    match johns_account.transfer(&mut janes_account, 300.0) {
        Err(message) => println!("{:?}", message),
        Ok(_) => ()
    }

    println!("{:?}", "Now let's try to transfer 100 pounds from John to Jane:");

    match johns_account.transfer(&mut janes_account, 100.0) {
        Err(message) => println!("{:?}", message),
        Ok(_) => ()
    }

    println!("{:?}", johns_account.info());
    println!("{:?}", janes_account.info());
}

{% endhighlight %}

And this should output:

{% highlight bash %}
"Let's look at John's and Jane's current accounts:"
"The account name is John Smith and the current balance is 200 pounds."
"The account name is Jane Doe and the current balance is 300 pounds."
"Now let's try to transfer 300 pounds from John to Jane:"
"Unable to withdraw. John Smith has insufficient funds!"
"Now let's try to transfer 100 pounds from John to Jane:"
"The account name is John Smith and the current balance is 100 pounds."
"The account name is Jane Doe and the current balance is 400 pounds."
{% endhighlight %}