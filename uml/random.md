    fn main() {
        // What is Rust:

        // Blurb from the Rust home page:
        // Rust is a systems programming language that runs blazingly fast,
        // prevents segfaults, and guarantees thread safety.

        // We'll focus on segfaults, and how Rust is designed to stop them from
        // happening.

        // So what is a segfault? A segfault (segmentation fault) is where a program
        // overwrites or gets access to a block of memory in an illegal way. For
        // example, if the program tries to overwrite something that's read-only or
        // if it accidentally modifies a block of memory that some other process
        // depends on.

        // Rust solves the segfault problem by having a complicated ownership model;
        // and this is what we'll talk about here.

        // Firstly:

        // Everyting is immutable in Rust by default. If we want to explicitly make
        // a variable mutable, then we have to use the mutate keyword. For example,

        let mut x = 3;
        x = 5;
        println!("{:?}", x);

        // Ownership:

        // So let's instantiate a vector:
        let v = vec![1, 2, 3, 4];

        // In rust, a vector is a type that dynamically allocates elements on the
        // heap at run-time. They are similar to reference types in C#.

        // In particular:

        // On the stack, a vector gets created and on the heap an area of
        // memory is created to hold its elements. As soon as the vector goes
        // out of scope, the vector and everything related
        // to it including the elements on the heap are collected by the
        // garbage collector. This is deterministic and happens instantly,
        // unlike C#, Erlang, etc...

        // For any resource, there can only be one explicit variable binding.

        // So suppose we write
        let v1 = v;

        // If we try to print out the first vector, then we'll get an error:
        // println!("{:?}", v);

        // However, changing v to v1 in the above statement fixes this.
        println!("{:?}", v1);

        // We also have copy types. Copy types are like value types in C#.

        // For example, let's say we define an integer
        let i = 1;

        // Now say we rebind this to a variable j. Then we get
        let j = i;

        // There was no error!

        // We find that we can still print out i and j
        println!("{:?}", i);
        println!("{:?}", j);

        // What's happened is that the data of i has been
        // copied and j is a new referance that points to the copied data.

        // Borrowing:

        // You could imagine that it would become quite tedious in Rust to
        // manipulate vectors since there can only be one explicit binding.

        // This is why Rust has introduced the concept of Borrowing references.
        // So lets see this in action.

        // Suppose we want to take a reference to a vector. Then this is written
        // using an ampsand before the variable name. If we set something equal
        // to the refence, then we have said that we have borrowed that reference.

        let vector = vec![1, 2, 3, 4];

        let borrowed_vector_reference = &vector;

        // If the resource is immutable, then you can borrow many references.
        // For example:
        let u = vec![1, 2, 3, 4];
        let u1 = &u;
        let u2 = &u;

        println!("{:?}", u);
        println!("{:?}", u1);
        println!("{:?}", u2);

        // While you can borrow many references of an immutable element in Rust,
        // you can only borrow one mutable reference. For example:

        // n is a mutable variable
        let mut n = 4;

        // n1 is a mutable reference to n
        let n1 = &mut n;

        // Unwrap the resource of n1 and set it equal to 3
        *n1 = 3;

        // This works, but if you change n1 to n in the following print statement,
        // then it will error, since you can only have one mutable reference to
        // a resource at a time.
        println!("{:?}", n1);

        // Why is this the case?
        // Recall that a segfault (segmentation fault) is where a program
        // overwrites or gets access to a block of memory in an illegal way. For
        // example, if the program tries to overwrite something that's read-only or
        // if it accidentally modifies a block of memory that some other process
        // depends on.

        // The ownership model in Rust stops this from happening. Since you can only
        // have multiple references to immutable objects, you can never overwrite the
        // referenced data so you are safe. Aditinally, since you can only have one
        // mutable reference to a resource, it is guaranteed that nothing else can
        // modify that resourse. This is how Rust solves segmentation fault problem.
    }