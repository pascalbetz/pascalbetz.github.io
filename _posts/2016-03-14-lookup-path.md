---
layout: post
title: What is the method lookup path in Ruby?
categories: ruby
---


A simple question but hard to answer. Why is that hard? Because Ruby has various ways of defining a method and add it to a class:

* Adding it to the singleton class
* Adding it to the class
* Include a module
* Prepend a module
* Extend a module
* Inherit from superclass

If this sounds complicated to you then that's __because it is__.

So first rule: try to avoid such situations where you have a multitude of classes and modules defining the same method.

If you have more than two definitions of a method then you most likely have bigger problems than knowing about the lookup path.
Also I haven't seen many good uses of adding a method to the singleton class so far.

So how do we go about finding the lookup path? How about a small piece of code that answers this question?

{% highlight ruby %}
module Include
  def call(level)
    puts "#{level} include"
    super(level + 1) rescue nil
  end
end

module Prepend
  def call(level)
    puts "#{level} prepend"
    super(level + 1) rescue nil
  end
end

module Extend
  def call(level)
    puts "#{level} extend"
    super(level + 1) rescue nil
  end
end

class Super
  def call(level)
    puts "#{level} super"
    super(level + 1) rescue nil
  end
end

class Klass < Super
  include Include
  prepend Prepend


  def call(level)
    puts "#{level} klass"
    super(level + 1) rescue nil
  end

end

thing = Klass.new


def thing.call(level)
  puts "#{level} singleton"
  super(level + 1) rescue nil
end

thing.extend(Extend)


thing.call(1)
{% endhighlight %}

What does this code do? It defines a method ```call``` for the six possibilities described above. They all print out some debugging info and then forward the call to ```super```. Since at the end of the hierarchy the ```call``` method is not implemented I added ```rescue nil```. Of course this would only be required for the last element in the hierarchy. But we don't know which one this is, yet. Lets run the code and see the output:

{% highlight console%}
1 singleton
2 extend
3 prepend
4 klass
5 include
6 super
{% endhighlight %}

What if you _extend_ or _include_ or _prepend_ multiple times? The last definition comes first. That is if you have:

{% highlight ruby%}  
class Foo
  include Bar
  include Baz
end
{% endhighlight %}

then the definitions from ```Baz``` will take precedence.

And of course if you do not call ```super``` then none of the other implementations will be called.

So, now that this is solved...let's look at another way this can be determined: ```ancestors```. The documentation says that this "Returns a list of modules included in mod (including mod itself).".
If we extend above code to print the list of ancestors:

{% highlight ruby %}
p thing.class.ancestors
{% endhighlight %}

Then we can see following:

{% highlight console%}
[Prepend, Klass, Include, Super, Object, Kernel, BasicObject]
{% endhighlight %}

This is the order that we determined before but not _complete_. We are missing the methods that have been added to the singleton class. Those can be seen if we check the ```singleton_class```
instead (note that this will create the singleton class if it does not yet exist):

{% highlight ruby %}
p thing.singleton_class.ancestors
{% endhighlight %}

This will print the full list of ancestors:

{% highlight console %}
[#<Class:#<Klass:0x007fe34b225480>>, Extend, Prepend, Klass, Include, Super, Object, Kernel, BasicObject]
{% endhighlight %}

The _#&lt;Class:#&lt;Klass:0x007fe34b225480&gt;&gt;_ is the singleton class. It exists solely for this object:

{% highlight ruby %}
p Klass.new.singleton_class
p Klass.new.singleton_class
{% endhighlight %}

{% highlight console%}
#<Class:#<Klass:0x007fc93913fbf0>>
#<Class:#<Klass:0x007fc93913fad8>>
{% endhighlight %}

This ancestry also shows how Ruby looks up methods. It does not make complicate decisions of where to look first. It just walks up the hierarchy and calls the first matching method it can find. So if the singleton class does not respond to the method, then the prepended modules will be checked until the root is reached.
