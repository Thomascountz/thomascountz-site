---
layout: lab
---

[← Back to Lab](/labs/peel)

# Day 0

## The Project

The goal of this project is to implement a feature-light ORM in Ruby, modeled after the Active Record _pattern_ and inspired by Gregory Brown's [Broken Record project](https://practicingruby.com/articles/implementing-the-active-record-pattern-1).

"Pattern?" Up until this point, I knew about the `ActiveRecord` ORM that ships with Ruby on Rails, but I wasn't aware that this was an implementation of the _active record pattern_...

## Init Research

In _Patterns of Enterprise Application Architecture_, Martin Fowler introduces the Active Record pattern as:

> _...an object that wraps a row in a database table or view, encapsulates the database access, and adds domain logic on that data..._

This sounds like how one might describe Ruby on Rails' `ActiveRecord`, and that's no mistake. In the Ruby on Rails guides' [_Active Record Basics_](https://guides.rubyonrails.org/active_record_basics.html#the-active-record-pattern), they describe it like so:

> It is an implementation of the Active Record pattern which itself is a description of an Object Relational Mapping system... In Active Record, objects carry both persistent data and behavior which operates on that data.

Let's say I have a database table called `users`, which has three columns, `id`, `firstname`, and `lastname`. I could represent this table, and some data in it, like this:

```
| id | firstname | lastname |
|----|-----------|----------|
|  0 | Simon     | Parker   |
|  1 | Mary      | Souza    |
|  2 | Tristen   | Lu-Chen  |
```

If I want to get the data for `Mary Souza`, I might write an SQL query like this:

```sql
SELECT * FROM users WHERE id = 1;
```

And I could expect to get some data back like this:

```
| id | firstname | lastname |
|----|-----------|----------|
|  1 | Mary      | Souza    |
```

However, from my _application_, I might instead want to write something like this:

```
user = User.select_where(id: 1)
```

And get back and object like this:

```
#{User "id": 1, "firstname": "Mary", "lastname":"Souza"}
```

And supply behavior to that data, such as with a method called `fullname`:

```
user.fullname
#=> "Mary Souza"
```

This object is an Active Record object! 

It 1) wraps data from a row in the database, 2) encapsulates the retrieval of that data with the `.select_where` method, and 3) adds domain-specific behavior, with the `.fullname` method.

Martin Fowler extends this descrption with the following:

>The Active Record class typically has methods that do the following:
>
>- Construct an instance of the Active Record from a SQL result set row
>- Construct a new instance for later insertion into the table
>- Static finder methods to wrap commonly used SQL queries and return Active Record objects
>- Update the database and insert into it the data in the Active Record
>-  Get and set the fields
>- Implement some pieces of business logic


This sounds pretty good, and more often than not, if you've used Ruby on Rails, you are familiar with the core feature-set of the active record implementation. However, throughout the apprenticeship, I've learned to always take a moment ask if this is the most cost-effective solution.

**Side Note**: What defines "cost" will always vary widely from project to project, feature to feature. Some general things to keep in mind are how easy a solution is to implement/maintain, how much time will the implementation take, does it directly solve our problem, is the expected return greater than the cost of delivery, and does it matter? The definition of cost must be defined for every feature, so for our purposes, I'll define cost as "does it directly solve our problem," and with that, we must first identify the problem!

For this project, the problem is, "Thomas needs a fun and exciting project to work on, he could benefit with some more time working with databases, and he loves working with Ruby, what should he build?" But, pushing that meta-problem aside, what problem does the active record pattern solve/when would we generally aim to implement it?

> Active Record is a good choice for domain logic that isn’t too complex, such as creates, reads, updates, and deletes. Derivations and validations based on a single record work well in this structure... Their primary problem is that they work well only if the Active Record objects correspond directly to the database tables... Another argument against Active Record is the fact that it couples the object design to the database design. 
>
> Martin Fowler, _Patterns of Enterprise Application Architecture_

## What's The Problem

Let's invent a little MVP scenario to get shake things loose and get us thinking creatively. 

> We want to build a simple CRUD app to manage a global list of books in our library. We should be able to record a book's title, author, and ISBN number.

The Active Record pattern solves our scenario. Let's also say that we've weighed all of the costs and the decision to implement Active Record has gotten the green light!

Where do we go from here?

## Initial Thoughts on Implementing Active Record

Without doing any more research at this point, I'll begin by reiterating what I understand as the three major features that need to be implemented in order to deliver an Active Record object:

1. Abstract database connections/queries
2. Encapsulate data into an "Active Record" object
3. Provide an interface to add domain-specific behavior

In the example presented in _Patterns of Enterprise Application Architecture_, Martin Fowler's houses all of this within a single object, and I think I'll begin with that approach. 

If we can implement a single Active Record object that connects to the database, wraps data, and provides an interface to that data, then maybe we can iterate until we end up with a reusable abstracted interfaces for composing new domain-specific Active Record objects. 


[Day 1 →](/labs/peel/Day_1)
