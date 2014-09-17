---
layout: post
title: 'Refactoring to REST : why context matters'
---

## The wrong path

The app I refactored licenses pictures. The licensee creates an order, selects pictures, finally submits the order, which is then reviewed by Mr licensor. The cart relied on on a current_order the same way current_user works (with a session id), to add or remove a picture from it.

A few months later we decided to extend the app to have automated invoices as well as  payments online. Naturally I created an "Invoice" and a "Payment" model which both belonged to an "Order" with a "one to one relationship".

So, to access payments and invoices, I also relied on current_order.

``` ruby
class PaymentsController < ApplicationController
  before_filter :find_payment
  layout "secure"
  
  def show(); end

private
  def find_payment
    @payment = current_order.payment
  end
end

#some view
= link_to "payment", current_order.payment #=> "/payment/:id"
```

That made sense at the begining : as soon as I accessed an Order#id it would check if the current_order was the same as the one found with params[:id] and if not, I updated it along with the cart.

``` ruby
#orders_controller.rb
 def set_current_order
   self.current_order==@order || self.current_cart= (self.current_order= @order).assets )
 end
 ``` 

But what if i wanted to access a payment directly, without first going through the orders controller (e.g. in some email inviting a customer to pay) ? I would have to make sure the payment I'm accessing belongs to the user and then update current_order with @payment.order so I don't endup having any mismatch with a trailing current_order.

Not unsolvable but : ...

Bad guess !!! I should only access a payment through the order it belongs to. What I'm doing wrong here is _setting the context (an order) after finding the object it lives in (it's payment)_.


## The Importance of Context

Rule of thumb : **get the context before the object it lives in !!!**

Why you ask ? Because I could easily end up having a Payment view linking to an order it doesn't belong to.

That meant : instead of having links like this

``` ruby
link_to "payment", current_order.payment
``` 

...they should all be like this :

``` ruby
link_to "payment", [@order, :payment]
``` 

That way you have a consistent before_filter for every singleton (payment, invoice, items etc) belonging to an order.

``` ruby
#route.rb
  map.resources "orders", order_hash do |order|
    order.resource "invoice"
    order.resource "payment", :new => {"express" => :get}
  end
  #map.resoures "payments" # not in use anymore !

#payments_controller.rb
  before_filter :require_order
  def show
    @payment = @order.payment
  end

#order_module.rb
module OrderModule #included in application_controller
  def require_order
    unless find_and_set_current_order
      flash[:notice]= "INVALID URL : the order you were trying to reach doesn't belong to your account"
      redirect_to "/account"
      return false
    end
  end

  def find_order
    @order = current_user.orders.find_by_id(params[:order_id] || params[:id])
  end

  def set_current_order
    @order && ( self.current_order==@order || self.current_cart= (self.current_order= @order).assets )
  end

  def find_and_set_current_order
    find_order &amp;&amp; set_current_order
  end
end
``` 

Although the idea of having a single point of access through current_order seemed a good starting point, not translating it with the conventions REST resources suggest, had me tangled up in inconsistencies.

It might seem obvious to many of you, but with the :shallow=>true option you get in your routes, which enables access to a resource without its parent, one might forget why, in many cases, the <b>context</b> of a resource is more than relevant : necessary.

Next I will talk of form_for which doesn't play very well with a singleton resource.
