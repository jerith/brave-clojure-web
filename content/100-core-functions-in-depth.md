--- 
title: Core Functions in Depth
link_title: Core Functions in Depth
---

# Core Functions in Depth

If you're a huge *Vampire Diaries* fan like me then you'll remember
the episode where the female teenage love interest, Elena, really
starts to question her mysterious crush's behavior: "Why did he
instantly vanish without a trace, which is impossible, when I scraped
my knee?" and "How come his face turned into a horrible mask of death
when I nicked my finger?" and so on.

You might be asking yourself similar questions if you've started
playing with Clojure's core functions. "Why did `map` return what
looks like a list when I gave it a vector?" and "How come my map got
turned into a list of vectors?" and so on. (With Clojure, though, you
at least don't have to contemplate the profound existential terror of
being a seventeen-year-old for eternity.)

In this chapter, you'll learn about Clojure's deep, dark,
bloodthirsty, supernatur&ndash; \**cough*\* I mean, in this
chapter, you'll learn a couple underlying concepts which are key to
understanding Clojure's core functions. You'll also see usage examples
of the functions you'll be reaching for the most. By the end, you'll
understand:

* Programming to abstractions
* The sequence abstraction
* Lazy sequences
* The collection abstraction
* Often-used functions

## Programming to Abstractions

Clojure emphasizes *programming to abstractions*. Since this phrase
doesn't mean anything without examples and explanation, let's explore
it a bit.

In general, programming to abstractions gives us power by letting us
use libraries of functions on a data structure regardless of that data
structure's implementation.

Take the sequence abstraction. First, some terminology: if a data
structure can be treated as a sequence, then we call that sequence a
"seq" because it's shorter and hipper, and that's how it's referred to
everywhere else.

If a data structure takes part in the sequence abstraction then it can
make use of the extensive seq library, which includes such superstar
functions as `map`, `reduce`, `filter`, `distinct`, `group-by`, and
dozens more. So, if you can treat a data structure as a seq then you
get oodles of functionality for free. Vectors, maps, lists, and sets
can all be treated as seqs.

A data structure can take part in the sequence abstraction if it's
possible to treat it as a *logical* list. "Logical" is emphasized in
"logical list" to distinguish it from the concrete implementation of a
linked list. Let's implement a linked list in Javascript so that we
can fully appreciate the distinction between the seq (logical list)
abstraction and the concrete implementation of a linked list.

In a linked list, nodes are linked in a linear sequence. Here's how
you might create one in Javascript:

```javascript
// "next" is null because this is the last node in the list
var node3 = {
  value: "last",
  next: null
};

// "next" points to node3 - that's the "link" in "linked list"
var node2 = {
  value: "middle",
  next: node3
};

var node1 = {
  value: "first",
  next: node2
};
```

Graphically, you could represent this list like this:

![Linked list](/images/core-functions-in-depth/linked-list.png)

There are three core functions that you can perform on a linked list:
`first`, `rest`, and `cons`. Once those are implemented, you can
implement `map`, `reduce`, `filter`, and so on top of them. Here's how
we would implement and use `first`, `rest`, and `cons` with our
Javascript example:

```javascript
// Note that the parameter is named "node" here. This might be
// confusing - you might think, "Ain't I getting the first element of
// a *list*? Well, you operate on the elements of a list one node at
// a time!
var first = function(node) {
  return node.value;
};

var rest = function(node) {
  return node.next;
};

// Append a new node to the beginning of the list
var cons = function(newValue, list) {
  return {
    value: newValue,
    next: list
  };
};

first(node1);
// => "first"

first(rest(node1));
// => "middle"

first(rest(rest(node1)));
// => "last"

first(cons("new first", node1));
// "new first"

first(rest(cons("new first", node1)));
// => "first"
```

As noted above, you can implement `map` in terms of `first`, `rest`,
and `cons`:

```javascript
var map = function (list, transform) {
  if (list === null) {
    return null;
  } else {
    return cons(transform(first(list)), map(rest(list), transform));
  }
}

// Let's see it in action:
first(
  map(node1, function (val) { return val + " mapped!"})
);

// => "first mapped!"
```

So here's the cool thing &mdash; since `map` is implemented completely
in terms of `cons`, `first`, and `rest`, you could actually pass it
any data structure and it would work as long as `cons`, `first`, and
`rest` work on that data structure. Those three functions are the
abstraction's *interface*. Here's how they might work for an array:

```
var first = function (array) {
  return array[0];
}

var rest = function (array) {
  var sliced = array.slice(1, array.length);
  if (sliced.length == 0) {
    return null;
  } else {
    return sliced;
  }
}

var cons = function (newValue, array) {
  return [newValue].concat(array);
}


var list = ["Transylvania", "Forks, WA"];
map(list, function (val) { return val + " mapped!"})
// => ["Transylvania mapped!", "Forks, WA mapped!"]
```

So, if you can implement `first`, `rest`, and `cons`, then you get
`map` for free, along with the aforementioned oodles of other
functions.

The takeaway here is that it's more powerful to focus on what we can
*do* with a data structure than on how it's implemented. Those
implementations rarely matter in and of themselves. They're only a
means to an end. We ultimately only care about what we do with them,
and by programming to abstractions we're able to do more with them.

By the way: Javascript doesn't provide an easy means of defining
different implementations of a function based on the type(s) of
argument(s), but Clojure does. It's pretty cool! But for now we won't
cover it. For now, just trust that Clojure makes it super easy.

## The Sequence Abstraction

Now that we understand the general approach of programming to
abstractions we can answer some of the questions we posed at the
beginning of the chapter:

* How come my map got turned into a list of vectors?
* Why did `map` return what looks like a list when I gave it a vector?
* Isn't Damon, my crush's hunky and troubled older brother, making
  lots of creepy puns involving consuming my blood as food? What's up
  with that?

### Seq Functions Convert Data Structures to Seqs

In the Javascript examples we gave above, we indicated one way that
you could allow a data structure to participate in the seq
abstraction: make `first`, `rest`, and `cons` work on that data
structure.

But! There's another way: explicitly convert the data structure to a
seq. So, instead of extending `first` etc. to work on your data
structure, you provide some way for your data structure to work with
those functions as they're currently implemented.

Clojure functions often use the `seq` function to
do this. From the [Clojure documentation](http://clojure.org/sequences):

> Clojure uses the ISeq interface to allow many data structures to
> provide access to their elements as sequences. The seq function
> yields an implementation of ISeq appropriate to the collection.

If those details don't really make sense, don't worry about it too
much. The important thing to know is that many functions will call
`seq` on a collection argument before doing anything else. The `map`
function does this, for example:

```clojure
; identity returns whatever was passed to it
(identity "Stefan Salvatore from Vampire Diaries")
; => "Stefan Salvatore from Vampire Diaries"

;; Map returns a new sequence consisting of the result of calling
;; "identity" on each member of the sequence it was given
(map identity {:name "Bill Compton" :occupation "Dead mopey guy"})
; => ([:name "Bill Compton"] [:occupation "Dead mopey guy"])
```

Since we know that `map` calls `seq` on its collection arguments, and
since `identity` returns whatever value was passed to it, we can
deduce that the `seq` function converts a map data structure into a
sequence of vectors, each of which is a key/value pair:

```clojure
(seq {:name "Bill Compton" :occupation "Dead mopey guy"})
; => ([:name "Bill Compton"] [:occupation "Dead mopey guy"])
```

I wanted to point out this example in particular because it might be
surprising and confusing. It was for me when I first started Clojure.
Knowing these underlying mechanisms will save you from the kind of
frustration and general mopiness of the kind of often seen among male
vampires trying to retain their humanity.

### Seq Functions Sometimes Return Lazy Seqs

Why do `map` and other functions return what looks like a list?

```clojure
(map identity {:name "Van Helsing" :occupation "Living angry guy"})
; => ([:name "Van Helsing"] [:occupation "Living angry guy"])
```

As we saw in the last section, `map` first calls `seq` on the
collection you pass to it. So that's part of the answer &mdash; the
functions which operate on seqs call `seq` on their arguments and
don't bother to convert them back.

But that's not the whole story. Some functions, like `map`, return
a "lazy seq". A lazy seq is a seq whose members aren't computed until
you try to access them. Computing a seq's members is called
"realizing" the seq.

There are two reasons for lazy seqs. First, they're more efficient
because they don't do unnecessary computations. For example, pretend
that you're part of a modern-day task force whose purpose is to
identify vampires. You know that there is a single vampire out a group
of one million suspects. Your boss gives you a list of one million
social security numbers and shouts, "Get it done, McFishwich!"

Here's one way that you could do that:

```clojure
(defn vampire?
  "Returns boolean"
  [record]
  (instant-computation record))

(defn vampire-related-details
  "Looks up vampire related details in super sophisticated database"
  [social-security-number]
  (ten-second-computation social-security-number))


;; To understand the function below, you need to understand
;; drop-while:
(drop-while neg? [-1 -2 0 1 2])
; => (-1 -2)

;; The strategy here is to keep dropping members of a sequence if
;; we know they're not a vampire. Then the first member of the
;; remaining sequence is a vampire
(defn identify-vampire
  [social-security-numbers]
  (first (drop-while #(not (vampire? %))
                     (map vampire-related-details
                          social-security-numbers))))
```

As you can see, it takes 10 seconds to pull up each potential
vampire's details. A non-lazy implementation of this would apply
`vampire-related-details` to every social security number before
passing the result to `drop-while`. This would take 116 days (10
million seconds), and half your city could be dead by then!

Instead, since `map` returns a lazy seq, `vampire-related-details`
doesn't get called until it's actually needed. At least, it's useful
to think of it that way. Sometimes lazy seqs are chunked, meaning that
they realize 32 members at a time:

```clojure
(def identities
  [{:alias "Batman" :real "Bruce Wayne"}
   {:alias "Spiderman" :real "Peter Parker"}
   {:alias "Santa" :real "Your mom"}
   {:alias "Easter Bunny" :real "Your dad"}
   {:alias "alias 5", :real "real 5"}
   ; ... Just pretend that there are actually maps here for 6-30
   {:alias "alias 31", :real "real 31"}
   {:alias "alias 32", :real "real 32"}
   {:alias "alias 33", :real "real 33"}
   {:alias "alias 33", :real "real 34"}])

(defn snitch
  "Announce real identity to the world"
  [identity]
  (println (:real identity))
  (:real identity))

(map snitch [{:alias "Batman" :real "Bruce Wayne}])

(def revealed-identities)
(first revealed-identities)
;; The following gets printed
Bruce Wayne
Peter Parker
Your mom
Your dad
real 5
... (real 6-30 would actually be printed)
real 31
real 32
```

Notice that `real 33` and `real 34` were *not* printed &mdash; only 32
lines were printed. Clojure doesn't realize a lazy list until you try
to read a value from it, and then it usually realizes 32 members at a
time. This is done for the sake of efficiency.

Note, also, that Clojure caches the values of the lazy seq. It doesn't
have to re-compute them when you try to access them again. Continuing
where we left off from the previous example:

```clojure
;; Since the lazy seq has already realized the first member, it
;; doesn't run the snitch function again and nothing gets printed
(first revealed-identities)
; => "Bruce Wayne"
```

Sometimes you need to realize the entire seq without bothering with
trying to take every member. Usually the only reason you'd want to do
this if you want to produce side effects &mdash; for example, if you
want to print every single real identity in the example above. In that
case, you use `doall` on the seq. The purpose of `doall` is to realize
the seq.

And that covers lazy seqs! Now you'll know what the heck is going on
next time you call `map` on a map!

### About Those Creepy Puns

He's a vampire, dammit! Why can't you see that!?!?

## The Collection Abstraction