---
layout: post
title:  "Rack's Contract in Pure Ruby!"
date:   2014-06-02 13:22:33
categories: ruby rack
---

Rack yo

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

app       = lambda { |env|  [env, "app says: "]}
donothing = DoNothing.new app
welcome   = Welcome.new donothing
hello     = Hello.new   welcome

puts hello.call("env says: ")

# OUTPUT
# => env says: hello request, welcome on the server...
# => app says: it was nice seeing you, have nice time on firefox!


#puts \
#Hello.new(
#  Welcome.new(
#    DoNothing.new(
#      app))).
#        call("env says: ")

~~~

hello 




