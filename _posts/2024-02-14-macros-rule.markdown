---
layout: post
title:  "macros_rule!"
categories: rust
---
This is a followup to my RustConf 2023 talk on `macro_rules!` and the fun patterns that you can take
advantage of when writing them. Apart from a quick refresher at the start, this post is going to
assume you've [watched](https://youtu.be/7uSM60jlkBE?si=4q3LacMmje8_SfMU) or
[read](https://github.com/AuroransSolis/rustconf-2023) the presentation - your choice which, they
should cover roughly the same things. So, let's get into it!

# A brief refresher #

The most important patterns we have to work with in declarative macros are:

1. Recursion
    - Macros calling themselves
2. Internal rules
    - General category for rules not intended to be called by users
3. Incremental TT munchers
    - Form of recursive macro call that takes some tokens off the front of the input and recurses
      with the rest
4. Push-down accumulation
    - Circumventing the requirement that macros expand to complete syntax elements (because macro
      calls are complete syntax elements!)
5. TT bundling
    - Shoving a whole bunch of tokens into one `[]`-delimited list and matching as a single `:tt`
6. Callbacks
    - Passing an `:ident` or `:path` to a macro to be called as a macro at some point

These cover just about everything you'll ever want to do with macros, but there's one other useful
pattern that I'd like to talk about:

# Macros as Lookup Tables

Lookup tables (LUTs, as I'll call them from here on out because I'm lazy) are an incredibly useful
tool for macros since we don't have many ways to do meaningful transformations of inputs at the AST
level. For instance, if we want to do addition, by taking tokens `$a` and `$b` and joining them with
a `+` we no longer have two (presumably) `:literal`s - we now have an `:expr`. That's not very
helpful! So to get around this, we can limit our input and define these transformations by hand.
Sure, it's not the most ergonomic thing to do, but it does allow us to do a surprising amount of
things.

First, let's look at something pretty simple - rock paper scissors.

```rs
macro_rules! rock_paper_scissors {
    // Input format: (you, opponent)
    (rock, rock) => { println!("tie"); };
    (rock, paper) => { println!("lose"); };
    (rock, scissors) => { println!("win"); };
    (paper, rock) => {println!("win") };
    (paper, paper) => { println!("tie"); };
    (paper, scissors) => { println!("lose"); };
    (scissors, rock) => { println!("lose"); };
    (scissors, paper) => { println!("win"); };
    (scissors, scissors) => { println!("tie") };
}
```

Note here how we're matching on literal tokens - `rock`, `paper`, and `scissors`. We aren't checking
if the two tokens are the same (~~because we can't~~although we could[^1]), instead we're spelling
out every state that the input _could_ be in. Let's take this idea and combine it with some other
macro patterns and do something a little more interesting, say, adding up every other literal in a
list of inputs.

```rs
macro_rules! add_every_other {
    // Easy case - the sum of nothing is nothing
    () => { 0 };
    // Non-empty input
    ($($val:literal),+$(,)?) => {
        add_every_other!(@add [add] [$($val),+])
    };
    (@add [add] [$first:literal $(, $rest:literal)*]) => {
        $first + add_every_other!(@add [skip] [$($rest),*])
    };
    (@add [skip] [$_:literal $(, $rest:literal)*]) => {
        add_every_other!(@add [add] [$($rest),*])
    };
    (@add $_:tt []) => { 0 };
}
```

Let's examine how this expands on some input.

```
add_every_other!(1, 2, 3, 4, 5, 6);
add_every_other!(@add [add] [1, 2, 3, 4, 5, 6]);
1 + add_every_other!(@add [skip] [2, 3, 4, 5, 6]);
1 + add_every_other!(@add [add] [3, 4, 5, 6]);
1 + 3 + add_every_other!(@add [skip] [4, 5, 6]);
1 + 3 + add_every_other!(@add [add] [5, 6]);
1 + 3 + 5 + add_every_other!(@add [skip] [6]);
1 + 3 + 5 + add_every_other!(@add [add] []);
1 + 3 + 5 + 0
```

In this I'm considering the LUT to be the `[add]` vs. `[skip]` match changing behaviour, though this
can be much more literal. Let's do something silly and, say, implement the [Luhn algorithm][luhn].
This one's gonna be a bit long, so [here's a link](#macros-are-functional) to the next section if
you'd like to skip over it.

First, let's define the top level macro. Should be pretty easy, we just want to know whether the
input length has an even or odd parity. We'll do this by seeing whether a multiple of two
metavariables needs a leading metavariable in order to match.
```rs
macro_rules! luhn {
    // Matches:
    //     - a
    //     - a b c
    //     - ...
    ($head:tt $($tail1:tt $tail2:tt)*) => {
        calculate_residue!([$head $($tail1 $tail2)*] [odd] [even] [0])
    };
    // Matches:
    //     - 
    //     - a b
    //     - ...
    ($($tail1:tt $tail2:tt)*) => {
        calculate_residue!([$($tail1 $tail2)*] [even] [even] [0])
    };
}
```
Next, the conditional part of the Luhn algorithm. If the parity of the current index doesn't equal
the parity of the length then we add the current digit to the accumulator, otherwise we add twice
the current digit (and subtract 9 if that's greater than 9). The LUT behaviour here is checking for
`[even] [odd]`, `[odd] [even]`, `[even] [even]`, and `[odd] [odd]`.
```rs
macro_rules! calculate_residue {
    // i % 2 != parity
    ([$head:tt $($tail:tt)+] [even] [odd] [$acc:tt]) => {
        add_mod10!([$($tail)*] [even] [even] [$acc, $head])
    };
    ([$head:tt $($tail:tt)+] [odd] [even] [$acc:tt]) => {
        add_mod10!([$($tail)*] [odd] [odd] [$acc, $head])
    };
    // i % 2 == parity
    ([$head:tt $($tail:tt)+] [even] [even] [$acc:tt]) => {
        mod10_aP2b!([$($tail)*] [even] [odd] [$acc, $head])
    };
    ([$head:tt $($tail:tt)+] [odd] [odd] [$acc:tt]) => {
        mod10_aP2b!([$($tail)*] [odd] [even] [$acc, $head])
    };
    // base cases
    ([$head:tt] $lparity:tt $iparity:tt [$acc:tt]) => {
        final_check!($head, $acc)
    };
}
```
Modular arithmetic does us a favour here - `mod10(a) + mod10(b) = mod10(a + b)`, so on each step we
can take the accumulator mod 10 and still end up with the right answer. This allows us to limit the
possible inputs to the `add_mod10!` and `mod10_aP2b!` macros, which will be defined as LUTs. So
let's get to doing that, shall we? For this we need to remember that macros can only "pass input" to
other macros by recursion, so in order for `add_mod10!` and `mod10_aP2b!` to "return" a value to
`calculate_residue!` they both must use a callback. That's why in the above definition of
`calculate_residue!` the calls to `add_mod10!` and `mod10_aP2b!` not only get our accumulator and
the digit at the head of the list, but the other arguments to `calculate_residue!` as well. This is
so that the callback to `calculate_residue!` has the necessary information to continue.

We can also note that both of the macros we want to define are going to have a very similar
structure: each rule is going to have two non-passthrough inputs which will define a single output
value. Additionally, each output will have several inputs that match it since everything is done
modulus 10. Since the structures are very similar and we would have to define 100 rules apiece
otherwise, let's write a macro to define `add_mod10!` and `mod10_aP2b!` for us.
```rs
macro_rules! def_lut_2in_1out {
    (
        // This is weird, yes, but see the next comment for an explanation.
        fix: [$d:tt],
        name: [$name:ident],
        map: [
            $($output:tt <= [$([$i0:tt, $i1:tt]),+$(,)?]),+$(,)?
        ],
    ) => {
        macro_rules! $name {
            $(
                $(
                    ($a:tt $b:tt $c:tt [$i0, $i1]) => {
                        calculate_residue! { $a $b $c [$output] }
                    };
                )+
            )+
            // If we were to try and just write `$($other:tt)*` here, Rust would
            // think we were trying to do another repetition using `$other`.
            // This is okay, except we aren't using any matched variables, which
            // isn't okay and will produce an error. To get around this, `$` can
            // be passed in as a `:tt` and used in the expansion.
            ($d($other:tt)*) => {
                compile_error!(concat!(
                    "input should be a literal token 0-9. found: `",
                    $d(stringify!($other)),*,
                    "`",
                ));
            }
        }
    };
}
```
And now we can define our LUTs:
```rs
// define a LUT for:
// (a + b) % 10
def_lut_2in_1out! {
    fix: [$],
    name: [add_mod10],
    map: [
        0 <= [[0, 0], [1, 9], [2, 8], [3, 7], [4, 6], [5, 5], [6, 4], [7, 3], [8, 2], [9, 1]],
        1 <= [[0, 1], [1, 0], [2, 9], [3, 8], [4, 7], [5, 6], [6, 5], [7, 4], [8, 3], [9, 2]],
        2 <= [[0, 2], [1, 1], [2, 0], [3, 9], [4, 8], [5, 7], [6, 6], [7, 5], [8, 4], [9, 3]],
        3 <= [[0, 3], [1, 2], [2, 1], [3, 0], [4, 9], [5, 8], [6, 7], [7, 6], [8, 5], [9, 4]],
        4 <= [[0, 4], [1, 3], [2, 2], [3, 1], [4, 0], [5, 9], [6, 8], [7, 7], [8, 6], [9, 5]],
        5 <= [[0, 5], [1, 4], [2, 3], [3, 2], [4, 1], [5, 0], [6, 9], [7, 8], [8, 7], [9, 6]],
        6 <= [[0, 6], [1, 5], [2, 4], [3, 3], [4, 2], [5, 1], [6, 0], [7, 9], [8, 8], [9, 7]],
        7 <= [[0, 7], [1, 6], [2, 5], [3, 4], [4, 3], [5, 2], [6, 1], [7, 0], [8, 9], [9, 8]],
        8 <= [[0, 8], [1, 7], [2, 6], [3, 5], [4, 4], [5, 3], [6, 2], [7, 1], [8, 0], [9, 9]],
        9 <= [[0, 9], [1, 8], [2, 7], [3, 6], [4, 5], [5, 4], [6, 3], [7, 2], [8, 1], [9, 0]],
    ],
}

// define a LUT for:
// if b > 4 {
//     (a + b * 2 - 9) % 10
// } else {
//     (a + b * 2) % 10
// }
def_lut_2in_1out! {
    fix: [$],
    name: [mod10_aP2b],
    map: [
        0 <= [[0, 0], [1, 9], [2, 4], [3, 8], [4, 3], [5, 7], [6, 2], [7, 6], [8, 1], [9, 5]],
        1 <= [[0, 5], [1, 0], [2, 9], [3, 4], [4, 8], [5, 3], [6, 7], [7, 2], [8, 6], [9, 1]],
        2 <= [[0, 1], [1, 5], [2, 0], [3, 9], [4, 4], [5, 8], [6, 3], [7, 7], [8, 2], [9, 6]],
        3 <= [[0, 6], [1, 1], [2, 5], [3, 0], [4, 9], [5, 4], [6, 8], [7, 3], [8, 7], [9, 2]],
        4 <= [[0, 2], [1, 6], [2, 1], [3, 5], [4, 0], [5, 9], [6, 4], [7, 8], [8, 3], [9, 7]],
        5 <= [[0, 7], [1, 2], [2, 6], [3, 1], [4, 5], [5, 0], [6, 9], [7, 4], [8, 8], [9, 3]],
        6 <= [[0, 3], [1, 7], [2, 2], [3, 6], [4, 1], [5, 5], [6, 0], [7, 9], [8, 4], [9, 8]],
        7 <= [[0, 8], [1, 3], [2, 7], [3, 2], [4, 6], [5, 1], [6, 5], [7, 0], [8, 9], [9, 4]],
        8 <= [[0, 4], [1, 8], [2, 3], [3, 7], [4, 2], [5, 6], [6, 1], [7, 5], [8, 0], [9, 9]],
        9 <= [[0, 9], [1, 4], [2, 8], [3, 3], [4, 7], [5, 2], [6, 6], [7, 1], [8, 5], [9, 0]],
    ],
}
```
And then the final check:
```rs
macro_rules! final_check {
    (1, 9) => { true };
    (2, 8) => { true };
    (3, 7) => { true };
    (4, 6) => { true };
    (5, 5) => { true };
    (6, 4) => { true };
    (7, 3) => { true };
    (8, 2) => { true };
    (9, 1) => { true };
    ($($other:tt),*) => { false };
}
```
And that's that! We now have implemented the Luhn algorithm with declarative macros. Here's a
[Playground link][luhn-playground] so you can mess around with actually using it.

What was even the point? The point was to try and expose readers to enough of these patterns that by
this point they may have begun to notice something feeling familiar about how this all works. Though
perhaps you may not have gone on the journey to this type of programming yet or I have done a poor
job of setting you up to make this realisation, so let me state it more explicitly:

# Macros Are Functional

Or, well, kind of. They're very, very close to functional programming. The only thing I can even
think of that makes macros impure functions is the side effect of mutating the module namespace[^2].
For example, calling a struct-defining macro twice with the same input will produce the same output,
which is a problem because there will now be two structs with the same name, producing a compile
error. Other than this, there's very few things that differ from pure functional programming:
- Functions (macros) may capture from their environments, but they can only capture things defined
  as items
- Equality checks can only be done where one or both of the LHS/RHS are not metavariables
- You cannot do something like Haskell's `where varaible = expr` (though internal rules do act a lot
  like `where rule args = expr`)
- Arguments can be taken in with regex-like repetition specifiers
- Different branches don't have to take in the same "type" arguments or use the same ordering

For the purpose of this section I'll be attempting to relate the macros we've defined so far in this
article to Haskell code (because it's the functional language I'm most familiar with). So, let's get
started with the rock paper scissors macro:
```hs
data Hand = Rock | Paper | Scissors

rockPaperScissors :: Hand -> Io ()
--               (rock, rock) => { println!("tie"); };
rockPaperScissors Rock Rock = putStrLn "tie"
--               (rock, paper) => { println!("lose"); };
rockPaperScissors Rock Paper = putStrLn "lose"
--               (rock, scissors) => { println!("win"); };
rockPaperScissors Rock Scissors = putStrLn "win"
--               (paper, rock) => { println!("win"); };
rockPaperScissors Paper Rock = putStrLn "win"
-- ...
```
Adding every other input also works:
```hs
data AddSkip = Add | Skip

addEveryOther :: Num a => [a] -> a
-- () => { 0 };
addEveryOther [] = 0
-- ($($val:literal),+$(,)?) => {
--     add_every_other!(@add [add] [$($val),+])    
-- };
addEveryOther list = addEveryOtherInner list Add
  where
    -- (@add [add] [$first:literal $(, $rest:literal)*]) => {
    --     $first + add_every_other!(@add [skip] [$($rest),*])
    -- };
    addEveryOtherInner (head : tail) Add = head + addEveryOtherInner tail Skip
    -- (@add [skip] [$first:literal $(, $rest:literal)*]) => {
    --     add_every_other!(@add [add] [$($rest),*])
    -- };
    addEveryOtherInner (head : tail) Skip = addEveryOtherInner tail Add
    -- (@add @_:tt []) => { 0 };
    addEveryOtherInner [] _ = 0
```
And so does the Luhn algorithm (I'm not going to do the direct comparisons to rule definitions this
time, sorry):
```hs
data LengthParity = Even | Odd

luhn :: Integral a => [a] -> Bool
luhn list = calculateResidue lengthParity Even 0
  where
    -- the way we determine length parity doesn't match super well, so I'll get
    -- that in a more standard way
    lengthParity = if length list `mod` 2 == 0 then Even else Odd

calculateResidue :: Integral a => [a] -> LengthParity -> LengthParity -> a -> Bool
calculateResidue [head] lParity iParity acc = finalCheck head acc
calculateResidue (head : tail) Even Odd acc = addMod10 tail Even Even (acc, head)
calculateResidue (head : tail) Odd Even acc = addMod10 tail Odd Odd (acc, head)
calculateResidue (head : tail) Even Even acc = mod10AP2B tail Even Odd (acc, head)
calculateResidue (head : tail) Odd Odd acc = mod10AP2B tail Odd Even (acc, head)

addMod10 :: Integral a => [a] -> LengthParity -> LengthParity -> (a, a) -> Bool
addMod10 list lParity iParity (0, 0) = calculateResidue list lParity iParity 0
addMod10 list lParity iParity (0, 1) = calculateResidue list lParity iParity 1
-- ...

mod10AP2B :: Integral a => [a] -> LengthParity -> LengthParity -> (a, a) -> Bool
mod10AP2B list lParity iParity (0, 0) = calculateResidue list lParity iParity 0
mod10AP2B list lParity iParity (0, 1) = calculateResidue list lParity iParity 2
-- ...

finalCheck :: Integral a => a -> a -> Bool
finalCheck 1 9 => True
finalCheck 2 8 => True
-- ...
```

Okay, so maybe it doesn't translate perfectly, but I hope that it's helped demonstrate the point
I'm trying to make.

With this in mind, let's see if we can generalise what the macro patterns look like in a functional
format (apart from recursion, because that one's pretty obvious).

### Internal Rules

Internal rules generally look like `where` bindings, though they can also be written as their own
functions. Functionally, this looks like
```
macroRules pattern = internalRule pattern
  where
    internalRule = ...
```

### Incremental TT Munchers

Incremental TT munchers look a lot like matching on `head : tail` or similar on part of our input,
and then recursing with `tail`. We then also need a base case to handle when this pattern fails to
match. This might be something like
```
macroRules [] = 0
macroRules head : tail = 1 + macroRules tail
```
We can think of the [example TT muncher][example-tt-muncher] from the presentation as something like
```hs
munchAndCrunch :: AstNode -> Io ()
munchAndCrunch [] = putStrLn "empty!"
munchAndCrunch (head : tail) = do
    putStrLn $ "munched: " ++ show head
    munchAndCrunch tail
```

### Push-down Accumulation

This one is more about the "function signature" of the macro. Because macros must expand to complete
syntax elements, the return value of the function must be a complete element. So if we were to think
of macros as Haskell functions, their types must always end in `-> CompleteSyntaxElement` for some
kind of complete syntax element. That can be an expression, a statement, an item, etc. etc.

For example, the [reverse tokens][reverse-tokens] example from the presentation might be something
like
```hs
reverseTokens :: [Ast] -> CompleteSyntaxElement
reverseTokens list = completeSyntaxElementFrom reversedList
  where
    reverseTokensInner [] = []
    reverseTokensInner (head : tail) = reverseTokensInner tail ++ [head]
```

### TT Bundling

This is sort of like packing a bunch of values into a struct. Instead of now having to match on a
large amount of inputs, you match on a single struct instance. The nice thing here is that you can
still access those struct fields later, as opposed to most other times you turn multiple tokens
into one single token.

### Callbacks

Generally this can be thought of as any function that takes another `[Ast] -> [Ast]` as a parameter
which is then called at some later point. Not much else to say here and I can't think of any trivial
examples for this.

# Alright, that's enough waffling

tl;dr: get comfortable with functional programming, it'll help you with writing declarative macros.

`macro_rules!` macros function so similarly to functional programming that thinking of them as such
is honestly way more useful than it ought to be, and I can genuinely recommend familiarising
yourself with a functional language or just with functional problem solving.

As I started picking up Haskell I noticed that my macro power level was increasing even faster than
usual, so if you're looking to buff up in this particular part of Rust programming then I'd suggest
learning Haskell (^: (or another functional language if you'd prefer)

[^1]:
    With some inspiration I did actually realise this is possible recently, see [my blog post on
    the topic]({% post_url 2024-02-26-uh-oh %})

[^2]:
    It's possible this is not actually a side effect of macros and that you could consider the
    mutation of the module namespace to be done by the compiler after the macro is expanded, however
    I'm not certain this is a wholly useful distinction to make since your code isn't going to
    compile either way. 

[luhn]: https://en.wikipedia.org/wiki/Luhn_algorithm
[luhn-playground]: https://play.rust-lang.org/?version=stable&mode=debug&edition=2021&gist=450bdd3ebfdaf2c6283d4c42648d1bf9
[example-tt-muncher]: https://github.com/AuroransSolis/rustconf-2023/blob/main/muncher-demo/src/main.rs
[reverse-tokens]: https://github.com/AuroransSolis/rustconf-2023/blob/main/reverse-tokens/src/main.rs
