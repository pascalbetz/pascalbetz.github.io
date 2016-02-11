---
layout: post
title: Case statement in ruby
categories: ruby benchmark
---
## Case

In other languages also know an ```switch``` statement, ruby uses ```case``` to build
statements that matches against conditions:

{% highlight ruby %}
something = rand(10)
result = case something
when 1 then 'one'
when 2 then 'two'
else 'other'
end
puts result
{% endhighlight %}

Unlike other languages ruby has some neat features that allow you to test against
ranges, multiple values or classes as well:

{% highlight ruby %}
something = ["a string", 1, Date.today, 100, 200].sample
result = case something
when String then 'found the class'
when 0..10 then 'was in the range'
when 100 then 'is an integer'
when 200, 300 then 'multi values'
else 'other'
end
puts result
{% endhighlight%}

You can even use it without passing a value. I think this is extremely ugly but
decide for yourself:

{% highlight ruby %}
result = case
when Date.today.wday == 0 then 'Sleep'
when Date.today.wday == 6 then 'Party'
else 'Work'
end

puts result
{% endhighlight %}

Well, no. Don't decide for yourself. It is ugly. And useless given that we have ```if/else```

### Case equality

So how does this work? How does ruby know when to match? Enter the _case equality operator_:
```===```

Everything that responds to ```===``` can be used in the _when_ clause:

{% highlight ruby %}
class MatchAll
  def ===(other)
    true
  end
end
something = ["a string", 1, Date.today, 100, 200].sample
result = case something
when MatchAll.new then 'match all'
else 'other'
end
puts result # => match all
{% endhighlight %}

### Sugar

As you might know writing

```foo === bar```

is just syntactic sugar for

```foo.===(bar)```

What Ruby does is to call the ```===``` on the object you passed to ```when```, supplying the value you passed to ```case``` as the single argument.

### Procs and Lambdas

In 1.9 ```===``` was added to ```Proc``` as a way to invoke it. This allows it to be the target of a ```when``` clause in ```case``` statement:

{% highlight ruby %}
even = -> (value) { value.even? }
odd = -> (value) { value.odd? }

something = rand(100)
result = case something
when even then 'even'
when odd then 'odd'
else 'what?'
end
puts result
{% endhighlight %}


### Strange world

When you use a ```Proc``` but do not supply a value to ```case``` then strangely ruby just evaluates the first ```when``` clause:

{% highlight ruby %}

callable = -> (value) { puts "was called"; true } # never called
result = case
when callable then 'when'
else 'else'
end
puts result # => will always print "when"
 {% endhighlight %}

I'd have expected an exception.
But you came to the conclusion that a ```case``` statement without a _value_ is ugly anyway, right?


### Performance

What about performance of a ```case``` statement compared to ```if/else```?
Well, if you don't use ```Procs``` then the performance is the same. But since calling ```Procs```
is expensive it might be worth rewriting a ```case``` to ```if/else```:

{% highlight ruby%}
require 'benchmark/ips'
one = -> (value) { value == 1 }
two = -> (value) { value == 2 }
thing = 1

Benchmark.ips do |x|
  x.report('case with inlined lambdas') do
    result = case thing
    when -> (value) { value == 1 } then 'one'
    when -> (value) { value == 2 } then 'two'
    else 'something else'
    end
  end

  x.report('case with lambdas') do
    result = case thing
    when one then 'one'
    when two then 'two'
    else 'something else'
    end
  end

  x.report('if/else') do
    result = if thing == 1
      'one'
    elsif thing == 2
      'two'
    else
      'something else'
    end
  end
end

{% endhighlight %}

Running above benchmark should show you something like:

{% highlight console %}
Calculating -------------------------------------
case with inlined lambdas
                        60.612k i/100ms
   case with lambdas    96.626k i/100ms
             if/else   117.181k i/100ms
-------------------------------------------------
case with inlined lambdas
                          1.203M (± 5.9%) i/s -      6.001M
   case with lambdas      3.259M (± 6.0%) i/s -     16.233M
             if/else      5.182M (±11.8%) i/s -     25.545M
{% endhighlight %}

Of course the runtime of the code depends on the value of ```thing``` adjust it so that first, second or third case is true to see the differences.
But in any case: using ```Procs``` is slower than the corresponding ```if/else```.
