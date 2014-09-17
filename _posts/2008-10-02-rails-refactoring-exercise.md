---
layout: post
title: Rails refactoring exercise
---

I started my blog after reading jay field's post on the <a href="http://blog.jayfields.com/2008/07/ruby-underuse-of-modules.html">underuse of modules</a> and <a href="http://www.designpatternsinruby.com/">'design pattern in ruby'</a> book. Mainly because i thought it would be interesting, as a learning process, to explore the silent power of modules.

So knowing Rails was doing a heavy use of alias_method_chain as it's AOP trick, i did a shallow dive in ActiveRecord's code to see how it could be transformed in the MultiInheritance-like-style of modules.

``` ruby
require "rubygems"
require "uninclude"

module Core
  def save; "saved" end
end

module Dirty
  def save; "clean + " + super end
end

module Transaction
  def save; "transaction + " + super end
end

module Validation
  def save;  "valid + " + super end
end

class Base
  include Core
  include Dirty
  include Transaction
  include Validation
end

# PLUGIN adding more validation
module MyValidation
  def save
    "more validation + " + super
  end
end

# PLUGIN overiding Transaction
module MyTransaction
  def save
    Base.send :uninclude, Transaction
    body = "my transaction! + " + super
    Base.send :include, Transaction
    body
  end
end

# Our everyday model
class A < Base
  include MyTransaction
end

class B < Base; end

class C < Base
  include MyValidation
end

a = A.new
b = B.new
c = C.new
puts a.save #=> my transaction! + valid + clean + saved
puts b.save #=> transaction + valid + clean + saved
puts c.save #=> more validation + transaction + valid + clean + saved
```



**UPDATE**
I published the post unfinished not thinking it would attract much attention without annoucement : so sorry for the lack of explanation.


* the main idea is to move (almost?) all public instance methods of ActiveRecord::Base in a Behavior module called ActiveRecord::Core - which makes sense since they're the methods to be deCORAted.
* The <a href="http://github.com/yrashk/rbmodexcl/tree/master">uninclude gem</a> is  here to enable the possibility for a plugin to override an AR "core" behavior in a sub "everyday" model.
* Another way I thought of was this (below), unfortunately i get : NoMethodError: super: no superclass method ‘save’.

So...

``` ruby
module MyTransaction
  def save
    Transaction.instance_eval do
      @@save = instance_method(:save)
      remove_method :save
    end

    body = "my transaction! + " + super

    Transaction.module_eval do
      define_method :save do
        @@save.bind(self).call
      end
    end

    return body
  end
end
```


Another solution would be to use the callstack to intercept the presence of `MyTransaction#save` and tell Transaction to do a transparent super.
