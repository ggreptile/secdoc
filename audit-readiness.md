# Audit Readiness

## Purpose

Overall goal:
Get the most value of the audit

Tactic:
* Create a frozen commit ID to be audited. It should be stable and contain all relevant tests, etc.
    - All automated tests should complete successfully
* Minimize time needed for auditor(s) to learn how the codebase works
* Identify most important areas of the codebase to triage auditing resources
    - areas of complexity 
    - state-changing logic
* Articulate system invariants (properties of the system that must always hold)
    - e.g. "a transfer of coins should not result in the creation of new coins. sum(inputs) == sum(outputs)"

## Tasks

### Audit kick-off
* Explain clearly our areas of concern
* Provide all relevant links to documentation, like whitepaper, invariants, how to get started, how to run tests, etc.
* Communicate security assumptions
    - e.g. "If we use .unwrap(), it means the system should crash."
    - e.g. "This function accepts arbitrary user input but we have validation in X.rs to handle insecure inputs"
    - e.g. "This structure may reach unsafe states but performance is more important to us in this case"

### Project Documentation

* Add as much documentation as possible to reduce confusion and reverse-engineering on the part of auditors, or 
extra time in meetings and communication overhead generally.
* Take advantage of Rust docs where possible
* Ensure darkfi book is up-to-date and accurate
* Glossary of terms in the codebase that are new or have a special meaning

### Automated Testing
* Add robust unit testing
* Make use of fuzz testing, especially for easy fuzzing targets (e.g. functions that accept bytes and do not require extensive mocking of data structures)

### Dev environment bootstrapping
* Provide auditors with an automated tool to create a working development environment
* Explain how to use interfaces like JSONRPC to investigate the network state
* Add tests to demonstrate functionality of the network:
    - zkas compilation
    - atomic swap
    - ...

## Appendix: Common issues in Rust codebases, with solutions

### Panic (especially in critical areas)
#### Problem
The code panics. If this happens in areas related to consensus, the network can fail catatrophically. If an attacker
can reliably trigger a panic in this area, they can use this to launch denial-of-service attacks against the network,
validators, etc.

#### Solution
Remove panic macros and unwrap() from the codebase unless the system should truly crash if it reaches a certain state
Handle errors properly

### Overflow/underflow and truncation

#### Problem
When converting to types with different sizes, mathematical and logical issues can occur. e.g.:

- converting from a u64 to a u32 can result in the higher bits being lost (truncated). 
- operations can overflow. if the sum of two values exceeds the maximum value that the type can hold, the result will become a very small number
- operations can underflow. if the difference of two values is smaller than the minimum value that the type can hold, the result will become a very large number
- converting from signed to unsigned types (or vice-versa) can change a {negative|postive} number into a {positive/negative} one

#### Solution
Check mathematical operations to ensure they fall within the expected bounds of their types. Add error handling to check for this.
Avoid converting between types where possible. Be consistent.
Use "checked" math functions (e.g. [checked_add in u32](https://doc.rust-lang.org/std/primitive.u32.html#method.checked_add))

### Arithmetic issues due to rounding

### Problem
Depending on the types and operations done in the code, rounding may be necessary when performing
tasks such as calculating funds.
If done inconsistently, this can cause loss of funds.

### Solution
Round up or down consistently across the codebase to avoid disagreement.

### Solution

### Non-determinism due to floating point arithmetic

#### Problem
Floating-point mathematics are non-deterministic and as such can cause consensus issues in blockchains. 

#### Solution
Work using big number code libraries instead, similar to Ethereum. 

### Division before multiplication can cause precision loss
#### Problem
Integer division and multiplication is not commutative.

e.g. If we multiply before dividing, 12345 * 100 = 1234500. Then 1234500 / 100 gives 12345.
however, if we divide before multipling: 12345 / 100 == 123.
Then if we multiply by 100, we get 12300.

This causes problem when e.g. calculating token/transfer amounts because tokens can appear to be created/destroyed.

#### Solution
Always multiply before dividing. When this is not done, the reason should be documented in the codebase to explain why.

### Hash collisions when hashing structured data

#### Problem
Often as blockchain code implements security properites like authentication, structured data will be hashed to create a sort of identifier.
Hash collisions can occur when adjacent inputs are hashed together.

```
let a = foo{
    prop1: b"1",
    prop2: b"23",
}
let b = foo{
    prop1: b"12",
    prop2: b"3",
}

let hash_a = Hash::hash(a);
let hash_b = Hash::hash(b);
```

Both of the above structs will generate the same hash because their inputs are both equal to b"123".

#### Solution
Use a character to separate the hashed inputs: "1|23", "12|3".
Embed further information in the hash, such as the name or type of the variable to act as a separator.

### Out of memory issues

#### Problem
If arbitrary user data is to be stored in a Vector or similar structure, it is possible for the user to submit
an extremely long input. This can slow down a validator or even cause an out-of-memory panic if the code tries to
allocate memory exceeds its capabilities.

#### Solution
Create an upper-bound limiting how much data can be stored in a struct. Reject user inputs that are too large.

### Index out of bounds panics

#### Problem
Developers may attempt to access indices within a Vector (or similar structure) that exceed its length. When this occurs, the code will panic.

#### Solution
Always check the index against the length of a data structure before trying to access its value.
Be cautious when using user-supplied values as an index/offset.

### Use of unsafe code

#### Problem

`unsafe{}` blocks in Rust can cause memory issues that undermine the guarantees of safe Rust.

#### Solution
Avoid using `unsafe{}`.
Review dependencies for their use of `unsafe{}` and understand the inherited risk.

### Non-determinism in data structure ordering and (de)serialization

#### Problem
Data strucures like HashMap are not ordered. If code used in consensus-critcal sections of the codebase assume
that HashMaps are ordered and perform operations on them, it is possible for validators to reach different states
and cause a consensus issue.

(de)serialization of data structures and formats like JSON is also non-deterministic.

#### Solution
Order unordered data structures before iterating over them.
Favour all-or-nothing patterns for state-changes; i.e. do not terminate early in a loop or change the state of only some elements and not others.

### Non-determinism in OS clocks

#### Problem
The use of system time and clocks in consensus code is very likely to cause consensus issues due to drift and sync issues between validators.

#### Solution
Do not use time in consensus-related code

### Bad randomness

#### Problem
If random number generators are used, they must be cryptographically secure or else they can be predicted/brute-forced.

#### Solution
Use only secure random number generators. OsRng in Rust is secure.

### Lack of testing

#### Problem
Untested areas of the codebase are blind spots and can contain bugs unknown to the team.

#### Solution
Make use of automated testing, including unit testing, fuzzing, and formal verification depending on the maturity and resources of the project.
Test error conditions, not just the "happy path".

### Use of insecure or abandoned dependencies

#### Problem
Dependencies of the codebase may contain security issues. They may no longer be actively developed and become insecure over time.

#### Solution
Reduce the use of dependencies.
Audit dependencies manually and/or make use of static analysis tools and community resources to determine the safety of dependencies.


<!--
### Issue

#### Problem

#### Solution
-->
