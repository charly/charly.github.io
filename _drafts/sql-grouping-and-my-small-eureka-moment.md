---
layout: post
title:  "Tail of my (small) Eureka moment"
date:   2014-06-02 13:22:33
categories: ruby activerecord
---

To determine if a picture is published or not, the **Picture table** has a column named _published_ : 

* if the value is 0 the image's unpublished. 
* if it's 1 the image will appear on the public section. 

Easy peasy...

Now I want to find out **people that _don't have any  pictures published_.**

### 1st Try : Filtering
So without thinking too much I start joining the People table with the Picture table  : 

~~~ ruby
Person.join(:pictures)
~~~

person   | picture    | published
-------- |------------|--------
Goddard  | pic_g1.jpg | 0 
Goddard  | pic_g2.jpg | 1 
Einstein | pic_e1.jpg | 1
Harris   | pic_h1.jpg | 0 

and then I ask for only unpublished pictures like so :


~~~ ruby
Person.join(:pictures).where("pictures.published = 0")
~~~

Great that was fun! 

But if you look closely at the results there's a twist...
The problem with this, is it will give me People that have unpublished pictures. Sure but that doesn't mean they can't have published pictures as well. For example all Einstein's pictures were published, so he won't appear. But Goddard who has many of both (pub & unpub) _will_ appear... I got rid of Einstein but Goddard is still sticking around despite Kono frowning at her.

### 2nd Try : Grouping
So the next thing I do is group the results. 
Grouping results in a relational Database is taking a column of a Table and throwing out all its duplicate values. 

For example if I say :

~~~ ruby
Person.join(:pictures).group("pictures.published") 
~~~

It will take all possible published values - which are (remember...) 0 or 1.
So I'll have 2 results : 0 and 1... Not very helpful

Now if I say :

~~~ ruby
Person.join(:pictures).group("people.id")
~~~

I'll have all people IDS (which are uniq)  but loose anything about their associated pictures... and their published status. Not helpful either.

However before the `group` operator throws out all information tied to someone, it can do a set of calculations on it. For example it can tell me the number of pictures by person :

~~~ ruby
Person.join(:pictures).group("people.id").select("people.*, count(*) as pictures_number")
~~~

And I'll know Goddard has 803 pictures and Einstein 6 - (although I don' know anymore which pictures precisely belongs to them, or wether they're published or not)

As a side note grouping can also calculate  : `sum`, `average`, `minimum`, `maximum` and much more stuff. 

A typical use case would be to calculate the average salary per job status : 

~~~ ruby
Employe.group("employees.status").select("AVG(employees.yearly_salary)")
~~~

Result : 

job       | salary
----------|--------
directors | 623 345
managers | 67 476
secretaries | 26 897 

You won't be able to know the individual salaries of each employee, but you get the average salary per job status.

...(Yawn)

But wait ! you can also group by a combination of columns. In our specific problem combining 2 columns actually gives us an interesting result.

``` ruby
Person.join(:pictures).group("people.id, pictures.published")
``` 

Since it behaves exactly as with a uniq grouping value, it will give us all the uniq values of that combination ("people.id, pictures.published") and ditch the rest. That is :

person | published
--------|--------
Goddard | 0 
Goddard | 1 
Einstein | 1
Harris | 0 

Intersting... This tells us :

- Goddard has both unpublished (0)  and published (1) pictures. 
- Einstein (who appears just once) _has only published pictures_ 
- Mildred Harris (who also appears once) _has only unpublished pictures_.
 
So by looking at the results I could already know who's eligible for my quest of finding **people that have _no_ published pictures.**

Mildred Harris only appears once _and_ with unpublished pictures, therefore, has _only_ unpublished pictures.... She wins, in some sort of twisted revenge !

### Hitting a quantum ceiling
Ok that's one interesting step but I still need to go through all people and check if they appear once or not. And if they appear once, is it with unpublished pictures... so I know if I have a winner.... 
Hum, if I have 678 000 people, I really need to find a way to filter those results... 
But by doing so the obvious way, I'd come into a seemingly untie-able paradox... 
Bare with me because this is where it becomes interesting. 

person | published
--------|--------
Goddard | 0 
Goddard | 1 
Einstein | 1
Harris | 0 

and we want o find a filtering process that gives us Harris, who has only unpublished asset. Now lets call the above temporary result : PeopleWithAssetsStatus. 

From there on let's find people with unpublished pictures.... 

``` ruby
PeopleWithAssetsStatus.where("pictures.published = 0")
``` 

Result : 

person | published
--------|--------
Goddard | 0 
Harris | 0 

Not sufficient Goddard is still here.

So lets count the number of times someone appears. If those people only appear once, that will tell us they have _only one_ published status, unlike the ambiguous Goddard !

``` ruby
PeopleWithAssetsStatus.group("people.id").having("count(*) = 1")
```

Result :  

person | published
--------|--------
Einstein | 1
Harris | 0 

Not sufficient either.... Einstein is still here. Indeed, as well as Harris, he only has one type of published status... but the wrong one...

So you ask me WHY DONT YOU COMBINE THESE RESULTS ? (find people which only appear once and when they do, with unpublished material)

Well, I answer, YOU CAN'T !

And you say, WHY ? 

And I lower my voice and say...

It's like the Heinseberg principle of uncertainty. You either know the position of the electron or the speed of the electron, but YOU CANNOT KNOW BOTH.

In other words if I count the number of time someone appear (once or twice) I need to do a grouping again : 

~~~ ruby
PeopleWithAssetsStatus.group("people.id").having("count(*) = 1")
~~~

But by doing so I loose the ability to know the status of the published column...

Covertly, if I don't do the grouping I know which people have published and unpublished pictures but I can't tell if they have both or not....

Heinsenberg...

Wait a second this is not quantum physics there must be a way ?

---

### Resorting to unsatisfying solutions & Eureka
Before going to sleep I did find a couple of solutions. One was intersecting 2 results and the other was through a filtering process out of the Database process. Both were costly, unscalable & deeply unsatisfying ( like having to push your Ferrari down hill to start the engine - but you live down the valley).

Around 3 o'clock in the morning some muse knocked at my door unless it was the caffeine of the bubble less coke I drank out of despair before pulling myself to bed. One way or the other my brain cells were freshened up by new questions. 

Wait a second I've been relying on counting results of a grouped set but there's other function I can use. Remember you can do an average on salaries. What if you do a such a silly thing on the published status.... Published status is 1 or 0. So... OMG wait a second, all that time I didn't see the obvious !!!!! 

person | published
--------|--------
Goddard | 0 
Goddard | 1 
Einstein | 1
Harris | 0 

Let's try on it some average (TODO explain)

``` ruby
PeopleWithAssetsStatus.group("people.id").select("AVG(pictures.published)")
```

Result :

person | average
--------|--------
Goddard | 0.5
Einstein | 1
Harris | 0 

So if I State : 

``` ruby
PeopleWithAssetsStatus.group("people.id").having("AVG(pictures.published) = 0") 
```

I only get : 

person | average
--------|--------
Harris | 0 

Eureka!

...

But wait again! I am working on a set of grouped values, but do I really need to do that in the first place. What if I take the original statement : 

``` ruby
Person.joins(:pictures)
```

and I sum up every published status for each person and the result is 0 that means _all their values are 0_ and that means _they're all unpublished_

``` ruby
Person.joins(:pictures).group("people.id, pictures.published").having("sum(published) = 0")
```

A posteriori so obvious yet... I'm sure thousands... millions of people went through this kind of process (programmers, physicists, politicians, fiscalists, magicians...) before reinventing the wheel. But it is so gratifying when you eat the one you fished yourself.

What I find mesmerizing is that once you know the solution and then (re)read the first statement of this text... it's already there.













