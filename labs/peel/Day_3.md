---
layout: lab
---

[← Back to Lab](/labs/peel)

# Day 3

## SQL Injection Vulnerability

```diff
class Book
...
        
  def update(title: nil, author: nil, isbn: nil)
    title = title || @title
    author = author || @author
    isbn = isbn || @isbn
    
-    SQLite3::Database.new("books.db").execute <<-SQL
-      UPDATE books  
-      SET title = '#{title}', author = '#{author}', isbn = '#{isbn}'
-      WHERE id = #{@id};
-SQL
+    SQLite3::Database.new("books.db").execute(
+      "UPDATE books  
+      SET title = ?, author = ?, isbn = ?
+      WHERE id = ?;",
+      [title, author, isbn, @id]
+    )
    Book.new(id: @id, title: title, author: author, isbn: isbn)
  end

...
end
```

Small update of the `Book#update()` method.  The original version of this method used string interpolation in order to set the values for `title`, `author`, `isbn`, and `id`. 

I did this because, the query is just a string and I needed that string to be "filled out" at runtime, dynamically. However, this has serious security implications. This is a  **prime** example of how easy it is to introduce an [SQL injection vulnerability](https://www.owasp.org/index.php/SQL_Injection). 

> A SQL injection attack consists of insertion or "injection" of a SQL query via the input data from the client to the application.

###  The Attack

Our innocent code gets a book from the database, using the `Book#find()` method:

```ruby
book = Book.find(1)
#=> #<Book:0x007fc3c00f9160 @author="Metz, Sandi", @id=1, @isbn="0115501237044", @title="Practical Object-Oriented Design in Ruby">
```

When we want to update our book, we call `book.update()`, with the parameters we want to update. In this case, we update the title:

```ruby
book.update(title: "POODR")
#=> #<Book:0x007fc34ee1160 @author="Metz, Sandi", @id=1, @isbn="0115501237044", @title="POODR">
```

This is the expected behavior.

Next, to visualize what an attack looks like, lets remind ourselves of the method signature and SQL query:

```ruby
def update(title: nil, author: nil, isbn: nil)
    ...
	"UPDATE books  
	SET title = '#{title}', author = '#{author}', isbn = '#{isbn}'
	WHERE id = #{@id};"
	...
end
```

Let's consider what would happen to our query if we called the `.update()` method like this:

```ruby
book.update(title: "PWNED' WHERE id = 2/*")
```

If we called our method like that, our query would look like this:

```sql
UPDATE books 
SET title = 'PWNED' WHERE id = 2/*, author = 'Metz, Sandi', isbn = '0115501237044' 
WHERE id = 1;
```

Notice that SQLite will consider everything after `/*` to be a comment, so if we strip that away, we're left with:

```sql
UPDATE books
SET title = 'PWNED'
WHERE id = 2;
```

What happened? Even though we were working with an Active Record object for a book row where `id = 1`, our interpolated string allowed us to effectively _change_ the original query in order to update a different row in our database, `WHERE id = 2` in our case. 

We can see the effect of our SQL injection by querying for the book with and `id = 2`

```ruby
Book.find(2)
#=> #<Book:0x007fc3c20dd120 @author="Martin, Robert C.", @id=2, @isbn="0187123641198", @title="PWNED">
```

This is not the intended use of our client code, but once it's out there, we can't protect against this kind of attack. Imagine if we were updating a user password instead of a book title!

### The Fix

The fix is to use _parameterized queries_. 

> The use of prepared statements with variable binding (aka parameterized queries) is how all developers should first be taught how to write database queries. They are simple to write, and easier to understand than dynamic queries. Parameterized queries force the developer to first define all the SQL code, and then pass in each parameter to the query later. This coding style allows the database to distinguish between code and data, regardless of what user input is supplied. 
>
> [OWASP](https://www.owasp.org/index.php/SQL_Injection_Prevention_Cheat_Sheet#Defense_Option_1:_Prepared_Statements_.28Parameterized_Queries.29)

The `SQLite3` gem makes using these types of statements, really easy. We replace the interpolated code with a `?`, and provide a list of arguments that will be used to fill in those blanks, later:

```ruby
def update(title: nil, author: nil, isbn: nil)
  ...
  SQLite3::Database.new("books.db").execute(
    "UPDATE books SET title = ?, author = ?, isbn = ? WHERE id = ?",
      [title, author, isbn, @id]
  )
  ...
end
```

In our case, it's as simple as passing in a second argument to `#execute()`. Let's looks at that function definition:

```ruby
def execute(sql, bind_vars = [], *args, &block)
```

You can read more about [Binding Variables in SQLite](https://sql.org/c3ref/bind_blob.html), but essentially, the abstraction in the `SQLite3` gem does exactly what you imagine: replace each, `?`, with the corresponding value in the  `bind_var` array, based on order.

If we try to do the same SQL injection now, we end up with this:

```ruby
book.update(title: "PWNED' WHERE id = 2/*")
#<Book:0x007fc3c22ff638 @author="Metz, Sandi", @id=1, @isbn="0115501237044", @title="PWNED' WHERE id = 2/*">
```

Instead of effecting any other rows in our database, the prepared statement escaped any characters which could otherwise be interpreted as SQL syntax.

[← Day 2](/labs/peel/Day_2)
[Day 4 →](/labs/peel/Day_4)
