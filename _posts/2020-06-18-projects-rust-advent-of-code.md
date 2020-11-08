---
layout: post
title: Advent of Code 2019 in Rust
categories:
  - Projects
---

I've always enjoyed solving programming puzzles. A month and a half of quarantine also gave my sufficient time for finally starting to pick up the Rust programming language. After skimming through the [Rust Book](https://doc.rust-lang.org/book/) and doing the [Rustlings Exercises](https://github.com/rust-lang/rustlings) (which I can highly recommend if you want are the learning-by-doing type), I felt prepared enough to tackle a few slightly larger challenges. I've (partially) done [Project
Euler](https://projecteuler.net/) and [cryptopals](https://cryptopals.com/) before, so I'm not entirely new to programming puzzles. 

If you would like to follow along with my Advent of Code 2019 journey, then you can find my code in [Github](https://github.com/jadeaffenjaeger/rust_aoc19)

## Advent of Code
[Advent of Code](https://adventofcode.com/) is an annual advent calendar that has a programming puzzle behind each door. I went through the puzzles from 2019 and implement them in Rust in order to get a better understanding of the language. Compared to other coding challenges I have done, Advent of Code spans a relatively large field of required skills/methods (at least that's how I perceived it). As of now, I have only halfway completed the 2019 edition and already had to implement
[graphs](https://en.wikipedia.org/wiki/Graph_(discrete_mathematics)), a [Virtual machine](https://en.wikipedia.org/wiki/Virtual_machine) and [Langton's Ant](https://en.wikipedia.org/wiki/Langton%27s_ant) among others. None of these concepts is strictly introduced however, and the challenges are demanding, though (mostly) solvable, which I find is quite an amazing achievement on the part of the creator. While the puzzles are often described as "small", I would certainly not call them
that, at least if you are doing them in a language you're not familiar with. Almost all of the challenges have taken me at least a few hours to implement, so I was glad that I didn't have anything else to do while working on them. How people are capable of solving them while also working full-time and having a family is absolutely beyond me.

I'll not go over every detail of every puzzle here. Only the bits that I found particularly interesting or that were particularly interesting to implement in Rust.
All the code that I've written can be found on [Github](https://github.com/jadeaffenjaeger/rust_aoc19).

### Day 1
One feature of Rust that I enjoy very much is how it allows you to use iterators to map/filter/reduce operations on lists. I find this functional way of working with lists extremely elegant and it even increases performance because the compiler can vectorize it better.
Mapping a function to an array of Integers and adding up the results simply becomes
```rust
fn fuel_required_simple(mass: i32) -> i32 {
    0.max((mass / 3) - 2)
}

let total_fuel_simple: i32 = masses
    .iter()
    .fold(0, |acc, x| acc + fuel_required_simple(*x));
```
and while we're in functional territory anyway, I felt like I more or less *have* to solve the second part using recursion:
```rust
fn fuel_required_complex(mass: i32) -> i32 {
    let fuel = fuel_required_simple(mass);
    if fuel <= 0 {
        0
    } else {
        fuel + fuel_required_complex(fuel)
    }
}

let total_fuel_complex: i32 = masses
    .iter()
    .fold(0, |acc, x| acc + fuel_required_complex(*x));
```
That was a nice warmup.

### Day 2
The first bits of my own Turing Machine! Nice! Putting everything together over the next days felt a bit like those subscription schemes that were around when I was kid: Each week, you'd get a magazine and a part of a model ship/dinosaur/whatever. This challenge is much more satisfying, though, because I never actually finished of the models as a kid. Also, building the Turing machine didn't actually cost any money.

Another lovely feature of Rust are strongly typed enums (I'm looking at you, C) that can also encapsulate data. Perfect for storing instructions.

### Day 3
Another impressive demonstration of the impact of picking the right data structure. My initial implementation stored every point of each grid in two separate lists and then calculated the overlap by iterating over both lists (quadratic complexity - doesn't really seem to do the job here.). Computing the solution that way took roughly 10 minutes on my computer. Storing each point in a [HashSet](https://doc.rust-lang.org/std/collections/struct.HashSet.html) instead brought that down to a few milliseconds. For the second part, I simply
added an additional list for each grid that stores the delays alongside the points. With the calculated overlapping points from the hash set, we can search for them in both lists and retrieve the delay. Not the most elegant solution, because I'm effectively storing every point twice, but I also didn't feel like optimizing it any further after it worked.

### Day 12
This effectively a crude Integer simulation of the [n-Body Problem](https://en.wikipedia.org/wiki/N-body_problem) which I already implemented once in C++ and CUDA for a university project (and for which I will do a write-up as soon as I find the time). One interesting thing that I found in Rust was that I struggled with the Update function for a while. In the update function, I would have normally used two nested for-loops in order to calculate the interactions between every pair of bodies.
However, Rusts concept of Ownership doesn't allow iterating over the same thing twice while mutating the underlying structure at the same time. The problem is that you could use an object to interact with itself, leading to potentially undefined behavior. To rule that out, Rust offers [`split_at_mut`](https://doc.rust-lang.org/nomicon/borrow-splitting.html) which splits a data structures into two at a given index, giving you two separate iterators over each of them. You can now let
objects from one part of the array interact with objects from the other part, circumventing the problem that objects can interact with themselves. Here's my code for the update function:
```rust
fn update_bodies(bodies: &mut Vec<Body>) {
    for i in 1..bodies.len() {
        let (left, right) = bodies.split_at_mut(i);
        for b2 in left {
            right[0].interact(&b2);
            b2.interact(&right[0]);
        }
    }

    for b in bodies {
        b.update_position();
    }
}
```
We are effectively iterating over the array and for every body, we are calculating the interactions *to and from* every body that comes before it in the array. Because of this, we never explicitly need to iterate over the first body, because all its interactions are covered in the subsequent loops. Here's what that would look like for three bodies `A, B, C`:
```
1. Iteration: B-A, A-B
2. Iteration: C-A, A-C, C-B, C-A
```
with that, we cover all possible combinations while *by design* ensuring that we are never accidentally computing `A-A`, `B-B` or `C-C`.

The second part of Day 12 ended up being the first problem that I couldn't solve on my own. Even knowing the solution, I'm fairly confident that I wouldn't have been able to come up with it myself. At first I - obviously - tried a brute-force approach which - obviously - didn't work. Two criticical bits of information that are required to come up with a working solution are:
1. The initial state of the universe has to be the first reoccurring state of the universe. This means that the chain of states will always be `A-B-C-D-A-B-C-D` and can never be `A-B-C-D-C-D-C-D`. This is because the forward transformation to get from one state to the next is invertible. So every state can only have exactly one state that leads to it. If we can go from `B` to `C`, that means that we can also always go from `C` to `B` by applying the inverse function (notice that this is not the case for the second example, where the previous state of `C`
   can either be `D` or `B`. With this, storing only the initial state and comparing to that is sufficient.
2. The computation for the x,y and z-axis are independent. They all exhibit individual cycles. The global cycle length is then when all of them reach the initial state at the same time. Conveniently, this is also the [LCM](https://en.wikipedia.org/wiki/Least_common_multiple) of the cycle lengths for the individual axes.

kudos to everyone who figured this out by themselves...

### Day 13
Nothing too special here, other than that I initially implemented a fully-fledged arcade with display and keyboard input because I wanted to play myself. After trying a couple of times, it turned out pretty quickly that my arcade skills were nowhere near good enough for this challenge, which meant that I had to have a `match` statement play the game for me and could only watch on my arcade screen. It was a good way of getting to play with [SDL2](https://crates.io/crates/sdl2), though!
[![AoC Day 13 Arcade](/assets/images/rust_aoc/day13.png)](/assets/images/rust_aoc/day13.png)

### Day 16
Of course I went with the obvious brute-force route for the first part of the problem. This means that I got to use Rust's iterators in all their functional glory. The [problem description](https://adventofcode.com/2019/day/16) states that each input is multiplied by a pattern of ones, negative ones and zeroes, which changes depending on the position in the input stream.

> While each element in the output array uses all of the same input array elements, the actual repeating pattern to use depends on which output element is being calculated. The base pattern is `0, 1, 0, -1`. Then, repeat each value in the pattern a number of times equal to the position in the output list being considered. Repeat once for the first element, twice for the second element, three times for the third element, and so on. So, if the third element of the output list is being calculated, repeating the values would produce: `0, 0, 0, 1, 1, 1, 0, 0, 0, -1, -1, -1`.
 When applying the pattern, skip the very first value exactly once. (In other words, offset the whole pattern left by one.) So, for the second element of the output list, the actual pattern used would be: `0, 1, 1, 0, 0, -1, -1, 0, 0, 1, 1, 0, 0, -1, -1, ....`

Here's how I ended up generating that pattern of a given length for any given position in the input vector:
```rust
fn generate_pattern(len: usize, pos: usize) -> Vec<i32> {
    let zeroes = iter::repeat(0).take(pos);
    let pos_ones = iter::repeat(1).take(pos);
    let neg_ones = iter::repeat(-1).take(pos);

    zeroes
        .clone()
        .chain(pos_ones)
        .chain(zeroes)
        .chain(neg_ones)
        .cycle()
        .skip(1)
        .take(len)
        .collect()
}
```
which feels much more elegant than anything that I would have come up with in any imperative paradigm.
Needless to say that my brute-force approach wasn't particularly fast. It took around 1-2s to come up with an answer for the first part of the problem, so applying it to the second part (which is the first part, but 1000 times larger) was out of the question.

Initially, I tried to leverage the fact that the problem is a matrix-vector multiply with the pattern being an upper-triangular matrix and the problem input being the vector. However, I couldn't find a general solution. I only realized that it is extremely helpful to leverage the fact that the _problem input_ itself has a property that vastly simplifies the solution. While that felt slightly disappointing, I found it a good reminder of the fact that it can sometimes be a good idea to
take a step back and look the entire problem. After looking up other people's solutions, I realized that there actually _is_ a general solution using partial sums.
