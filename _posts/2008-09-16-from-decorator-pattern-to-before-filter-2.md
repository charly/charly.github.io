---
layout: post
title: From Decorator Pattern to Before Filter 2
---

In my previous post <a href="http://ruby.simapse.com/2008/09/from-decorator-pattern-to-before-filter.html">From Decorator Pattern to Before Filter 1</a> we got to the point where we could write this: 

``` ruby
coffee = Coffee.new
coffee.mixin Callback
coffee.before_filter :spoon_required, :cost
coffee.cost
``` 

Now the thing missing is that we're only filtering one method ('cost' in this case) wheras the before_filter in rails filters all methods by default. So how do we do this? 

Quit easily, we just have to list all the instance methods inside the class Coffee (ruby provides this possiblity with "instance_methods(false)").... and... iterate!:

``` ruby
require 'rubygems'
require 'mixology'
 
module Callback
  def filter_with filter, filtered_method
    Callback.method_factory(filter, filtered_method)
  end
  
  def before_filter filter
    methods = self.class.instance_methods(false)
    methods.delete(filter.to_s)
    methods.each do |filtered_method|
      filter_with filter, filtered_method
    end
  end
    
  
  def self.method_factory(filter, filtered_method)
    define_method(filtered_method) do
      send filter
      super
    end
  end
end

class Coffee
  def spoon_required
    puts "get a spoon to steer sugar || creme"
  end
  
  def cost
    puts "3"
  end
  
  def smell
    puts "good"
  end
  
  def color
    puts "black"
  end
end

coffee = Coffee.new
coffee.mixin Callback
coffee.before_filter :spoon_required

# Every call beneath is prepended by :
#  - "get a spoon to steer sugar || creme" 
coffee.cost
coffee.smell
coffee.color
``` 

After that adding options like :except=>"smell" is easy. Dealing with inheritance is probably less obvious. The one main differance we have with the rails version of before_filter is the fact that it is set inside the classes body. Not at object creation, and for a good reason since controllers instances are created under the hood.

Of course the easy path is adding: extend(Creme) and before_filter(:spoon_required) in the Coffee's initialize method, but my question was: "is it possible to make it look like this" : 

``` ruby
class Coffee
  include Creme
  before_filter :spoon_required
  
  def spoon_required
    puts "get spoon"
  end
  
  def cost
    puts "3"
  end
end
``` 


The answer is yes although it's a bit convoluted (it looks like a rail plugin with module InstanceMethod and so on)  AND we still have to overide initialize somewhere which makes it dangerous.

Now one word of explanation: the hole elegance of the Decorator Pattern in ruby relies on the fact we're calling super instead of aliasing twice the method we're decorating (or unbinding and binding). Calling super() is possible because when we extend an object with a module (like in coffee.extend Creme) we're actually including the module in the ghostclass (or metaclass or singleton class) which sits right underneath the 'real' class in the inheritance chain.

(TODO add figure)

So wouldn't be nice if we could mixin a module 'underneath' a class by saying so in the body of the class so we could easily do all that aspect oriented stuff without having to worry about it at instanciation time ?

The great Library <a href="http://facets.rubyforge.org/">Facets</a>, gives it a try with <a href="http://facets.rubyforge.org/doc/api/core/classes/Class.html#M000192">prepend</a> which can be used this way :

``` ruby
require 'rubygems'
require 'facets'

module Creme  
  def cost
    puts "before_filter"
    super
  end
end

class Coffee
  prepend Creme
  
  def cost
    puts "3"
  end
end

coffee = Coffee.new
coffee.cost

#But guess what, it's by overriding new:

# facet library
  def prepend( aspect )
    _new      = method(:new)
    _allocate = method(:allocate)
    (class << self; self; end).class_eval do
      define_method(:new) do |*args|
        o = _new.call(*args)
        o.extend aspect
        o
      end
      define_method(:allocate) do |*args|
        o = _allocate.call(*args)
        o.extend aspect
        o
      end
    end
  end
  ``` 

Comments welcome !
