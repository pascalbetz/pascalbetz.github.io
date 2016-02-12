---
layout: post
title: XML Anyone?
categories: ruby benchmark
---

Recently we had to deal with importing about 120Mb of XML data. Daily. It was split up into files of around 6Mb each.
Those files contain information about employees, with around 60 attributes per employee and around 5000 employees per file.
We need to read the files and create/update a record in the DB with the values from the XML file.

An XML file looked something like this:

{% highlight xml %}
<root>
  <record>
    <attr1>A name</attr1>
    <attr2>A street</attr2>
    ...
  </record>
  <record>
    ...
  </record>
</root>
{% endhighlight %}

Basically a CSV file in XML format...

Processing these files took ~16h. Whaaat?
I don't need to mention, that there was a bug in our code, do I (we accessed the nodes through NodeSet, see benchmark samples)? By fixing it we were able to cut down the time to about 20min. Not bad.
Looked further into the code with [rubyprof](https://github.com/ruby-prof/ruby-prof), I found that still a big part of time was spent in XML parsing. So I set out to build a benchmark __for our use case__. I was wondering if we could improve performance by replacing Nokogiri with one of the alternatives:

* [OGA](https://github.com/YorickPeterse/oga)
* [OX](https://github.com/ohler55/ox)
* [REXML](http://www.germane-software.com/software/rexml/)

As Mike Perham described in his [Kill Your Dependencies](http://www.mikeperham.com/2016/02/09/kill-your-dependencies/) article, relying on
fewer libraries (and using STDLIB instead) is better. Since Rails has a dependency I always resort to Nokogiri for XML parsing. Even though REXML works  and in cases where performance does not matter could be used instead of adding another dependency to your lib.

A simplified benchmark can be found [here](https://github.com/pascalbetz/benchmarks/blob/master/xml_parsing.rb). I was suprised that OGA (pure ruby) was that much faster than Nokogiri. OX tops that and is ~12 times faster than Nokogiri.

{% highlight console %}
Nokogiri: NodeSet    280.336  (±17.1%) i/s -      1.368k
Nokogiri: Element      2.428k (± 7.4%) i/s -     12.312k
               OX     29.533k (± 3.6%) i/s -    148.188k
              OGA     18.220k (± 6.7%) i/s -     92.750k
            REXML    771.185  (± 9.2%) i/s -      3.871k
{% endhighlight %}

So it looks like I could improve the performance once more by switching to OGA (preferred, since pure ruby) or OX. Next step is to test the performance in real life with real input under real conditions.

Note: It looks like OX does return nil for empty bodies whereas Nokogiri returns empty string.
