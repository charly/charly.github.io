---
layout: post
title:  "Single Responsibility Part I: The Unknown Knowns"
date:   2015-08-16 13:22:33
categories: ruby srp octopuss
---

We've all heard of the Single Responsibility Principle which stands on top of the SOLID graal of Object Oriented Programming. And there has been a strong trend in the ruby community to apply this principle to enhance the reliability and maintainability of our code. 

So it's all well'n good but agreeing on the benefits of SRP isn't enough and cries for another question that is : 

> where (or when) do you draw the line of an object responsibility ?

Some "responsibilities" are obvious especially when they're direct reflections of the "real world". A User and a Book are clearly seperated because there is, at the expense of sounding pedantic, an 'Ontological' differentiation between the two. A book is not a user and vice versa because well... our brain is wired to not understand the opposite.

But here's a more subtle one. Is A User a Person ? At first you're tempted to answer yes and your gut feeling doesn't even give it a second thought. YES a user is someone behind his computer or mobile phone intertacting with your application, so you swiftly move on and give it attributes like first_name, last_name, date_of_birth etc. 

Is a User a Contact ? Filled with confidence from your previous Yes you through yourself with contempt in this positive Assertion feast and give another resounding "Yes" to that question : obviously a user has an email to authenticate so yeah it's a contact, obviously... Duh 

And here the God syndrome starts  creeping in...

The problem is you're still relying on the 'ontological differentiation' molded by your human experience, but which isn't enough anymore. What you should do instead is think from the standpoint of your Application. To do that you must shift you thoughts from Human Oriented thinking to Application Oriented Thinking. Yes take the blue the pill.

So what's a user, what do we know about the user. Do you remember the famous [statement of Rumsfeld](https://en.wikipedia.org/wiki/There_are_known_knowns) about arms of mass destruction. He said you have 3 sorts of knowns :

* the known knowns (e.g : position of all Irak's AA radars...)
* the known unknowns (e.g. the precise number of tanks. 10 or 20 thousand...)
* AND the unknown unknowns (whouhha they're maybe secretley working whith aliens...)

Zizek latter added to this category the _unknown knowns_. In this case the _unknown knowns_ are all the presumptions & perception biases that inconsciously drove American leaders to take bad decisions...

Well this is exactely what happens with the User god object : you just couldn't help assuming it's a human being with a face, name and sex. But from the Application standpoint it might as well be a bot, an alien or an octopuss, it doesn't give a... care less. 

Same with the Contact. You can be drawn to think the User is a contact since it has an email and you can therefore make contact. But remember the login field used to be 'username' or 'login'. Or it could be something else, why not a picture file with your face on it. And while we're at it, why limit your contacts to the authenticated users ? Why couldn't you have a whole bunch of contacts in your database for newsletters for exemple ?

Again : Person, Contact, Address are all concepts that tresspass the responsability of the sole User. A User is just a couple of matching attributes which permit outside interaction with other parts of the App. And that's it. So if you're eager to give your User an avatar, a name and body parts... remember your _unknown knowns_ !

The next part of this post will tackle with SRP in the MVC. And no it's not about Services...


