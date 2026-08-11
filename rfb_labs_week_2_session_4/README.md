# Rust for Bitcoin 2.0 — Week 2, Session 4

Build a small lending library while practising structs, enums, traits,
ownership, borrowing, collections, and `Result`-based error handling. No
Bitcoin and no external crates — just Rust.

The crate is intentionally incomplete. Search for `TODO` and implement each
part; do not change the public type names or function signatures.

## Recommended workflow

1. Read [ASSIGNMENT.md](ASSIGNMENT.md).
2. Complete Part 2 in `error.rs`, then Part 3 in `library.rs`.
3. Remove `#[ignore]` from the relevant test and run it.
4. Complete the traits in Part 4 and the two operations in Parts 5–6.
5. Run the ownership experiments and record the errors.
6. Build the demo in `main.rs`.
7. Add the remaining required tests yourself.

```bash
cargo test
cargo test -- --ignored
cargo run
cargo fmt --check
cargo clippy --all-targets --all-features -- -D warnings
```

`cargo test` checks the starter project. Ignored tests intentionally exercise
unfinished code; enable them progressively rather than leaving them ignored in
the submission.

## Written answers

Answer in your own words. Add both ownership compiler errors from Part 7 as
fenced text blocks, then explain what caused each.

### 1. Why is `LoanStatus` an enum rather than a `bool` plus two `Option` fields?
Because `LoanStatus` has mutual exclusive possible status, if they were a `bool` plus 2 `Option`, a inconsistent state. For example `Available` and `Lost` at the same time.

### 2. What does `match` force you to do when a fourth `MediaKind` is added later?
The `match` forces the code for the new `MediaKind` to be handled. In this way, all possible options for the enum are garanteed to be treated.

### 3. `Item::new` takes `String` rather than `&str`. Who owns the title afterwards?
Passing a `String` to `Item::new` moves ownership of `title` into the new `Item`. In this case the `title String` is passed by value. When `new` is called, the passed String is transferred into the function. Inside `Item::new`, `title` is assigned to `Self { title, ... }`, so the `Item` now owns that `String`. After the call, the original title variable can’t be used, because it has been consumed. So afterwards, `item.title` is owned by the `Item` instance.

### 4. Why does `add_item` take `self` by `&mut` but `item` by value?
`add_item` takes `&mut self` because the method needs to mutate the Library. It should push the new `Item` into `self.items`, which requires mutable access. `Item` needs to be pushed into a `Vec<Item>`, so the `Libray` is supposed to take ownership of the item passed in. If item were `&Item`, it would only be borrowed, but `Library` needs to own it long-term. Passing by value allows `self.items.push(item)`.

### 5. When `add_item` returns `Err`, what happened to the `Item` the caller passed in? Was that a good design choice, and what is the alternative?
The `item` argument is moved into the function scope. If an error occurs, the original `Item` passed by the caller is not returned or somehow restored. It's consumed in the process of attempting to add it to the library. The item passed into the function is entirely lost.

I believe it's a good design because the Vector needs the ownership of its values. If the caller needs an `Item`, he must use the `Libray` interface. 

If a reference was passed to the `add_item`. It would be necessary to clone it before inserting into the Vector. This could be desirable if the whole `Item` was still necessarry after a `Err`. But the enum `LibraryError` already indicates the reason of the error, so I believe the original design decision was correct.


### 6. Why does `find_item` return `Option<&Item>` rather than `Option<Item>`?
`find_item` returns Option<&Item> to make possible to look at an item that’s stored inside the Library without taking ownership of it. The library must keep owning items so it can update their status and keep them consistent.

### 7. What is the lifetime `'a` in `items_by_author` actually saying?
That the references of `Item`s returned by the function `items_by_author` will live for at least as long the object that owns the method is alive. That  happends because `&self` and `&Item` have the same lifetime. 

It's beind said to the compiler that a `Vec<&Item>` is being returned, but not any `Vector`, it's a Vector of `Item`s that will live at least for the same time that the object that `items_by_author` is called.

This ensures memory safety while allowing efficient borrowing without unnecessary copying of entire data structures.

### 8. Why can't `checkout` hold a `&mut Item` and a `&mut Member` from the same `Library` at once, and how did you structure the method around that?
Because the ownership system does not permit to hold two mutable references to different parts of the same data structure simultaneously. In this case `struct Library` has references to both members and items. This rule avoids mutable borrows to overlapping memory locations.

### 9. Why are `Library`'s fields private?
Because the library is responsible for keeping item's `LoanStatus` and a member's borrowed-id list in agreement, if the caller could access the values directly, these could lead to inconsistent states inside the Libray

### 10. What duplication does the provided `late_fee_cents` remove, and what would you lose by making it a free function instead?
The `late_fee_cents(&self, days_held)` default method in the `LoanTerms` trait removes duplication by centralizing the shared late-fee formula: implementers only need to provide `loan_days()` and `daily_late_fee_cents()`, while the trait supplies the common logic.

If it was a free function instead, the reusable default behavior tied to the `LoanTerms` was going to be lost. It should pass the needed values explicitly,  rather than via `late_fee_cents(days_held)` for any `T: LoanTerms`, which would be likely to duplicate logic. In the way with the trait, the code works with different types implementing `LoanTerms`.

### 11. Why is `Result` preferable to `panic!` for validation failures? Name a place in this crate where a panic would be defensible.
`Result` is preferable to `panic!` for validation failures because validation errors are expected part of normal program flow (bad input, duplicates, limits reached, etc.). With `Result`, the caller can handle the error gracefully and the program doesn’t abort unexpectedly.

`panic!`, on the other hand, is appropriate for situations that represent an internal logic bug or something that should never happen. For example: finding the library in an impossible state where an item’s `LoanStatus::OnLoan { member_id }` does not match the borrower’s borrowed_item_ids. In this case the internal consistency was violated. It could be defensible to `panic!` in this scenario. 

### 12. Which derive did you deliberately leave off a type, and why?
`Item` and `Library` structs don't have a `Clone` type because is not desirable to have multiple libraries available, that could result in inconsistent states. And `Item` does not have the `Clone` type also, because `Library` fields owns items, so if we clone these items, that could lead to inconsistent behavior.


## Design notes

Describe any choices you made, including how you kept an item's status and its
borrower's list from drifting apart, and (if attempted) the optional generic
search.

## Example output

Paste the output of `cargo run` here once Part 8 is complete.
