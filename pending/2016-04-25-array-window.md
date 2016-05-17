layout: post
title: Window iterator
categories: ruby
date: 2016-04-28
---

Calculate the moving average and find the peaks in an array of integers. Both tasks where you can make use of a "window iterator".

Let's look at those tasks in detail:

## Finding the peaks

A value ```a[i]``` in an array ```a``` is a peak if ```a[i] > a[i - 1]``` AND ```a[i] > a[i + 1]```

So in the following array the value at index ```3``` is a peak because the values to the left and right are lower.

{% highlight ruby %}
a = [10, 14, 16, 20, 19, 0]
{% endhighlight %}


Here is a simple enumerator that can help with this problem:

{% highlight ruby%}

class WindowEnumerator
  def initialize(array, window_size)
    @array = array
    @window_size = window_size
  end

  def each_with_index
    return enum_for(:each_with_index) if !block_given?
    0.upto(array.size - window_size) do |lower|
      upper = lower + window_size
      window = array[lower...upper]
      yield(window, lower)
    end
  end

  private

  attr_reader :array, :window_size
end

{% endhighlight %}
