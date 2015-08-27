---
layout: post
title:  "Single Responsability Part II: Fighting the Consensus"
categories: ruby omniauth devise
---

This post is rather coincidental since it wasn't planned when I started my SRP series. But while I was [illustrating SRP][last] with the canonical `User` God object, I was also fiddling with a recent app, fine tuning some interactions with my Users.  And you guessed right,  I was using  [Devise][dev] by José Valim - who's a sort of God himself BTW, especially since he drank a strange [Elixir][eli].

The problem was the more I was fine tuning my User, the more Devise was getting in my way. Nothing insurmountable but I was starting to fear the time some change in the API would uncover my hacks and bring mayhem to my app (I _was_ bitten before). So the idea of "Authentication from scratch" started creeping in, which triggered this very, very... very long internal debate summed up by : "shoulda ditch devise ?" 

"or not".

## The Consensus
And this is when the consensus fought back. The consensus, also known as the DRP (Devise Responsibility Principle*) has a few widely accepted ideas which blow from almost every direction : 

* it's mature
* it's secure
* it's modular

The DRP also tells you with a hint of contempt : "we understand that, as a newbee, you wish to practice authentication from scratch, as it makes you familiar with the ins and outs of its mechanisms, but experienced developers cut the crap, use devise, and work on their domain model..."

And so in my dreams devise started appearing as this overly sexy MILF with shiny red jewelry, whispering : "don't worry darling, everything's fine, you can let go, I'm modular"... and I would weak up screaming and sweating : "BUT I AM an experienced developper...."

Other dreams involved killing kittens with the face of José Valim. But enough of that.

I wasn't alone actually. A very old monk with a shit load of wisdom was spreading the good news, for some time already: [watch it he's quit funny][diy]

<iframe width="560" height="315" src="https://www.youtube.com/embed/hfsC5Hj6qVI" frameborder="0" allowfullscreen></iframe>

---  
.  

## But Is it Really Worth the Trouble
Beyond that my upmost concern was : is this really worth it ? All this time I'm going to spend re-implementing `rememberable`, `confirmable`, `trackable`, `omniauthable`... Yeah maybe not... 

So came the leap of faith. "I could write a blog post about it" I told myself. "Become famous with my experience fighting devise." 

"Become a hero."

* "The American heroes from Thalys were decorated by François Hollande" 
* "The French hero from Devise got a badge from Stackov...." 

Meanwhile looking at Devise my skeptisism was growing. First of all I had a closer look at all the attributes it adds for `confirmable`. And I started questioning their use.

* `confirmed_at` : ok why not `User.order(:confirmed_at)` gives you a better hint on the user's involvement with the app than created_at. You could average the time between creation and confirmation to see how long it takes to open their email etc...
* `confirmation_token` : ok, wait.... NO this has nothing to do her. It's used once and then remains empty. Clearly this column is polluting the User table...

So I imagined a UserEvent model that could handle the confirmation business. And the sweet glow of SRP kept pouring in my brain. By that time I finished daydreaming MILF had grown old and a had a mustache. José was no more a cute Brazilian but a Polish drunk.

Also there was another victim I'm sorry to say : Ryan Bates.... 

## Challenging Ruby heroes
[José Valim][plata] & [Ryan Bates][rb] ? SRSLY ? Without them we would be lost in the  rainforest crying for help with wild bugs and _unknown unknowns_  roaming beside us. 

Okay, but hear me. I gave many examples in my [last article][last] on what was and wasn't a user. Here's two of them :

* **Contacts** :  you wouldn't hold your users in the same table as the contacts one because maybe you want to pour all your gmail contacts in it without sending them confirmation emails for their membership in your latest MILF app !!!
* Same for **People** : a Person can represent someone on a picture in an [imagebank], but being dead for a long time, there no chance they'll be actual Users ! 

You probably noticed the common trend her for drawing the line of SRP. It is : where does the data belong ? I even came up with a name for it : Data Driven Responsability (because in the realm of programming everybody has a car, don't we ?).
 
Ok, but what's the problem with Ryan Bates, you ask. Well it's complicated... No it's not complicated ! He betrayed me ! He betrayed the whole rails community ! When he made this [fine tutorial][tut] on [Omniauth-Identity][omid] he finally had the chance to separate Authentication concern with User concern, but he didn't ! And considering the huge influence he had (and has) he left us not only orphans (please come back Ryan) but also dazed & confused.

## User is a _Uniq ID_, Authentication is a _Process_
I'll cut to the point : if you're going to take authentication seriously you need to decouple it form the user's responsibility which is only here to provide scope on data. What I mean is that from the standpoint of the application, the only thing that distinguishes a user from another user is its `id` which by definition is uniq. So a user in the eyes of your app is just this number in the `user_id` column of various tables to gather data under a common denominator. That's all. It doesn't care if you're a hacker, a dog or who you claim to be. It's just here to ensure the integrity of data through the uniqueness of a number. 
 
On the other hand authentication is the _process_ which aims to correlate the uniqueness of the user id with the uniqueness of the person claiming that id. Think of this in the real world : you have a passeport, an ID card a driver's license, etc. and they're all supposed to point to the same name and picture of you. Think of your name + picture as the user's id. What happens if you have several passeports with different names : you cheat the system. 

"No I'm not Hannibal Lecter, my name is John Smith and I ain't killed nobody". 

Conversely if the uniqueness of your ID isn't guaranteed and you wish to visit the U.S and the name on your passeport happens to be "Osama _something_" it'll put you in deep trouble. Imagine if your supposed-to-be-uniq-email appeared several times in the user table : at on point you'd get your data and the next day nada....

## Ok Great Concepts Captain Obvious, Give me some Code
You might be thinking : "I already knew all this". 

And I'll ask you : "Did you really ?". 

Because I kinda knew all that as well but it only became clear as I dug into it to make my point in this article. 

So let me restate again before I throw some code: 

"**Identity** (of the user) is a concept based on _uniquness_.  **Authentication** is the _process_ to correlate the User ID with a uniq real world someone."

And these are very, very _different responsibilities_ !!! 

Just think of it : How many time do you authenticate with User ? Just once when you create the session. The rest of the time you just use it to scope data `@orders = current_user.orders` That should've told us something earlier don't you think ?

So how do you dispatch them SRPs ? Well we'll use the gem that got it right in the first place : Omniauth. But unlike the many tutorial and examples hanging around (including the devise way) Omniauth *should not be* in the same db_table/model as the User. It should have its own and you may want to call it `Authentication`. 

The Authentication Table is going to store the "omniauth.auth" data it retrieved from your provider (aka facebook, google_oauth2, twitter, etc). It's also going to hold the `user_id` 

~~~ ruby
# attribute provider
# attribute uid
# attribute user_id
# attribute profile_data
#... 
class Authentication < AR
  belongs_to :user

  def self.from_omniaut(provider:, uid:)
    where(provider: provider, uid: uid).first_or_initialize
    # ...
    # Logic to create a user or link to an existing one...
  end
  
end

# attribute email or just username
# attr.. oh wait that's all we need !!! 
class User < AR
  has_many :authentications
end

~~~

Finally the missing provider : `omniauth-identity`

AS the README points out in a sidenote : 
> Note: OmniAuth Identity is different from many other user authentication systems in that it is not built to store authentication information in your primary User model. Instead, the Identity model should be associated with your User model giving you maximum flexibility to include other authentication strategies such as Facebook, Twitter, etc.

~~~ ruby
class Identity < OmniAuth::Identity::Models::ActiveRecord
  validates_presence_of :name
  validates_uniqueness_of :email
  validates_format_of :email, with: /^[-a-z0-9_+\.]+\@([-a-z0-9]+\.)+[a-z0-9]{2,4}$/i
end
~~~  

And this little guy up there is just going to be another provider for authentication.

Here's some of the bonuses :

* you can do all sorts of email confirmation in identity without polluting the user
* you have a single (DRY) point of authentication for session creation since its all handled by omniauth
* User can handle trackable stuff (current_ip etc) since it's closely related to the `session[:user_id]`
* you can correlate data given from various providers to raise authentication security (something airbnb does)
* you could even let your user login by just entering the email, provided he's already signup with an external service and is still logged into it
* .... many more I can't think of right now
* last but not least Identity _can be pulled out of your app DB_ and act as a general purpose provider for many apps since it only needs to be called once per new session ! (to be tested, I'm not absolutely sure of this yet...)

To conclude because I've got to eat something, sleep, feed the yack, etc : many articles discussed the notions of Authorization vs Authentication since they were often perceived as something interchangeable and it needed clarification. I find it weird that (seemingly) so little has been done to distinguish Authentication vs Identity. But I'm sure there's articles/discussions out there tackling the problem and I'd be happy if you point them out to me.

PS : one could argue a user is almost/theoretically not necessary in this scheme. If user's only a number, it could be provided by the user_id column of authentications table and brought to life in session[user_id] without any call to the DB... Scoping data would just have to be handled differently... But I'm only speculating...

<small>\* DRP states if you have users, delegate them to devise</small>

[last]: http://ruby.simapse.com/ruby/srp/octopuss/2015/08/16/single-responsability-part-unknown-knowns.html 
[tat]: http://www.tousatable.org/
[dev]: https://github.com/plataformatec/devise
[eli]: http://elixir-lang.org/
[plata]: http://blog.plataformatec.com.br/
[rb]: http://railscasts.com/
[xx]: https://practicingruby.com/articles/responsibility-centric-vs-data-centric-design
[tut]: http://railscasts.com/episodes/304-omniauth-identity
[omid]: https://github.com/intridea/omniauth-identity
[diy]: http://confreaks.tv/videos/nickelcityruby2013-actually-invented-here-a-paean-to-reinventing-the-wheel
[imagebank]: http://photo.charliechaplin.com/

