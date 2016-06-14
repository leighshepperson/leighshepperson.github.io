---
layout: post
title:  "Build a register virtual machine in Rust"
date:   2016-06-03 23:21:31 +0100
categories: jekyll update
---
# Let's build a virtual machine in Rust

In this post we create a simple register based virtual machine to compute the greatest common divisor of a pair of small integers. Virtual machines are usually written in system level languages, so for this example, we will build the virtual machine in Mozilla's Rust language.

### Registers

In a physical processor, a *register* is an area of data storage available to a central processing unit (CPU). They are used to speed up programs so they don't have to access the main memory for simple operations like add and subtract. 

Furthermore, *registers files* are arrays of registers.

The virtual machine needs four registers. An *instruction pointer* `ip` to keep track of the next instruction in the program and three arithmetic registers in a register file called `registers`. We'll initialise these inside the cpu function:

{% highlight rust %}
type RegisterFile = [u16; 3];

fn cpu() {
    let mut ip = 0;
    let mut registers = RegisterFile::default();
}
{% endhighlight %}

Since `RegisterFile` is a type alias of `[u16; 3]` it has access to all its inherent methods. So the line

{% highlight rust %}
let mut registers = RegisterFile::default();
{% endhighlight %}

calls the default function on `[u16; 3]` giving

{% highlight rust %}
registers = RegisterFile::default() = [0, 0, 0];
{% endhighlight %}

Alternatively, you could declare the registers as global mutable variables outside the `cpu` function so other functions or methods can use them without passing them in as parameters. However, it’s unsafe to do this in multithreaded applications, so the Rust language forces you to use the `unsafe` code block if you want to do this. For example, if you wanted to print out the value of a global mutable version of the instruction pointer, then you would have to do it like so:

{% highlight rust %}
unsafe {
    println!("{:?}", ip);
}
{% endhighlight %}

### The Fetch, Decode, Execute Cycle

The `cpu` function performs the fetch, decode, execute cycle. First, the instruction pointer **fetches** an instruction from the instruction set. Here, the *instruction set* is an array of instructions encoded as 16 bit words, such as `[0xFF0D, 0xAB0A]`. Then, the instruction is **decoded** and the CPU **executes** it by performing an appropriate action. Finally, the instruction pointer is incremented and the process continues until it halts.

{% plantuml %}
start

repeat
  :fetch;
  :decode;
  :execute;
repeat while (halt?)

stop
{% endplantuml %}

We give more details in the following sections:

### Instructions

The virtual machine reads from an array of instructions encoded as 16 bit words. For example:

{% highlight rust %}
let encoded_instructions = [0xFFFD, 0xFF2A, 0x1121];
{% endhighlight %}

An *opcode* is the part of the encoded instruction that specifies the operation that gets performed. In our encoding, the first 4 bits of an encoded instruction represent the opcode; the remaining bits specify the parameters.

The following table defines the opcodes used by the virtual machine:

| Opcode        | Description           |
| ------------- |-------------|
| Halt      | terminates the program and prints out the value of the 0th arithmetic register |
| Load      | loads an 8 bit unsigned integer into the destination register      |
| Swap | swaps the data in the destination register with the data in the source register.  You must specify a temporary register   |
| Mod   | computes the modulus of two register values and puts the result in a new register   |
| BEZ   | moves the instruction pointer back by offset-many places only if the register's value is equal to zero


<br />
The following table defines the instruction encoding:

| Opcode   | 1st hex digit   | 2nd hex digit   | 3rd hex digit   | 4th hex digit   |
| ------------- |-------------| ------------- |-------------| -------------| 
| Halt   | 0   | -   | -   | -   |
| Load   | 1   | destination register   | value   | *   |
| Swap   | 2   | destination register   | source register   | temporary register   |
| Mod   | 3   | dividend  | divisor  | destination   |
| BEZ   | 4   | offset   | *   | *   |

<br />
Here, we use "*" to mean the hex digits to the right are part of the same value. For example, the value in the **Load** instruction is an 8 bit integer, not two 4 bit integers. The "-" means we ignore that hex digit.

In Rust, enum variants can have data associated to them. So we use an enum to represent instructions, where each variant represents the opcode and its data are the parameters it operates on:

```rust
enum Instruction {
    Halt,
    Load { destination: usize, value: u16 },
    Swap { destination: usize, source: usize, temp: usize },
    Mod { dividend: usize, divisor: usize, destination: usize},
    BEZ { offset: usize }
}
```


### Fetch

To implement the *fetch* operation, we will first define the `Program` struct:

{% highlight rust %}
struct Program<'a> {
    instructions: &'a [u16],
}
{% endhighlight %}

The `instructions` field is an array slice that uses an explicit `'a` lifetime. Here, an *array slice* is an array with size not known at compile time and the *explicit lifetime* `<'a>` means the `Program` instance won't outlive the referenced `instructions` object. 

We use the `Program` struct to wrap the encoded instructions read by the virtual machine. 

Now define an implementation of `Program` that defines a *fetch* method that returns the instruction indexed by the instruction pointer:

{% highlight rust %}
impl<'a> Program<'a> {
    fn fetch(&self, ip: usize) -> u16 {
        self.instructions[ip]
    }
}
{% endhighlight %}

We use the fetch method like so:

{% highlight rust %}
let encoded_instructions = Program { instructions: &[0x1110, 0x2100, 0x3010, 0x0] };

let encode_instruction = program.fetch(0);
{% endhighlight %}

### Decode

Since all the instructions are encoded as 16 bit hexidecimal words, we'll need a way of decoding them. Given the instruction encoding table above, we can do this by using bit shifts and masks. 

For example, consider the encoded instruction `0x13D2`. Writing this out in binary gives us the following representation:

```rust
0001 0011 1101 0010
```

We easily get the opcode **Load** by bit shifting it 12 places to the right:

```rust
        0001 0011 1101 0010 = 0x13D2 
>> 12   0000 0000 0000 0001 = 0x1 = 1
```

Since the operation is **Load**, we know from the encoding table that the next 4 bits represent the register and the last 8 bits must be the value. To get the register we need to shift `0x13D2` 8 bits to the right and then apply the bit mask `0xF` to kill off the unwanted bits:

```rust
        0001 0011 1101 0010 = 0x13D2
>> 8    0000 0000 0001 0011 = 0x13
    
        0000 0000 0001 0011 = 0x13
        0000 0000 0000 1111 = 0xF
AND     0000 0000 0000 0011 = 0x3 = 3
```

We get the value of 210 by performing the bitwise **AND** operation with `0xFF` and converting from hexidecimal to decimal:

```rust
        0001 0011 1101 0010 = 0x13D2 
        0000 0000 1111 1111 = 0xFF
AND     0000 0000 1101 0010 = 0xD2 = 210
```

In our virtual machine, we'll implement the decode step by defining an implementation of `Instruction` that defines a `decode` method that returns the appropriate `Instruction` variant. If we are not able to decode the instruction, then we'll use the generic `Return` type to return the variant `Err` with an informative message. Otherwise, it just returns success `Ok` and its value.

```rust
impl Instruction {
    fn decode(encoded_instruction: u16) -> Result<Self, &'static str> {
        let operator = encoded_instruction >> 12;
        let reg1 = ((encoded_instruction >> 8) & 0xF) as usize;
        let reg2 = ((encoded_instruction >> 4) & 0xF) as usize;
        let reg3 = (encoded_instruction & 0xF) as usize;
        let offset = (encoded_instruction & 0xFFF) as usize;
        let value = encoded_instruction & 0xFF;

        match operator {
            0 => Ok(Instruction::Halt),
            1 => Ok(Instruction::Load { destination: reg1, value: value }),
            2 => Ok(Instruction::Swap { destination: reg1, source: reg2, temp: reg3 }),
            3 => Ok(Instruction::Add { destination: reg1, source: reg2 }),
            4 => Ok(Instruction::Branch { offset: offset }),
            _ => Err("Failed to decode the instruction")
        }
    }
```

### Execute

After decoding an instruction, we'll need to execute it. This step involves performing an arithmetic operation on one or more of the registers. For example, let's consider the **Swap** operation:

In our virtual machine, we have a register file containing three registers that we can read and write to. So suppose we want to swap the data in register 1 with the data in register 2:

{% plantuml %}
node "Data: 200" <<Register 1>> as a
node "Data: 300" <<Register 2>> as b
{% endplantuml %}

First, we need to copy the data of register 1 into a temporary register (which in our case is register 3):

{% plantuml %}
node "Data: 200" <<Register 1>> as a
node "Data: 300" <<Register 2>> as b
node "Data: 200" <<Register 3>> as c

a -[hidden]right-> b
a -down-> c
{% endplantuml %}

Then, we need to copy the data of register 2 and use it to overwrite the data in register 1:

{% plantuml %}
node "Data: 300" <<Register 1>> as a
node "Data: 300" <<Register 2>> as b

b -left-> a
{% endplantuml %}

Finaly, we just copy the contents of register 3 into register 2:

{% plantuml %}
node "Data: 300" <<Register 1>> as a
node "Data: 200" <<Register 2>> as b
node "Data: 200" <<Register 3>> as c

a -[hidden]right-> b
a -[hidden]down-> c
c --> b
{% endplantuml %}

<br />
Given the above, we can readily implement the **Swap** operation in Rust as follows:

```rust
fn swap(source: usize, destination: usize, temp: usize, registers: &mut [u16]) {
    registers[temp] = registers[source];
    registers[source] = registers[destination];
    registers[destination] = registers[temp];
}
```

Finally, we'll extend the implementation of the `Instruction` enum by defining an `execute` method that uses patten matching on the `Instruction` variant to determine how to execute itself:  

```rust
fn execute(&self, registers: &mut [u16], ip: &mut usize) -> bool {
    match *self {
        Instruction::Load { destination, value } => {
            registers[destination] = value;
        },
        Instruction::Swap { destination, source, temp } => {
            swap(destination, source, temp, registers);
        },
        Instruction::Add { destination, source } => {
            registers[destination] = registers[destination] + registers[source];
        },
        Instruction::Branch { offset } => {
            *ip -= offset - 1;
        },
        Instruction::Halt => {
            println!("{:?}", registers[0]);
            return false;
        },
    }

    true
}
```

### The Greatest Common Divisor

The greatest common divisor (gcd) of a pair of integers *a* and *b*, where at least one of *a* or *b* is non zero, is the largest integer *c* that divides both *a* and *b* without remainder. For example, the gcd of 9 and 12 is 3 and the gcd of 20 and 52 is 4. 

In general, the gcd is found by applying the following recursive algorithm:

$$\mbox{gcd}(a, b) = \begin{cases} a &\mbox{if } a \equiv 0 \\ 
\mbox{gcd}(b, a \bmod b) & \mbox{if } a \not\equiv 0 \end{cases} \pmod{b}.$$

Although it didn't appear in this form, this algorithm is over 2000 years old and it was featured in Euclid's Elements. It is a good algorithm for our virtual machine since it can be implemented easily and it uses all the instructions Load, Mod, Swap, BEQ, and Halt. The instructions it needs to execute to compute the gcd of the pair 15 and 12 is given by the following diagram:

{% plantuml %}
start
: Load 0 12;
note right
  Load the number 12 into register 0
  ====
  0x110C
end note
: Load 1 15;
note right
  Load the number 15 into register 1
  ====
  0x110F
end note
repeat
  :Mod 0 1 2;
note right
  Compute the modulus of register 0
  and register 1 and store the result 
  into register 2
  ====
  0x2012
end note
  :Swap 1 0;
note right
  Swap the data in register 1 with the 
  data in register 0
  ====
  0x3010
end note
  :Swap 2 1;
note right
  Swap the data in register 2 with the 
  data in register 1
  ====
  0x3021
end note
repeat while(BEZ 1 3)
: Halt;
note right
  Terminate the program and print out  
  the value of register 0
  ====
  0x0000
end note

stop
{% endplantuml %}

[x86-regs]: http://www.eecg.toronto.edu/~amza/www.mindsec.com/files/x86regs.html 