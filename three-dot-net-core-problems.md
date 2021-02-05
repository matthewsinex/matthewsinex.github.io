---
title: "Three .NET Core Problems, and How I Solved Them"
date: "2020-01-14"
---

Sometimes it takes a while to get with the times. Even though it's been out and stable for a number of years, I still hadn't taken the plunge with .NET Core. All of our projects at work were still being done with .NET Framework in Visual Studio 2015. In this post, what I'd like to do is to share some thoughts about migrating from to .NET Core, some stumbling blocks I found along the way, and how I fixed them.

To be fair, I had dabbled in .NET Core for a while, as you might notice from a number of posts in this blog. I've done a couple of small personal projects in the framework, but nothing I took too seriously or that was worth bragging about.

To my workplace's credit, when I brought up the idea of creating our newest project in .NET Core, I didn't meet any resistance. It can really help to work in an organization that encourages experimentation and education.

So, here are my top three things that confused me about .NET Core, and how I fixed them.

## Problem: Entity Framework and Raw SQL

All of my work projects obviously contain some tables that are unique to that project. However, in our organization, almost every project needs to reference existing legacy databases, some of them created and utilized by vendors.

Since I'm joining on that data a lot, a good deal of my queries need to be written in raw SQL.

In Entity Framework, this was easy enough to do with a view model. Just make a POCO model, then write code like:

string sql = @"
SELECT
  \*
FROM
  Users U
  INNER JOIN HREmployees HR ON HR.EmployeeID=U.EmployeeID
WHERE
  HR.HireDate > DATEADD(m, -6, GetDate())
";

var newHires = context
  .Database
  .SqlQuery<NewHireView>(sql)
  .ToList();

And we'd be good. You could also return primitive types, like a list of `int`s if you needed to.

Entity Framework Core is pickier.

In .NET Core, raw SQL queries can only be used on entities that are listed in your context as a `DBSet`.

So, I'd have to, in addition to creating the POCO, include it in the `DbContext`.

\[NotMapped\]
public DbSet<NewHireView> HireView { get; set; }

And then the database call would look like:

var newhires = context.HireView.FromSql(sql).ToList();

Is this really that big of a deal? I mean, not _really_, though it is annoying to have to add the extra lines (and having to remember to do that every time I make a new view model). What I really don't like is that I now have view models polluting my list of entities when I'm using Intellisense on a DbContext. In any given project, I can also have a _lot_ of different view models, and I don't like the clutter in both the autocompletion and in the DbContext.

## Solution: Dapper

I like Dapper a lot, and end up using it throughout projects anyway, so with this project I decided to use it from the beginning.

Using [this blog post](https://exceptionnotfound.net/using-dapper-asynchronously-in-asp-net-core-2-1/) as a guide, I can make repositories like this:

public interface IUserRepository
{
    Task<IEnumerable<NewHireView>> GetNewHireView();
}

public class UserRepository : IUserRepository
{
  private readonly IConfiguration \_config;

  public UserRepository(IConfiguration config)
  {
    \_config = config;
  }

  public IDbConnection Connection
  {
    get
    {
      return new SqlConnection(\_config.GetConnectionString("Default"));
    }
  }

  public async Task<IEnumerable<NewHireView>> GetNewHireView();
  {
    using IDbConnection conn = Connection;
    string sql = @"
    SELECT
      \*
    FROM
      Users U
      INNER JOIN HREmployees HR ON HR.EmployeeID=U.EmployeeID
    WHERE
      HR.HireDate > DATEADD(m, -6, GetDate())
    ";

    return await conn.QueryAsync<NewHireView>(sql);
  }
}

No problem with mapping data to arbitrary POCO models. Now, having done this, I realize that this is way more code than just adding an extra two lines in the `DbContext`, but I like the other features that Dapper affords, like list support, multi mapping, and multiple results. So, meh.

## Problem: Understanding Dependency Injection

If you already know about and understand DI, you can probably skip this explanation. If you're like I was in the beginning of this, stick with me for just a second.

You notice how, in the UserRepository we just encountered, that we pass in the `IConfiguration` object during construction instead of just declaring it directly? That's an instance of dependency Injection.

You can also see something similar in your razor pages, with code like:

private readonly IRolesRepository \_roles;

public DetailsModel(IRolesRepository roles)
{
  \_roles = roles;
}R

public RoleView Role { get; set; }

public async Task<IActionResult> OnGetAsync(int? id)
{
  if (id == null)
  {
    return NotFound();
  }

  Role = await \_roles.GetByID((int)id);

  if (Role == null)
  {
    return NotFound();
  }
  return Page();

The question you should ask yourself at this point is: how does the framework determine which implementation of `IRolesRepository` to use?

It all happens on this line in **Startup.cs**:

services.AddTransient<IRolesRepository, RolesRepository>();

In other words, we've told .NET Core that whenever the project uses the `IRolesRepository` interface, use the implementation as written in `RolesRepository`.

So, okay, that's the _how_, which took a little getting used to. What I was more confused by is the _why_.

Why is this better than just declaring the service as a static helper class? With dependency injection, I have to create an interface and register the service every time.

Well, here's [what Microsoft says](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/dependency-injection?view=aspnetcore-3.0), as well as my thoughts on each point:

> To replace MyDependency with a different implementation, the class must be modified.

That's true, I suppose. However, and this is just me, personally, if I need to modify a class, I just... modify it. I can't think of a single time when I needed to rewrite an implementation of a class, but keep both the new implementation and old implementation in the live project at the same time.

Not saying it doesn't happen, just saying I can't recall a time when this was actually a problem for me.

> If MyDependency has dependencies, they must be configured by the class. In a large project with multiple classes depending on MyDependency, the configuration code becomes scattered across the app.

Okay, I'm trying to wrap my head around this. I can sort of see why this would be useful, but again, I just can't think of a project I've built (and I've built some decent-sized ones) where this was an issue.

> This implementation is difficult to unit test. The app should use a mock or stub MyDependency class, which isn't possible with this approach.

Well, this is the point where I admit that our organization doesn't unit test. I'm not going to argue against unit testing. In fact, it's probably something that we should be doing. But the fact is, right now, we don't have that as part of our workflow.

But I can definitely understand why DI would be useful in that case.

So, in total, I either don't understand the reasoning behind it or the reasons don't apply to my situation.

## Solution: Grin and Bear it?

This may seem like a cop-out, but on this one I'm just going to keep at it and see what happens. I'm trusting (at least temporarily) that there's a good reason for this industry standard.

Hey, I'm open to new experiences! Using DI seems baked in to .NET Core, so at least for now, I'm just going to keep moving on this project and see where it goes. DI doesn't seem to be _hurting_ anything, at any rate.

## Problem: Figuring out Permissions

At my workplace, authorization doesn't fall into the neat boxes that most of the .NET Core tutorials cover. It's easier to, for example, give all department heads limited admin control over their own departments than to manually swap out permissions on an individual. With a few thousand employees, so there's a moderate amount of turnover.

However, this information comes from an external database instead of Active Directory. So, while we use Windows authentication as a basis, it's not a complete solution by itself. Almost none of the tutorials out there cover this kind of situation.

## Solution: Custom Claims Transformation

Nonetheless, I have figured out that using custom _Claims_ is part of the recommended solution.

A claim is just something that you know about the user, like their ID or job description.

Windows authentication provides you with some limited claims about the user, like their name, but not much else.

In any case, my first step was to create a class that implements the `IClaimsTransformation` interface. As you might imagine from the name, you are transforming the claims into something else. Here's an early version of the class I made ([this blog post](https://philipm.at/2018/aspnetcore_claims_with_windowsauthentication.html) was really helpful):

public class ClaimsLoader : IClaimsTransformation
{
  public const string RoleKey = "Role";
  private readonly IHttpContextAccessor \_httpContextAccessor;
  private readonly IUserRepository \_userRepository;

  public ClaimsLoader(IHttpContextAccessor httpContextAccessor, IUserRepository userRepository)
  {
    \_httpContextAccessor = httpContextAccessor;
    \_authorization = authorization;
  }

  public async Task<ClaimsPrincipal> TransformAsync(ClaimsPrincipal principal)
  {
    var identity = (ClaimsIdentity)principal.Identity;

    // create a new ClaimsIdentity copying the existing one
    var claimsIdentity = new ClaimsIdentity(
        identity.Claims,
        identity.AuthenticationType,
        identity.NameClaimType,
        identity.RoleClaimType);

    var username = \_httpContextAccessor.HttpContext
                      .User.Identity.Name;

    //the user repository is a service I wrote
    //to get the user object from a database query                  
    User user = await \_userRepository.GetUser(username);

    if(user != null)
    {
      //the roles enum is a list of possible roles, like "Admin"
      foreach(var p in user.Permissions)
      {
        var userRole = Enum.GetName(typeof(Roles), p.Role);

        claimsIdentity.AddClaim(
            new Claim(RoleKey, userRole));
      }
    }

    return new ClaimsPrincipal(claimsIdentity);
}

So, that gets me a lot of the way to where I want to be. By the end of it, we now have a list of the User's custom roles in the user's claims. If we wanted to do something like check whether a user was an admin, I could register an authorization policy in **Startup.cs**.

services.AddAuthorization(options =>
{
    options.AddPolicy(
        "IsAdmin",
        policy => policy.RequireClaim(ClaimsLoader.RoleKey, "Admin"));
});

Finally, in the page model of a razor page, I could use this line to indicate a user has to be an Admin to access the page:

\[Authorize(Policy = "IsAdmin")\]

Or, if I wanted an entire folder to fall under that policy, you'd add this to **Startup.cs**:

services.AddRazorPages(options => {
    options
    .Conventions
    .AuthorizeFolder("/Admin/Roles", "IsAdmin");
});

There are a few things I don't like about that implementation. It makes a database request every time every page loads, which I don't find necessary when you can just store that in session.

Later versions of this led to me to doing exactly that: storing the User object in session.

The claims method, as written, also doesn't store the user's department, which is going to be important moving forward. (Department heads often only get access to data regarding their own department.)

When I get to that part of the project, I may need to come back to this post and write an update.

## Learning About .NET Core: Conclusion

These three hurdles notwithstanding, I really like .NET Core. It seems to run faster, and Razor Pages seems to organize the code in a way that I find less confusing than having the view and controller completely separate.

I obviously still have a lot to learn and hope to write more about this journey as I progress. Happy coding!
