---
title: "Solving Schema Bloat with Postgresql's JSONB"
layout: post 
date: 2018-01-06
tags:
  - consulting
  - web-development
  - postgresql
  - ruby
---

<div class="home-link-container">
  <a class="blog-home-link" href="/blog">Blog Home</a>
</div>

I often find myself inheriting projects with database schema’s that suffer from _schema bloat_.  What is “schema bloat”, one might ask ?  Great question, since I just invented the term.  Here is my definition:

Schema Bloat:  The act of adding more and more non-essential columns to a database table in order solve problems for a limited group of your application’s users.

“Table Bloat” would probably make more sense, however, the term “table bloat” has a specific [meaning](https://dba.stackexchange.com/questions/126258/what-is-table-bloating-in-databases).  So we will stick with “schema bloat” for the remainder of this article.

## Schema Bloat
Typically there is a central table for an application - often _users_ or _companies_ or something similar.  The schema bloat problem typically happens when the following occurs:

- Company A needs to track some data that only effects Company A.  To solve this, columns are added to various tables.
- Rinse and repeat for companies B, C, D ….

It’s reasonable to ask, especially if inheriting an existing database, why bother refactoring this column-bloated schema ?  What do we gain ? 

The initial costs of doing this refactoring (which adds zero new features) are significant.  We will cover the specifics later, but for now, let’s go with the assumption that this is not going to be a simple task.  Despite the scariness (and difficulty of selling this kind of refactoring), the gains are	 significant.

#### By getting rid of these non-essential columns, we gain:

*A clearer picture of the data-model*.  There is much room for error and confusion when working on abstract concepts like a DB schema.  Anything we can do to simplify and clarify how our data is organized will help programmers working on the app.  If we can see that one group of values only applies to Company A and another group to Company B, **at the DB level**, it makes our data much easier to understand.

*Working with the data can be easier*.  I’ve inherited tables with 150+ columns.  Most of them non-essential.  How many hours do programs spend squinting at their screens trying to find the proper column name ?  The vast majority of the columns they need to scroll by might not even apply to the company/user that is needing attention.  And one can imagine how this problem can become exponentially more difficult if columns have obscure and similar sounding names:  `xprt_vs_cd_tps` or `xprt_vs_cd_rts` .  Like finding a needle in a haystack!

![Pic](http://foolishlego.com/wp-content/uploads/2014/10/lego_2_277.jpg)


This is a real cost to consider.   In addition, not addressing this kind of problem makes day to day life much more unpleasant for your programmers.  Worker happiness is important ! 

![Pic](https://uproxx.files.wordpress.com/2014/09/george-costanza.jpg?quality=100&w=650&h=400)

---

## What is JSONB ?
JSONB is a column type (data type) that is part of the wonderful Postgresql, open-source database.  Essentially we can store nested JSON (Javascript Object Notation) objects in this column.  _We are going to assume the reader is familiar with JSON.  If not, the internet is your friend_.  This column type gives Postgres the sort of key/value store that is associated with NoSQL databases.&nbsp; JSONB also provides tools for indexing and querying these JSON objects which we shall cover presently.

#### JSONB Basics

Let’s briefly go over some of the basics ways to work with JSONB.  Adding a JSONB column is quite easy:

```sql
CREATE TABLE companies (
  optional_data jsonb
);
```

We also want to (usually) index these JSONB columns.  And we want to use a GIN index.  The reasons for this index are beyond the scope of this talk, as is the discussion about when and why to use indexes.  For the vast majority of situations, the GIN index will work nicely.  This index can also be placed on an individual attribute in the JSONB column.  Here we will apply it to the entire column.   

`CREATE INDEX optional_data_idx ON companies USING GIN (optional_data);`

Once we have a JSONB column we can insert JSON objects into that column.  

```sql
INSERT INTO companies(optional_data) values ('{"opt_att": "Some Val 1", "another_opt_att": "Another Val 1"}');
INSERT INTO companies(optional_data) values ('{"opt_att": "Some Val 2", "another_opt_att": "Another Val 2", "nested_obj": {"nested_att": "some nested value"}}');
```

The syntax for querying values from a JSONB column is fairly straightforward.  There are lots of JSONB functions and operators that we are not going to cover.  You can [read about them here](https://www.postgresql.org/docs/9.5/static/functions-json.html)).

Use the `->>` operator to select a value at the attribute level.   The following query:

`SELECT optional_data->>’opt_att’ FROM companies LIMIT 1;`

Should return `Some Val 1`.


You can select nested values using a combination of the `->` and `->>` operators.

`SELECT optional_data->’nested_obj’->>’nested_att’ FROM companies;`

Should return `some nested value`.

#### JSONB solution for Schema Bloat - The Big Idea

Our big idea is actually quite simple.  We are going to move any column that is “optional” (not used by every company) into a JSONB column, thus slimming down our bloated table(s) significantly.  It’s a fairly simple idea at the DB level.  

Once we get into the Ruby (or insert other language here) level things can become more complex.  We need to make sure our getters and setters are doing the correct thing. The thought of refactoring a lot of views in the application can be daunting enough to scare off would-be-refactorers.  Luckily for us (in the Ruby world and beyond), we have some tools that can help us handle this Ruby refactoring at a low level, and save us from having to change the view layer at all (in most cases).

---

## Ruby and the Storext Gem
Welcome to the Ruby part of this article!  The Storext gem makes it easy for ActiveRecord to work with JSONB.  The Storext gem also wraps the Virtus gem, so let’s talk about that briefly.

#### Virtus Gem

The Virtus gem assists in creating virtual attributes for a Ruby object.  In our case, it is an object that inherits from ActiveRecord::Base - a model.  When can think of [Virtus](https://github.com/solnic/virtus) as a pumped up version of  `attr_accessor`.  The important part of Virtus is that it can enforce type restrictions, which helps a virtual attribute behave more like a column.  Ideally, we want Rails to see no difference between fetching data from a “normal” column, versus our JSONB, optional_values attribute.

#### Using Storext with Rails

Let’s jump into the nuts and bolts of how we accomplish this JSONB refactoring.  We’ll be working with the `Foobar` object - which is backed by a table with WAY too many optional columns (let’s use our imaginations).  

We can store a hard-coded (or if you want to get really flexible, store this data in the DB instead) file in our `app/models/concerns` directory that has information about all of our custom attributes, for a given company.

```ruby
module Company1CustomAtts 
  COMPANY1_CUSTOM_ATTS = [
   {:key=>"att_1", :type=>"Boolean", :default=>false},
   {:key=>"att_2", :type=>"DateTime"},
   {:key=>"att_3", :type=>"String"},
   # ...
  ]
end
```

We’ve stored info about our custom attributes in a hash.  Now we just need to tell our model about these custom attributes.  Here is where Storext and Virtus make our lives much easier!

```ruby
class Foobar < ActiveRecord::Base
  include Company1CustomAtts
  include Company2CustomAtts
  include Storext.model
  @@all_custom_atts = COMPANY1_CUSTOM_ATTS + COMPANY2_CUSTOM_ATTS
  @@all_custom_atts.each do |hash|
    store_attribute :custom_attributes, hash[:key].to_sym, hash[:type], default: hash[:default]
  end
  # ...
end
```

We need to include our collection of custom attributes.  We also need to include the `Storext.model` module to get the Storext functionality for our Foobar model.  Once that is done, we can cycle through our collection of attributes and pass them to the `store_attribute` method.   We pass some other metadata along about the attributes (type and default values).  The `store_attribute` method, sets up setters and getters for each JSONB attribute we have in our collection.  After we execute this code, we can do things like this:

```ruby
foobar = Foobar.first
foobar.att_1 = 42
foobar.save
new_var = Foobar.first
new_var.att_1 # returns 42
```

and Rails will fetch the value from the JSONB attribute, just as if it was a regular database column.

The amazing win here (DO NOT forget to read the “Gotchas” section below) is that once we have this set up, Rails is not even aware that the values are coming from JSONB.  Our getters and setters are magically working!  And things like `form_for`  *just work* (so far to my knowledge - I’m sure there are edge-cases).

Once we realized that we did not have to refactor our views, forms etc… this refactoring became a lot more attractive.  It seems we can solve this at the Ruby level with only a few lines of code!

#### Gotchas - JSONB and Rails

There are definitely some columns that even if optional, may not be good candidates for our JSONB refactoring.   Let’s examine a few cases.

*Foreign Keys*.  ActiveRecord expects foreign keys to be legitimate database columns.  I’m sure one could get this to work using some of the macros provided by ActiveRecord or even diving down to a low level in ActiveRecord.  I would not recommend it.  I think schema bloat is usually not caused by excessive foreign keys.  If that is a problem with a database table, I suspect there are deeper schema problems to address.

*Low level database queries*.  Although we are working with Rails, there often comes a time when we need to drop down to the SQL level and write our query.  Maybe we need to run a Postgresql function.  Maybe we just need to access an attribute outside of ActiveRecord.  Obviously, if you’ve written raw SQL, you will need to refactor it to use the JSONB syntax.  This is not so much a “gotcha” as a friendly reminder.

*Data migration (over to JSONB) can be tricky*.  If we are inheriting a bloated schema, we will need to migrate that data over to it’s new JSONB home.  Once we write the Storext code, the model will start using those getters/setters because they are defined last in the model.  When a Rails model inherits from ActiveRecord it automagically inherits setters and getters for each database column.  That is defined in line 1 of our model (`class Foobar < ActiveRecord::Base`).  Since the Storext code runs after line 1, those getters and setters over-write the ActiveRecord getters and setters.  One way to help get around the trickiness this code introduces is by remembering the following:

```ruby
foobar = Foobar.new
foobar.att_1 # uses the ruby getter for "att_1" (in our case the JSONB attribute) to fetch the value
foobar.columns_hash['att_1'] # uses the foobars.att_1 database column to fetch the value
```
For more complex data migrations, doing things at a lower level in ActiveRecord or at the SQL level might be required.  Once the migration is complete, it is possible to delete or hide (rename) the legacy columns.

*Optional columns that require lots of aggregate data calculations*.  It is possible that one may face performance issues if there needs to be lots of aggregate functions / calculations run on the JSONB attributes.  This is definitely beyond the scope of this article, but I think it’s safe to say that this issue probably only effects those that have large data sets.  Large enough that we need to start thinking about scale.   This last gotcha probably does not apply to most web applications.

---

## Solving Schema Bloat Outside of JSONB
Even without JSONB, we can handle non-essential, optional attributes without adding a bunch of columns to a bloated table.  It requires more moving parts and can perform worse (lots of JOINS needed), but let’s take a look at that schema pattern.

We can set up on table to store our attribute names and a second table to store the related values, scoped to each Foobar object.

```
optional_attributes
-----------------+------------------------+
 id              | integer                |
 company_id      | integer                |
 attribute_name  | character varying(255) |
-----------------+------------------------+
```


```
foobar_optional_attributes
--------------------------+------------------------+
 id                       | integer                |
 foobar_id                | integer                |
 optional_attribute_id    | integer                |
 optional_attribute_value | character varying(255) |
--------------------------+------------------------+
```


We choose the simple example because it illustrates our point clearly.  We can add as many rows as needed to cover the optional attributes.  We no longer need these optional attributes as database columns.  Therefore, schema bloat can be solved without JSONB, however, JSONB makes it much easier.  The example above is only meant to illustrate a schema pattern/idea.  I’m not suggesting it as a viable solution for most applications.


## Summary
Discovering how to easily use Postgresql’s JSONB to manage optional data can really help slim down some bloated tables.  So if you have a bloated schema,  and you are in the Postgresql world, consider JSONB to solve schema bloat!

<div class="home-link-container">
  <a class="blog-home-link" href="/blog">Blog Home</a>
</div>

