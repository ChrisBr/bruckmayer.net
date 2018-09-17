---
layout: post
title:  "How I broke bundler for JRuby"
date:   2018-09-12 00:00:00
categories: JRuby
---

As you might remember from [my last blog article](/jruby/2018/09/04/jruby-hash.html), I recently refactored JRuby's hash tables to use open addressing which improved the performance significantly. While doing so, I broke bundler for JRuby :cry: And as bundler is the most important dependency management tool for the Ruby programming language, this was severe. If you want to know what happened in detail, read on.

<br>

## The issue
[Yasuo Honda](https://github.com/yahonda) reported in issue [#5280](https://github.com/jruby/jruby/issues/5280) that ``bundle install`` of ``rails`` fails since the new hash table implementation. Although the refactoring was already merged into master, luckily there was no new JRuby version released. But how was this issue then discovered? Because with rvm and rbenv it is also possible to install development versions of Ruby. And for instance Rails runs their CI system against these development versions. And after my changes were merged, the CI started to fail. The reported stack trace was the following:

```ruby
NoMethodError: undefined method 'dep' for #<Gem::Dependency:0x7ee955a8>
Did you mean?  dup
                == at /home/travis/.rvm/gems/jruby-head/gems/bundler-1.16.3/lib/bundler/dep_proxy.rb:18
                add? at org/jruby/ext/set/RubySet.java:639
                for at /home/travis/.rvm/gems/jruby-head/gems/bundler-1.16.3/lib/bundler/spec_set.rb:27
                loop at org/jruby/RubyKernel.java:1412
                  ...
```

<br>

## Lets debug
When fetching a value from the hash table, the key is used to quickly find the bucket with the key-value pair. However, as there is only a limited number of buckets, keys with different hash values can get stored in the same bucket (or different keys can have the exact same hash value). If this happens, we need to search this bucket and compare each key with the one we're looking for.

So before my refactoring, JRuby stored every hash element in a ``RubyHashEntry`` object with ``key``, ``value``, ``hash`` attributes. When looking now for a key in a bucket, the algorithm first compared the hash value and only if the hash value is the same it compared the actual key. The code looked like this:

```java
private boolean internalKeyExist(RubyHashEntry entry, int hash, IRubyObject key) {
    return (entry.hash == hash
        && (entry.key == key || (!isComparedByIdentity() && key.eql(entry.key))));
}
```

<br>

However, with my refactoring we removed the ``RubyHashEntry`` objects and stored the keys and values in a single array with a wrapper object. We did this to decrease the number of object allocations which significantly improved the performance. In the beginning we also tried out to store the hash values in this array, but the hash values are primitive integer and the key and value are objects. Now to store everything in the same array, we would need to convert the hash value to an ``Integer`` object (boxing) which was just not efficient. And as collisions should not happen very often anyway, we decided to not store the hash value anymore and just do an equal comparison on the keys instead.

In Ruby small hashes are also very common, e.g. for keyword arguments or parameters in Rails. To support that, hashes which are smaller than eight elements just perform a linear search and iterate over all keys without the overhead of calculating a bucket. In worst case we would need to compare eight keys which is also not very efficient. So we introduced another array just to store the hash values for small hashes. While debugging this issue, I figured out that for instance installing only ``a`` gem (yes, there is really [a gem called a](https://rubygems.org/gems/a)) was successfully but ``rails`` not. The difference was that ``a`` has no dependencies at all but rails has a lot more than eight dependencies. The former was using the small hash specialization while the latter used a normal hash. And although it didn't seem like we changed the external API, it was obvious that these changes caused the issue.

The reported stack trace lead us to the ``SpecSet`` class in bundler. The simplified code there looked like this:

```ruby
handled = Set.new
deps = dependencies.dup
specs = []

loop do
  break unless dep = deps.shift
  next if !handled.add?(dep)
  # do some fancy stuff with the dependency
end
```

<br>

The code iterates basically over all dependencies and adds them to a set if it is not already in the set. If you're wondering what set has to do with an issue in the hash class: Set is using a hash table internally. Following the provided stack trace, it seems like the ``add?`` is causing the problem which first checks if the object exists before it's getting added to the set. The code tries to add ``DepProxy`` object. And as described, we do not compare the hashes anymore but do **always** an equal check on the object now. According to the Ruby documentation, the ``==`` method should get overridden to provide class specific meaning. This is how the ``==`` method looked in bundler:

```ruby
def ==(other)
  return if other.nil?
  dep == other.dep && __platform == other.__platform
end
```

<br>

Do you already see the issue? In the first line, the method is only checking if the ``other`` object is ``nil``. If you pass now anything other than ``nil`` or a ``DepProxy`` object, this method will crash :cry:. For instance, the following code will produce a similar error than the reported issue:

```ruby
d = DepProx.new("foo", "bar")
d == "foobar" # -> NoMethodError: undefined method `dep' for #<String>
```

<br>

## Solution

We now know the issue and can start programming a fix for it. The bundler fix is quite straight forward: We just need to check that the ``other`` object is also an instance of the ``DepProxy`` class and if not return ``false``. The fix ([#6669](https://github.com/bundler/bundler/pull/6669)) looks as simple as this:

```ruby
def ==(other)
  return false if self.class != other.class
  dep == other.dep && __platform == other.__platform
end
```
<br>

But we can not expect that all JRuby users update their bundler gem when we release the new JRuby version. And while working on this fix, I also realized that our implementation was different to the CRuby implementation, one could even claim we changed the public API of hash tables. Although it wouldn't be necessary to store the hash value, it has benefits. First of all we should provide similar behavior than CRuby (which we did not). Furthermore doing always a full object comparison might also not be very efficient. For 'primitive' objects like integers or strings it shouldn't make a difference but for more complex objects it can have quite some impact, so our benchmarks were a best case scenario. Last but not least, when resizing the hash, it was necessary to calculate all hashes again which also had some impact on the performance on insertions of the new implementation. Long story, short: We now also store the hash values in a separate hashes array.

<br>

## Summary

To wrap it up, there are several things we can learn from this. If your software provides a public API, you should always investigate if an internal change might change your public API nevertheless. We might did it to easily in our case and didn't think enough about the implications this 'simple' change might cause.

As mentioned, rvm and rbenv provide development versions of Ruby which are based on the master branches. You should leverage this feature and try out future Ruby versions before they get officially released. Your software will benefit because you will discover deprecations or changes early and the Ruby community will benefit from your feedback. And also don't forget that there are other Ruby implementations than CRuby and test your code also against alternative implementations like JRuby, [mruby](https://github.com/mruby/mruby) or TruffleRuby. 

<img src="/img/imposter.png" alt="Impostor syndrome pie chart" style="max-width: 50%; margin: 0 auto;" class="img-responsive"/>
<p class='text-center'>Impostor syndrome <small>by https://errantscience.com (CC-BY-NC)</small></p>

<br>

Maybe you've already heard of [impostor syndrome](https://en.wikipedia.org/wiki/Impostor_syndrome). If not, it basically means that sometimes you doubt your accomplishments. Many people in the software industry struggle with this issue. And finding bugs like this and reading other peoples code helps me to realize that everyone's code sucks (sometimes). Even the smartest developers of libraries which are used by hundreds of thousands of people (like bundler) sometimes produce 'stupid' code. **Everyone** does it. The only thing which helps is to realize this and to continue learning, teaching and collaborating so that everyone gets a little bit better at the end of the day.

Last but not least I also hope you learned now how to (not) implement the ``==`` method in Ruby. If you're still struggling, I recommend to read [this blog article](http://batsov.com/articles/2011/11/28/ruby-tip-number-1-demystifying-the-difference-between-equals-equals-and-eql/). Or you just use the awesome [dry-equalizer](https://dry-rb.org/gems/dry-equalizer/) library which will do it for you.

<br>
