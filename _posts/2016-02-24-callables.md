---
layout: post
title: Callables in Ruby
categories: ruby
---

In Ruby there are several objects that respond to ```call```. I usually refer to them as _callables_:

* Proc
* Lambda
* Method

There are various ways that you can invoke those _callables_:

* ```.call()```
* ```[]```
* ```.()```
* ```.===```

I've talked about the case equality operator ```===``` over [here](/ruby/benchmark/2016/02/10/ruby-case-when/).
But what about the other ones? IMHO you should stick to ```.call()``` because it does not require knowledge about
a special syntax.
