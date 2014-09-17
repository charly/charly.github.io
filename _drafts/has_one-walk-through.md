---
layout: post
title:  "Post has_many walks through associations"
date:   2014-06-02 13:22:33
categories: ruby activerecord
---

Associations is, I believe, what really makes RoR shine. When I started using the framework back in 2005, ActiveRecord had many limitations and the 80/20 percent use case marketed by DHH felt more like 20/80. My Models were covered with gigantic SQL strings, my controllers were obese and everything suffered of HypoSyntacticGlycemia. Today when faced with complex queries, the unfamous impedence mismatch is just a minor nuisance, thanks to Arel, Scopes, Squeel & ActiveRecord diverse strategies. However all the goodness it provides isn't always well understood by the newcomers, considering the amount of questions I see on stackoverfolw. I thought paying a litlle tribute by a simple walkthrough of the Associations internal structure, would contribute to a better general understanding of it all.

## But first the Main Characters
Many times before diving in the code, I heard of macros, reflections, proxies and other weird animals with a vague idea of their roles, but : where they lived... what they'd do... how they'd interact was a complete mystery. So let me present them to you first.

* The keyword __macro__ points to the type of association we're dealing with : `belongs_to`, `has_many`, `has_one`, `has_many through`, `has_and_belongs_to_many ... Why the term macro (in greeek BIG) ? Because calling it inside the class body will unfold a truck load of methods & behaviours.
* The keyword __reflection__ was more opaque to me (like a tinted mirror I guess). And thinking about reflection would often throw me in an infinite loop.... However I am able to say now a Reflection is a class that holds all the informations a Model needs to know about its association. In particular, the reflection will be helpfull for the model to create the association_proxy.
* Which brings us the notion of __Proxy__. As you may know already a Proxy is a bit like your lawyer or spokesman. It'l stand in front of a heard of hungry journalist & IP requests while you're on holidays in Paris sucking up a coffeescript at Café de Flore. When you call 'articles' on the the Author model and ask for interesting insights, the author isn't going to pull out all his work for you, he'll keep enjoying his caffeine spleen while sending his editor proxy to pull out some bait to keep you away from a life of lust and total lack of inspiration. I'll illustrate that later

To gently massage the information in your brain, I'll use a concrete super simple case and in each abstract step translate what's doing with it. Here's the mandatory :

```ruby
class Book
  belongs_to :author
end
```

...

## Ready to Dive in? YourAttention belongs_to me

Since all the magic comes from the simple call of a class method inside your Model's guts, first things first, lets find where these are defined. A quick search shows us they're, _surprisingly_, in the top Association class.

```ruby
# associations.rb
module Association
  # ...
  module ClassMethods
    # ...
    def belongs_to(name, scope = nil, options = {})
      reflection = Builder::BelongsTo.build(self, name, scope, options)
      Reflection.add_reflection self, name, reflection
    end

    def has_many(name, scope = nil, options = {}, &extension)
       reflection = Builder::HasMany.build(self, name, scope, options, &extension)
       Reflection.add_reflection self, name, reflection
     end
    # ...
  end
end
```

So if we put the macro inline and replace the arguments by there value here's what we get

```ruby
class Book
  # belongs_to :author
  # same as...
  reflection = Builder::BelongsTo.build(self, :author)
  Reflection.add_reflection self, :author, reflection
end
```
...

### Builder the other name for Factory (ruby is not the entreprisy one!)

Ok so here it is , the mystery of associations unfolds with a Builder. `belongs_to :bla` is going to instantiate a Builder with `self` the `Book` class, the name of the association `:author` & a few options (eg: inverse_of: :books).
The rest of all the association's logic is neatly tucked in the associations folder & the builders are in the builder folder. It shows us that `Builder::BelongsTo` inherits from two super models in the builder namespace :

`BelongsTo < SingularAssociation < Association`

And that build is just short hand for `new(args).build`

Now we see `build` is defined in both BelongsTo and Association and that the BelongsTo#build calls super & then deal with the specifics of the belongs_to association (not interesting at this stage). So the meat of it is in topp level `Association`.  

```ruby
# associations/builder/association.rb
module ActiveRecord::Associations::Builder
  class Association
    #...
    def self.build(model, name, scope, options, &block)
      #...
      builder = create_builder model, name, scope, options, &block
      reflection = builder.build(model)
      define_accessors model, reflection
      define_callbacks model, reflection
      builder.define_extensions model
      reflection
    end

    def self.create_builder(model, name, scope, options, &block)
      #...
      new(model, name, scope, options, &block)
    end

    def initialize(model, name, scope, options)
      # setting @model = model etc
    end

    def build(model)
      ActiveRecord::Reflection.create(macro, name, scope, options, model)
    end

  end
end
```

We're still in some setup building phase, nothing really happened yet, but we're getting close.
Remember Model is the Book, refereed as self in Builder::HasMany.build(self, name, options, &extension). So Book is creating a reflection. There's no Reflection Model in the association folder but if you lookup the modules included in ActiveRecord::Base, you'll find it :

```ruby 
# base.rb
module ActiveRecord
  class Base
    # ...
    include Association
    #...
    include Aggregations, Transactions, Reflection, Serialization, Store
  end
end
```

Ok hold that Thought. Reflection are usefull and I'll come back to them, but what we _really_ want to know now is where do we get the famous methods which proxy the association class. In our case how does a book know about it's author by just calling author or self.author= ?

### The Association Accessor 



```ruby
# reflection.rb
module Reflection
  module ClassMethods
    def create_reflection(macro, name, options, active_record)
      case macro
        when :has_many, :belongs_to, :has_one, :has_and_belongs_to_many
          klass = options[:through] ? ThroughReflection : AssociationReflection
          reflection = klass.new(macro, name, options, active_record)
        when :composed_of
          reflection = AggregateReflection.new(macro, name, options, active_record)
      end

      self.reflections = self.reflections.merge(name => reflection)
      reflection
    end
  end
end
```

  
And further down the initialize method for AssociationReflection that it gets through it's ancestor 

```ruby
# reflection.rb
MacroReflection
module Reflection
  class MacroReflection
    def initialize(macro, name, options, active_record)
      @macro         = macro
      @name          = name
      @options       = options
      @active_record = active_record
      @plural_name   = active_record.pluralize_table_names ?
                          name.to_s.pluralize : name.to_s
    end
  end
end
```


So we see that after a serie of some hooplas we all this leads to a Book holding a 



[association.rb]: https://github.com/rails/rails/blob/master/activerecord/lib/active_record/associations.rb
[builder/association.rb]: https://github.com/rails/rails/blob/master/activerecord/lib/active_record/associations/builder/association.rb











