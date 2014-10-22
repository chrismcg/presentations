slidenumbers: true

# [fit] What's new in Ruby since 2.0.0

### Chris McGrath
### @chrismcg

---

# Speed

![inline](http://discuss.samsaffron.com/uploads/default/_optimized/122/34d/72ad939a71_690x386.png)

---

# Garbage Collector

---

# Mark and Sweep in Ruby 1.8

* Stop everything Ruby is doing
* Go through and mark all objects that have a reference to them
* Sweep all the objects that aren’t marked

---

# 1.8

![inline](http://tmm1.net/ruby21-rgengc/ruby18.png)

#### (Images from [Aman Gupta's blog](http://tmm1.net/ruby21-rgengc/))

---

# 1.9.3

![inline](http://tmm1.net/ruby21-rgengc/ruby19.png)

Ruby 1.9.3 made the sweep phase "lazy" so the pause is just the mark phase

#### (Images from [Aman Gupta's blog](http://tmm1.net/ruby21-rgengc/))

---

# Ruby 2.0
# **Copy On Write**

![inline](http://tmm1.net/ruby21-rgengc/ruby20.png)

#### (Images from [Aman Gupta's blog](http://tmm1.net/ruby21-rgengc/))

---

# Ruby 2.1
# **Generational GC**

![inline](http://tmm1.net/ruby21-rgengc/ruby21.png)

#### (Images from [Aman Gupta's blog](http://tmm1.net/ruby21-rgengc/))

---

# Ruby 2.2
## **Symbols**
## **Incremental GC**

---

# GC Tuning

---

# Refinements

* Extend classes (monkeypatch) locally instead of globally
* Introduced as experimental in 2.0
* Reduced in scope and complexity since initial idea
* Not experimental in 2.1

---

# Refinements

```ruby
module Stats
  refine Array do
    def mean
      inject(0) { |sum, x| sum + x } / size
    end
  end
end
```

---

# Refinements

```ruby
a1 = [1, 2, 3, 4]
a1.mean rescue "no mean method" # => "no mean method"

using Stats

a2 = [2, 4, 6, 8]
a2.mean # => 5
a1.mean # => 2
```

---

# Refinements

```ruby
class Analyzer
  using Stats

  def initialize(array)
    @array = array
  end

  def summary
    { mean: @array.mean }
  end
end
```

---

# Refinements

```ruby
a1 = [1, 2, 3, 4]
analyzer = Analyzer.new(a1)

a1.mean rescue "no mean method" # => "no mean method"
analyzer.summary # => {:mean=>2}
```

---

# Refinements - Lexically scoped

* `using` to end of file
* `using` to end of module
* `using` to end of string being evaled
* Don't apply when re-opening classes

---

# Refinements

```ruby
class Analyzer
  def short_summary
    { m: @array.mean }
  end
end

a1 = [1, 2, 3, 4]
analyzer = Analyzer.new(a1)
analyzer.summary # => {:mean=>2}
analyzer.short_summary rescue "no mean method" # => "no mean method"
```

---

# Refinements

* Seem like a great fit for gem authors
* Not in JRuby yet ([but coming](https://github.com/jruby/jruby/issues/1062))
* Not in Rubinius
* What should something like ActiveSupport do?
* Haven't seen any benchmarks yet (but not expecting them to be slow)

---

# Keyword Arguments

* Introduced in Ruby 2.0
* Improved in 2.1
* Backwards compatible with hash syntax
* Better error messages for missing / wrong arguments

---

# "Fake" Keyword Arguments in Ruby 1.8/1.9

```ruby
def render(content, opts = {})
  width = opts[:width]
  height = opts[:height]
  ...
end

render "* foo"
```

---

# Need some defaults

```ruby
def render(content, opts = {})
  width = opts[:width] || 1920
  height = opts[:height] || 1080
  ...
end

render "* foo"
```

---

# :full means use width or height of page

```ruby
def render(content, opts = {})
  width = opts.fetch(:width) { 1920 }
  width = @page.width if width == :full
  height = opts.fetch(:height) { 1080 }
  height = @page.height if height == :full
  ...
end

render "* foo", :width => :full
render "* bar", :hieght => 786 # doesn't pick up on typo
```

---

# With Ruby 2.x keyword arguments


```ruby
def render(content, width: 1920, height: 1080)
  width = @page.width if width == :full
  height = @page.height if height == :full
  ...
end

render "* foo"
render "* bar", hieght: 786 # => 
# ~> -:7:in `<main>': unknown keyword: hieght (ArgumentError)
```

---

# If the user must specify width

```ruby
def render(content, width:, height: 1080)
  width = @page.width if width == :full
  height = @page.height if height == :full
end

render "* foo"
# ~> -:6:in `<main>': missing keyword: width (ArgumentError)
```
---

# Benefits

* Better errors and programmer doesn't have to manually check arguments
* Easier to add named arguments to those methods you pass four numbers into
* Named arguments mean you can change argument position without needing to refactor
* Named arguments make it clear where method call is what the arguments are

---

# Drawbacks

* More typing if you're making lots of calls with named arguments
* ...

(Check out RubyTapas episodes 186 and 187 to learn how to handle extra arguments with the `**` operator and just how flexible they are.)

---

# Module#prepend (:heart:)

* `include` puts module after current class in method lookup chain
* `prepend` puts it before
* Removes need for `alias_method_chain`, breaking `super` etc.

---

# Module#include / prepend

```ruby
module Stats
  def mean
    puts "Stats#mean"
    super rescue puts "Stats#mean no superclass mean method"
  end
end
```

---

# Module#include / prepend

```ruby
class Analyzer
  include Stats

  def mean
    puts "Analyzer#mean"
    super
  end
end

Analyzer.ancestors # => [Analyzer, Stats, Object, Kernel, BasicObject]
Analyzer.new.mean # => nil
# >> Analyzer#mean
# >> Stats#mean
# >> Stats#mean no superclass mean method
```

---

# Module#include / prepend

```ruby
class Analyzer2
  prepend Stats

  def mean
    puts "Analyzer2#mean"
    super rescue puts "Analyzer2#mean no superclass mean method"
  end
end

Analyzer2.ancestors # => [Stats, Analyzer2, Object, Kernel, BasicObject]
Analyzer2.new.mean # => nil
# >> Stats#mean
# >> Analyzer2#mean
# >> Analyzer2#mean no superclass mean method
```

---

# Cache decorator

```ruby
class Analyzer
  def expensive_method
    puts "thinking really hard!"
    # do lots of calculations
    42
  end
end
```

---

# Cache decorator

```ruby
module AnalyzerCacher
  def expensive_method
    @expensive_method_result ||= super
  end
end

class Analyzer
  prepend AnalyzerCacher
end
```

---

# Cache decorator

```ruby
a = Analyzer.new
a.expensive_method # => 42
# >> thinking really hard!
a.expensive_method # => 42
a.expensive_method # => 42
```

(This could of course be generalized with meta programming)

---

# Method Cache

* Pre Ruby 2.1 any change to any class would clear all the method caches
* Performance hit, especially if you're doing something like DCI
* Post Ruby 2.1 just the cache for the class and it's subclasses is changed
* Can use [Busted](https://github.com/simeonwillbanks/busted) tool to see where cache is busted on 2.1+

---

# Other optimizations

* [Kernel#require optimized to make rails loading fast](https://www.ruby-lang.org/en/news/2013/02/24/ruby-2-0-0-p0-is-released/)
* Backtrace generation optimized
* [Hash#shift performance increased](https://bugs.ruby-lang.org/issues/8312), so along with preserving insertion order can [make a nice little LRU Cache with it](http://globaldev.co.uk/2014/05/ruby-2-1-in-detail/#hashshift-much-faster)

---

# `__dir__`

(instead of `File.dirname(__FILE__)`)

```ruby
  require __dir__ + '/foo'
  File.read(__dir__ + '/.config')
```

**(note lower case)**

---

# %i and %I

Create arrays of symbols

```ruby
[1] pry(main)> n = 10
=> 10
[2] pry(main)> %i(foo bar#{n})
=> [:foo, :"bar\#{n}"]
[3] pry(main)> %I(foo bar#{n})
=> [:foo, :bar10]
```
---

# def returns method name as symbol

## (Ruby 2.1+)

```ruby
method = def foo
  puts "bar"
end

method # => :foo
```

---

# def returns method name as symbol

## (Ruby 2.1+)

Handy for metaprogramming and just making one method private

```ruby
  private def foo
    #...
  end
```

---

# Exception#cause (2.1)

```ruby
class Downloader
  class DownloadException < StandardError; end

  def download!
    begin
      raise "boom!"
    rescue RuntimeError
      raise DownloadException
    end
  end
end
```

---

# Exception#cause (2.1)

```ruby
begin
  Downloader.new.download!
rescue => e
  e.to_s # => "Downloader::DownloadException"
  e.cause # => #<RuntimeError: boom!>
end
```
---

# Lazy enumerators

`#lazy` can be called on any enumerable. Can do FP style stuff like:

```ruby
[25] pry(main)> [1, 2, 3, 4].lazy.cycle.take(20)
=> #<Enumerator::Lazy: ...>
[26] pry(main)> [1, 2, 3, 4].lazy.cycle.take(20).to_a
=> [1, 2, 3, 4, 1, 2, 3, 4, 1, 2, 3, 4, 1, 2, 3, 4, 1, 2, 3, 4]
```

---

# Lazy enumerators

Find first three four letter words in `/usr/share/dict/words` without loading whole file into memory:

```ruby
File.open('/usr/share/dict/words').lazy.
                                   map(&:chomp).
                                   select { |w| w.length == 4 }.
                                   take(3).
                                   force
=> ["Aani", "Aaru", "abac"]
```

---

# Module.const_get


Knows about namespaces since 2.0

```ruby
module Foo
  class Bar
  end
end

Object.const_get("Foo::Bar") # => Foo::Bar
```
---

# Kernel#itself (2.2)

```ruby
[1, 2, 3, 4, 1, 5, 2, 3, 5, 6].group_by(&:itself)
=> {1=>[1, 1], 2=>[2, 2], 3=>[3, 3], 4=>[4], 5=>[5, 5], 6=>[6]}
```

(This is my favourite new method name)

---

# warn accepts multiple arguments

(warn == puts for stderr)

```ruby
pry(main)> warn "warning 1", "warning 2"
warning 1
warning 2
```

---

# Thread and Fibre stack sizes

* Fibre stack size hardcoded at 4K in 1.9.x
* Now settable via ENV variable in 2.x

---

# Thread and Fibre stack sizes

* RUBY\_THREAD\_VM\_STACK\_SIZE: vm stack size used at thread creation. default: 128KB (32bit CPU) or 256KB (64bit CPU).
* RUBY\_THREAD\_MACHINE\_STACK\_SIZE: machine stack size used at thread creation. default: 512KB or 1024KB.
* RUBY\_FIBER\_VM\_STACK\_SIZE: vm stack size used at fiber creation. default: 64KB or 128KB.
* RUBY\_FIBER\_MACHINE\_STACK\_SIZE: machine stack size used at fiber creation. default: 256KB or 256KB.

---

# Thanks!!

* [Ruby 2.0 NEWS](https://github.com/ruby/ruby/blob/ruby_2_0_0/NEWS)
* [Ruby 2.1 NEWS](https://github.com/ruby/ruby/blob/ruby_2_1/NEWS)
* [Ruby 2.2 (trunk) NEWS](https://github.com/ruby/ruby/blob/trunk/NEWS)
* [Ruby 2.0.0 in detail](http://globaldev.co.uk/2013/03/ruby-2-0-0-in-detail/)
* [Ruby 2.1 in detail](http://globaldev.co.uk/2014/05/ruby-2-1-in-detail/)
* [Ruby 2 Keyword Arguments](http://robots.thoughtbot.com/ruby-2-keyword-arguments)

---

# More links

* [Ruby 2.1 Garbage Collection: ready for production](http://samsaffron.com/archive/2014/04/08/ruby-2-1-garbage-collection-ready-for-production)
* [Demystifying the Ruby GC](http://samsaffron.com/archive/2013/11/22/demystifying-the-ruby-gc)
* [Visualizing Garbage Collection in Ruby and Python](http://patshaughnessy.net/2013/10/24/visualizing-garbage-collection-in-ruby-and-python)
* [Introducing Incremental GC algorithm](https://bugs.ruby-lang.org/issues/10137)
* [Ruby 2.1: Out-of-Band GC](http://tmm1.net/ruby21-oobgc/)
* [MRI's Method Caches](http://jamesgolick.com/2013/4/14/mris-method-caches.html)
