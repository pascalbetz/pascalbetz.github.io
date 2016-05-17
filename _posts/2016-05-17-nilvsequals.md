---
layout: post
title: nil? vs. ==
categories: ruby
---

Just stumbled over [this question](http://stackoverflow.com/questions/1972266/obj-nil-vs-obj-nil) on Stackoverflow.com.

What is the difference between ```nil?``` and ```== nil```?.

There is no difference.
At least when looking at the observable outcome. And I prefer ```nil?``` because of readability. But that is just a matter of taste.

However there is a slight difference in how this outcome is calculated.


### nil?

```nil?``` is a method defined on ```Object``` and ```NilClass```.

{% highlight ruby %}

# NilClass
rb_true(VALUE obj)
{
  return Qtrue;
}

# Object
static VALUE
rb_false(VALUE obj)
{
return Qfalse;
}

{% endhighlight %}

Unless you mess with this implementation through monkeypatching a ```nil?``` check is a simple method call.
For your own objects (unless they inherit from ```BasicObject``` which does not implement ```nil?```) the implementation stems from
```Object``` and will always return ```false```.

### == nil

```a == b``` is just syntactic sugar for sending the ```==``` message to the left hand side, passing the right hand side as sole argument. This translates to
```a.==(b)``` in the general case. And to ```a.==(nil)``` in our specific case.

```==``` is a method defined on ```BasicObject```:

{% highlight ruby%}
rb_obj_equal(VALUE obj1, VALUE obj2)
{
    if (obj1 == obj2) return Qtrue;
    return Qfalse;
}
{% endhighlight %}

From [the docs](http://ruby-doc.org/core-2.2.0/BasicObject.html#method-i-3D-3D):

> At the Object level, == returns true only if obj and other are the same object.

So this returns ```true``` if two variables are pointing to the same object or if the receiving class has overridden the ```==``` method in another way. And subclasses are meant to override ```==``` to implement class specific behavior.

This also means that the performance depends on the implementation of ```==```. Unlike ```nil?``` which should not be overridden by subclasses.

## Another solution

In ruby everything besides ```nil``` and ```false``` is considered _truthy_. If this interpretation fits your use case, then you can avoid the check and pass the object to the if clause directly:

{% highlight ruby %}
if some_object
  puts "some_object is neither nil nor false"
end
{% endhighlight %}


### Performance


I'd expect ```nil?``` to be as fast as ```== nil```. And because ```==``` might be overridden by subclasses performance should depend upon the receiver.
And omitting the check should be fastest.
Here is a simple benchmark to test my assumptions. As usual, I use the wonderful [benchmark/ips](https://github.com/evanphx/benchmark-ips) gem by Evan Phoenix.

{% highlight ruby %}

require 'benchmark/ips'

class ExpensiveEquals
  def ==(other)
    1000.times {}
    super(other)
  end
end

string = 'something'
number = 123
expensive = ExpensiveEquals.new
notnil = Object.new
isnil = nil

Benchmark.ips do |x|

  x.report('isnil.nil?') do
    if isnil.nil?
    end
  end

  x.report('notnil.nil?') do
    if notnil.nil?
    end
  end

  x.report('string.nil?') do
    if string.nil?
    end
  end

  x.report('number.nil?') do
    if number.nil?
    end
  end

  x.report('expensive.nil?') do
    if expensive.nil?
    end
  end

  x.report('isnil == nil') do
    if isnil == nil
    end
  end

  x.report('notnil == nil') do
    if notnil == nil
    end
  end

  x.report('string == nil') do
    if string == nil
    end
  end

  x.report('number == nil') do
    if number == nil
    end
  end

  x.report('expensive == nil') do
    if expensive == nil
    end
  end

  x.report('nil == isnil') do
    if nil == isnil
    end
  end

  x.report('nil == notnil') do
    if nil == notnil
    end
  end

  x.report('nil == string') do
    if nil == string
    end
  end

  x.report('nil == number') do
    if nil == number
    end
  end

  x.report('nil == expensive') do
    if nil == expensive
    end
  end


  x.report('isnil') do
    if isnil
    end
  end

  x.report('notnil') do
    if notnil
    end
  end

  x.report('string') do
    if string
    end
  end

  x.report('number') do
    if number
    end
  end

  x.report('expensive') do
    if expensive
    end
  end

  x.compare!
end
{% endhighlight %}

{% highlight ruby %}

notnil: 11840655.4 i/s
expensive: 11801608.9 i/s - 1.00x slower
number: 11679119.8 i/s - 1.01x slower
string: 11598016.4 i/s - 1.02x slower
 isnil: 11529034.6 i/s - 1.03x slower
notnil == nil: 10397262.7 i/s - 1.14x slower
nil == expensive: 10319767.0 i/s - 1.15x slower
nil == string: 10188393.9 i/s - 1.16x slower
nil == number: 10167930.4 i/s - 1.16x slower
isnil == nil: 10120560.0 i/s - 1.17x slower
nil == notnil: 10069779.1 i/s - 1.18x slower
nil == isnil: 10055165.2 i/s - 1.18x slower
expensive.nil?:  9970905.5 i/s - 1.19x slower
string.nil?:  9967045.3 i/s - 1.19x slower
number.nil?:  9893974.5 i/s - 1.20x slower
notnil.nil?:  9581974.5 i/s - 1.24x slower
isnil.nil?:  8390963.6 i/s - 1.41x slower
string == nil:  6915330.1 i/s - 1.71x slower
number == nil:  6803969.3 i/s - 1.74x slower
expensive == nil:    25382.6 i/s - 466.49x slower

{% endhighlight %}


I did not expect ```nil?``` to be slower, still looking into this. But one can see that if you go for the ```==``` check, then it should be faster if you do ```nil == other``` instead of ```other == nil```. Usual micro benchmark warnings apply.
