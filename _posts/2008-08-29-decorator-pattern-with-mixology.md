---
layout: post
title: Decorator Pattern with Mixology
---

There are [many][1] [articles][2] [discussing][3] the Decorator Pattern in Ruby.

Also Design Patterns in Ruby gives an overview of every approach and drawback.

In a nutshell, Decorator pattern is to object functionnality what tagging is to classification. It gives developpers the ability to add a combination of behaviors to the core feature of an object just like inheritance would, but without the constraint of hierarchy.

To achieve it in Ruby you have 3 solutions :

* the classic java way which is a little, ug... mmm... verbose.
* the alias_method_chain which is the chosen way in rails, and probably the easiest to understand since it kind of shows what it does by ripping the method guts in three with its chainsaw.
* and the 'object.extend Amodule' way which I personnaly find the most elegant, since it takes no magic tricks, only uses the singleton class of all instances, mixes in a module and lets the 'super' method complete the task - instead of some 'process_without_my_decorator' alias.

However the later has a major flaw. In the life span of an object you may require the core/original method of its object... undecorated. Take the example below :

``` ruby
module Creme
  def cost
    super + 0.4
  end
end

class Coffee
  def cost
    3
  end
end

x = Coffee.new
x.extend Creme

x.cost # => 3.4
``` 

The thing is I'm french and I only like my coffee black. Well there is no way I can revert to the coffee without creme. I could ask the waitress to make me a Coffe.new but i don't want to pay my coffee twice. I could as well add an extra module UnCremeMyCoffe, or... i could just write java and mold my own coffee.

You got the point. As Russ Olsen points it out : "you simply cannot un-include a Module"

## Meet Mixology

This was somewhere laying in the back of my head, when I stumbeled upon the mixology gem. It gives ruby the possibility to unmix a module out of a class. Yes, simple and brillant and one wonders why it is not in ruby core. Hopefully it gets in ruby 1.9 or 2.0.

As demonstrated in their [blog post][mixology] it can be used to easily implement the State Pattern. My Eureka moment probaly fired before i reached the end of that post because i don't recall reading then the use of it in the Decorator Pattern. But as they state :
"It's also handy for decorating objects that hang around in memory and need undecorating."

Anyhow I jumped on my terminal fired : sudo gem mixology. And then gave it a try on Textmate:

``` ruby
require 'rubygems'
require 'mixology'

module Creme
  def cost
    super + 0.4
  end
end

class Coffee
  def cost
    3
  end
end

coffee = Coffee.new

coffee.mixin Creme
coffee.cost # => 3.4 

# Waitress, please I asked for a black coffee

coffee.unmix Creme
coffee.cost # => 3
``` 

Amen


[1]: http://cfis.savagexi.com/2007/09/05/rails-unusual-architecture
[2]: http://www.lukeredpath.co.uk/2006/9/6/decorator-pattern-with-ruby-in-8-lines
[3]: http://blog.jayfields.com/2008/04/extend-modules-instead-of-defining.html
[dpr]: http://jroller.com/rolsen/
[mixology]: http://www.somethingnimble.com/bliki/mixology
