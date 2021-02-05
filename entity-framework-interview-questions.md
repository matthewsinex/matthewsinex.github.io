---
title: "Entity Framework Interview Questions"
date: "2018-06-14"
categories: 
  - "entity-framework"
---

Every developer who has started to learn .NET web development encounters the Entity Framework in the beginning tutorials. Entity Framework, and other ORMs, are ubiquitous in modern web development. I've put together this list of Entity Framework interview questions based on my own experience with the framework, including common "gotchas." This knowledge could help you out the next time you're in the interview room, and allow you to show off your Entity Framework know-how.

### What is Entity Framework?

Entity Framework is an ORM (Object-Relationship-Mapper) that lets developers work with databases using objects instead of focusing on the underlying database directly.

For instance, the database could have a table called `Customers`. Instead of creating SQL statements to create and read rows from this table, Entity Framework could **map** the values in the table to a C# object, also called `Customers`. You can then work on those objects just like you would any other C# object. Entity Framework helps to separate concerns, simplify queries, and reduce overall development time.

Entity Framework can interface with a variety of database systems, not just SQL Server. Compatible systems include SQLite, MySQL, and PostgreSQL. You can find a complete list at Microsoft's [page about database providers](https://docs.microsoft.com/en-us/ef/core/providers/).

Also note that Entity Framework is not the only ORM for .NET environments. Other ORMs include [Dapper](https://github.com/StackExchange/Dapper) and [Tortuga Chain](https://github.com/docevaad/Chain).

### What is a model?

A model is a class that represents data. In terms of Entity Framework, a model represents the data from a table in the database. For example, your `Customer` model might look like this:

_Customer.cs_

public class Customer
{
  public int ID { get; set; }
  public string Name { get; set; }
  public DateTime JoinDate { get; set; }
}

This model would represent a table in the database called _Customers_. Each of the properties in this model class represents a column in the table, including the name and datatype of the column.

The concept of a model is not specific to Entity Framework, and is utilized by any number of ORMs across languages.

### What is a DbContext?

The `DbContext` class manages the different model classes and is used to perform actions on the database.

For example, in MVC 5, a typical DbContext class might look like this:

public class ApplicationContext : DbContext
{
  public ProjectContext() : base("name=default") //name of connection string
  {
  }

  public DbSet<Customer> Customers { get; set; }
  public DbSet<Orders> Orders { get; set; }
}

The DbContext class can then be used to generate tables, as well as MVC controllers and views. You would use the DbContext class to perform read, write, and delete operations on the tables, as well.

### What development methods can you use in Entity Framework?

Entity Framework offers three different development methods.

#### Database First

In database first development, the tables already exist before you write any code. You would simply make the database tables yourself, possibly using SQL Server management studio. Then, you could either write the model classes by hand, or you could [use Visual Studio's wizards to connect to the database and generate the models for you](https://docs.microsoft.com/en-us/aspnet/mvc/overview/getting-started/database-first-development/creating-the-web-application). It's also possible to create these models from your existing database using the Package Manager Console. Specifically, you would use this command:

Scaffold-DbContext "Server=(localdb)\\mssqllocaldb;Database=MyDb;Trusted\_Connection=True;" Microsoft.EntityFrameworkCore.SqlServer -OutputDir Models

#### Code First

Code first development requires you to code the models first. You would create C# classes, like the example earlier. Then, after creating the DbContext, you would run the following commands in the Package Manager Console:

`enable-migrations`

`add-migration InitialCreate`

`update-database`

Provided you have everything set up correctly, Entity Framework will create the tables in the database that you've specified in your connection string. It will also set up primary key and foreign key relationships, as long as you've named your properties conventionally. (For example, your primary key columns are name "ID" and your foreign keys are named something like "CustomerID".)

#### Model First

Model first is not very common, and typically not included in current tutorials about .NET development. You use the in-built menus in Visual Studio to create a database schema. This is stored in an EDMX file.

To start the process, right click your project, and then select _Add -> New Item..._ Choose **ADO.NET Entity Data Model** and you'll be guided through a wizard where you will create the models and properties.

I personally haven't heard much about this method, and it doesn't even seem to be included in .NET Core, as far as I can tell. Don't worry about this method as much as the other two.

### How can you indicate a different table or column name than one defined in the class?

You can use **data annotations** to indicate that the columns or tables are named differently from your model properties. This can be especially helpful when working with legacy tables that have odd naming conventions, and you're trying to clean up the names when working in your application. In this example, the table in the database is called "tblCustomer", and the Name field in the database is unhelpfully titled "custNm" :

using System.ComponentModel.DataAnnotations.Schema;

\[Table("tblCustomer")\]
public class Customer
{
    public int ID { get; set; }

    \[Column("custNm")\]
    public int Name { get; set; }

    public DateTime JoinDate { get; set; }
}

### What is LINQ, and how does it relate to Entity Framework?

LINQ is a way of writing SQL-style queries in C#. It stands for _Language Integrated Query_. You can use LINQ to query a number of datastores, including databases, XML files, and C# collections. Queries in Entity Framework are formed using LINQ syntax.

### How would you implement the standard CRUD operations for a given table?

LINQ provides two basic methods of syntax: method and query syntax. I'll provide simple examples of the standard CRUD operations using both method and query syntax when applicable. Assume that, for each of the examples, we have an existing DbContext and an instance of a `Customer` model. I'm thinking something like the following:

ApplicationContext context = new ApplicationContext();
Customer customer = new Customer
{
    ID = 1,
    Name = "John Doe",
    JoinDate = DateTime.Now
};

#### Create

context.Customers.Add(customer);
context.SaveChanges();

Note that, after `SaveChanges` is called, the ID property of the model will be set to whatever the new ID is in the database.

#### Read

Here's how you would get all records:

//method syntax
var customers = context.Customers.ToList();

//query syntax
var customer = from c in context.Customers
                select c;

If you wanted to retrieve just one record by the ID, use the following:

int customerID = 1;

//method syntax
var customer = context.Customers.Where(c => c.ID == customerID).SingleOrDefault();

//query syntax
var customer = (from c in context.Customers
               where c.ID == 1
               select c)
               .SingleOrDefault();

//alternate method, using the primary key
var customer = context.Customers.Find(customerID);

If you're unfamiliar with the `SingleOrDefault` method, I would suggest reading [my article on the topic](https://sensibledev.com/linq-firstordefault-vs-first/), which also covers similar types of LINQ methods.

#### Update

var customer = context.Customers.Where(c => c.ID == customerID).SingleOrDefault();

//make some changes to the customer object model

context.Entry(customer).State = EntityState.Modified;
context.SaveChanges();

#### Delete

context.Customers.Remove(customer);
context.SaveChanges();

### What are navigation properties?

Navigation properties on a model represent the related tables in the database. Another way of thinking about it is that a navigation property represents a foreign key relationship. For instance, if there's an `Orders` table and we add a one-to-many relationship on our _Customer_ model, that might look like this:

public class Customer
{
  public int ID { get; set; }
  public string Name { get; set; }
  public DateTime JoinDate { get; set; }

  public virtual ICollection<Order> Orders { get; set; }
}

public class Order
{
  public int ID { get; set; }
  public int CustomerID { get; set; }
  public DateTime Date { get; set; }

  public virtual Customer Customer { get; set; }
}

The properties marked virtual are the navigation properties.

### What is the difference between lazy loading and eager loading?

Lazy and eager loading are two different ways that Entity will load related data (like the Orders attached to our Customers).

**Lazy loading** means that when an entity is first read, its related entities are _not_ loaded. Entity Framework waits until you try to access those entities before querying the database again. This can lead to multiple database queries. For instance, the following code will result in two different database queries:

var order = context.Orders.Find(id);
var customer = order.Customer;

Used improperly, this can lead to N+1 queries and slow down performance.

**Eager loading** means that when you retrieve an entity, its related properties are loaded at the same time. This typically means that Entity Framework will create a single query with joins. You can specify eager loading by using the `Include` method. The following code is an example of eager loading:

var orderID = 1;
var order = context.Orders
    .Include(x => x.Customer)
    .Where(o => o.ID == orderID)
    .SingleOrDefault();

### What is model binding?

Model binding maps the values from an HTTP request to an object. This is most commonly found in your controller actions. For instance, this is a typical POST method:

\[HttpPost\]
\[ValidateAntiForgeryToken\]
public ActionResult Create(\[Bind(Include = "Name,JoinDate")\] Customer customer)
{
    //write the customer object to the database, do other operations...
}

Once MVC gets the POST request, it will set the Name and JoinDate values to the appropriate properties in the Customer object. You can then use the model object and insert the entity into the database.

### How can you validate the model properties? For example, how would we make a property required?

You can validate the model properties using data annotations. For example, on the Customer model, if you wanted to make the Name field required, you would add the `Required` attribute, like this:

public class Customer
{
  public int ID { get; set; }

  \[Required\]
  public string Name { get; set; }

  public DateTime JoinDate { get; set; }

  public virtual ICollection<Order> Orders { get; set; }
}

This will also automatically validate the forms where this model is used, if you're including the jQuery Unobtrusive Validation script. In the controller action, you would then check to see if the `ModelState` is valid.

\[HttpPost\]
\[ValidateAntiForgeryToken\]
public ActionResult Create(\[Bind(Include = "Name,JoinDate")\] Customer customer)
{
  if (ModelState.IsValid) //check for model errors
  {
    context.Customers.Add(customer);
    context.SaveChanges();
    return RedirectToAction("Index");
  }

  return View(customer);
}

If the Name field doesn't have a value, the controller will return the View again, and display the model errors.

For a better overview of data annotations and other ways you can validate models, you can check out [my post about the compare validator](https://sensibledev.com/compare-validator-in-mvc/).

### How can you execute a Raw SQL statement in Entity Framework?

If you want to write the queries yourself, this can still be done in Entity Framework. Writing your own SQL can sometimes be preferable with more complicated queries.

In MVC 5, use the `Database.SqlQuery` method:

string sql = "SELECT \* FROM Customers";
var customers = context.Database.SqlQuery<Customer>(sql).ToList();

The syntax in .NET Core is a little different. When you're using EF Core, you use the `[Entity].FromSql` method instead.

using Microsoft.EntityFrameworkCore;

string sql = "SELECT \* FROM Customers";
var customers = context.Customers
    .FromSql(sql)
    .ToList();

### Entity Framework Interview Questions: Conclusion

I hope you've found this overview of the Entity Framework useful. If you have any questions about Entity Framework, feel free to post them in the comments. The best way that you can learn about Entity Framework is fire up Visual Studio and start creating a project. Get out there and build something!
