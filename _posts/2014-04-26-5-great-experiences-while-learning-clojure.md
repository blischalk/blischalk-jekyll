---
layout: post
title: "5 Great Experiences While Learning Clojure"
description: "Lately I have been exploring functional programming and have been spending a lot of time working with Clojure.  As I spend more and more time with it I find that there are particular aspects of the language that I find beautiful and enjoyable.  Some of these aspects come from the fact that Clojure is a LISP dialect of programming language.  Other aspects come from the fact that Clojure is a functional language.  Here are 5 great experiences I have had while learning the language:"
redirect_from:
  - /posts/16-5-great-experiences-while-learning-clojure
tags: [clojure, functional, programming]
---
Lately I have been exploring functional programming and have been spending a lot of time working with Clojure.  As I spend more and more time with it I find that there are particular aspects of the language that I find beautiful and enjoyable.  Some of these aspects come from the fact that Clojure is a LISP dialect of programming language.  Other aspects come from the fact that Clojure is a functional language.  Here are 5 great experiences I have had while learning the language:

### Everything Revolves Around Lists

Since I had previously been spending most of my time in Ruby I was sort of off put by all the parenthesis I saw when I first encountered Clojure.  One of the things I loved most about Ruby when I started learning it was that parenthesiss and semi-colons where optional.  Coming from a PHP and Javascript background prior to Ruby, this was very delightful.  Now I was going back to these parens and to top it off they were often nested inside each other.  You would end up with code that would look like the following:

{% highlight clojure %}
(fn my-grouping [coll]
  (loop [coll coll state {}]
    (if (empty? coll) state
      (let [h (first coll) t (rest coll)]
        (if-let [counter (get state h)]
          (recur t (conj state [h (inc counter)]))
          (recur t (conj state [h 1])))))))
{% endhighlight %}

Look at that stack of trailing parens at the end.  There are seven of them there!  Is this insanity?  Well as it turned out, after working with the language I found out that this little nuisance was actually a small trade off for the simplicity that a Lisp dialect language can provide.  As unwieldy as that code may seem, it is actually simpler than other languages in the sense that there is a very defined pattern to how code is written in the language.  You might notice in the above code for instance that the if statement on the third line looks similar to all the other lines.  "if" is the first item in that particular list just as "empty?" is the first item in the list following the if.  This may seem confusing but the moral here is that control statements e.g if / else sort of constructs are written in the same manor as executing a function.  They are not executed in a separate type of construct as opposed to a lot of other programming languages.

### Simple Made Easy

A co-worker of mine saw my interest in functional programming and Clojure and thought I might enjoy a talk given by its creator Rich Hickey.  The talk was titled "Simple Made Easy":http://www.infoq.com/presentations/Simple-Made-Easy and talked about how a problem you are trying to solve has its own inherent complexity and then most object-oriented programmers proceed to "Complect" the problem by adding all sorts of abstractions around the problem at hand.  Rich Hickey says if we just treat data as data we can keep our code much easier to reason about.  This was an extremely profound light bulb moment for me as I work as an object oriented programmer and spend a lot of my time building these sort of abstractions he was criticizing.  Could he be right?  Could I / we actually be "Complecting" things by building abstractions in an object oriented fashion?  I haven't worked with Clojure and functional programming long enough to determine how correct Rich Hickey's theory is but I must say that he definitely made an impact on my thinking.

### No Side Effects

Another profound moment for me was working on "4clojure.org":http://www.4clojure.com/ problems and wracking my brain to find ways to solve these exercises in ways that don't mutate variables (don't have side effects).  These exercises were game changers for me because I hadn't previously considered that problems such as these could be solved without keeping counts, references to things, or altering things.  Most of the problems are solved through combinations of functions (hence the functional programming aspect of the language) and often recursion.  At first it was sort of like riding a unicycle when you've only ever ridden a bike.  After a while though my brain started to think differently about things and I actually feel that the experience has made me a better programmer for it.

### Data Transformation

Analyzing problems as a series of data transformations has been very interesting.  Because Clojure is a functional language the core mechanism for manipulating data comes through chaining various small focused functions together which combine to output a desired result.  This is contrary to the object oriented paradigm which models data through objects and message passing.  Some people have commented that functional programming focuses on the verbs while object oriented programming focuses on the nouns.  There is actually a great article about this called [The Kingdom of Nouns](http://steve-yegge.blogspot.com/2006/03/execution-in-kingdom-of-nouns.html)  As my experience with Clojure and functional programming increase it will be interesting to re-visit Rich Hickey's theory of OOP making the complex even more complex and see if viewing all problems as just series of data transformations is the true path to enlightenment.

### Code as Data

I haven't had a whole lot of experience with the notion of "Code as Data" (Homoiconicity) yet but from what I have read and what little experience I do have with it I can see how powerful it can be.  One of the features of Ruby that I love is its ability for metaprogramming.  With metaprogramming in Ruby you have the ability to add methods to classes and objects at runtime or react when a method can't be found.  Through the idea of homoiconicity Clojure gains metaprogramming capabilities as well.  Clojure's macros allow you to actually extend the language itself.  The canonical example that is normally given is the control flow "unless" construct.  Lets say you like this construct found in the Ruby language.  You want it in Clojure too?  No problem, you just create a macro composing "if" and "not" and you have it.  Don't like how Clojure uses Prefix notation?  You can create a macro for translate that for you too.  I have yet to really dig in to Clojures macros but feel that this could potentially be one of Clojures greatest strengths.

All in all my experiences with Clojure have been very enjoyable thus far.  My time with Clojure has introduced me to concepts and techniques that I had never experienced before.  I strongly believe that moving outside ones general comfort zone can be greatly rewarding experience and my time with Clojure has been a great example of that.  I am so interested in functional programming and different ways of approaching programming challenges now that I am strongly considering taking the time to learn Haskell.  I have heard that Haskell can push you to think in even more new and exciting ways that you may have never encountered in your programming experience before.  Maybe there will be a 5 Great Experiences with Haskell post in my future.



