---
layout: lab
---

[← Back to Lab](/labs/peel)

# Day 4

## Adding Functionality to Client Classes

The first step in tackling this problem is to add functionality to a model class. If we have a class `Book`, we want to add a class method `Book.find` , and some instance methods `book#title`, `book#author`, and `book#isbn`, without needed to explicitly write them, like we did in the original spike.

#### A Note on Mix-Ins & Inheritance

When you want to create a model with `ActiveRecord`, you have to inherit from the `Base` module:

```ruby
class Book < ActiveRecord::Base; end
```

There are endless debates about composition v. inheritance, and, from my perspective, they all mostly lean towards composition, but what about using mix-ins versus using string inheritance?

In Ruby, inheriting and including a module _both_ add a new entity in the method look-up path:

**Inheritance** 

```ruby
class Foo; end
class Bar < Foo; end
Bar.ancestors
#=> [Bar, Foo, Object, Kernel, BasicObject]
```

**Module Inclusion**

```ruby
module Foo; end
class Bar
 include Foo
end
Bar.ancestors
#=> [Bar, Foo, Object, Kernel, BasicObject]
```

Which is what we want. When we think of the task at hand, we want to insert an object that defines `.find()` for a object we want to make an Active Record object.

For our intents and purposes, either solution will do, and perhaps the same could be said for the folks working on `ActiveRecord`. There are a few considerations to be made when deciding however; here's a quote from [Well Grounded Rubyist, 2nd Ed.](https://www.manning.com/books/the-well-grounded-rubyist-second-edition)

> - _Modules don't have instances._ It follows that entities or things are generally best modeled in classes, and characteristics or properties of entities or things are best encapsulated in modules... class names tend to be nouns, whereas module names are often adjectives...
> - _A class can have only one superclass, but it can mix in as many modules as it wants._ If you're using inheritance, give priority to creating a sensible superclass/subclass relationship. Don't use up a class's one and only superclass relationship to endow the class with what might turn out to be just one of several sets of characteristics.

With that said, I want to think of making client classes "model-able," rather than making client classes "a model." This pulls us towards using a module and including it, rather than using inheritance, _a la_ `ActiveRecord`.

#### A Note about Include & Extend

There are a few different ways to add behavior from a module into our classes. For our purposes, we'll focus on `include` and `extend`.

As mentioned above, `include` will add an entity in the method look-up path. So for our `book#title`, `book#author`, and `book#isbn`, this sounds like the kind of functionality we want. The users of our ORM shouldn't have to define these methods, they should be provided to them. If the method is missing from there class, we can define them in our module and place it next in line.

`extend` on the other hand, defines a singleton class and places the method definitions from our module there. This is the same as if the users of our ORM defined class methods in their class. This is a good candidate for `Book.find`, which, again, we'll define and place in singleton class of the client's class.

>  Often, you will want to use a module to import instance methods on a class, but at the same time to define class methods. Normally, you would have to use two different modules, one with `include` to import instance methods, and another one with `extend` to define class methods.
>
> — [Léonard Hetsch](https://medium.com/@leo_hetsch/ruby-modules-include-vs-prepend-vs-extend-f09837a5b073)

The idiomatic way of doing this like so:

```ruby
module Foo
  def self.included(base)
    base.extend(ClassMethods)
  end
    
  module ClassMethods
    def bar
      "It works!"
    end
  end
end

class MyClass
  include Foo
end
```

The hook `.included()` gets called whenever a module is included into a class, and it's passed the class that included it. The above code adds the method `.bar` to class `MyClass`'s singleton class, `#MyClass`:

```ruby
MyClass.bar
#=> "It works!"
```

## Bits & Bobs

Now we have all the bits & bobs to get started. We know we want to define `Book.find`, `book#title`, `book#author`, and `book#isbn`, but we'll also want to define a constructor `Book.new`, which Ruby has us do by defining the instance method `book#initialize`.

The workflow will go like this: we'll define a method in our client class, *e.g*. in `Book`, and then abstract those method out into modules.

## Onward to the Database

First, well want a way to read our database columns in order to know what accessor methods we need to build. In SQLite, we can use `PRAGMA`

> The PRAGMA statement is an SQL extension specific to SQLite and used to modify the operation of the SQLite library or to query the SQLite library for internal (non-table) data.

This works with the `sql3` gem like this:

```ruby
SQLite3::Database.new("books_app")
  .execute("PRAGMA table_info(books)")
#=> [[0, "id", "INTEGER", 0, nil, 1], [1, "title", "VARCHAR(255)", 0, nil, 0], [2, "author", "VARCHAR(50)", 0, nil, 0], [3, "isbn", "VARCHAR(13)", 0, nil, 0]]
```

Each array has the following values:

```ruby
["cid", "name", "type", "notnull", "dflt_value", "pk"]
```

Because we want the "name" of each column, we can extract them like this:

```ruby
SQLite3::Database.new("books_app")
  .execute("PRAGMA table_info(books)")
  .map { |column| column[1].to_sym }
#=> [:id, :title, :author, :isbn]
```

Notice the call to `.to_sym` which turns each of the strings into Ruby symbols. This makes them a bit easy to work with later on.

Notice in our use of the `sql3` gem, there are two pieces of information that tightly couple this to our implementation: the database name, `books_app`, and the table name `books`.

**Note**: we're going to ignore the hardcoded database name, for now, and only address the table name.

Our solution is going to be to define an instance variable with the table name, and use that to build our query.

```ruby
class Book
  @table_name = "books"
  class << self
    def columns
      SQLite3::Database.new("books_app")
        .execute("PRAGMA table_info(#{@table_name})")
        .map { |column| column[1].to_sym }
    end
  end
end
```

`Book.columns` is a class method, because the columns will be the same for all instances of class `Book`. We could pass the table name into `.columns`, but likewise, the table name will be the same for all instances. However, we want to extract this into a module, and the table name will vary depending on the class that `include`s it. There are at least two ways to solve this problem.

`ActiveRecord` solves this by implicitly linking classes and tables based on the name of the class, _i.e._, a class of `Book` would be linked to a table `books`, and a class `User` would be linked to a table, `users`. Similar to the Broken Record project, I believe there's value in *explicitness*, so instead of inferring a table name, we'll pass it into our module and store it there as an instance variable.

```ruby
module Modelable
  def self.included(base)
    base.extend(ClassMethods)
  end
    
  module ClassMethods
    def link_to(table_name)
      @table_name = table_name.to_s
    end
    ...
  end
end
```

This allows us to do the following from our `Book` class:

```ruby
class Book
  include Modelable
  link_to(:books)
  ...
end
```

There is still an instance variable called `@table_name`, but we've assigned it using a method. That instance variable is available both to our `Book` class, as well as in the `Modelable` module, so we can now move the `#columns` class method into our module.

```ruby
module Modelable
  def self.included(base)
    base.extend(ClassMethods)
  end
    
  module ClassMethods
    def link_to(table_name)
      @table_name = table_name.to_s
    end
      
    def columns
      SQLite3::Database.new("books_app")
        .execute("PRAGMA table_info(#{@table_name})")
        .map { |column| column[1].to_sym }
    end
  end
end
```

**Note**: Here were see string interpolation in an SQL query again, and we discussed how that produces a security vulnerability. Unfortunately, SQLite doesn't allow you to bind variables in PRAGMA queries. We could find the column names another way, using a query that _does_ allow us to use bound parameters, or we could even sanitize `@table_name` ourselves. For now, we'll make a note of it and place it in the icebox where stories go to die.

[← Day 3](/labs/peel/Day_3)
[Day 5 →](/labs/peel/Day_5)
