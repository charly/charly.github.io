---
layout: post
title:  "Combining Scopes & Ransack thanks to Virtus & Siphon"
categories: ruby ransack siphon
---


There's a common feature request on the [ransack][2] issue page and that's to integrate your usual ActiveRecord scopes within the search. Indeed ransack is a fantastic tool to quickly setup a form for selecting table rows depending on column values. But it falls a little short when you want more complex, relational searches. So you almost naturally add a scope in your ransack powered search form before you realize it doesn't work and there's no clear way to do it.

## Ransack won't take it all

The problem with that attractive and somewhat obvious feature request is [ransack][2] relies on the typecasting activerecord performs on its attributes, to coerce all the string values of the parameter's hash into their real value. When `params[:user][:age_gt] => "18"` barges into your model it knows "18" should be an Integer (because the corresponding DB column says so) and will pursue accordingly. 

On the flip side your model has no reason to know what's the argument's type you'd be sending to your scope. 
Take this :

```ruby
scope :active, ->(bool) { where(active: bool) } 
```

Unlike ActiveRecord attributes, it says nowhere that `bool` is a Boolean.

```ruby
params[:user][:active] # => "false"
# and since "false" is just a string
!!"false" # => true 
```

Yeah.... Oups! Since you _only get strings from your parameters_ you need a layer of configuration to tell what type your argument is before it reaches your scope.

Of course you could delegate the coercing work to each & every scope by having them only take strings. They would then turn the arguments into the right type. But that's not very elegant : first you run the risk of breaking legacy code and second you're adding another layer of responsability to activerecord which [we're all trying to unbloat][10]. Finally ransack is already being a Form object _and_ an ActiveRecord extension, adding more responsability to it seems well.. irresponsible.

## Why Siphon

Once you ransacked your database you may want to flee with your car but OMG it's out of gas and all the data's in it... So you pull out that little plastic tube out of your pocket and stick in another car suck out the first drops and then let it flow in your car... Siphoning is a very discrete activity next to ransacking and it shows in the codebase : the gist of it is around 50 lines. 
 
So [Siphon's][2] just a tiny convenience gem similar to [has_scope] which is still  experimental, but it does its job of applying scopes to an ActiveRecord model thanks to a Form Object (created with [Virtus][3]) containing the coercing info. 

Now let's see it in action. Imagine you have the canonical "Orders, Products and Items" data set. What would be a case where ransack alone doesn't cover the conditions you want to apply ? Well I had to lookup ransack again because I couldn't remember where it fell short. It does cover quit complex conditions with 'OR', 'AND', matches, greater than,  even joins etc. If course I could come up with complex queries to illustrate the necessity of a scope, but what would be the simplest situation in which ransack wouldn't cut it and you would need a custom scope. Well for one you _can_ combine columns but you _can't_ combine predicates (e.g equals and bigger than). So if you want to display all stale orders and need to disjoin 2 columns with different types:

With a scope it comes naturally (notice the 'OR'):

```ruby
 class Order < ActiveRecord::Base
   scope :stale, -> { where(["state = 'onhold' OR  submitted_at < ?", 2.weeks.ago]) }
 end
```


With ransack alone : 

```haml
= search_form_for @q do |f|
  = f.text_field :description_or_name_cont  #=> Ok but...
  = f.text_field :state_eq_or_submitted_at_gt #=> Impossible.
```

... you're screwed!

And if you wish to put them in different fields and rely on the user to do the right combination
    
```haml
= search_form_for @q do |f|
  = f.text_field :state_eq 
  = f.date_field :submitted_at_gt
```

... you're still screwed because different fields only do exclusive conjunctions (aka: condition1 _AND_ condition2) not disjunctions (aka: condition1 _OR_ condition2).

Ok point made let's get on with applying scopes within a form.

## Siphon in action

### The Scopes :

```ruby
# order.rb
class Order < ActiveRecord::Base
  scope :stale, ->(duration) { where(["state='onhold' OR (state != 'done' AND updated_at < ?)", duration.ago]) }
  scope :unpaid -> { where(paid: false) }
end
```

### The Form :

```haml
= form_for @order_form do |f|
  = f.label :stale, "Stale since more than"
  = f.select :stale, [["1 week", 1.week], ["3 weeks", 3.weeks], ["3 months", 3.months]], include_blank: true
  = f.label :unpaid
  = f.check_box :unpaid
```

### The Form Object:

```ruby
# order_form.rb
class OrderForm
  include Virtus.model
  include ActiveModel::Model
  #
  # attribute are the named scopes and their value are : 
  # - either the value you pass a scope whith arguments
  # - either a Siphon::Nil value to apply (or not) on a scope whith no argument
  #
  attribute :stale, Integer
  attribute :unpaid, Siphon::Nil
end
```



### Aaaaand... TADA siphon :

```ruby
# orders_controller.rb
def search 
  @order_form = OrderForm.new(params[:order_form])
  @orders = siphon(Order.all).scope(@order_form)
end
```

You may want to read some [insights on what siphon does][11] or let's dive right into it...


## Ransack hand in hand with Siphon & Virtus

The main idea is to separate the ransack fields from the siphon/scope fields and therefore nest one of them. So let's nest the ransack fields in the q param (since it's ransack's convention) and leave the scopes on top :

```haml
-# admin/products/index.html
= form_for @product_search, url: "/admin/products", method: 'GET' do |f|
  = f.label "has_orders"
  = f.select :has_orders, [true, false], include_blank: true
  -#
  -# And the ransack part is right here... 
  -#
  = f.fields_for @product_search.q, as: :q do |ransack|
    = ransack.select :category_id_eq, Category.grouped_options
```

ok so now `params[:product_search]` holds the scopes and `params[:product_search][:q]` has the ransack goodness. We need to find a way, now, to distribute that data to the form object. So first let ProductSearch swallow it up in the controller:

```ruby
# products_controller.rb
def index
  @product_search = ProductSearch.new(params[:product_search])
  @products ||= @product_search.result.page(params[:page])
end
```

And now the gist of it :

```ruby
# product_search.rb
class ProductSearch
  include Virtus.model
  include ActiveModel::Model
  
  # These are scopes for the siphon part
  attribute :has_orders,    Boolean
  attribute :sort_by,       String
  
  # The q attribute is holding the ransack object
  attr_accessor :q
  
  def initialize(params = {})
    @params = params || {}
    super
    @q = Product.search( @params.fetch("q") { Hash.new } )
  end
  
  # siphon takes self since its the formobject
  def siphoned
    Siphon::Base.new(Product.scoped).scope( self )
  end
  
  # and here we merge everything
  def result
    Product.scoped.merge(q.result).merge(siphoned)
  end
end
```

As you see here Virtus will handle all the siphon attributes automagically (thanks to `super` which really deserves its name here). Then the line :

```ruby
@q = Product.search( @params.fetch("q") { Hash.new } )
```

...will assign a Ransack Form Object to q which will bravely hold the values in :
    
```haml
= f.fields_for @product_search.q, as: :q do |ransack|
```

Then calling `@q.result` on it will give you an ActiveRelation which you'll merge with the other ActiveRelation given by siphon :

```ruby
Siphon::Base.new(Product.scoped).scope( self )
```

And voilà, the controller just collects all the fruits of the hardworking Form Object :

```ruby
@products ||= @product_search.result.page(params[:page])
```

... and you can go on applying more scope (like pagination) it's still your good ol' regular ActiveRelation...

---

To quickly wrap it up : there's no magic and it just works!
Feel free to ask me questions or suggest stuff to improve the article
I'd be glad to update it.


[1]: https://github.com/ernie/ransack
[2]: https://github.com/charly/siphon 
[3]: https://github.com/solnic/virtus
[4]: https://github.com/yo/has_scope
[10]: http://blog.codeclimate.com/blog/2012/10/17/7-ways-to-decompose-fat-activerecord-models/
[11]: https://github.com/charly/siphon#some-insights
