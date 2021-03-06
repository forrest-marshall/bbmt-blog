# The Birds, the Bees, and the Merkle Trees ep[1]: Hashing Things Out

*This is episode 1 of 'The Birds, the Bees, and the Merkle Trees',
a developer-focused educational series on blockchain technology.
The purpose of this series is to give a bottom-up view of how blockchains
work, by experimenting with and building core blockchain technologies.
This series is for educational purposes only and should not be construed
as advice or recommendations concerning best-practices for security,
cryptography, etc... This series is developed openly on github,
at `forrest-marshall/bbmt-blog`.*

---

## Introduction

In [episode zero](./episode-0.md) I introduced the goal for this series:
to explore blockchain technology from the ground up by building toy versions of
the foundational systems and structures that make blockchains possible.  I
gave a brief introduction to the properties that make blockchains interesting and
useful, as well as my thoughts on why a bottom-up view of blockchain technology
is important.

In this episode, we will begin meeting the family of cryptographic tools that we will
use throughout this series.  Don't worry if you aren't familiar with cryptography
at all; we'll start from the beginning.

> [Cryptography] is the practice and study of techniques for secure communication
> in the presence of third parties... --Wikipedia

Cryptography has its roots in secrets; early cryptography dealt primarily with
how to keep secrets such that only the intended parties could read them.  Cryptography
was typically the domain of spies, military leaders, and the like.  In more recent
times, cryptography has branched out significantly to encompass just about any
concern related to passing information in an adversarial scenario.  Most people use
a significant amount of cryptography every day, even if they are not aware of it.
Cryptography guards the integrity of software updates, assures us that the websites
that we visit are who they claim to be, and is integral to the functioning of
all digital financial systems, blockchain or otherwise.

Blockchain technology isn't actually very ideal for keeping secrets.  There are ways
to do it, but the core value-proposition of a blockchain is its ability to ensure
the integrity of a system's information.  There is nothing which prevents blockchains
from housing secrets, but it isn't their strength, and, in some cases, storing secrets
on a blockchain can actually be a *very* bad idea (we will discuss this further at
a later time).  Since integrity is the core goal, we will focus on
the cryptographic tools that allow us to ensure integrity.  Two key challenges arise
when we want to secure a system against invalid state or inputs: ensuring the
accuracy of information within the system, and validating the identity of actors within
the system.  We will focus on the first problem for this episode, and tackle the second
problem in the next episode.


## Setup

If you are planning to follow along with examples, make sure that you have installed
[Rust](https://www.rust-lang.org/en-US/), as well as its associated package-manager
`cargo` (covered in [episode zero](./episode-0.md)).  If you are not following along,
you can safely skip this section.

The `cargo` package-manager includes a set of helpful commands for instantiating,
building, and running Rust projects.  Create a new binary (executable) project
with `cargo new ep1-examples --bin`.  This command will create the following
file-structure:

```
ep1-examples/
├── Cargo.toml
└── src/
    └── main.rs
```

The `Cargo.toml` file specifies dependencies, as well as the name and authorship
of the project.  The `main.rs` file is the core of our program, and is instantiated
with the following code by default:

```rust
fn main() {
    println!("Hello, world!");
}
```

If you `cd` into the main project directory and execute `cargo run`, the project
should compile and print out `Hello, world!`.

For this demonstration we will need to use one dependency, the `eth-crypto` crate
(Rust refers to its libraries as "crates", in keeping with Rust's industrially
inspired naming system).  Tell cargo where to acquire the dependency by adding the
following line to the `Cargo.toml` file under the `[dependencies]` section:

```toml
eth-crypto = { git = "https://github.com/forrest-marshall/eth-crypto.git" }
```

Once the dependency has been listed, we can bring it into scope within our code
by adding these lines to the top of the `main.rs` file:

```rust
extern crate eth_crypto;
use eth_crypto::prelude::*;
```

The `extern crate` line tells the compiler to look for a crate named `eth_crypto`.
Once a crate has been brought into scope, we can import items from the crate
with `use` statements.  In our case, we are importing the entire contents of
of the `prelude` module, which contains most of the commonly used types and
functions from this library.  The use of a `prelude` module as a shortcut for
importing a library's most commonly used elements is a very common pattern in
Rust.


## Cryptographic Hashing Functions

The general definition of a hashing function is a function which produces a
fixed-size output for an arbitrary sized input.  A simple example of this
property would be a function which divides any whole number by 2 and returns
the remainder of the division.  The output of this function would always
be either 0 or 1, regardless of the size of the input number.  The output
of this function is also *deterministic*, meaning that it will always produce
the same output for a given input.

Cryptographic hashing functions are a special subset of hashing functions
which are suitable for use in cryptography.  The goal of a cryptographic
hashing function is to be a one-way function.  This means that it should,
in theory, be impossible to reverse a hashing function and thereby determine what
input values produced a given output.  Furthermore, it should be infeasible
to find two similar inputs which produce the same output (an event referred
to as a "collision").  If built correctly, a cryptographic hashing function
can be thought of as a finger-printer for data.  If a piece of data produces
a given output, that output serves as a unique identifier for the data.  If
the data is tampered with in any way, it will no longer match the original
hash, and can easily be shown to be invalid.

The `eth-crypto` crate uses the `keccak-256` hashing function, which is the
default hashing function used by the
[Ethereum Virtual Machine](https://en.wikipedia.org/wiki/Ethereum#Ethereum_Virtual_Machine).
The `keccak-256` hashing function produces a 256 bit output value, and
is generally considered, as of the time of writing, a secure and reliable
hasher.  Lets take a look at the difference between two similar inputs:

```rust
let mut a = [1,2,3,4,5];
let hash1 = hash(&a);
a[0] = 0;
let hash2 = hash(&a);
println!("hash-1: {:x}",&hash1);
println!("hash-2: {:x}",&hash2);
```

Adding the above code to your `main` function and running the new code
(`cargo run`) should produce this output:

```
hash-1: 7d87c5ea75f7378bb701e404c50639161af3eff66293e9f375b5f17eb50476f4
hash-2: c23ee272307dfe7f763e5c0e5534dc158fa262bdd0e5801ea863024db84fe507
```

So, what did we just do here?  First, we declared a mutable array of bytes.
For security and optimization reasons, Rust always forces us to be explicit
about wanting to be able to mutate (modify) any variables that we declare.
Next, we pass an immutable reference to the array into the `hash` function, saving the
output.  Immutable references, indicated by the `&` symbol, allow a function to
temporarily read a piece of data without allowing the function
to modify that data, another key element in Rust's security and safety guarantees.
The third line is where things get interesting.  By changing the first byte of the
array from `1` to `0`, we have actually only modified a single binary bit.
A byte of value `1` is represented in binary as `00000001`.  A byte of
value `0` is, unsurprisingly, represented as `00000000`.  After flipping
this bit, we hash again.  When we print out the two hashes,
we see that the two values are wildly different even though their
respective inputs only differed by a single bit.  This is the real
magic of a cryptographic hashing algorithm; the apparent difference
or similarity of two inputs has no correlation with the difference
or similarity of their respective outputs.


## Hashing in the Wild

The above example demonstrated an important property, but it wasn't particularly
interesting or useful.  Lets take a look at how we might use hashing to build
a rudimentary cryptographic trust protocol.  Suppose Alice has a secret clubhouse.
Anyone who knows the password may enter the clubhouse.  Bob believes he knows the
password, and wants to enter the clubhouse.  Unfortunately, Eve is outside the clubhouse
listening, so Bob cannot simply speak the password.  Alice could just ask Bob
for the hash of the password, but if she did that then Eve could use the
hash to get via the same system.  How can Alice check if Bob knows the password without
making the clubhouse vulnerable to Eve?

Hint:

```rust
fn verify(password: &str, challenge: &str, response: &Hash) -> bool {
    let expected = hash_many(&[password,challenge]);
    &expected == response
}
```

So what exactly does the above function do?  It takes two strings, `password` and
`challenge`, and a hash named `response`.  The function returns a `bool` (true/false)
value, indicated by the `->` operator.  Internally, we are hashing the `password`
and `challenge` strings together by passing them to `hash_many` inside of a reference
to an array.  We then check whether or not the hash we produced is equal to the
`response` hash.  Note that the function is returning the result of the `==` comparison
operator automatically.  If the last line of a function does not end with a semi-colon,
Rust infers that the value of that expression is the value to return.

The solution is to have Alice provide a randomly generated challenge phrase.
Bob can hash the password *and* the challenge phrase together.  This will
allow Alice to easily check if Bob knows the password, but won't provide
Eve with any information that she could use to enter the clubhouse by the
same process.  If we model the entire process in code, it looks like this:

```rust
// bob knows the password.
let password = "decentralize all the things";
// alice gives bob a challenge phrase.
let challenge = " where appropriate";
// bob hashes the password and challenge together and
// gives alice the result as his response.
let response = hash_many(&[password,challenge]);
// eve observes the response, but cannot use it
// to reconstruct the password.
// if the response is valid, alice lets bob into
// the clubhouse.  otherwise she turns him away.
if verify(password,challenge,&response) {
    println!("welcome!");
} else {
    println!("access denied!");
}
```

The above process is not wholly dissimilar from how user passwords are
stored in many systems.  Storing a password is very dangerous, as any
party which gained access to the system could read all the passwords.
Instead of storing the password itself, a random value known as a 'salt'
is stored along with a hash of the salt and password combined.  This allows
a password to be verified without needing to save the password itself to disk.
The randomness of the salt prevents attackers from using a pre-computed
database of the hashes of common passwords to attempt to determine the password
that is in use.

## Conclusion

Cryptographic tools give us the ability to secure information and
information-based systems against malicious actors.  Different
cryptographic tools are useful for different kinds of securement.
Cryptographic hashing functions offer us the ability to uniquely
identify information in a manner which is incredibly difficult to
forge or reverse.  With a little creativity, hashing can be used to
secure systems from many different threats, including tampering,
forgery, and surveillance.  In a future episode we will learn
about how hashing may even be used to measure investment of
computational resources.

## Up Next

In the next episode we will be discussing identity-based cryptography.
We will experiment with Elliptic Curve Cryptography to generate secure
digital signatures, and look at how digital signature algorithms allow
us to make secure public identities and accounts.
