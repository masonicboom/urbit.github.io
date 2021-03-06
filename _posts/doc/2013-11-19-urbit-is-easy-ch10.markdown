---
layout: post
category: doc
title: Urbit Is Easy&#58; Chapter X (Type Inference)
---

*Ever, as before, does Madness remain a mysterious-terrific, altogether infernal boiling-up of the Nether Chaotic Deep, through this fair-painted Vision of Creation, which swims thereon, which we name the Real.*

**(Carlyle)**

[**Prev**: Tiles](urbit-is-easy-ch9.html)

##On type inference algorithms##

Hoon is a higher-order typed functional language.  Most languages
in this class, Haskell and ML being prominent examples, use
something called the Hindley-Milner unification algorithm.  Hoon
uses its own special sauce instead.

Why?  There are two obvious problems with H-M as a functional
type system, the main one being the wall of heavy-ass mathematics
that greets you instantly when you google it.  We have heard some
claims that H-M is actually quite simple.  We urge all such
claimants to hie themselves to its Wikipedia page, which they'll
surely be able to relieve of its present alarming resemblance to
some string-theory paper in Physics Review D.

(Nor is this in any way an an anti-academic stance.  Quite the
contrary.  Frankly, OS guys really quite seldom find themselves
in the math-department lounge, cadging stray grants by
shamelessly misrepresenting the CAP theorem as a result in
mathematics.  It doesn't seem too much to expect the
mathematicians to reciprocate this basic academic courtesy - how
about it, old chaps?)

Furthermore, besides the drawback that it reeks of math and
programmers who love math are about as common as cats who love a
bath - a problem, but really only a marketing problem - H-M has a
genuine product problem as well.  It's too powerful.  

Specifically, H-M reasons both forward with evaluation, and
backward from constraints.  Pretty unavoidable in any sort of
unification algorithm, obviously.  But since the compiler has to
think both forward and backward, and the programmer has to
predict what the compiler will do, the programmer has to think
backward as well.

Hoon's philosophy is that a language is a UI for programmers, and
the basic test of a UI is to be, ya know, easy.  It is impossible
(for most programmers) to learn a language properly unless they
know what the compiler is doing, which in practice means mentally
stepping through the algorithms it uses (with the exception of
semantically neutral optimizations).  Haskell is a hard language
to learn (for most programmers) because it's hard (for most
programmers) to follow what the Haskell compiler is thinking.

It's true that some programmers have an effective mathematical
intuition that let them "see" algorithms without working through
them step by step.  But this is a rare talent, we feel.  And
even those who have a talent don't always enjoy exercising it.

If a thorough understanding of any language demands high-grade
mathematical intuition in its programmers, the language as a UI
is like a doorway that makes you duck if you're over 6 feet tall.
The only reason to build such a doorway in your castle is if you
and all your friends are short, and only your enemies are tall.
Is this really the case here?

Although an inference algorithm that reasons only forward must
and does require a few more annotations from the programmer, the
small extra burden on her fingers is more than offset by the
lighter load on her hippocampus.  Furthermore, programs also
exist to be read.  The modern code monkey is above all things a
replaceable part, and some of these annotations (which a smarter
algorithm might infer by steam) may annoy the actual author of
the code but be a lifesaver for her replacement.

##Low-powered type inference##

Broadly speaking, type inference in Hoon has three general
limitations as compared to H-M inference.  

*One*, Hoon really does not think backwards. For instance, it
cannot infer a function's argument type (or to use Hoonese, a
gate's sample type) from its body.  

*Two*, Hoon can infer through tail recursion, but not head
recursion.  It can *check* head recursion, however, given an
annotation.

*Three*, the compiler catches most but not all divergent
inference problems - ie, if you really screw up, you can put the
compiler into an infinite loop or exponential equivalent.  That's
ok, because an interrupt will still show you your error location.
Also, this never happens once you know what you're doing.

Our experience is that these limitations are minor annoyances at
worst and prudent restrictions at best.  Your mileage may vary.

A good basic test for the power of a type inference algorithm is
whether, given a grammar for a complex AST, it can verify
statically that the noun produced by an LL combinator parser for
the grammar actually fits in the AST type.  (It would be quite
surprising to see any language solve this for an LR parser.)

Hoon indeed checks its own LL parser.  Moreover, the core of
the Hoon compiler - `++ut`, which handles both type inference and 
Nock generation, is less than 2000 lines (and feels a bit bloated
at that, since for obvious reasons it is relatively old code).
Hoon is pretty expressive, but not `that` expressive.  

So we feel Hoon's approach to type inference offers, beneath its
rich Corinthian leather, an unrivalled balance of simplicity and
power.  Still, don't take it out on the freeway till you're
pretty sure you know how to drive it.

##Facts to keep in mind##

Type inference is a frightening problem, especially if you've
been exposed to the wall of math.  Your only real problem in
learning Hoon is to learn not to fear it.  Once you work past
this reasonable but entirely unfounded fear of inference, your
Hoon experience will be simple, refreshing and delightful.  So
first, let's talk through a few reassuring facts:

*One*, it's important to remember - having just read about tiles - 
that type inference in Hoon *never sees a tile*.  It operates
exclusively on twigs.  All tiles and synthetic twigs are reduced
to natural twigs for the inference engine's benefit.

*Two*, the semantics of Hoon are in `++ut` in `hoon.hoon`, and 
nowhere else.

*Three*, within `++ut`, all the semantics of Hoon are in the call
graph of one arm - `++mint`.  `++mint` has a case for every
natural hoon.  So do `++play` and `++mull`, but their semantics
are redundant with `++mint`.

*Four*, one leg in the sample of `++mint` - `gol` - which looks
for all the world like a mechanism for backward inference, is
not.  It is semantically irrelevant and only exists to get better
localization on error reports.

*Five*, we've already explained what `++mint` does, but let's
repeat it one more time:

When we have a type that describes the subject for the formula
we're trying to generate, as we generate that formula we want to
also generate a type for the product of that formula on that
subject.  So our compiler computes:

    [subject-type twig] => [product-type formula]

As long as `subject-type` is a correct description of some
subject, you can take any twig and compile it against
`subject-type`, producing a `formula` such that `*(subject
formula)` is a product correctly described by `product-type`.

`++mint` is the gate that maps `[type twig]` to `[type nock]`.
So, if you know `++mint`, you know Hoon.

*Six*, most of the things `++mint` does are obvious.  For
instance, let's quickly run through how `++mint` handles a
`=+` (`tislus`) twig, `[%tsls p=twig q=twig]`.  This is a 
synthetic twig:

    ++  open
      ^-  twig
      ?-  gen
        [%tsls *]  [%tsgr [p.gen [~ 1]] q.gen]
      ==

ie, `=+(a b)` is `=>([a .] b)`.  We thus turn to the `%tsgr`
twig in `++mint`.  Simplifying broadly:

    ++  mint
      |=  [sut=type gen=twig]
      ^-  [p=type q=nock]
      ?-    gen
          [%tsgr *]
        =+  fid=$(gen p.gen)
        =+  dov=$(sut p.fid, gen q.gen)
        [p.dov (comb q.fid q.dov)]
      ==
    ++  comb
      |=  [mal=nock buz=nock]
      ^-  nock
      ?:  &(?=([0 *] mal) !=(0 p.mal))
        ?:  &(?=([0 *] buz) !=(0 p.buz))
          [%0 (peg p.mal p.buz)]
        ?:  ?=([2 [0 *] [0 *]] buz)
          [%2 [%0 (peg p.mal p.p.buz)] [%0 (peg p.mal p.q.buz)]]
        [%7 mal buz]
      ?:  ?=([^ [0 1]] mal)
        [%8 p.mal buz]
      ?:  =([0 1] buz)
        mal
      [%7 mal buz]

It may not be obvious what `++comb` is doing, but we can guess
that it's composing two Nock formulas, and in particular doing
its best to get back the `%8` shorthand that we might have
otherwise thrown away by making `=+` a synthetic hoon.

If you understand this, you understand the general problem that
`++mint` is solving.  Now, let's dive into the details.

##Pseudolazy evaluation: %hold##

We'll start by introducing a new type stem, `%hold` (yes, we are
bringing them in one by one like Gandalf's dwarves):

    ++  type  $|  ?(%noun %void)
              $%  [%atom p=term]
                  [%cell p=type q=type]
                  [%core p=type q=coil]
                  [%cube p=* q=type]
                  [%face p=term q=type]
                  [%fork p=type q=type]
                  [%hold p=(list ,[p=type q=twig])]
              ==

(There is now only one type stem that's missing - and a
relatively frivolous and unimportant stem at that.)

`%hold` looks a little bit funky there.  Let's indulge in a
little more unlicensed simplifying, and pretend it was instead

    [%hold p=type q=twig]

(The actual `%hold`, with its list, is just a big `%fork`; and can
be adequately represented, at least semantically, by a nested
`%fork` of simple holds like this one.  The reasons it doesn't
work this way are unimportant and we can forget them for now.)

What are the semantics of a `%hold`?  Simple - this type
delegates to the value, also of course a type, `p:(mint p q)`.
(Once again, in the real `hoon.hoon`, the `++mint` interface
is also slightly different from our caricature here.)

Here we see how infinite types, such as a linked list, can be
constructed.  Hoon, of course, is a strict language (ie, one
without lazy evaluation, like Haskell) and cannot construct an
infinitely long linked list.  We certainly can describe, as a
`++type`, the infinite set which contains all linked lists - or
all lists of a given item type, eg, `[p=type q=twig]`.  But this
infinite set must be defined in a very finite noun.

When we traverse this finite noun by expanding `%hold` stems, we
implicitly produce the illusion of an infinite type - for
instance, one that in the case of a linked list of @tas, was

    [ %fork
      [%cube 0 %atom %n]
      [ %cell
        [%atom %tas]
        [ %fork
          [%cube 0 %atom %n]
          [ %cell
            [%atom %tas]
            [ %fork
              [%cube 0 %atom %n]
              ... 

and it's turtles all the way down.  (`[%cube 0 %atom %n]` is of
course the null terminator, `~`).

Infinite state is not required to produce this sequence ad
infinitum.  In any language, this is essentially what an infinite
list *is* - an indefinite pattern generated from finite state..
But in Hoon, the illusion is produced at the user level, not
hidden in the language semantics.

Perhaps it's not clear why this is superior...

Again, you want to define an infinite list.   But your physical
computer is finite.  So, your infinite list must consist of some
kind of generated pattern.  Lots of ways to do this.

What is the right way to manage this finite state?  State of this
kind can be expressed in three forms: (a) the actual data
structure that the pattern contains; (b) a core, which takes this
payload and wraps a pattern generator around it; (c) an
abstraction, which makes the generator indistinguishable from
simple list data.

It seems clear that since (a) can be converted into (b), and (b)
into (c), the best representation is (a).  For instance, (a) is
easy to send over a network, (b) is clunky at best (we really try
to never send nock over the network), (c) is a research project.
In short, lazy evaluation is inherently a leaky abstraction.

##How we use %hold##

There is one advantage of explicit over implicit laziness which
would leave us no choice but to use explicit laziness in `++ut`, 
even if we expressed the same semantics in a language with
built-in implicit laziness (like Haskell).

The advantage is that the explicit pattern state makes a good
search key for an associative array, ie, `++map` in Hoon.  The
problem of using an infinite expansion as a search key in a lazy
language is one I'd be surprised to see solved, and even more
surprised to see solved well. 

Consider the function computed by `++nest`.  Again simplifying
the interesting core structure of `++ut`, `(nest sut ref)` is yes
if *it can verify that* every noun which is a member of type
`ref` is also a member of type `sut`.  In other words, `++nest`
is a conservative algorithm - it sometimes busts the innocent,
it always nails the guilty.

(`++nest` and everything else in the Hoon type system practice a
policy of pure "structural equivalence."  Even %face is ignored
for compatibility purposes; only noun geometry is considered.  If
it walks like a duck, etc.)

(You can look at `++nest` yourself.  It is a big function for
Hoon: 150 lines, though arranged quite loosely (it should be 
more like 120) with a complex internal core.)

If you look again at `++type` above and forget about `%hold`, 
it is probably not hard to imagine building a `++nest` for 
*finite* type nouns.  (The problem is not even equivalent to SAT,
because there is a unification stem but no intersection stem.)

But for (logically) infinite types?  Yes - actually, the problem
is straightforward.  But the solution requires us to use `%hold`
types as search keys.

Essentially, `++nest` can begin by assuming that every `ref` is a
`sut`, then traverse the parallel trees searching for any case
of a `ref` that might conceal a non-`sut`.  In this traverse, we
can simply keep a set of the `[sut ref]` problems we are solving:

    =+  gil=*(set ,[p=type q=type])

Again, `gil` is just the `++nest` stack as a live problem set.

If we descend into another instance of the search tree we're
actually in, ie, currently searching for exceptions to the
compatibility assumption, there is no reason to restart our
search.  So, we can prune and ignore these recurrences.  And
so, we can write a decent `++nest`.

Every nontrivial algorithm that traverses types keeps one of
these recurrence sets to detect repeating patterns.

This leaves the onus on the programmer to design types that recur
regularly.  An infinite type *cannot* be traversed if the actual
`[%hold p=type q=twig]` in it does not recur, *with the exact
same type and twig nouns*.  Indeed, doing anything significant
with this type will cause the compiler to hang (requiring an
interrupt, which will tell you where you made this mistake).

Fortunately, doing the right thing here is much easier than doing
the wrong thing.  Why would you roll your own `list`, anyway?
Use the standard tools and everything will work.

##Where we use %hold##

Everywhere.  But more specifically, every time a wing resolves to
an arm, we don't trace into the callee; we make a `%hold`.
Hence, in practice the type in a `%hold` is always a `%core`.

When other typed functional languages construct or define a gate,
lambda, closure, function, etc, they generally compute a "type
signature" for the function.  Ie, it accepts *this* and produces
*that*.  You will see things in Hoon gates that look to all the
world like type signatures.  They aren't type signatures.

In Hoon, if the compiler wants to traverse the type a gate
produces, it simply iterates a subject through the AST - the
twig.  In doing so, we start with the subject

    [%hold gate-type gate-twig]

and iterate into it, reducing as needed, and checking wherever
possible for logic loops, until we find what we want.

Logically this iteration is always the same as `q:(mint p q)`,
but in actual practice `++mint` is making a formula we don't
need.  Also, we only need to actually verify correctness when we
generate a formula - not when we traverse pseudolazy types.
Hence, we have the lighter `++play` gate for traversal.

##Head and tail recursion##

The `^-` (`kethep`, `%kthp`) twig, 

    [%kthp p=tile q=twig]

produces the type `p:(mint sub ~(bunt al p.gen))` and the formula
`q:(mint sub q.gen)`, checking that `p:(mint sub q.gen)` nests in
`p:(mint sub ~(bunt al p.gen))`.  

It is difficult to avoid the word "cast" when talking about
`%kthp`, so we relent and use it.  Essentially we are casting the
product of twig `q` to the icon of tile `p`.
 
(Casting should not be confused with fishing.  `%kthp` is a purely
static operation with no runtime effect whatsoever.)

It's a good general principle to cast the product of every arm.
First, because `%face` is ignored by `++nest`, the cast is an
opportunity to get the names right - for example, your arm can
produce `[foo bar]` and cast it at the end to `[p=foo q=bar]`.
The type produced by mere inference on practical code may be
funky, duplicative, poorly named, etc, etc.  Even if it's not,
it's often good to remind the reader what just got made.

Furthermore, when an arm is used in head recursion, casting its
product becomes essential.  Our simplistic inference algorithm
cannot follow head recursion; but it can check that every step in
head recursion is correct, leaving no loophole.  

A key point is that since `++play` does not verify, it does not
need to descend into twig `q` at all.  Thus when we type-check
head recursion, we check that any step is correct, assuming any
further steps are correct - a positive answer is dispositive.

If you don't completely understand (or believe) this, or if you are
not quite sure on the difference between head and tail recursion,
just cast the product of every arm.

`%kthp` also has an irregular wide form.  `^-(@tas foo)` can also
be written

    `@tas`foo

For example, although Hoon is perfectly capable of inferring that
decrement produces an atom, the decrement in `hoon.hoon` casts
the product of the loop step anyway:

    =+  b=0
    |-  ^-  @
    ?:  =(a +(b))
      b
    $(b +(b))

Note that this is just a fancy modern arrangement of the classic Hoon 

    =+  b=0
    |-  
    ^-  @
    ?:  =(a +(b))
      b
    $(b +(b))

Ie, it is only by casting *within* the arm that 

The idiom of "barhep kethep tile" is common enough that you
should rarely see a `|-` without a `^-`.  Especially while
still a beginning Hoon programmer - when in doubt, cast.

##Branch analysis##

Type inference wouldn't be very useful or interesting if we
couldn't learn things about our nouns at runtime.  Actually,
we can't even really *use* most nouns without type inference.

For instance, let's make a list:

    ~waclux-tomwyc/try=> =foo `(list ,@)`[1 2 3 4 ~]
    ~waclux-tomwyc/try=> foo
    ~[1 2 3 4]
    ~waclux-tomwyc/try=> :type; foo
    ~[1 2 3 4]
    it(@)

Looks good.  (Our prettyprinter has traversed the `list` type and
matched it to its own list detection heuristic - which is the
only way we can print a list as such, since `++list` is just a
gate and no trace of the word `list` remains in the type.)

So let's try to use it.  What's the first item?

    ~waclux-tomwyc/try=> i.foo
    ! -type.it(@)
    ! -find-limb.i
    ! find-fork
    ! exit

Doh!  We got a `find-fork` because, with the cast, we threw away
our knowledge that we had `[1 2 3 4 ~]` and replaced it with a
repeating structure that is either `~` or `[i=@ t=(list ,@)]`.
We cannot pull `i` from `foo`, because we don't know that `foo`
is not `~`.

Since we've lost that information, we need a way to test for it
dynamically, and change the type of `foo` on either side of the
branch.  To wit:

    ~waclux-tomwyc/try=> ?~(foo 42 i.foo)
    1

What is `?~` - `wutsig`, `%wtsg`?  A simple synthetic that
use our fishing hoon, `?=` (`wuttis`):

    ~waclux-tomwyc/try=> ?:(?=(~ foo) 42 i.foo)
    1

On the `|` side of the `?:`, we know that `foo` is not `~` -
and because it's not `~`, it has to be a cell.  We can see the
type system learning this: 

    ~waclux-tomwyc/try=> :type; ?~(foo !! foo)
    [i=1 t=~[2 3 4]]
    [i=@ t=it(@)]

Notice also the use of `!!`.  `!!` always crashes - so it
produces the type `%void`.  We're essentially asserting that
`foo` is a non-empty list.   A branch that produces `%void` will
never return a value (if taken, it will always crash), so its
product can be ignored.  If we changed that:

    ~waclux-tomwyc/try=> :type; ?~(foo %foobar foo)
    [i=1 t=~[2 3 4]]
    { %foobar [i=@ t=it(@)] }

It's important to note that the *only* hoons recognized in branch
analysis are `?=` (`wuttis`, `%wtts`, fish), `?&` (`wutpam`,
`%wtts`, logical and), and `?|` (`wutbar`, `%wtbr`, logical or).
(Boolean logic is fully understood within the test itself, so the
second twig in a `?&` can depend on the result of the first.)

Of course, synthetic hoons that reduce to these work as well.
However, we don't learn from any other test, even if we could:

    ~waclux-tomwyc/try=> ?:(?=(~ foo) 42 i.foo)
    1
    ~waclux-tomwyc/try=> ?:(=(~ foo) 42 i.foo)
    ! -type.it(@)
    ! -find-limb.i
    ! find-fork
    ! exit

Type inference in Hoon is anything but magic.  It's a relatively
simple algorithm, which you nonetheless need to understand if
you're going to make it as a Hoon programmer.  "It just works"
and "do what I mean" are not in the building.

All branches (such as `?~`) reduce to `?:`.  It would appear, for
instance, that Hoon has (like many other functional languages)
pattern-matching primitives, `?-` (and `?+`, which takes a
default).  Au contraire - these are just synthesized from `?:`
and `?=`.

Finally, another thing that branch analysis can do is statically
detect that branches aren't taken.  For instance, with our little
`foo` list, we cannot know *statically* whether or not it's null,
but we can test it *dynamically*.  Statically, however, we know
one thing - `foo` cannot have the value `%foobar`.

So what happens if we fish for it?

    ~zod/try=> ?:(?=(%foobar foo) 42 i.foo)
    ! mint-vain
    ! exit

For lo, we fish in vain.  We get this error whenever a branch is
not taken.  This is tremendously useful when, for example, `?-`
switches on a kelp - if we write a case for a stem that doesn't
exist, or miss a stem that must be handled, the compiler knows.

##Geometric and generic polymorphism##

One interesting question in any language is what happens when you
pass a function an argument more specialized than its declaration.

Consider a function that takes a pair `[a b]` and produces [b a].
Any noun will obviously do for `b` and `a`.  So the sample tile
is `[a=* b=*]`.  But suppose the caller knows something more
about `a` and `b`.  What happens?  Let's give it a try:

    ~zod/try=> :type; (|=([a=* b=*] [b a]) %foo %bar)
    [7.496.034 7.303.014]
    [* *]

With `|=` (`bartis`, `%brts`), the normal way of building a gate,
we lost all our type information.  Sometimes this is fine.
Sometimes it's exactly what we want.  But sometimes...

Here's something else we can do:

    ~zod/try=> :type; (|*([a=* b=*] [b a]) %foo %bar)
    [%bar %foo]
    [%bar %foo]

By using `|*` (`bartar`, `%brtr`), we seem to have produced the
same noun, but not lost any type information.  Interesting...

The difference is that while both `|=` and `|*` are polymorphic,
`|=` uses *geometric* polymorphism and `|*` uses *generic*.
Which should you choose?

There's a simple rule for this, which is that unless you are a
wizard doing wizardly things, you should use `|=`.  Generic
polymorphism is a way to build and use tools like containers
(lists, etc), which in other, inferior languages might simply be
built in to the language semantics.  Especially as a new Hoon
programmer, you are probably not building heavy machinery of this
kind, and should probably make do with the standard components.

But you will be *using* containers, etc.  So there's no shortcut
for understanding both systems.

###Geometric polymorphism###

We've already met the fundamental function of geometric
polymorphism, `++nest`.  The question `(nest sut ref)` asks is:
can I take any noun in type `ref`, and use it as if it was within
`sut`?

Consider the trivial example above - the question is, can we use
our noun [%foo %bar] as if it was a `[a=* b=*]`?  

When we use `%=` (`centis`, `%cnts`) to modify the sample in a
core, we actually change the type of the core.  (Of course we are
not *modifying* the core per se, but creating a new one with the
given changes.)

    [ %core
      [ %cell 
        [%cell [%face %a %noun] [%face %b %noun]]
        context
      ]
      (map term twig)
    ]

to

    [ %core
      [ %cell 
        [%cell [%cube %foo [%atom %tas]] [%cube %bar [%atom %tas]]]
        context
      ]
      (map term twig)
    ]

This poses a couple of problems.  First, remember, when we infer
into a gate call or any other use of a core arm, we infer lazily
by creating a `[%hold type twig]` with the subject core and the
arm body.  When we want to find out what type the arm produces,
we simply step through it.

But among other possible problems here, we've actually *destroyed
our argument names*.  If we try to step through the body of the
function with this modified core, we can't possibly play `[b a]`.
The names `a` and `b` simply won't resolve.

This is why a core is actually more complicated than our original
explanation would suggest.  The `%core` frond is actually:

    ++  type  $%  [%core p=type q=coil]
              ==
    ++  coil  $:  p=?(%gold %iron %lead %zinc)
                  q=type
                  r=[p=?(~ ^) q=(map term foot)]
              ==
    ++  foot  $%  [%ash p=twig]
                  [%elm p=twig]
              ==

Geeze, man, what is all this nonsense?  Okay, fine.  Let's
explain the real `%core`.

First, `q` in the `coil` is the original payload type for use in
a `%hold`.  Hence geometric polymorphism.  The question we have
to answer whenever we use an arm is: is this core corrupt?  As
in: is the payload that's in it now geometrically compatible with
the payload that its formulas were compiled with?  Can we use the
present payload as if it was the original, default payload?

It is actually not a type error of any kind to produce a modified
core with the payload set to any kind of garbage at all.  It's
just a type error to *use* it - unless the payload is compatible.
And when we use a geometric arm, we test this compatibility and
then treat the present sample as if it was the original.

`q.r` in the coil is the map of arms.  The polymorphism model is
not an attribute of the core, but of the arm - `%ash` means
geometric, `%elm` means generic.  

`p.r`, if not null, is the actual generated battery of Nock
formulas.  Why do we need this at compile time?  Remember
`%ktsg`, which folds constants.  

`p.r` is null while we are compiling the core's own arms, for
obvious reasons (though we could get a little smarter about
circularities), but once we complete this task we can put the
battery in the type.  The result is that, if we are building a
reef of cores, we can fold arms of this core in the next one
down.  

For example, one of the main uses of `%ktsg` is simply to make
default samples constant, so that we don't have to perform some
simple but unnecessary computation every time we use a gate.
Because we can only fold an arm in a completed core, a good
general practice in building applications is to use a reef of at
least two cores - inner for the tiles, outer for the functions.

All that remains is this mysterious `p` field.  If you are an OO
geek of a certain flavor who was once busted with a Sharpie for
writing "BERTRAND MEYER IS GOD" on a Muni bus, you may be
familiar with the broad language concept of *variance*.

Polymorphism in Hoon supports four kinds of variance: `%gold`
(invariant), `%lead` (bivariant), `%zinc` (covariant), and
`%iron` (contravariant).

The question of variance arises when we want to test whether one
core is compatible with another.  Hoon is a functional language,
for example, so it would be nice to pass functions around.  Gosh,
even C can pass functions around.

For core A to nest within core B (any A can be used as a B), it
seems clear that A should have the same set of arms as B (with
formulas at exactly the same axes in the battery), and that any
product of an A arm should nest within the product of the
corresponding B arm.

But what about the payloads?  Worse - what about the *contexts*?
A simple rule might be that the payload of A must also nest
within the payload of B.  Unfortunately, this doesn't work.

Suppose, for instance, that I write a sort gate, one of whose
arguments is a comparison gate producing a loobean.  (This is
generally a problem calling for generic polymorphism, but let's
assume we're sorting lists of a fixed type.)  Okay, great.  We
know the *sample* type of the comparator - but what about the
*context*?  The sort library cannot possibly have the same
context as the application which is using the sort library.  So,
the cores will be incompatible and the invocation will fail.

This rule, which doesn't work in this case, is the rule for
`%gold` (invariant) cores.  Every core is created `%gold`, and
remains `%gold` from the perspective of its own arms.

But the type of some arbitrary comparator, which is an argument
to our sort function, cannot possibly be `%gold`.  Rather, we
need an `%iron` (contravariant) core.  You can turn any `%gold`
core into an `%iron` one with `^|` (`ketbar`, `%ktbr`), but the
transformation is not reversible.

The rules for using an `%iron` core are that (a) the context is
opaque (can neither be read nor written), and (b) the sample is
write-only.  Why?  Because it's absolutely okay to use as your
comparator a gate which accepts a more general sample than you'll
actually call it with.  You can write a more specialized noun
into this sample - but if you read the default value and treat it
as more specialized, you have a type loophole.

A `%zinc` (covariant) core is the opposite - the context remains
opaque, the sample is read-only.  We don't use any `%zinc` at
present, but this is only because we haven't yet gotten into
inheritance and other fancy OO patterns.  (Hoon 191 had
inheritance, but it was removed as incompletely baked.) You make
a `%gold` core `%zinc` with `^&` (`ketpam`, `%ktpm`).

Finally, the entire payload of a `%lead` (invariant) core is
immune to reading or writing.  So all that matters is the product
of the arms.  You make a lead core with `^?` (`ketwut`, `%ktwt`).

###Generic polymorphism###

When the arm we're executing is `%elm`, not `%ash`, there is
actually *no* check that the payload in the actual core, type
`p`, nests in the original payload `q.q`.

Moreover, `%elm` arms (defined not with `++` or `slus`, which
means `%ash`, but `+-` or `shep`) are actually *not even
type-checked when compiled*.  They have to generate valid nock,
but all verification is disabled.

Why?  How?  Because there's another way of dealing with a core
whose payload has been modified.  In a sense, it's incredibly
crude and hacky.  But it's perfectly valid, and it lets Hoon
provide the same basic feature set provided by generic features
in other languages, such as Haskell's typeclasses.

Logically, when we use an `%elm`, generic or "wet" arm, we simply
*recompile the entire twig* with the changed payload.  If this
recompilation, which *is* typechecked, succeeds - and if it
*produces exactly the same Nock formula* - we know that we can
use the modified core as if it was the original, without of
course changing the static formula we generated once.

Why does this work out to the same thing as a typeclass?  Because
with a wet core, we are essentially just using the source code of
the core as a giant macro in a sense.  Our only restriction is
that because the Nock formula that actually executes the code
must be the same formula we generated statically for the battery.

In a sense this defines an implicit typeclass: the class of types
that can be passed through the arm, emerging intact and with an
identical formula.  But no declaration of any sort is required.
You could call it "duck typeclassing."

Of course, a lot of caching is required to make this compile with
reasonable efficiency.  But computers these days are pretty fast.

This description of how wet arms work is not quite correct,
though it's the way ancient versions of Hoon worked.  The problem
is that there are some cases in which it's okay if the modified
core generates a different battery - for example, if the original
battery takes branches that are not taken in this specific call.

So we have a function `++mull` which tests whether a twig
compiled with one subject will work with another.  But still,
thinking of the wet call check as a simple comparison of the
compiled code is the best intuitive test.

Again, your best bet as a novice Hoon programmer is to understand
that this is how things like `list` and `map` work, that someone
else who knows Hoon better than you wrote these tools, and that
in general if you want to use generic polymorphism you're
probably making a mistake.  But if you wonder how `list` works -
this is how it works.

###Generic polymorphism action sequences###

Let's deploy this boy!  Here is `++list`:

    ++  list  |*(a=_,* $|(~ [i=a t=(list a)]))

Don't worry.  This is just grievously badass hardcore Hoon - as
irregular as Marseilles street slang.  As a novice Hoon monkey,
you won't be writing `++list` or anything like it, and you can
stick to French as she is spoke in Paris. 

On the other hand, at least this grizzled old baboon has no
trouble parsing the apparent line noise above.  Why so funky?
Why, oh why, `_,*`?  Because for various irrelevant reasons,
the `++list` here is trying as hard as possible to build itself
out of tiles.

For instance, in general in Hoon it is gauche for a gate to use
its core's namespace to recurse back into itself, but tiles do
not expose their own internals to the twigs they contain
(otherwise, obviously, they could not be hygienic).

manipulate the subject such that it's not possible to loop
properly.  So in fact there is no alternative but to use `(list
a)` within `++list` - a normal usage in complex tiles, but only
in complex tiles.

But a normal person wouldn't use tiles to prove a point.  They'd
do it like this - let's use the REPL to build a `list` replacement, 
without one single "obfuscated Hoon trick."

    ~zod/try=> =lust |*(a=$+(* *) |=(b=* ?@(b ~ [i=(a -.b) t=$(b +.b)])))
    ~zod/try=> ((lust ,@) 1 2 ~)
    ~[1 2]
    ~zod/try=> ((list ,@) 1 2 ~)
    ~[1 2]
    ~zod/try=> `(list ,@)``(lust ,@)`[1 2 ~]
    ~[1 2]

Note that `list` and `lust` do the same thing and are perfectly
compatible.  But sadly, `lust` still looks like line noise.
Let's slip into something more comfortable:

    |*  a=$+(* *)
    |=  b=*
    ?@  b  ~
    [i=(a -.b) t=$(b +.b)]

We haven't met `$+` (`buclus`) yet.  `$+(p q)` is a tile for a
gate which accepts `p` and produces `q`.  The spectre of function
signatures once again rears its ugly head - but `$+(p q)` is no
different from `$_(|+(p _q))`.

Otherwise, when we think of a wet gate (`|*`) as a macro, we see
(list ,@) producing

    |=  b=*
    ?@  b  ~
    [i=(,@ -.b) t=$(b +.b)]

This function is easily recognized as a gate accepting a noun and
producing a list of atoms - in short, perfectly formed for an
herbaceous tile.  Pass it some random noun off the Internet, and
it will give you a list of atoms.  In fact, we could define it
on its own as:

    ++  atls
      |=  b=*
      ?@  b  ~
      [i=(,@ -.b) t=$(b +.b)]

and everywhere you write `(list ,@)`, you could write `atls`.
Which would work perfectly despite its self-evident lameness.
Note *in particular* that the inner `b` gate is *not* wet, but
rather dry - once we have done our substitution, there is no need
to get funky.

Let's do some little fun thing with `list` - eg, gluing two lists
together:

    ++  weld
      ~/  %weld
      |*  [a=(list) b=(list)]
      =>  .(a ^.(homo a), b ^.(homo b))
      |-  ^+  b
      ?~  a  b
      [i.a $(a t.a)]

What is this `homo` thing (meaning "homogenize", of course)?
Exactly that - it homogenizes the type of a list, producing its
sample noun unchanged:

```
    ++  homo
      |*  a=(list)
      ^+  =<  $
        |%  +-  $  ?:(_? ~ [i=(snag 0 a) t=$])
        --
      a
```

Here is more dark magic that should clearly be ignored.  But
essentially, to homogenize a list `a`, we are casting `a` to a
deranged pseudo-loop that produces an infinite stream of the
first item in `a`, selected via the `++snag` function:

    ++  snag
      ~/  %snag
      |*  [a=@ b=(list)]
      |-
      ?~  b
        ~|('snag-fail' !!)
      ?:  =(0 a)
        i.b
      $(b t.b, a (dec a))

`++snag` of course selects any item in the list; but if `b` has a
type more complex than a homogeneous list (eg, the type system
might well know the number of items, etc, etc), Hoon is nowhere
near enough to see that the counter is always 0.  So the type
produced by `++snag` is the union of all list elements, which is
precisely the type we want for our homogenized list.

As for `^.` (`ketdot`, `%ktdt`), we can see it in `++open`:

      ++  open
        ^-  twig
        ?-  gen
          [%ktdt *]  [%ktls [%cnhp p.gen q.gen ~] q.gen]
        ==

Ie, `^.(a b)` is `^+((a b) b)` is `^-(_(a b) b)`.

`++weld` prudently casts its product to the type of the base list
`b`.  In future this `^+`  will probably be removed, since we are
perfectly capable of inferring that when you weld `(list a)` and
`(list b)`, you get a `(list ?(a b))`.  But there's old code this
change might confuse.

In short: generic polymorphism is cool but wacky.  Leave it to
the experts, please!

[**Prev**: Tiles](urbit-is-easy-ch9.html)
