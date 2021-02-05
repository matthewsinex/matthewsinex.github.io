---
title: "How to Get the Base URL in ASP.NET"
date: "2018-09-28"
categories: 
  - "asp-net-core"
---

How to get the base URL in ASP.NET is one of those trivial things that I can never seem to remember. When I've gone to look it up, a dozen different sources tell me a dozen different things. That’s partly because the context matters. Are you in the controller? The view? A separate class? Let’s comb through the ways to get this done and talk about how they work. We'll also look at the different ways to do this in .NET Standard with MVC and with .NET Core and dependency injection.

![](//pluralsight.pxf.io/i/1316549/480967/7490)

## How to Get the Base URL in an MVC Controller

Here’s a simple one-liner to get the job done.

var baseUrl = string.Format("{0}://{1}{2}", Request.Url.Scheme, Request.Url.Authority, Url.Content("~"));

Let’s pull this apart.

The `Request.Url` object is really useful for building URLs. It contains all the information regarding the URL that was requested by the browser.

For instance, imagine that you create a standard MVC application in Visual Studio. Then, you debug the program locally and navigate to the default Contact page, here:

http://localhost:YOURPORT/Home/Contact

Here are some of the more useful properties of `Request.Url`, and what their values would be if you're running locally:

\[table id=22 /\]

We can make a few interesting observations here.

`Authority` and `Host` are almost the same. But `Authority` will include the port number while `Host` will not.

So, in our one-line solution, we take the Scheme (“http”) and combine it with the Authority (“localhost:50808”).

That leaves the `Url.Content("~")` at the end. This is not the Url of `Request.Url`, but the UrlHelper in `Controller.Url`. The method call to  `Url.Content()` converts a relative path to an absolute path. This method call doesn’t make too much of a difference when we’re running locally. It’ll just add the slash ("/") at the end.

However, let's imagine that we’re deploying our application to a subdirectory, like `https://sensibledev/MyApp`. If you omit the Url.Content method call, we lose the subdirectory, and we’ll be left with the address `https://sensibledev` instead.

In fact, let’s continue on that thought, just to further illustrate how Request.Url works. In our app in the subdirectory, here are the different values of the Request.Url properties:

\[table id=23 /\]

Again, we’re assuming that we’re making the request from the Contact action in the Home controller.

With that bit of knowledge in our heads, let’s look at another method of finding the base URL in an MVC controller.

### Solution #2

var urlBuilder =
    new UriBuilder(Request.Url.AbsoluteUri)
    {
        Path = Request.ApplicationPath,
        Query = null,
        Fragment = null
    };

string url = urlBuilder.ToString();

This solution uses the controller’s UriBuilder class. This will let us build a URI from various pieces.

It takes as its argument the `AbsoluteUri` from the request. In our live subfolder example, that would be:

https://sensibledev.com/MyApp/Home/Contact

Then, we need to set some of our object’s properties. The Path property is set to Request.ApplicationPath, which is:

/MyApp/

The `Query` and `Fragment` properties are set to null. This will strip out any query strings or fragments (anything after a hash # sign).

The result, when calling `ToString()` on the resulting object, is exactly the same as in our first method. You can also get the root path by accessing the `Uri` property of urlBuilder:

string url = urlBuilder.Uri;

What you should start to see is that there are a number of ways to get the parts of the request URL. It’s almost frustrating how many options there are. I’d advise you to pick one of these two methods, stick with it, and not worry too much.

![](//pluralsight.pxf.io/i/1316549/512012/7490)

## How to Get the Base URL in an MVC View

Let’s focus our attention on the View now.

Our one-liner will still work, as long as we wrap it in a Razor expression:

@(string.Format("{0}://{1}{2}", Request.Url.Scheme, Request.Url.Authority, Url.Content("~")))

There isn’t much difference here because the Razor view can access the same methods as we did in the controller earlier.

This is a pretty messy solution, however, and not something you’d want to use over and over again in your markup. Remember, once you start copying something more than once, it's wise to find a way to abstract that out into a method.

Instead of the mess above, we can use the built-in `Url.Action` helpers to create an absolute path. Let’s say that we use `Url.Action` like this:

@Url.Action("Index", "Home")

Assuming our app lives in a subdirectory, like `https://sensibledev.com/MyApp/`, the helper will render on the page as:

/MyApp/

If we want to have the fully qualified path, we’ll need to add in the protocol, and leave the `routeValues` parameter null (this is the third parameter).

@Url.Action("Index", "Home", null, "http")

//Renders as:

http://sensibledev.com/MyApp/

You’ll notice that we’ve hard-coded the protocol, which makes this less than ideal. A better solution is to write a custom Url Helper, as described in [this blog post](https://blog.mariusschulz.com/2011/06/30/how-to-build-absolute-action-urls-using-the-urlhelper-class):

/// <summary>
/// Generates a fully qualified URL to an action method by using
/// the specified action name, controller name and route values.
/// </summary>
/// <param name="url">The URL helper.</param>
/// <param name="actionName">The name of the action method.</param>
/// <param name="controllerName">The name of the controller.</param>
/// <param name="routeValues">The route values.</param>
/// <returns>The absolute URL.</returns>
public static string AbsoluteAction(this UrlHelper url,
    string actionName, string controllerName, object routeValues = null)
{
    string scheme = url.RequestContext.HttpContext.Request.Url.Scheme;
 
    return url.Action(actionName, controllerName, routeValues, scheme);
}

You can then write this in your Razor view to get the base URL:

@Url.AbsoluteAction("Index", "Home")

The code underlying the method call is also smart enough to know that the Index action is your home page, so it doesn’t render as `https://sensibledev.com/MyApp/Home/Index`, but rather as `https://sensibledev.com/MyApp/`.

## How to Get the Base URL in a Class Outside the Controller

Let’s say that you have a business layer that sends emails in your application. You want to include links in your emails to the application, and you want to grab the root URL to make those.

This is a little trickier. We’ll have to get at the `Request` object differently. But more importantly, we don’t have access to `Url.Content`.

Here’s how you would get the base URL in a class outside the controller:

/// <summary>
/// Generates the fully qualified base URL for
/// the application.
/// </summary>
/// <returns>The application's base URL.</returns>
public static string GetBaseUrl()
{
    var request = HttpContext.Current.Request;
    var appRootFolder = request.ApplicationPath;

    if (!appRootFolder.EndsWith("/"))
    {
        appRootFolder += "/";
    }

    return string.Format(
        "{0}://{1}{2}",
        request.Url.Scheme,
        request.Url.Authority,
        appRootFolder
    );
}

I’m very grateful to [this](https://stackoverflow.com/a/16691930) StackOverflow post for providing the solution which replaces the need for Url.Content.

## How to Get the Base URL in .NET Core

In this section, I'm assuming that you're using .NET Core's Razor pages instead of MVC views and controllers. So, let's see how to get the root URL...

### In the Razor PageModel

We have just a few differences between .NET Core and .NET MVC. There's no controller, obviously, but you can access the `Request` object directly from the `PageModel`. Also, `Request.Authority` is gone and replaced with `Request.Host`.

We’ll also need to use the Extensions namespace by including this using statement:

using Microsoft.AspNetCore.Http.Extensions;

Here, then, are the same techniques as above, but slightly re-coded to comply with .NET Core.

#### Solution #1

string baseUrl = string.Format("{0}://{1}{2}", Request.Scheme, Request.Host, Request.PathBase);

#### Solution #2

//GetDisplayUrl provides us with the full URL made in the request
var displayUrl = UriHelper.GetDisplayUrl(Request);

var urlBuilder =
new UriBuilder(displayUrl)
{
    Query = null,
    Fragment = null
};
string url = urlBuilder.ToString();

### As Middleware

However, we have the same problem as before, with the code being too verbose to reuse. We could abstract this out to a helper class. But let's leverage .NET Core's built-in dependency injection instead.

A better way to get at the root URL is to build our code as middleware so that we can use it anywhere in our application.

Put these two classes in a separate C# file anywhere you’d like:

public class MyHttpContext
{
    private static IHttpContextAccessor m\_httpContextAccessor;

    public static HttpContext Current => m\_httpContextAccessor.HttpContext;

    public static string AppBaseUrl => $"{Current.Request.Scheme}://{Current.Request.Host}{Current.Request.PathBase}";

    internal static void Configure(IHttpContextAccessor contextAccessor)
    {
        m\_httpContextAccessor = contextAccessor;
    }
}

public static class HttpContextExtensions
{
    public static IApplicationBuilder UseHttpContext(this IApplicationBuilder app)
    {
        MyHttpContext.Configure(app.ApplicationServices.GetRequiredService<IHttpContextAccessor>());
        return app;
    }
}

And then add a few lines to the methods in _Startup.cs_:

public class Startup
{
    public IConfiguration Configuration { get; }

    // This method gets called by the runtime. Use this method to add services to the container.
    public void ConfigureServices(IServiceCollection services)
    {
        //I've omitted a good deal of code here to just include what you need to add

        services.AddSingleton<IHttpContextAccessor, HttpContextAccessor>();
        //you MAY need to put AddSingleton before AddMvc, I'm not sure if the order matters 
        services.AddMvc().SetCompatibilityVersion(CompatibilityVersion.Version\_2\_1);
    }

    // This method gets called by the runtime. Use this method to configure the HTTP request pipeline.
    public void Configure(IApplicationBuilder app, IHostingEnvironment env)
    {
        app.UseHttpContext();
        //other app.Use... statements omitted

        app.UseMvc();
    }
}

A big thank you to [this StackOverflow post](https://stackoverflow.com/a/47051481). I had to update a few things to account for some changes in .NET Core 2.1.

Now, you can just use this middleware anywhere (including the Razor views):

MyHttpContext.AppBaseUrl

## How to Get the Base URL in ASP.NET: Conclusion

We’ve looked at a number of different ways to get the base URL in ASP.NET and .NET Core. There are a lot of options (almost too many), but I find the ones discussed here to be simple enough to get the job done. I also like how this shows how fundamental dependency injection is in .NET Core, and how it can make some tasks a lot easier than in .NET Standard.

Happy, sensible coding!
