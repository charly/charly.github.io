---
layout: post
title: Some (old) thoughts on Mongoid
---

(disclaimer)  
I am <a href="http://groups.google.com/group/mongoid/browse_thread/thread/b5e2bccf77457043/395c08f08cc4777e?hl=en&amp;q=#395c08f08cc4777e">pasting a post I wrote a year ago</a> after discovering <a href="http://mongoid.org/">mongoid</a> on hashrockets liveshow and beeing very excited yet disappointed by the idea mongoid is for hierarchy only.... The reason is I planning to write another one soon and want to expand a little on this one.  
(disclaimer end)

Lets take the canonical Book app. My Book app is very book centric, and for many good/bad reasons I do not want my books to be embedded in the Author class.

2 solutions :

1. the _SQLlike solution_, where book has the reference author_id and Author is a seperate collection. The problem is you can't do joins, so finding conditions with association (eg books where author.age > 18 )
2. the _Document Oriented_ solution, where the Author is embedded in Books. After all why not, you can live with that level of duplication if its just a name. But wait you want to add the Author's bio, date of birth, and plenty more attributes... Yup Author really is a collection by itself, it can't just be embedded or it becomes a consistency nightmare.

So from there one possible move is keep the SQL set of mind and do some caching : on top of author_id you add the field author_name and Book pulls out the Author#name on a before_save callback so its available as a condition on Book.find and for direct display in your index files.

But that is stupid. We're still thinking like SQL junkies here, carefully denormalizing a little our data for performance... without the goodness of SQL! DocumentObject is half way through the SQL and the View, its like a show.erb file without the html tags plus all the power of Business Logic. It is not some poor Database intermediate pledging for a little content through complex ORMs asking you to<br />fullfill more queries before they indulge in providing it.

No ! The hole point of DocumentObject is to mirror the content of the ObjectModel as closely as possible. Which means if a Model is composed of another Model, it should be reflected in its data structure, and that is precisely what an embedded object is in mongoDB/mongoid.

In practice that means having Author stored both in an independent collection "authors" !!!!and!!!! embedded in Book. The embedded author would have a reference (e.g :master_id) to its couterpart in authors collection to keep in sync. You could then decide if the embedded is a light copy of its master or a full blown nested mirror... The main idea is you are not trying to shape your programm to the way your data is stored, you are bending the data to the way you domain model is shaped, for the same reasons you chose Ruby over Java. Happiness :-)

Call it glorified caching if you wish, I think this could be a small yet true paradigm shift. Why ?

* first of all your **domain model rules** the data not the contrary !
* you are **not constrained anymore to hierarchy vs reference**, they both complete themselves
* you keep the **consistency of the dot notation** (no ugly author_name here)
* **caching is not an after thought**, it is part of your model right from the beginning.
* duplication may be a problem for some but consider it also like a new possiblity : Author can be tied for some attributes to its master (name, date_of_birth) but then the bio could change/or not depending on the parent's scope (book).

Of course one should not ignore the drawbacks, it would be poorly suited for a heavy writing model app. But I think that suddenly widens the horizon, you don't have to torture yourself figuring out which from mongomapper or mongoid is best suited for your app, or should i stick to SQL.... etc.

Here's some more thoughts :

You cannot avoid the performance hit on the writing part, so you should be able to decide which attributes are going to live on the embedded version to minor that. Those would typically be the ones with small footprints (booleans, integers, small strings, etc) unlikely updated, &amp; usefull for aggregation or find conditions. To keep data in sync nothing extraordinnary : a timestamp on each embedded version to compare with the "master" timestamp versions. The DSL could look something like that :

``` ruby
class Books
  has_one :author, :exclude =>["bio", "ratings"]
end
``` 

To go a little further I was imagining what a user's library app would look like.<br />In a SQL environment you'd have  a library table with user_id, and book_id, and : 

``` ruby
class User
  has_many :books, :through => :library
end
```

whereas in mongoid  :

``` ruby
class Library
  has_one :user
  has_many :books, :only => ["title", "author.name"] do
    field :rating, integer
    field :read, boolean
    include Book::LibraryMethods #could be a convention
  end
end
```

Each row of the library collection would contain all the users book. But those books wouldn't be a simple proxy of Book, but a striped out/ enhanced version, specially suited for the Library. The Developper should be encouraged to use the power of modules so that each set of methods are consistent with each scope.

``` ruby
class Author
  has_many :books, :only =>["title", "publication_date"] do
    field :writing_context
    include Book::AuthorMethods
  end
end
``` 

Another approach would be to simply dump the proxy design and go with modules. It strikes me that an embedded object in Mongo is very similar to a Ruby module : it only lives in the context of the class/docObject it is included in. That drifts us away from the current dsl of mongoid and I haven't put much thouth in yet. However I have a few more ideas. I'm sure we haven't yet scratched the surface of the doc object possibilities....

Meanwhile here's a some experimental code for accessing embedded association without a proxy : <a target="_blank" rel="nofollow" href="http://www.google.com/url?sa=D&amp;q=http://gist.github.com/285877">http://gist.github.com/285877</a>

charly
