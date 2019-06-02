---
layout: lab
---

[← Back to Lab](/labs/peel)

# Day 2

## Picking Up

It seems silly that it has taken me so long to realize that the goal isn't to create an Active Record object, it's to create an interface for dynamically adding Active Record behavior to a class.

I'm not sure I worded that well, so let me try to break it down, for my own sake.

In our class, `Book`, we shouldn't define a class-level method called `.find()`. Instead, this should be dynamically added to the ancestry chain, so that when we call `Book.find()`, and it doesn't exist inside `Book`, Ruby knows to look up the hierarchy to find where that method is defined. 

Likewise for the instance methods, like `#update()`. Instead of "hardcoding" that method inside of the `Book` class, we'll want to establish an interface that will allow Ruby to delegate to where that method is defined.

With that said, the challenge of defining these methods, (`.find()`, `#update()`, etc.) continues. How do we define them in an abstract way so that the their behavior is adjusted during runtime? What interface do we make available to classes who wish to add this behavior?

## Behaviors

**Construct an instance of the Active Record from an SQL result set row.**

We want to define a class...

```ruby
class Book
end
```

Where we can call a method like `.find(1)`...

```ruby
Book.find(1)
```

Which will in turn make a call to `SQLite3` and execute a `SELECT` query...

```ruby
SQLite3::Database.new("books_app")
	.execute("SELECT * FROM books WHERE id = 1;")
```

And return to us a `Book` instance that contains the data from the result set row, as attributes on the object.

```ruby
# => #<Book:0x007fa903965e98 @author="Metz, Sandi", @id=1, @isbn="9780321721334", @title="Practical Object Oriented Design in Ruby">
```

The bulk of the work in this flow lies in the `SQLite3` execute method. This method has a lot of knowledge:

1. `SQLite3::Database.new()` means it depends on the `SQLite3` library and knows how to get a new instance of a database object.
2. `"books_app"` is the name of the database.
3. `execute()` is the name of the method on the SQLite database instance that executes an SQL query.
4. `"SELECT * FROM ... WHERE ..."` is the SQL query string to be run.
5. `books` is the name of the database table.
6. `id` is that name of the column.
7. `1` is the id we're querying for.

Managing how all of that knowledge gets to our persistence layer is a vital part of this problem.

Another piece of the problem is how `Book` ends up with a `.find()` class method and how our persistence layer knows how to construct a new `Book` instance, once it's retrieved that data from the database.

[← Day 1](/labs/peel/Day_1)
[Day 3 →](/labs/peel/Day_3)
