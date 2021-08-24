---
layout: post
title: 'How to arrive at simple code'
date: 2021-11-08 18:42:59 +0100
categories: programming design
---

## Introduction

Programmers argue about many things, but we all agree that simple code is desirable.
We also know that writing simple code is not so easy. In this post I will try give some tips,
aimed at beginning programmers.

## Tips

### Tip: do some upfront analysis

.. example:

    If you have to lay a railway track from A to B, it's good to have a general sense
    of where the track will run. You will not be able to foresee how to build every bridge or
    solve every swamp crossing, but you at least you will be able to rule out some obvious
    bad choices. For example, if there is a shortcut that cuts off a chunk of the trajectory,
    you will be able to spot that early on.

This approach works because developing "on paper" is magnitudes faster than writing actual code.
By spending an hour on thinking how to get from A to B (resisting the temptation to stop halfway)
you've already started on the simplification process.

### Tip: focus first on getting the code to work

Because simple code is easy to understand, it gives the impression that any reasonable person
would produce that code at the first attempt. But simple code requires judicious choices, even though
these choices will look obvious in hindsight. That's why I first focus on getting the code to work.
Remember, while you are working on the code, you haven't actually solved the problem yet. If you
simultaneously focus on writing simple code, you are making your life quite hard. It's often faster to
solve the problem with clunky code, and then simplify.

### Tip: make your data-structures work for you

.. example:

    Let's say your code needs to work with very large vectors, that don't fit into memory.
    To add two such vectors, you might process the data in chunks. Such code will get complex
    very soon. Instead, you need a data-structure that hides the fact that not all the data is in memory
    (for example, by dynamically loading data when it's needed). With this high-level data structure,
    you can add vectors without having to think about their size.

In general, create and use data structures that allow you to express your intentions in a natural way.
If your data-structures are sub-optimal but not horrible to use in practice, then focus first on
getting the code to work. However, if the introduction of better data-structures helps you to
get the code to work, then go for it straight away.

### Tip: be precise with terminology and naming

.. example:

    Let's assume we have a website that shows article prices, where some prices are checked with an
    external server, and other prices are only estimates. On some parts of the website we are allowed
    to show price estimates, but on other parts only checked prices. In this case, we should try to reflect
    in our terminology and naming whether we are working with prices or estimates. For example, we shouldn't
    use the term `price_service` for a service that returns price estimations. Without clear terminology
    we will probably be so confused by our own code that simplifying it will be a big challenge.

Usually, the problem illustrated in the example plays out on a more subtle level. For example, we might use
a function argument called 'article' that actually stores an article id. This is a problem because it obscures
the fact that the function is not dependent on the article record. Spotting these cases allows
us to simplify the code, because we can now use this function in other places where we only have the article id.
We should be precise about terminology, so that we can prevent this problem instead of having to cure it.

### Tip: write less code

Short solutions are not always simple solutions, but they do have some a priori advantages:

- they fit into your head more easily
- they are less likely to contain unnecessary steps (you can usually reduce a big
  function by 5 lines, but not a small one)
- they are easier to change because there is less code to deal with (we all experience this when
  we start a new project)
- they don't scare people off. Few people are eager to dig through a big code base.

The advice to "Keep It Short and Simple" is sound, because "short" and "simple" do tend to go hand in hand.
Try to develop an internal "complexity alarm bell" for when the size of modules and functions grows.

### Tip: pay attention to roles and responsibilities

Functions and classes become harder to understand when they have more responsibilities. Therefore, the
following exercise is useful:

- for each function or class in your module, try to complete the phrase
  "this function/class is responsible for ...".
- if there is an "and" in your answer, then ask yourself whether this means that the function or class
  should be split.

Although source files and modules usually have several responsibilities, the same type of reasoning applies.
Formally this can be stated as: source files and modules should have high cohesion and low coupling. This means
that the functions in a source file or module should ideally be working towards the same goal.

.. example:

    We might have a source file that contains a `UserProfile` class, a `find_unused_profiles` function
    and a `sort_profiles_on_creation_date` function. Let's assume that `sort_profiles_on_creation_date` takes
    care of an implementation detail of `find_unused_profiles`. In this case, you don't want the
    reader to think that `sort_profiles_on_creation_date` and `UserProfile` are somehow directly related.
    This can be achieved by moving `sort_profiles_on_creation_date` and `find_unused_profiles` to a
    separate source file.

### Tip: think both bottom-up and top-down

We often write code bottom-up, getting the basic building blocks in place and then combining them to get closer
and closer to the goal. If we get stuck, we may search for ways to simplify the constructs that we have so far
produced. It's beneficial to also use a top-down approach, where we ask ourselves: if there already existed a library
for my problem, then what functions and use-case examples would I ideally like to see in that library? If we are lucky,
this might opens up new ways of solving the problem that avoid the complexity of our bottom-up solution.

## Conclusion

Arriving at simple code requires effort. It's only partially true that just by programming a lot, we will write ever simpler code.
We also need to examine our programming habits and patterns. This is similar to how a pianist will only make real progress
if they practice in an effective way. If they just repeat the pieces they already know without paying attention to how
they play them, then they will only improve a little. Even code review is not a guarantee that we will write better and better code,
because it often focuses on more superficial aspects. Finally, writing simple code is not a
mechanical process: it often requires some inspiration or a new insight. And this is also what makes it enjoyable.
