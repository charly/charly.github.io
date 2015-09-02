---
layout: post
title:  "Devise and Omniauth Upside Down"
date:   2015-09-02 13:22:33
categories: ruby devise omniauth
comments: true 
---

This is a follow-up to the previous article I wrote about [mixing concerns of Identity and Authentication][1]. I encourage you to read it, however I'll attempt to put it in a nutshell : 

*The User Model's fundamental purpose is to give scope and integrity to various parts of the data in an application based on user interaction and ownership. Authentication on the other hand is the process which attempts to tie the `user_id` you provide to some person you claim to be. Yet all popular authentication gems (Devise, Sorcery, Authlogic...) mix both.*

A strong clue for this separation of concern is authentication's only _done once_ (when you signin) while data integrity is ensured the _rest of the time_ (`current_user.orders` etc.)

Based on that finding my proposal was to consider the usual email authentication process as just another Omniauth provider and have a separate User table to scope data & interactions. 

## From Omniauth-Identity to Omniauth-Devise

[Omniauth-Identity][omnid] was actually meant to to be used that way but nowhere in the various tutorials or implementations did I see the idea taken all the way down to its conclusion. So I thought lets go for it and make a proof of concept.

One minor detail annoyed me though, the impossibility of using nested params to send `email` and the password. I like my tools ([SimpleForm][simplef], Form Objects, etc.) and being forced to write `form_tag` with none of the niceties I get with `simple_form_for` led me to want to create my own strategy for omniauth.


So I looked at the omniauth-identity gem to see how it worked. There was some cruft in it (form builder, many options..), but the crux of it all actually lied in one line : 

~~~ ruby
module Omniauth::Strategies
  def request_phase
    ::Identity.authenticate(conditions, request['password'] ) or fail!(:invalid)
    super
  end
end
~~~ 

At that point a small light bulb popped out of my brain : hey wait!... Why wouldn't I use Devise for that. After all it has all the bells and whistles, why reinvent the wheel...

And the immediate after thought was the realization of **Devise's Original Sin** : 

*instead of Devise using Omniauth with omniauthable it should've been the other way round from the very beginning, that is _Omniauth using Devise_ - as  another provider. Devise's Omniauthable module is like putting the bottle (omniauth) inside the water (devise).*

NO Omniauth is the container ! and Devise's the kool-aid.

Here's a little diagram to illustrate the principle:

![Omnidevisable]({{ site.url }}/assets/omnidevisable.png)



## The Proof of Concept
So what does Omniauth-Devise look like ? It's almost identical to omniauth-identity. 

~~~ ruby
module OmniAuth
  module Strategies
    class Devise
      include OmniAuth::Strategy

      def callback_phase
        return fail!(:invalid_credentials) unless devise_model
        super
      end

      # some more stuff.....

      def devise_model
        resource = model.
          find_for_database_authentication(request["session"].slice("email"))
        resource if resource.valid_password?(request["session"]["password"])
      end
    end
  end
end
~~~ 

Here's what happens.

1. You register your email and password in a model created by devise.  
I called mine EmailId which is a terrible name please don't do it !!!!
2. `bin/rails g devise email_id` ... I know yuck! But I wanted to make clear this is not our beloved User !!! You may then go through the process of confirming your email or not this is basic devise stuff.
3. Now the **big difference (drumm rolllll)** is you DO NOT LOGIN with DEVISE NOMORE !  
Instead you point your login form to:

`= simple_form_for UserSession.new, url: "/auth/devise/callback" do |f|`

which points to

`get "/auth/devise/callback", to: "session#create"`

~~~ ruby
class SessionsController < ApplicationController

  def create
    auth = AuthenticateUser.build( env["omniauth.auth"] )

    if auth.save
      session[:user_id] = auth.user.id

      redirect_to "/secret",
        notice: "yeyy got there with omniauth and #{params[:provider]} as provider !"
    else
      render text: "failed sorry try again"
    end
  end


  def destroy
    session[:user_id] = nil
    redirect_to root_path, notice: "You got outa omni box"
  end
end
~~~ 

Now all we nedd is some secret sauce which ties Authentication and User together.

You're all grown up developpers, I think it'll be clear enough  if you read it yourself :

~~~ ruby
class AuthenticateUser

  include Virtus.model
  include ActiveModel::Model

  attribute :provider
  attribute :uid
  attribute :info, Hash
  attribute :credentials, Hash

  validates :provider, :uid, :info, presence: true

  # instantiate AuthenticateUser with controller.env["omniauth.auth"]
  def self.build( omni_hash )
    self.new omni_hash.slice("provider", "uid", "info", "credentials")
  end

  def email
    info.fetch("email")
  end

  def save
    link_or_create_user
    authentication.save
  end

  def authentication
    @authentication ||= Authentication.
      where(attributes.slice(:provider, :uid)).first_or_initialize
  end

  def user
    @user ||= User.where(email: email).first_or_initialize
  end

private
  def link_or_create_user
    authentication.user || authentication.user = user
  end
end
~~~ 

Remember all this is just a proof of concept. I haven't tested nor worked on many quirks. 
But believe me it works ! 

You don't ??? 

Try it out for yourself [the code is on github][gitomni]

Cheers,  
Charly







[1]: {{site.url}}/ruby/omniauth/devise/2015/08/27/single-reponsibility-fighting-consensus.html
[omni]:  https://github.com/intridea/omniauth
[omnid]: https://github.com/intridea/omniauth-identity
[simplef]: https://github.com/plataformatec/simple_form
[gitomni]: https://github.com/charly/omnidevisable

