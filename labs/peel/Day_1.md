---
layout: lab
---

[← Back to Lab](/labs/peel)

# Day 1

## Where to Begin

First, let's quickly revisit our problem domain:

> We want to build a simple CRUD app to manage a global list of books in our library. We should be able to record a book's title, author, and ISBN number.

Now, let's start with some initial, sometimes arbitrary, design decisions so that I can zero-in on the meat of the problem. First, like Gregory Brown's [Broken Record project](https://practicingruby.com/articles/implementing-the-active-record-pattern-1), I think I'll begin with an SQLite database. SQLite is the default Ruby on Rails development database, and its light footprint means I should be able to get up and running quickly. We'll utilize the `SQLite3` gem, which you can read up on [here](https://github.com/sparklemotion/sql3-ruby/).

I'm also going to assume that the database table already exists. Later in the project, I hope to take a look at managing schema migrations, but for now, the creation and alteration of database tables/columns will happen outside of the active record object. This is a good example of how the `ActiveRecord` implementation from Ruby on Rails goes above and beyond the basic definition of the Active Record pattern. 

The database is build like this:

```sql
CREATE TABLE IF NOT EXISTS books (
  id 	 INTEGER PRIMARY KEY,
  title  VARCHAR(255),
  author VARCHAR(50),
  isbn   VARCHAR(13)
); 
```

The line `CREATE TABLE IF NOT EXISTS` does exactly what it sounds like, it will either create the table based on the schema provided, or, if a `books` table exits, it will do nothing. 

This is important to note because a change to this SQL statement will not _update_ the `books` table. For example, if we wanted to add another column `pages`, to record the number of pages in our books,  we couldn't simply add another line to our statement and run it again.

```sql
/* This new statement will not update an existing table */
CREATE TABLE IF NOT EXISTS books (
  id 	 INTEGER PRIMARY KEY,
  title  VARCHAR(255),
  author VARCHAR(50),
  isbn   VARCHAR(13),
  pages  INTEGER
);
```

The next few lines create the columns of our table. `id`, `title`, `author`, and `isbn`, each hold the the data belonging to each book. The interesting one to note is `id`, which is declared as `INTEGER PRIMARY KEY`

> If you declare a column of a table to be [INTEGER PRIMARY KEY](https://www.sql.org/lang_createtable.html#rowid), then whenever you insert a `NULL` into that column of the table, the `NULL` is automatically converted into an integer which is one greater than the largest value of that column over all other rows in the table, or `1` if the table is empty. 
>
> -[SQLite FAQ](https://www.sql.org/faq.html#q1)

## SQLite3 Gem

In order to build an Active Record object, we need a database, and in order to use a database, we need an adapter. Fortunately for us, it is not the responsibility of an Active Record object to facility the infrastructure of communication between itself and the database. For our purposes, we'll begin with the [SQLite3 gem](https://github.com/sparklemotion/sql3-ruby) that abstracts the database away from our object. 

Later we'll see how we can wrap this gem in an object we own, so that we can move towards becoming database agnostic. 

## Spiking an Active Record Object

```ruby
require 'sql3'

db = SQLite3::Database.new("books.db").execute <<-SQL
  CREATE TABLE IF NOT EXISTS books (
    id 	   INTEGER PRIMARY KEY,
    title  VARCHAR(255),
    author VARCHAR(50),
    isbn   VARCHAR(13)
  ); 
SQL

class Book
  class << self
    def create(title:, author:, isbn:)
      db = SQLite3::Database.new("books.db")
      db.execute(
        "INSERT INTO books (title, author, isbn) VALUES (?, ?, ?)",
        [title, author, isbn]
      )
      id = db.last_insert_row_id
      Book.new(id: id, title: title, author: author, isbn: isbn)
    end
    
    def find(id)
      result = SQLite3::Database.new("books.db").execute(
        "SELECT * FROM books WHERE id = ? LIMIT 1", id
      )
      result.empty? ? result : Book.new(id: result[0][0], 
                                        title: result[0][1], 
                                        author: result[0][2], 
                                        isbn: result[0][3]
                                        )
    end
  end

  def initialize(id: nil, title:, author:, isbn:)
    @id = id
    @title = title
    @author = author
    @isbn = isbn
  end

  def update(title: nil, author: nil, isbn: nil)
    title = title || @title
    author = author || @author
    isbn = isbn || @isbn

    SQLite3::Database.new("books.db").execute <<-SQL
      UPDATE books  
      SET title = '#{title}', author = '#{author}', isbn = '#{isbn}'
      WHERE id = #{@id};
SQL
    Book.new(id: @id, title: title, author: author, isbn: isbn)
  end

  def destroy
    SQLite3::Database.new("books.db").execute("DELETE FROM books WHERE id = ?", @id)
  end
end
```

## Example Usage ##

```ruby
Book.create(title: "Practical Object Oriented Design in Ruby", author: "Metz, Sandi", isbn: "9780321721334")
# => #<Book:0x007fa904193e80 @author="Metz, Sandi", @id=1, @isbn="9780321721334", @title="Practical Object Oriented Design in Ruby">

Book.create(title: "Patterns of Enterprise Apllication Architecture", author: "Fowler, Martin", isbn: "9780321127426")
# => #<Book:0x007fa9039290b0 @author="Fowler, Martin", @id=2, @isbn="9780321127426", @title="Patterns of Enterprise Apllication Architecture">

book = Book.find(1)
# => #<Book:0x007fa903965e98 @author="Metz, Sandi", @id=1, @isbn="9780321721334", @title="Practical Object Oriented Design in Ruby">

book.title
# => "Practical Object Oriented Design in Ruby"

new_book = book.update(title: "POODR")
# => #<Book:0x007fa90291f638 @author="Metz, Sandi", @id=1, @isbn="9780321721334", @title="POODR">

Book.find(2)
# => <Book:0x007fa9028a72c8 @author="Fowler, Martin", @id=2, @isbn="9780321127426", @title="Patterns of Enterprise Apllication Architecture">

Book.find(2).destroy
# => []

Book.find(2)
# => []
```

This is an Active Record object. It's not pretty, but it illustrates the core features of the pattern.

From the client's perspective, they're only dealing with a POOR, _plain old Ruby object_. Without debating the pros and cons of obfuscating database interactions, I think our `Book` class does a pretty good job!

Here is a list that Martin Fowler describes as typical Active Record object behaviors and a mapping to what our object does:

1. Construct an instance of the Active Record from a SQL result set row *(e.g. `Book.find()`)*
2. Static finder methods to wrap commonly used SQL queries and return Active Record objects *(e.g. `Book.find()` & `Book.create()`)*
3. Update the database and insert into it the data in the Active Record *(e.g. `Book#update()`)*
4. Get and set the fields *(e.g. `Book#id()`, `Book#title()`, etc)*

Two other behaviors are:

1. Implement some pieces of business logic
2. Construct a new instance for later insertion into the table

Which aren't currently implemented, however, we could follow the pattern above and implement these features pretty quickly. 

## What I Learned

The spiked implementation highlights a few things for me.

1. The `ActiveRecord` implementation provided by Ruby on Rails does more than just implement the Active Record pattern. 

   `ActiveRecord` isn't just an *implementation* of a pattern, it's an _abstraction_ of the pattern. `ActiveRecord` allows its clients to _create_ Active Record objects, rather than being one. It uses object-orientation to produce interfaces which allow you or I to more easily create a `Book` class that interacts with a database. Using the Active Record pattern to build an ORM is not the same as using Active Record pattern to build an object.

2. Testing is tricky.

   The clearest way of testing the spiked version of `Book` is to write end-to-end tests that actually touch the database. This isn't ideal, so it's worth taking a step back and considering #1, above. The goal is to build an ORM abstraction that other classes can use, not just build an Active Record class, so there's some pieces missing in the current iteration.

3. It's all about generalization.

   The spike has left me with ugly code, but "working" code. I say, "working," because I've only got manual tests to verify what "working" means. However, with working code in hand, now I can begin to see abstractions that I could only guess about before.

## Next Steps

How can I make SQL queries dynamic? How can I generalize a class' attributes, based on database columns? How can I remove hardcoded dependencies on the `SQLite3` gem? 

Tomorrow, it's into the fire with these questions in hand. Thankfully, Ruby allows us to answer these questions in many different ways. Hopefully, tests can drive us to the answers!

[← Day 0](/labs/peel/Day_0)
[Day 2 →](/labs/peel/Day_2)
