---
layout: post
title: default_scope use case
---

Ryan Bate in <a href="http://railscasts.com/episodes/152-rails-2-3-extras">rails 2.3 extras episode</a> throws 2 sugestions on when to use default_scope (which is a sort of global named_scope, that can be overriden).

* `default_scope :conditions => "deleted_at is not NULL"`
* `default_scope :order => "position DESC"`

Ok thats fine but recently i've come up with a more interesting use.

I've built an online payment system for <a href="http://photo.charliechaplin.com/">chaplin's image bank</a> and going through all the hastle of getting ssl certificate, a new ip for the subdomain etc etc it certenly not something I wanted to repeat. (BTW thanks again to Ryan bates for the great serie of episodes on paypal). Well since I would be using it for other parts of charliechaplin.com domain I decided to share the payment table, invoice table and user table through differant rails application. That way I could also have an app gathering all the info for budget management.

Yes <a href="http://blog.rubybestpractices.com/posts/gregory/rails_modularity_1.html">Rails modularization</a> is definitely one aspect of the framework I was eager to explore at some point specially when it came to membership. However you still need to know if a payment comes from the image bank app, or the live perfomance app, or the funny hat app.

So instead of using STI with the risk of breaking things by renaming Payment with PhotoPayment or something, i brought default_scope in the game along with a before_save (or before_create if you prefer)

``` ruby
# In my Image Bank Rails App 
class Payment < ActiveRecord::Base
  default_scope :conditions => { :application => 'ImageBank' }
  before_save { |p| p.application="ImageBank"  }
end

# In my Live Performances Rails App 
class Payment < ActiveRecord::Base
  default_scope :conditions => { :application => 'LivePerformances' }
  before_save { |p| p.application="LivePerformances"  }
end
``` 

And voilà, nothing else to change.


