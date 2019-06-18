---
layout: tutorial
title: Building a JSON:API Backend with Node and Lux
logo: /images/node.svg
logo_alt: Node.js logo
---

[Lux][lux] is a framework for creating JSON:API backends in Node.

To try it out, let's create a web service for rating dishes at restaurants. We'll call it "Opinion Ate".

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
Lux persists data to the database using classes called Models.

First let’s create a model representing a restaurant. Run the following command in the terminal:

```bash
$ lux generate model restaurant name:string address:string
```

This tells Lux to create a new model called `restaurant` and to define two `string` fields on it: `name` and `address`.

The generator created a number of files; let’s take a look at a few of them. First, open the file in `db/migrate` that ends with `-create-restaurants.js` — the date on the file will be different than mine, showing the time you ran the command.

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

Now let’s set up the model for a dish itself:

```bash
$ lux generate model dish name:string rating:integer restaurant_id:integer
```

You’ve seen a `string` column before, and you can probably guess what `integer` does. We create a `rating` column, as well as a `restaurant_id` that creates a foreign key column that references another model, in this case Restaurant.

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
+  }
 }
```

And add a `belongsTo` field to `Dish` indicating that it belongs to a restaurant:

```diff
 class Dish extends Model {
+  static belongsTo = {
+    restaurant: {
+      inverse: 'dishes',
+    },
+  }
 }
```

## Setting Up the Web Service

Now that we’ve got our data all set, let’s set up Lux so we can access it via a web service. First we need to create "serializers", which translate models to their end-user-facing format.

Generate a serializer for each model:

```bash
$ lux generate serializer restaurant
$ lux generate serializer dish
```

First add the following to the generated `app/serializers/restaurant.js`:

```diff
 import { Serializer } from 'lux-framework';

 class RestaurantsSerializer extends Serializer {
+  attributes = ['name', 'address'];
+  hasMany = ['dishes'];
 }

 export default RestaurantsSerializer;
```

Add similar entries to `app/serializers/dish.js`:

```diff
 import { Serializer } from 'lux-framework';

 class DishesSerializer extends Serializer {
+  attributes = ['name', 'rating'];
+  hasOne = ['restaurant'];
 }

 export default DishesSerializer;
```

Now that our serializers are set, we need to create controllers that handle the HTTP requests for restaurants and dishes:

```bash
$ lux generate controller restaurant
$ lux generate controller dish
```

For security purposes, controllers don't allow just any arbitrary parameters to be passed in; we need to indicate which can be sent. Add the following to `app/controllers/restaurants.js`:

```diff
 import { Controller } from 'lux-framework';

 class RestaurantsController extends Controller {
+  params = ['name', 'address']
 }

 export default RestaurantsController;
```

And the following to `app/controllers/dishes.js`:

```diff
 import { Controller } from 'lux-framework';

 class DishesController extends Controller {
+  params = ['name', 'rating', 'restaurant']
 }

 export default DishesController;
```

Note that the `restaurant` relationship is listed in the `params` along with the plain attributes.

The last piece of the puzzle is hooking up the routes. Open `routes.rb` and add the following:

```diff
 export default function routes() {
+  this.resource('restaurants');
+  this.resource('dishes');
 }
```

This will set up all necessary routes. For example, for restaurants, the following main routes are created:

- GET /restaurants — lists all the restaurants
- POST /restaurants — creates a new restaurant
- GET /restaurants/:id — gets one restaurant
- PATCH or PUT /restaurants/:id — updates a restaurant
- DELETE /restaurants/:id — deletes a restaurant

That’s a lot we’ve gotten without having to write almost any code!

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
- `relationships` provides data on the relationships for this record. In this case, the record has a `restaurant` relationship. We're given a "resource identifier object," containing the type and ID of the record, but not its attributes.

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

Let’s see how we can create related data as well. To add a new dish associated with restaurant 1, `POST` to `http://localhost:3000/dishes`:

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

- Make a `PUT` request to `http://localhost:3000/restaurants/1`, passing in updated `attributes`.
- Make a `DELETE` request to `http://localhost:3000/restaurants/1` with no body to delete the record.

## There’s More
We’ve seen a ton of help Lux has provided us: the ability to create, read, update, and delete records, including record relationships. But it offers a lot more too! It allows you to configure validation errors, allows you to request only a subset of the fields you need, allows you to include related records in the response, as well as sorting, filtering, and pagination. To learn more, check out [the Lux Guide][lux].

Now that you have a JSON:API backend, you should try connecting to it from the frontend. Choose a tutorial from the [How to JSON:API home page](/)!

[lux]: https://lux.postlight.com/
[postman]: https://www.getpostman.com/
[sqlite]: https://www.sqlite.org/index.html
