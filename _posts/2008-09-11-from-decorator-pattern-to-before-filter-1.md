---
layout: post
title: From Decorator Pattern to Before Filter 1
---

Looking at my previous post on <a href="http://ruby.simapse.com/2008/08/test.html">the Decorator Pattern in ruby</a>, I thought: 'here's an elegant solution to inject behaviour in an object method while respecting encapsulation... but what could be the use of it, a part from unmixing creme out of my coffee'.

\- Hummm well in this rails plugin i started writing it could fit in....  
\- yes but as a more general solution...  
\- mmmm...Filtering of course !!! Just like the 'before_filter' in rails controller.  
\- you mean like that...  


``` ruby
require 'mixology'

module Callback
  def before_filter filter, filtered_method
    Callback.method_factory(filter, filtered_method)
  end

  # Needed defin_method here so it's called in module's context
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

  def taste
    puts "African Arabica"
  end
end

coffee = Coffee.new
coffee.mixin Callback
coffee.before_filter :spoon_required, :cost
coffee.cost
coffee.cost
```


\- as a rough start it gives you the idea. But we would need something more dsl-ish, like our reference in rails controller: 'before_filter :login_required'  
\- so i guess the next step is to give it a method to call on like spoon_required:  

``` ruby
require 'mixology'

module Callback
 def cost
   spoon_required
   super
 end
end

class Coffee
 def spoon_required
   puts "get a spoon to steer sugar || creme"
 end

 def cost
   puts 3
 end
end

coffee = Coffee.new
coffee.mixin Callback

coffee.cost
```

\- ok, that was easy but the module is still tied to the class it's decorating. We need to abstract away the method we're filtering ('cost') in the module so it could be any method in the class. So this requires a little bit of ruby's magic :  

``` ruby
require 'mixology'

module Callback
 def before_filter filter, filtered_method
   Callback.method_factory(filter, filtered_method)
 end

 # Needed defin_method here so it's called in module's context
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

 def taste
   puts "African Arabica"
 end
end

coffee = Coffee.new
coffee.mixin Callback
coffee.before_filter :spoon_required, :cost
coffee.cost
coffee.cost
```



\- I think we've done the hardest part. The rest in <a href="http://ruby.simapse.com/2008/09/from-decorator-pattern-to-before-filter_16.html">episode 2</a>.  


[dpr]: http://jroller.com/rolsen/
[mixology]: http://www.somethingnimble.com/bliki/mixology
