---
layout: post
title: Search across multiple models
categories: ruby
---

Consider a Rails application with a search feature. Type in a name and it lists the matching ```Artist```s. Sounds simple.
What if we also want to search for the city the artists were born in? Simple as well: add a ```join```, include the city name in the ```where``` clause. Done.
But what if we not only want to search for the ```Artist```s but also the ```Album```s? And have those included in the result list as well?

Usually I try to go without additional libraries or even additional services whenever possible. Because every dependency comes at a cost. And honestly there are quite a few dependencies in Rails already. So let's try and see how far we can get without any additional gem.


### The models:

Somewhat contrived but you should get the idea.

{% highlight ruby%}  
class Artist < ActiveRecord::Base
  has_many :nicknames
end

class Nickname < ActiveRecord::Base
  belongs_to :artist
end

class Album < ActiveRecord::Base
end

{% endhighlight %}


### The simple solution

Just run two queries. As simple as it can get. Perhaps wrap it in an object so your controller stays clean and the view as well:

{% highlight ruby%}
class Search
  def initialize(query)
    @query = query
  end

  def albums
    Album.where('name like :query', query: "%#{@query}%")
  end

  def artists
    Artist.joins(:nicknames).where('nicknames.name like :query OR artists.name like :query', query: "%#{@query}%")
  end
end

{% endhighlight %}

It works but starts to get complicated as soon as you have pagination or you want/need to display the results in the same list. How do you merge those results?

### The DB-view solution

Another solution that does not need any external dependencies: Database views. Database views can be seen as a predefined ```select``` statement that is accessible like a table. Depending on your database you can use different view types (materialized) but for this sample I want to keep it simple.

We create a database view which acts as a reverse index. It combines all the attributes we want to be part of the search and add a reference back to the model. To have multiple models included in the view we can use ```union``` which combines results from multiple tables.

This is the migration which will create the view:

{% highlight ruby%}
class CreateSearchView < ActiveRecord::Migration
  def up
    sql = <<-SQL
      CREATE VIEW searches AS
        SELECT
          a.name || GROUP_CONCAT(n.name) AS reverse_index,
          a.id AS searchable_id, 'Artist' AS searchable_type,
          a.name AS label,
          MAX(a.updated_at, n.updated_at) AS updated_at
        FROM artists a
        JOIN nicknames n on n.artist_id = a.id
        GROUP BY a.id

        UNION ALL

        SELECT
          a.name AS reverse_index,
          a.id AS searchable_id, 'Album' AS searchable_type,
          a.name AS label,
          a.updated_at AS updated_at
        FROM albums a
    SQL

    execute(sql)
  end

  def down
    execute('DROP VIEW searches')
  end
end

{% endhighlight %}

Multiple things to notice:

* This is for SQLite, some functions might be different for other databases (string concatenation, max)
* Because an ```Artist``` can have multiple ```Nickname```s we need to group the results. In order to get all the nicknames in our ```reverse_index``` column we use ```GROUP_CONCAT```
* I added a ```label``` column to avoid N+1 selects when displaying the results
* The ```searchable_id``` and ```searchable_type``` column are named like this to make use of Rails polymorphic ```belongs_to``` association
* The different selects need to return tables of the same size/column order
* There is an ```updated_at``` column. I've added it to have a value i can use for odering

With this table we can create a ```Search``` model and use it to search inside all ```Artist``` and ```Album``` records.

{% highlight ruby%}
class Search < ActiveRecord::Base
  belongs_to :searchable, polymorphic: true
  scope :execute, -> (query) {
    where('reverse_index like :query', query: "%#{query}%")
  }
end
{% endhighlight %}

That's it. You can now use it like:

{% highlight ruby%}
2.2.0 :009 > Search.execute('slash')
  Search Load (0.4ms)  SELECT "searches".* FROM "searches" WHERE (reverse_index like '%slash%')
 => #<ActiveRecord::Relation [#<Search reverse_index: "Saul HudsonSlash", searchable_id: 2, searchable_type: "Artist", label: "Saul Hudson", updated_at: "2016-04-19 14:00:21.146916">, #<Search reverse_index: "Slash", searchable_id: 1, searchable_type: "Album", label: "Slash", updated_at: "2016-04-19 14:00:21.151739">]>
{% endhighlight %}

Some notes about this solution:

* The view does not have an id column, default ordering will not work
* Weighting attributes is not possible. Weighting can be used to improve the order of hits. Consider this example: when searching for "john" then a match on the name "John Doe" should be ranked higher than on a company name "Johnson & peterson"
* Performance: ```union``` can be costly, consider using ```union all```

### The takeaway

Depending on your needs there might a simple solution that does not depend on additional libraries and does not add a dependency. Don't be afraid of SQL.
