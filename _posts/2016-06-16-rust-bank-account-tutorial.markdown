---
layout: post
title:  "Let's Build a Bank Account in Rust"
date:   2016-06-16 23:21:31 +0100
categories: jekyll update
---
### Introduction

In this post, we'll build a simple bank account in Rust. We'll cover the material in a whirlwind fashion, so if you need more details, have a read of [this](https://doc.rust-lang.org/book/README.html).

### Install Rust

Make sure you've installed Rust - go to the [getting started guide](https://doc.rust-lang.org/book/getting-started.html) if you haven't done so already.

### Create a new project

We're going to use cargo to create the new project. Lets call it `simple_bank` and create it using the command

{% highlight bash %}
cargo new simple_bank --bin
{% endhighlight %}

Now navigate to the root of the project and run the command `cargo run`. You should see the words "Hello, world!" in the console. We're now ready to build our simple bank.

### Structs

We're going to model a bank account using a simple struct. In Rust, structs are complex data types that have named fields. So let's start by trying to define an `Account` struct that has two fields `name` and `balance` where the name is a string `str` and the balance is a 32 bit float `f32`:

{% highlight rust %}

struct Account {
    name: str,
    balance: f32
}

{% endhighlight %}

Whoops - if you compile it using `cargo run`, then you'll see the following error:

{% highlight bash %}
"2:14 error: the trait `core::marker::Sized` is not implemented for the type `str` [E0277]"
{% endhighlight %}

You get this error because `str` is an unsized type - in particular, it's an immutable UTF-8 sequence held in memory and is normally accessed using a `&str` reference. (Here, `&str` is called a string slice and, in Rust, a slice is just a view into a block of memory). So let's try this instead:

{% highlight rust %}

struct Account {
    name: &str,
    balance: f32
}

{% endhighlight %}

Double whoops - if you compile it, then you'll see the following error:

{% highlight bash %}
"2:15 error: missing lifetime specifier [E0106]"
{% endhighlight %}

To understand this, suppose we initialise a variable called `account_name` of type `&str` and then create an `Account` struct that borrows a reference to `account_name`:

{% highlight rust %}
let account_name = "John Smith";
let account = Account { name: account_name, balance: 100.0 };
{% endhighlight %}

Now suppose the `account_name` resource gets deallocated but the `Account` struct still holds a reference to it. If we try to access `account_name` using the `Account` struct, then the program will probably break. To stop this from happening, the `Account` struct needs to have the **same** lifetime as the `account_name`. We can do this by using the `'a` lifetime:

{% highlight rust %}

struct Account<'a> {
    name: &'a str,
    balance: f32
}

{% endhighlight %}

Lifetimes are a complicated subject; you can find out more about them here:

[https://doc.rust-lang.org/book/lifetimes.html](https://doc.rust-lang.org/book/lifetimes.html)

[http://rustbyexample.com/scope/lifetime/explicit.html](http://rustbyexample.com/scope/lifetime/explicit.html)

Now lets create an `Account` and print a message to the console:

{% highlight rust %}
fn main() {
    let account = Account { name: "John Smith", balance: 200.0 };

    println!("The account name is {} and the current balance is {} pounds", account.name, account.balance);
}
{% endhighlight %}

If you run `cargo run`, then you should see this printed on the screen:

{% highlight bash %}
The account name is John Smith and the current balance is 200 pounds
{% endhighlight %}

### Functions

Now lets write a function to return some information about an `Account`.

{% highlight rust %}
fn info(account: &Account) -> String {
    format!("The account name is {} and the current balance is {} pounds.", account.name, account.balance)
}

fn main() {
    let account = Account { name: "John Smith", balance: 100.0 };

    println!("{:?}", info(&account));
}

{% endhighlight %}

If you run `cargo run`, then you should see the following printed out:

{% highlight bash %}
"The account name is John Smith and the current balance is 100 pounds."
{% endhighlight %}

Lets talk about the implementation. 

First, notice that we are using `&Account` as the function's parameter type. This means it accepts a reference to an `Account` struct. So to use it, we need to call the `info` method like this: `info(&account)`. 

The `format!` macro replaces the empty brackets with the values `account.name` and `account.balance`, just like the `string.Format` method in C#.

The return type of the `format!` macro is `String`. This is different from `str` in the sense that `str` is immutable and `String` is not. 

Notice that there is no `return` statement. Essentially, when we want to return from a function/method, then we don't use a semicolon. If a function/method does not return anything, then we use a semicolon. (Note, the `return` keyword does exist, and it's useful for early returns.)

### Implementations

Given what the `info` function does, it makes sense to associate it with the `Account` struct. The way to do this is by defining an implementation of `Account`:

{% highlight rust %}
impl<'a> Account<'a>{
    fn info(&self) -> String {
        format!("The account name is {} and the current balance is {} pounds.", self.name, self.balance)
    }
}
{% endhighlight %}

Since we are using the `'a` lifetime, you need to make sure this matches up with the `impl` definition. 

You use the `info` method via the dot notation like this:

{% highlight rust %}

fn main() {
    let account_name = "John Smith";
    let account = Account { name: account_name, balance: 100.0 };

    println!("{:?}", account.info());
}
{% endhighlight %}

Now let's define the deposit method by extending the implementation of the `Account` struct: 

{% highlight rust %}
fn deposit(&mut self, amount: f32) {
    self.balance += amount;
}
{% endhighlight %}

Note, we get access to the `Account` struct's fields by using the `self` keyword. Since we want to modify its data, we need to use the `mut` keyword. By default, everything in Rust is immutable. So to make an object mutable, you need to explicitly state that fact. 

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

Now for the withdraw method. A simple implementation is to do the reverse of the deposit method. But our simple bank does not allow overdrafts! So we want to tell the user that they are not allowed to withdraw more money than what's in their current account. To do this, we are going to use Rust's `Result` type. The `Result` type is an enum with the two variants `Ok` and `Err`. In `Rust`, enums can have data, so we can give the `Err` variant a string message to return. However, we don't want to return anything specific in the `Ok` variant, so we just return do `Ok(())`. Let's define the withdraw method:

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

There are a few things to note about this: We've used pattern matching on the boolean `is_sufficient_funds`. If it matches `true`, then we return the success variant `Ok`. Otherwise we just return the `Err` variant with a useful error message.

To use the method, we'll again use pattern matching to print out the error message if it occurs: 

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

Finally, lets define a transfer method. This method will simply move money from one account to another. We can implement this by reusing the withdraw and deposit methods defined earlier. We do this as follows:

{% highlight rust %}
fn transfer(&mut self, recipient: &mut Account, amount: f32) -> Result<(), String> {
    try!(self.withdraw(amount));
    recipient.deposit(amount);
    Ok(())    
}
{% endhighlight %}

The thing to notice about this is the `try!` macro. Error handling from Result types can be tedious in Rust, especially if we want the program to continue when everything is OK and you have to do do lots of patten matching to get there. Essentially, if the line `try!(self.withdraw(amount))` returns the `Err` variant, then that variant is returned from the `transfer` function. Otherwise, it continues and deposits the amount into the recipient's account and finally returns `Ok`.

So let's end by putting everything together:

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

fn info(account: &Account) -> String {
    format!("The account name is {} and the current balance is {} pounds.", account.name, account.balance)
}

fn main() {
    let mut johns_account = Account { name: "John Smith", balance: 200.0 };
    let mut janes_account = Account { name: "Jane Doe", balance: 300.0 };

    println!("{:?}", "Lets look at John's and Jane's current accounts:" );

    println!("{:?}", johns_account.info());
    println!("{:?}", janes_account.info());

    println!("{:?}", "Now lets try to transfer 300 pounds from John to Jane:");

    match johns_account.transfer(&mut janes_account, 300.0) {
        Err(message) => println!("{:?}", message),
        Ok(_) => ()
    }

    println!("{:?}", "Now lets try to transfer 100 pounds from John to Jane:");

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
"Lets look at Johns and Janes current accounts:"
"The account name is John Smith and the current balance is 200 pounds."
"The account name is Jane Doe and the current balance is 300 pounds."
"Now lets try to transfer 300 pounds from John to Jane:"
"Unable to withdraw. John Smith has insufficient funds!"
"Now lets try to transfer 100 pounds from John to Jane:"
"The account name is John Smith and the current balance is 100 pounds."
"The account name is Jane Doe and the current balance is 400 pounds."
{% endhighlight %}