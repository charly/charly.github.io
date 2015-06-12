---
layout: post
title:  "Rack's Contract in Pure Ruby!"
categories: ruby rack
---

Most of you probably know [what Rack does][1] but here's a quick reminder : 

> Rack's a web server interface powering most of Ruby's web frameworks (Rails, Sinatra...) with a nice little protocol which gives you the possibility to manipulate the incoming http Request hash and the outgoing Responses in nicely pilled up stack of "Middlewares".

For example if you `bin/rake middleware`  a Rails app you get this : 

~~~ ruby
# http request hash
use Rack::Sendfile
use ActionDispatch::Static
use Rack::Lock
# ... loooong list
use Rack::ETag
run Bookstore::Application.routes
# Bookstore response
~~~

so...

1. The Request you send from the browser gets processed in `Rack:Sendfile` all _the way down_ to `Bookstore::Application.routes` 
2. Then the Response of `Bookstore` is similarly processed all _the way up_ back to `Rack::Sendfile` and in your your browser.

Ok now that we have the basics how does this actually work in pure ruby terms ? Because the concept is sure easy to grasp but does it need plenty of magick to be used ? Or can it be easily reproduced in a different context ? 

Well let me first create my own little middleware stack.

## My own little Rack stack

As you can see below, to comply with Rack's protocol, each middleware has a `#call(env)` method which takes the request hash (aka: env). It also initialize itself with the next middleware in the stack : 

~~~ ruby
class Hello
  def initialize(app)
    @app = app
  end
  
  def call(env)
    env, body = @app.call(env + "hello request, ")
    
    [env, body + "have nice time on firefox!"]
  end
end
    
class Welcome
  def initialize(app)
    @app = app
  end
  
  def call(env)
    env, body = @app.call(env + "welcome on the server...")
    
    [env, body + "it was nice seeing you, "]
  end
end

class DoNothing
  def initialize(app) @app = app; end

  def call(env) @app.call(env); end
end  

app = lambda { |env|  [env, "app says: "]}
#...
~~~ 

Rack's DSL (as seen with the Rails middlewares) will use it like this :

~~~ ruby
use Hello
use Welcome
use Donothing
run app
~~~ 

Which is just some syntactic sugar for :

~~~ ruby
donothing = DoNothing.new app
welcome   = Welcome.new donothing
hello     = Hello.new   welcome

puts hello.call("env says: ")

# OUTPUT
# => env says: hello request, welcome on the server...
# => app says: it was nice seeing you, have nice time on firefox!
~~~ 

As you see the order is inverted in Rack's DSL to reflect the actual order in which every middleware processes (calls) the request : from top to bottom.

And that's it ! Super easy !

Another way of seeing it is like this :

~~~ ruby
Hello.new( Welcome.new( DoNothing.new( app ))).call("env says: ")
~~~

It's interesting to note at this stage that this is conceptually close to a well known pattern which wraps methods at runtime : the [Decorator Pattern][2]. 

And indeed it's not difficult to take another step and consider [Rack middleware as a general purpose abstraction][3].


[1]: http://code.tutsplus.com/articles/exploring-rack--net-32976 
[2]: https://sourcemaking.com/design_patterns/decorator
[3]: https://youtu.be/i6pyhq3ZvyI




