---
layout: lab
---

[← Back to Lab](/labs/peel)

# Day 5

## Modelable

Yesterday, we left off with module, `Modelable`, that, when included, adds two class methods, `.link_to()`, and `.columns()`. 

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

We use this module like this:

```ruby
class Book
  include Modelable
  link_to(:books)
end

Book.columns
#=> [:id, :title, :author, :isbn]
```

Notice, that if we don't call the `.link_to()` method, we would get an error:

```ruby
class Book
  include Modelable
end

Book.columns
#=> SQLite3::SQLException: near ")": syntax error
```

Because we haven't defined the `@table_name` class instance variable, so the query is being interpolated with `nil` instead of a string. 

**Note**: If `nil` were just an empty string, we would get back an empty list, rather than an exception. This is design choice to take into consideration. Perhaps if we were test driving this, this is the kind of behavior we would account for in our design.

Now that we have our list of column names, we want to create accessor methods for each of the columns. We can do this by creating instance variables for each column and then creating getter and setter methods that retrieve and set the instances variables, respectively. 

### Metaprogramming

```ruby
class Book
  columns.each do |column|
    define_method("#{column}") { instance_variable_get("@#{column}")}
    define_method("#{column}=") { |value| instance_variable_set("@#{column}", value)}
  end
end
```

Ruby gives us access to metaprogramming, which I like to think of as just programming with meta-data. In this specific example, we use the `#define_method()` method, which does exactly like what is sounds like.

We pass into `#define_method()` the name of the method, and a block containing the body of the method. The two calls to `define_method()`, above, create a getter and a setter for each column. The body of the first retrieves and instance variable using `#instance_variable_get()`, and the second sets an instance variable using `#instance_variable_set()`. 

An important thing to note here is that `#instance_variable_get()` will return `nil` if that instance variable hasn't been defined. So our getter method will either return a value that was set using the setter's `#instance_variable_set()`, or it will return `nil`. 

**Note about `nil`**: It's important to realize the downfalls of using `nil` throughout a codebase in this way. `nil` represents both a _state_, (nothing has been set), as well as _data_, (the value of `foo` *is* `nil`, or `null`, as represented in the database). This conflation around `nil` is a topic I find really interesting, and has been tackled in functional languages with sum types; like Haskell's `Maybe` and Elixir's `Option`.

These calls to `#define_method()` are just plopped into the `Book` class, but we can just as easily move them to the `Modelable::ClassMethods` module and call them from our `.link_to` method:

```ruby
module Modelable
  def self.included(base)
    base.extend(ClassMethods)
  end
    
  module ClassMethods
    def link_to(table_name)
      @table_name = table_name.to_s
      create_accessors
    end
  
    def create_accessors
      columns.each do |column|
        define_method("#{column}") { instance_variable_get("@#{column}")}
        define_method("#{column}=") { |value| instance_variable_set("@#{column}", value)}
      end
    end
      
    def columns
      SQLite3::Database.new("books_app")
		.execute("PRAGMA table_info(#{@table_name})")
		.map { |column| column[1].to_sym }
    end
  end
end
```

Now, whenever a client class calls `.link_to()`, or module will retrieve the names of the column for the given table, and automatically generate getters/setters for each of them:

```ruby
class Book
  include Modelable
  link_to(:books)
end

book = Book.new
book.title
#=> nil
book.author
#=> nil
```

### Find

After creating some infrastructure in the client class, we've gotten a good pattern here with our modules. When we want to add a class instance method, we add it to the `Modelable::ClassMethods` module, when we want to add an instance method, we place in `Modelable` module. 

If we think about our interface for `.find()`, we know we want to be able to call it on the class, e.g., `Book.find(1)`. And, we want that class instance method to return a new instance of `Book`, with all of the instances variables set to the values in the respective columns. 

First, we'll tackle the retrieval of information from the database:

```ruby
def find(id)
  SQLite3::Database.new("books_app")
    .execute("SELECT * FROM #{table_name} WHERE id = ? LIMIT 1", id)
end
```

_Note: we have a similar issue as before with the string interpolation..._

Here we're selecting all the rows from the `books` for all rows that have an `id` that match the `id` we pass into the method. `SQLite3`'s `Database#execute()` method allows us to pass in bound arguments as an array as the second argument. Here, it also allows us to pass in just one variable as a single parameter.

By default, `SQLite3` returns the results in an array of arrays:

```ruby
[[2, "Practical Object-Oriented Design in Ruby", "Metz, Sandi", "0311237841549"]]
```

We can use this, with our `columns` array to create a key-value pair, but `SQLite3` also gives us access to a `Database#execute2()` method, which is exactly the same as `Database#execute()`, except that is returns the column names in an array, as well as the resulting values from the query:

```ruby
[
  ["id", "title", "author", "isbn"], 
  [2, "Practical Object-Oriented Design in Ruby", "Metz, Sandi", "0311237841549"]
]
```

This allows us to run one query, instead of two, to get the column names. Now we can `zip` the two arrays, and convert it into a hash:

```ruby
def find(id)
  result = SQLite3::Database.new("books_app")
    .execute2("SELECT * FROM #{table_name} WHERE id = ? LIMIT 1", id)
  result[0].zip(result[1]).to_h
end

Book.find(2)
#=> {"id"=>2, "title"=>"Practical Object-Oriented Design in Ruby", "author"=>"Metz, Sandi", "isbn"=>"0311237841549"}
```

The last thing we'll want to take in account is if our query returns an empty result. If that is the case, we'll return an empty hash:

```ruby
def find(id)
  result = SQLite3::Database.new("books_app")
    .execute2("SELECT * FROM #{table_name} WHERE id = ? LIMIT 1", id)
  
  if result.empty?
    {}
  else
    result[0].zip(result[1]).to_h
  end
end

Book.find(1337)
#=> {}
```

Like before, we can write this function in the `Modelable::ClassMethods` module, an it will be available to our client class, `Book`.

### Construction

Now that we have our hash, we're just a stone throw's away from having a new instance of `Book`. If we tried to pass our hash into `Book.new`, we'd get the following error:

```ruby
Book.new(Book.find(2))
ArgumentError: wrong number of arguments (given 1, expected 0)
```

That's because we haven't defined a constructor for `Book` that accepts a hash.

We could start out with something like this:

```ruby
class Book
  def initialize(**args)
    @id = args.fetch(:id, nil)
    @title = args.fetch(:title, nil)
    @author = args.fetch(:author, nil)
    @isbn = args.fetch(:isbn, nil)
  end
end
```

Now, if we pass in our hash, we get a new instance with the correct values!

```ruby
Book.new(Book.find(2))
#<Book:0x007f8b8fafe6b0 @author="Metz, Sandi", @id=2, @isbn="0311237841549", @title="Practical Object-Oriented Design in Ruby">
```

However, we want to abstract that our generalized `.find()` method can return an instance of _any_ class.

Instead of hardcoding instance variables, we can use the same `#instance_variable_set()` method we used earlier. We'll iterate over the hash, and for each key/column we'll create an instance variable and set it to the value/row data:

```ruby
def initialize(**args)
  args.each do |attribute, value|
    instance_variable_set("@#{attribute.to_s}, value)
  end
end
```

Notice the `@` symbol, without it, our instance variables wouldn't have the conventional `@` in front of it.

Now, we can initialize our `Book` with a hash, and it will assign instance variables for each of the key-value pairs!

```ruby
Book.new(Book.find(2))
#<Book:0x007f8b8fafe6b0 @author="Metz, Sandi", @id=2, @isbn="0311237841549", @title="Practical Object-Oriented Design in Ruby">
```

We can also initialize our `Book` manually, like before:

```ruby
Book.new(title: "POODR")
#<Book:0x0078e21fafe6b0 @title="POODR">
```

However, we can also initialize _any_ instance variables we pass in:

```ruby
Book.new(foo: "bar")
#<Book:0x0078e21fafe6b0 @foo="bar">
```

To solve this, we can limit the instance variables based on the columns in the database, if we pass in a key that doesn't match a column, we will raise an `ArgumentError`:

```ruby
def initialize(**args)
  args.each do |attribute, value|
    if self.class.columns.include?(attribute)
      instance_variable_set("@#{attribute.to_s}", value)
    else
      raise ArgumentError, "Unknown keyword: #{attribute}"
    end
  end
end
```

### Tying it All Together

The last step is to move this constructor into the `Modelable` module. We do not want to move it into the `Modelable::ClassMethods` module, even though we're defining the `.new` class method...

In ruby, when we define the _instance_ method, `#initialize()`, ruby will automagically call it after we call the _class_ method, `.new`.

The final step is to add a call to `.new` with the result of the call `.find()`.

Our entire module looks like this:

 ```ruby
module Modelable
  def initialize(**args)
    args.each do |attribute, value|
      if self.class.columns.include?(attribute)
        instance_variable_set("@#{attribute.to_s}", value)
      else
        raise ArgumentError, "Unknown keyword: #{attribute}"
      end
    end
  end

  def self.included(base)
    base.extend(ClassMethods)
  end
    
  module ClassMethods
    def link_to(table_name)
      @table_name = table_name.to_s
      create_accessors
    end
  
    def create_accessors
      columns.each do |column|
        define_method("#{column}") { instance_variable_get("@#{column}")}
        define_method("#{column}=") { |value| instance_variable_set("@#{column}", value)}
      end
    end
      
    def find(id)
      result = SQLite3::Database.new("books_app")
      .execute2("SELECT * FROM #{table_name} WHERE id = ? LIMIT 1", id)
  
      if result.empty?
        {}
      else
        result_hash = result[0].zip(result[1]).to_h
        new(result_hash)
      end
    end
      
    def columns
      SQLite3::Database.new("books_app")
		.execute("PRAGMA table_info(#{@table_name})")
		.map { |column| column[1].to_sym }
    end
  end
end
 ```

### Grande Finale

We can now create Active Record objects with our module:

```ruby
class Book
  include Modelable
  link_to(:books)
end

book = Book.find(1)
#=> {"id"=>1, "title"=>"Practical Object-Oriented Design in Ruby", "author"=>"Metz, Sandi", "isbn"=>"0311237841549"}

book.title
#=> "Practical Object-Oriented Design in Ruby"
```

## Next Steps

There are quite a number of directions to go in with our new `Modelable` module, here are few key ones that interest me:

1. Abstracting the Database 

   We're relying directly on the `SQLite3` gem, which couples our implementation directly to SQLite. Even further, our implementation hardcodes the database `book_app`. Wrapping this library, abstracting a `Repository` object, and/or using a `Configuration` object, are all possible next steps. Moving towards database agnosticism is an even better move towards Rails' `ActiveRecord` 

2. Tests

   We didn't write any tests for `Modelable`, and that's because we couldn't see where we were going before we got there. Now is a good time to write some tests and drive this out again, or at least writing some integration tests that describe the current behavior.

3. Extend to Other Active Record Methods

   We've implemented `.find()`, and have established a good abstraction patter in our module. It should be fairly straight forward to implement other common methods like `#save()`, `.all`, `#update()`, `#destroy`, etc. Soon, more patterns are sure to surface and lead to further abstractions of the module. This, of course, could really benefit from TDD!

4. Remove String Interpolation from SQL Queries

   See [Day_3](../Day_3)

5. Replace `nil` with `Null` Objects?

   Above, we talked about the downfalls of using `nil` to take on many meanings throughout the code base. Ruby is notorious for this, but often times the Null Object Pattern can step in and provide our `nil`s with more context. Perhaps this could be usefully employed to solve our `nil` problem. 

[← Day 4](/labs/peel/Day_4)
