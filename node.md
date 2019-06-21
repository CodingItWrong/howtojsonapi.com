---
layout: tutorial
title: Building a JSON:API Backend with Node and Lux
logo: /images/node.svg
logo_alt: Node.js logo
---

[Lux][lux] is a framework for creating JSON:API backends in Node.

To try it out, let's create a web service for rating dishes at restaurants. We'll call it "Opinion Ate".

_Looking for something slightly different? Check the official JSON:API implementations page for [alternative Node.js server libraries][jsonapi-js-servers]._

First, install Lux globally:

```bash
$ npm install -g lux-framework
```

Create a new Lux app:

```bash
$ lux new opinion-ate
$ cd opinion-ate
```

This will create an app configured to store data in a [SQLite][sqlite] database, which is just a flat file. This is the simplest way to go for experimentation purposes.

## Models
JSON:API represents the data in your apps as "resources", and Lux provides a single command to set up everything you need for a given resource. First let’s create a resource representing a restaurant. Run the following command in the terminal:

```bash
$ lux generate resource restaurant name:string address:string
```

This tells Lux to create a new resource called `restaurant` and to define two `string` fields on it: `name` and `address`.

The generator created a number of files; let’s take a look at each of them.

First, open the file in `db/migrate` that ends with `-create-restaurants.js` — the date on the file will be different than mine, showing the time you ran the command.

```javascript
export function up(schema) {
  return schema.createTable('restaurants', table => {
    table.increments('id');
    table.string('name');
    table.string('address');
    table.timestamps();

    table.index('created_at');
    table.index('updated_at');
  });
}

export function down(schema) {
  return schema.dropTable('restaurants');
}
```

This file contains a *migration*, a class that tells Lux how to make a change to a database. This file will `createTable('restaurants')`, which is just what it sounds like. The arrow function passed to `createTable()` receives a parameter `table`, representing a table. `table.increments()` creates an auto-incrementing primary key column. `table.string()` creates a new string column, and `table.timestamps()` creates `created_at` and `updated_at` columns that Lux will manage for us automatically. Finally, `table.index()` creates database indexes allowing for performant sorting of the `created_at` and `updated_at` columns.

The `restaurants` table hasn’t actually been created yet; the migration file just records *how* to create it. You can run it on your computer, when a coworker pulls it down she can run it on hers, and you can run it on the production server as well. Run the migration now with this command:

```bash
$ lux db:migrate
```

Next let's look at the `app/models/restaurant.js` file created:

```javascript
import { Model } from 'lux-framework';

class Restaurant extends Model {

}

export default Restaurant;
```

That’s…pretty empty. We have a `Restaurant` class that inherits from `Model`, but nothing else. This represents a Restaurant record, but how does it know what columns are available? Lux will automatically inspect the table to see what columns are defined on it and make those columns available; no configuration is needed.

Next, take a look at `app/serializers/restaurant.js`:

```javascript
import { Serializer } from 'lux-framework';

class RestaurantsSerializer extends Serializer {
  attributes = [
    'name',
    'address'
  ];
}

export default RestaurantsSerializer;
```

Serializers translate models to their end-user-facing format. In this case, `name` and `address` are configured as columns that should be made available to end users. If we wanted some columns to only be used on the backend, we could remove them from the serializer.

Finally, take a look at `app/controllers/restaurants.js`:

```javascript
import { Controller } from 'lux-framework';

class RestaurantsController extends Controller {
  params = [
    'name',
    'address'
  ];
}

export default RestaurantsController;
```

For security purposes, controllers don't allow just any arbitrary parameters to be passed in; they need to be explicitly listed in the `params` property. If we wanted some columns to be read-only we could remove them from here.

The last piece of the puzzle that was set up for us is the route. Check out `app/routes.js`:

```javascript
export default function routes() {
  this.resource('restaurants');
}
```

This will set up all necessary routes for restaurants:

- GET /restaurants — lists all the restaurants
- POST /restaurants — creates a new restaurant
- GET /restaurants/:id — gets one restaurant
- PATCH or PUT /restaurants/:id — updates a restaurant
- DELETE /restaurants/:id — deletes a restaurant

Next, let’s create a dish resource:

```bash
$ lux generate resource dish name:string rating:integer restaurant_id:integer
```

You’ve seen a `string` column before, and you can probably guess what `integer` does. We create a `rating` column, as well as a `restaurant_id` that creates a foreign key column that references another model, in this case Restaurant.

This command generates a model, serializer, and controller for `Dish`es, and added the appropriate route.

Go ahead and migrate the database again:

```bash
$ lux db:migrate
```

Although we don't need to configure columns in the model classes, we do need to specify relationships between models. Add a `hasMany` field to `Restaurant` indicating that it has many dishes:

```diff
 class Restaurant extends Model {
+  static hasMany = {
+    dishes: {
+      inverse: 'restaurant',
+    },
+  };
 }
```

And add a `belongsTo` field to `Dish` indicating that it belongs to a restaurant:

```diff
 class Dish extends Model {
+  static belongsTo = {
+    restaurant: {
+      inverse: 'dishes',
+    },
+  };
 }
```

We also need to add these relationships to the serializers. First, `app/serializers/restaurant.js`:

```diff
 class RestaurantsSerializer extends Serializer {
   attributes = [
     'name',
     'address'
   ];
+  hasMany = ['dishes'];
 }
```

Add similar entries to `app/serializers/dish.js`:

```diff
 class DishesSerializer extends Serializer {
   attributes = [
     'name',
     'rating',
     'restaurantId'
   ];
+  hasOne = ['restaurant'];
 }
```

Note that although the model uses `belongsTo`, the serializer uses `hasOne`.

Finally, make one change to `app/controllers/dishes.js`:

```diff
 class DishesController extends Controller {
   params = [
     'name',
     'rating',
-    'restaurantId'
+    'restaurant'
   ];
 }
```

Instead of sending in the restaurant ID field, we'll be sending in the `restaurant` as a relationship.

With that, we're done building out our app, and we hardly had to write any code!

## Trying It Out

Now let’s give it a try. Start the Lux server:

```bash
$ lux serve
```

Since we don't have any data in our server yet, let's create a restaurant. We won’t be able to do this in the browser; we’ll need a more sophisticated web service client to do so. One good option is [Postman][postman]—download it and start it up.

Create a POST request to `http://localhost:4000/restaurants`. Go to the Headers tab and enter key “Content-Type” and value “application/vnd.api+json”—this is the content type JSON:API requires.

Next, switch to the Body tab. Leave the dropdown as "Text"; if you change it to "JSON", Postman will change the "Content-Type" to "application/json", which our server won't accept. Enter the following:

```json
{
  "data": {
    "type": "restaurants",
    "attributes": {
      "name": "Sushi Place",
      "address": "123 Main Street"
    }
  }
}
```

Now that our request is set up, click Send and you should get a “201 Created” response, with the following body:

```json
{
  "data": {
    "id": "1",
    "type": "restaurants",
    "attributes": {
      "name": "Sushi Place",
      "address": "123 Main Street"
    },
    "relationships": {
      "dishes": {
        "data": []
      }
    }
  },
  "links": {
    "self": "http://localhost:4000/restaurants"
  },
  "jsonapi": {
    "version": "1.0"
  }
}
```

This is a JSON:API response for a single record. Let’s talk about what’s going on here:

- The top-level `data` property contains the main data for the response. In this case it’s one record; it can also be an array.
- The record contains an `id` property giving the record’s publicly-exposed ID, which by default is the database integer ID. But JSON:API IDs are always exposed as strings, to allow for the possibility of slugs or UUIDs.
- Even if you can infer the type of the record from context, JSON:API records always have a `type` field recording which type they are. In some contexts, records of different types will be intermixed in an array, so this keeps them distinct.
- `attributes` is an object containing all the attributes we exposed. They are nested instead of directly on the record to avoid colliding with other standard JSON:API properties like `type`.
- `relationships` provides data on the relationships for this record. In this case, the record has a `dishes` relationship, but it doesn't have any related records in it yet.

Now that we have a restaurant, let's retrieve the data for it. In a new tab, send a GET request to `http://localhost:4000/restaurants`. You should get the following response:

```json
{
  "data": [
    {
      "id": "1",
      "type": "restaurants",
      "attributes": {
        "name": "Sushi Place",
        "address": "123 Main Street"
      },
      "relationships": {
        "dishes": {
          "data": []
        }
      },
      "links": {
        "self": "http://localhost:4000/restaurants/1"
      }
    }
  ],
  "links": {
    "self": "http://localhost:4000/restaurants",
    "first": "http://localhost:4000/restaurants",
    "last": "http://localhost:4000/restaurants",
    "prev": null,
    "next": null
  },
  "jsonapi": {
    "version": "1.0"
  }
}
```

Note that this time `data` is an array. For now it only contains one record.

Let’s see how we can create related data as well. To add a new dish associated with restaurant 1, `POST` to `http://localhost:4000/dishes`:

```json
{
  "data": {
    "type": "dishes",
    "attributes": {
      "name": "Volcano Roll",
      "rating": 4
    },
    "relationships": {
      "restaurant": {
        "data": {
          "type": "restaurants",
          "id": "1"
        }
      }
    }
  }
}
```

You should get the following response:

```json
{
  "data": {
    "id": "1",
    "type": "dishes",
    "attributes": {
      "name": "Volcano Roll",
      "rating": 4
    },
    "relationships": {
      "restaurant": {
        "data": {
          "id": "1",
          "type": "restaurants"
        },
        "links": {
          "self": "http://localhost:4000/restaurants/1"
        }
      }
    }
  },
  "links": {
    "self": "http://localhost:4000/dishes"
  },
  "jsonapi": {
    "version": "1.0"
  }
}
```

If you’d like to try out updating and deleting records:

- Make a `PUT` request to `http://localhost:4000/restaurants/1`, passing in updated `attributes`.
- Make a `DELETE` request to `http://localhost:4000/restaurants/1` with no body to delete the record.

## There’s More
We’ve seen a ton of help Lux has provided us: the ability to create, read, update, and delete records, including record relationships. But it offers a lot more too! It allows you to configure validation errors, allows you to request only a subset of the fields you need, allows you to include related records in the response, as well as sorting, filtering, and pagination. To learn more, check out [the Lux Guide][lux].

Now that you have a JSON:API backend, you should try connecting to it from the frontend. Choose a tutorial from the [How to JSON:API home page](/)!

[jsonapi-js-servers]: https://jsonapi.org/implementations/#server-libraries-node-js
[lux]: https://lux.postlight.com/
[postman]: https://www.getpostman.com/
[sqlite]: https://www.sqlite.org/index.html
