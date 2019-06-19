---
layout: tutorial
title: Building a JSON:API Backend with JSON API .Net Core
logo: /images/dotnetcore.svg
logo_alt: .NET Core logo
---

[JSON API .Net Core][json-api-dotnet-core] is a library for creating JSON:API backends in Java, and features integration with Spring Framework.

First, [install .NET Core][install-dotnet-core].

Create a new .NET Core Web API:

```bash
$ mkdir OpinionAte
$ cd OpinionAte
$ dotnet new webapi
```

This will create an app configured to serve up a web service.

Next, let's install the JSON API .Net Core library:

```bash
dotnet add package JsonApiDotnetCore
```

## Models
JSON API .Net Core (JANC (😄)) provides classes and attributes for defining your models in a way JANC can work with. To start building our app we'll create these models.

First let’s create a model representing a restaurant. Create a `Models` folder at the root of your project, then a `Models/Restaurant.cs` and add the following:

```csharp
using System.Collections.Generic;
using JsonApiDotNetCore.Models;

public class Restaurant : Identifiable
{
    [Attr("name")]
    public string Name { get; set; }

    [Attr("address")]
    public string Address { get; set; }
}
```

This defines a model with two `string` fields on it: `name` and `address`.

Now let’s set up the model for a dish itself. Create `Models/Dish.cs` and add the following:

```csharp
using JsonApiDotNetCore.Models;

public class Dish : Identifiable
{
    [Attr("name")]
    public string Name { get; set; }

    [Attr("rating")]
    public int Rating { get; set; }
}
```

Now that our two models exist, we can connect them to one another. In `Dish.cs` add the following fields to give it a reference to a `Restaurant`:

```diff
     [Attr("rating")]
     public int Rating { get; set; }
+
+    [HasOne("restaurant")]
+    public virtual Restaurant Restaurant { get; set; }
+    public int RestaurantId { get; set; }
 }
```

Now let's add a collection of `Dish`es to `Restaurant`:

```diff
     [Attr("address")]
     public string Address { get; set; }
+
+    [HasMany("dishes")]
+    public virtual List<Dish> Dishes { get; set; }
 }
```

Now that our models are set up, we need an Entity Framework `DbContext` to handle them. Create `Models/AppDbContext.cs`:

```csharp
using Microsoft.EntityFrameworkCore;

public class AppDbContext : DbContext
{
    public AppDbContext(DbContextOptions<AppDbContext> options) : base(options) { }

    public DbSet<Restaurant> Restaurants { get; set; }

    public DbSet<Dish> Dishes { get; set; }
}
```

Let's hook up this `DbContext` in our `Startup.cs` file. For the sake of this tutorial, we'll just use an in-memory database:

```diff
+using Microsoft.EntityFrameworkCore;
...
     public void ConfigureServices(IServiceCollection services)
     {
         services.AddMvc().SetCompatibilityVersion(CompatibilityVersion.Version_2_2);
+
+        services.AddDbContext<AppDbContext>(opt =>
+        {
+            opt.UseInMemoryDatabase("OpinionAte");
+        });
     }
```

Now that our models are set up, we can create some records. When using JANC, a convenient place to seed the database is in the `Startup.Configure()` method:

```diff
-public void Configure(IApplicationBuilder app, IHostingEnvironment env)
+public void Configure(IApplicationBuilder app, IHostingEnvironment env, AppDbContext context)
 {
     if (env.IsDevelopment())
...
     }

+    context.Database.EnsureCreated();
+    if (context.Restaurants.Any() == false)
+    {
+        Restaurant sushiPlace = new Restaurant { Name = "Sushi Place" };
+        Restaurant burgerPlace = new Restaurant { Name = "Burger Place" };
+        context.Restaurants.Add(sushiPlace);
+        context.Restaurants.Add(burgerPlace);
+        context.SaveChanges();
+
+        context.Dishes.Add(new Dish {
+            Restaurant = sushiPlace,
+            Name = "California Roll"
+        });
+
+        context.Dishes.Add(new Dish {
+            Restaurant = sushiPlace,
+            Name = "Volcano Roll"
+        });
+
+        context.Dishes.Add(new Dish {
+            Restaurant = burgerPlace,
+            Name = "Barbecue Burger",
+        });
+
+        context.Dishes.Add(new Dish {
+            Restaurant = burgerPlace,
+            Name = "Slider"
+        });
+
+        context.SaveChanges();
+    }
...
 }
```

Note that we can assign a `Restaurant` to a `Dish`'s `Restaurant` field, and we don't have to mess with ID fields directly.

## Setting Up the Web Service

Now that we’ve got our data all set, let’s set up our JANC controllers so we can access it via a web service.

Create a `Controllers/RestaurantsController.cs` file and add the following:

```csharp
using JsonApiDotNetCore.Controllers;
using JsonApiDotNetCore.Services;
using Microsoft.Extensions.Logging;

public class RestaurantsController : JsonApiController<Restaurant>
{
    public RestaurantsController(
        IJsonApiContext jsonApiContext,
        IResourceService<Restaurant> resourceService,
        ILoggerFactory loggerFactory)
        : base(jsonApiContext, resourceService, loggerFactory)
    { }
}
```

That's it--we don't need to add any functionality, because JANC's parent class provides it for us.

Now let's make one for `Dish`es, `Controllers/DishesController.cs`:

```csharp
using JsonApiDotNetCore.Controllers;
using JsonApiDotNetCore.Services;
using Microsoft.Extensions.Logging;

public class DishesController : JsonApiController<Dish>
{
    public DishesController(
        IJsonApiContext jsonApiContext,
        IResourceService<Dish> resourceService,
        ILoggerFactory loggerFactory)
        : base(jsonApiContext, resourceService, loggerFactory)
    { }
}
```

Now we just need to hook up JANC when our app boots. Add the following lines to `Startup.cs`:

```diff
+using JsonApiDotNetCore.Extensions;
...
     public void ConfigureServices(IServiceCollection services)
     {
         services.AddMvc().SetCompatibilityVersion(CompatibilityVersion.Version_2_2);

         services.AddDbContext<AppDbContext>(options =>
         {
             options.UseInMemoryDatabase("OpinionAte");
         });
+
+        services.AddJsonApi<AppDbContext>();
     }
...
     public void Configure(IApplicationBuilder app, IHostingEnvironment env, AppDbContext context)
     {
...
         app.UseHttpsRedirection();
         app.UseMvc();
+        app.UseJsonApi();
     }
```

Now, our app should be set. The `JsonApiController` subclasses we created will set up all necessary routes. For example, for restaurants, the following main routes are created:

- GET /restaurants — lists all the restaurants
- POST /restaurants — creates a new restaurant
- GET /restaurants/:id — gets one restaurant
- PATCH or PUT /restaurants/:id — updates a restaurant
- DELETE /restaurants/:id — deletes a restaurant

That’s a lot we’ve gotten without having to write almost any code!

## Trying It Out

Now let’s give it a try. Start the .NET server:

```bash
$ dotnet run
```

Visit `https://localhost:5001/restaurants/1` in your browser and accept the security certificate. You should see something like the following:

```json
{
  "data": {
    "attributes": {
      "name": "Sushi Place",
      "address": null
    },
    "relationships": {
      "dishes": {
        "links": {
          "self": "https://localhost:5001/restaurants/1/relationships/dishes",
          "related": "https://localhost:5001/restaurants/1/dishes"
        }
      }
    },
    "type": "restaurants",
    "id": "1"
  }
}
```

If you’re using [Firefox][firefox] you should see the JSON data nicely formatted. If your browser doesn’t automatically format JSON, you may be able to find a browser extension to do so. For example, for Chrome you can use [JSONView][jsonview].

This is a JSON:API response for a single record. Let’s talk about what’s going on here:

- The top-level `data` property contains the main data for the response. In this case it’s one record; it can also be an array.
- `attributes` is an object containing all the attributes we exposed. They are nested instead of directly on the record to avoid colliding with other standard JSON:API properties like `type`.
- `relationships` provides data on the relationships for this record. In this case, the record has a `dishes` relationship. Two `links` are provided to get data related to that relationship:
	- The `self` link conceptually provides the relationships themselves, which is to say just the IDs of the related records
	- The `related` link provides the full related records.
- Even if you can infer the type of the record from context, JSON:API records always have a `type` field recording which type they are. In some contexts, records of different types will be intermixed in an array, so this keeps them distinct.
- The record contains an `id` property giving the record’s publicly-exposed ID, which by default is the database integer ID. But JSON:API IDs are always exposed as strings, to allow for the possibility of slugs or UUIDs.

Try visiting the `related` link, `https://localhost:5001/restaurants/1/dishes`, in the browser. You’ll see the following:

```json
{
  "data": [
    {
      "attributes": {
        "name": "California Roll",
        "rating": 0
      },
      "relationships": {
        "restaurant": {
          "links": {
            "self": "https://localhost:5001/dishes/1/relationships/restaurant",
            "related": "https://localhost:5001/dishes/1/restaurant"
          },
          "data": {
            "type": "restaurants",
            "id": "1"
          }
        }
      },
      "type": "dishes",
      "id": "1"
    },
    {
      "attributes": {
        "name": "Volcano Roll",
        "rating": 0
      },
      "relationships": {
        "restaurant": {
          "links": {
            "self": "https://localhost:5001/dishes/2/relationships/restaurant",
            "related": "https://localhost:5001/dishes/2/restaurant"
          },
          "data": {
            "type": "restaurants",
            "id": "1"
          }
        }
      },
      "type": "dishes",
      "id": "2"
    }
  ]
}
```

Note that this time the `data` is an array of two records. Each of them also has their own `relationships` getting back to the `restaurants` associated with the record. These relationships are where JR really shines. Instead of having to manually build routes, controllers, and queries for all of these relationships, JR exposes them for you. And because it uses the standard JSON:API format, there are prebuilt client tools that can save you the same kind of code on the frontend!

Next, let’s take a look at the restaurants list view. Visit `https://localhost:5001/restaurants` and you’ll see all the records returned.

Next, let’s try creating a record. We won’t be able to do this in the browser; we’ll need a more sophisticated web service client to do so. One good option is [Postman][postman]—download it and start it up. Go to Preferences > General and turn off "SSL certificate verification"--this will allow Postman to accept the app's self-signed SSL certificate.

You can use Postman for GET requests as well: set up a GET request to `https://localhost:5001/restaurants` and see how it displays the same data as the browser.

Next, let’s create a POST request to the same URL, `https://localhost:5001/restaurants`. Go to the Headers tab and enter key “Content-Type” and value “application/vnd.api+json”—this is the content type JSON:API requires.

Next, switch to the Body tab. Leave the dropdown as "Text"; if you change it to "JSON", Postman will change the "Content-Type" to "application/json", which our server won't accept. Enter the following:

```json
{
  "data": {
    "type": "restaurants",
    "attributes": {
      "name": "Spaghetti Place",
      "address": "789 Third Street"
    }
  }
}
```

Notice that we don't have to provide an `id` because we’re relying on the server to generate it. And we don’t have to provide the `relationships` or `links`, just the `attributes` we want to set on the new record.

Now that our request is set up, click Send and you should get a “201 Created” response, with the following body:

```json
{
  "data": {
    "attributes": {
      "name": "Spaghetti Place",
      "address": "789 Third Street"
    },
    "relationships": {
      "dishes": {
        "links": {
          "self": "https://localhost:5001/restaurants/3/relationships/dishes",
          "related": "https://localhost:5001/restaurants/3/dishes"
        }
      }
    },
    "type": "restaurants",
    "id": "3"
  }
}
```

Our new record is created and the data is returned to us!

Let’s see how we can create related data as well. To add a new dish associated with restaurant 1, `POST` to `http://localhost:5001/dishes`:

```json
{
  "data": {
    "type": "dishes",
    "attributes": {
      "name": "Chicken Fettucine Alfredo",
      "rating": 4
    },
    "relationships": {
      "restaurant": {
        "data": {
          "type": "restaurants",
          "id": "3"
        }
      }
    }
  }
}
```

Notice that now, instead of `links` inside the relationship, we provide `data` that specifies the type and ID of the record the dish is related to.

If you’d like to try out updating and deleting records:

- Make a `PUT` request to `https://localhost:5001/restaurants/3`, passing in updated `attributes`.
- Make a `DELETE` request to `https://localhost:5001/restaurants/3` with no body to delete the record.

## There’s More
We’ve seen a ton of help JSON API .Net Core has provided us: the ability to create, read, update, and delete records, including record relationships. But it offers a lot more too! It automatically exposes Rails validation errors, allows you to request only a subset of the fields you need, allows you to include related records in the response, as well as sorting, filtering, and pagination. To learn more, check out [the JSON API .Net Core guide][json-api-dotnet-core].

Now that you have a JSON:API backend, you should try connecting to it from the frontend. Choose a tutorial from the [How to JSON:API home page](/)!

[firefox]: https://getfirefox.com
[install-dotnet-core]: https://dotnet.microsoft.com/download
[jsonview]: https://chrome.google.com/webstore/detail/jsonview/chklaanhfefbnpoihckbnefhakgolnmc
[json-api-dotnet-core]: https://json-api-dotnet.github.io/JsonApiDotNetCore/
[postman]: https://www.getpostman.com/
