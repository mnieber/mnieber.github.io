---
layout: post
title: "How to avoid bad codebases?"
date: 2021-02-20 13:45:59 +0100
categories: programming
---

# Introduction

Over the years I have participated in many software developer teams as a permanent member or freelancer.
In most cases we worked on codebases that were in poor shape. In this blog post, I will try to reflect
on why that is the case, and what could be done to improve things.
I will start out by giving a general overview of problems that often manifest themselves and then zoom in
on particular problems. In some case, I will present this as an advice to do "this" instead of "that", and
in other cases I will try to point out a frequently occurring problem and give suggestions for improvement.
I believe most of these advices can be absorbed by beginning programmers, even though almost none of this
advice is easy to apply (which does not mean that it's not worth trying, in my opinion it really is).

## Frequently occurring problems

### Lack of conceptual clarity and consistency

Imagine that you come across a piece of spaghetti code with terrible names. The only way to figure out what it
does is to tediously inspect every line and mentally calculate its effect. This is of course what we don't want.
The opposite situation exists when the code is well structured and has good names. The names tell us about the
intention of the code. The structure allows us to either focus on low-level details or zoom out and think in terms
of higher-level concepts. We can only write such well structured code when we have a suitable conceptual model
of our system. Therefore, it's a problem when programmers only think in terms of code and not in terms of concepts.

### Lack of modularity

When you mention modularity many programmers think of bigger structures such as libraries, layers, and tools.
However, a source file, a function and a class are also modules, so we should ask ourselves what makes for a
good "source file module", "function module", etc. I argue that "good" means that they have the right responsibilites.
Taking the example of a class: it can have too few responsiblities (which means that things that it ought to
control are not under its control), too many (which means that you are not using divide and conquer properly), or just
about right. The first case is a bit subtle because small classes tend to look okay from the outside. But the problem
with them becomes more apparent when you realize that classes are entities, and every entity represents some kind of
concept. If you have too many classes, it indicates a bloated conceptual model. We've seen the importantance of
our conceptual model, and bloated is not good!

### Ad hoc solutions

Ad hoc solutions are another symptom that appears when there is no strong conceptual model. For example, a programmer
might decide to create a new Project instance when a project request is received. When the request is turned down you need to get
rid of these spurious project instances, perhaps using a SpuriousProjectDeleter class. When you are about to intrudoce such a
class, which seems disconnected from the problem domain (do any of your domain experts talk about spurious projects?)
then it's good to pause and reconsider. In this case, a better conceptual model is to have a ProjectRequest class (especially
if project requests _are_ something your domain experts talk about), and to create the Project instance only if the request was approved. Of course, pragmatic solutions (and even ad-hoc solutions) are a part of programming too, they have their place, just
be on the lookout for better solutions when possible.
