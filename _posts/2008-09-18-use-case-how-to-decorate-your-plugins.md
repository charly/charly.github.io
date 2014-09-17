---
layout: post
title: 'Use case: How to Decorate your Plugins'
---

Rails being ruby's killer app, I believe it's vital to illustrate whatever theoretical consideration with real life rails cases (and not just <a href="http://ruby.simapse.com/2008/09/from-decorator-pattern-to-before-filter_16.html">Coffee</a>, spoon_required gibberish)

So take my <a href="http://photo.charliechaplin.com/">image bank</a>: class Asset holds the pictures and i gave it a tagging system. I quikly realized i needed to downcase all tags for consistency so I would not end up with both "Winston Churchill" and "winston churchill" as seperate tags.

<a href="http://github.com/mbleigh/acts-as-taggable-on/tree/master">acts_as_taggable_on</a>, gives you a whole set of methods... some of which are dynamic given its argument (e.g acts_as_taggable_on(:tags, :people) will give you tag_list, and person_list)

Well 2 years ago i would of done something extremely ugly in my controller like:

``` ruby
params[:asset][:tag_list].downcase.... 
```

And just the thought of it makes me sick and consider zen boudhism as an alternative to eating cats.

So now that my controllers are all skinny, i was looking at a way to downcase the argument of tag_list=(arg) in my model. Lets make a short story shorter: tag_list= and person_list= being defined dynamically, they're just stubs for a more general method:  set_tag_list_on(context, new_list, tagger=nil).

Below i've summed up my 4 or 5 attempts in two:

<script src="http://gist.github.com/170539.js"></script>

To tell you the truth my second attempt was actually

``` ruby
module TagHack
  def tag_list=(new_tags)
    super(new_tags.downcase)
  end
end
``` 

but that didn't work because tag_list is actually dynamically defined _directly_ in the body of the Asset class wheras set_tag_list_on lies in ActiveRecord::Acts::TaggableOn::InstanceMethods so it is higher up in the inheritance chain. And that is true for 95% of all instance methods of any active record plugin.

So as you see, nothing is more easy than "enhancing", "decorating", "altering", "bending" any method


* _without_ touching the original code or endangering other classes using it...
* _without_ resolving to black magic... 
* and _without_ polluting your code _with_ methods saying _without_ !

The only constraint is that both the original code and your hack must be in modules 'bracket'. The good news is that a lot of code (like ActiveSupport and so on) is already concieved that way.

Voilà.

(Next I'll look at some rails internals and a suggested way to <a href="http://rails.lighthouseapp.com/projects/8994/tickets/285-alias_method_chain-limits-extensibility">dump alias_method_chain</a>)
