---
layout: page
title: Building a JSON:API Backend with Rails and JSONAPI::Resources
---

[JSONAPI::Resources][jr] is a library for creating JSON:API backends using the Ruby on Rails application framework.

To try it out, let's create a web service for rating dishes at restaurants. We'll call it "Opinion Ate".

First, [install Ruby on Rails][install-rails].

Create a new Rails app:

```bash
$ rails new --api opinion_ate
```

This will create an app configured to store data in a [SQLite][sqlite] database, which is just a flat file. This is the simplest way to go for experimentation purposes. If you'd like to use another SQL database like Postgres or MySQL, the `--database=` flag can be used. Run `rails new --help` to see a list of valid options for the flag.

Let's make sure our app will work. Run the rails server:

```bash
$ rails server
```

Then in a browser go to `http://localhost:3000`. You should see the “Yay! You’re on Rails!" page.

## Models
Rails persists data to the database using classes called Models. JSONAPI::Resources uses the same models, so to start building our app we’ll create models in the typical Rails way.

First let’s create a model representing a restaurant. Run the following command in the terminal:

```bash
$ rails generate model restaurant name:string address:string
```

This tells Rails to create a new model called `restaurant` and to define two `string` fields on it: `name` and `address`.

You’ll see output like the following:

```bash
Running via Spring preloader in process 32912
      invoke  active_record
      create    db/migrate/20190604100704_create_restaurants.rb
      create    app/models/restaurant.rb
      invoke    test_unit
      create      test/models/restaurant_test.rb
      create      test/fixtures/restaurants.yml
```

The generator created a number of files; let’s take a look at a few of them. First, open the file in `db/migrate` that ends with `_create_restaurants.rb` — the date on the file will be different than mine, showing the time you ran the command.

```ruby
class CreateRestaurants < ActiveRecord::Migration[5.2]
  def change
    create_table :restaurants do |t|
      t.string :name
      t.string :address

      t.timestamps
    end
  end
end
```

This file contains a *migration*, a class that tells Rails how to make a change to a database. This file will `create_table :restaurants`, which is just what it sounds like. The `do` keyword introduces a block passed to the `create_table` method. It receives a parameter `t`, representing a table. `t.string` creates a new string column, and `t.timestamps` creates `created_at` and `updated_at` columns that Rails will manage for us automatically. Rails will also create a primary key on the table; we don’t need to specify anything for it to do so.

The `restaurants` table hasn’t actually been set up yet; the migration file just records *how* to set it up. You can run it on your computer, when a coworker pulls it down she can run it on hers, and you can run it on the production server as well. Run the migration now with this command:

```bash
$ rails db:migrate
```

You’ll see the following output:

```bash
== 20190604100704 CreateRestaurants: migrating ================================
-- create_table(:restaurants)
   -> 0.0019s
== 20190604100704 CreateRestaurants: migrated (0.0020s) =======================
```

Next let’s look at the `app/models/restaurants.rb` file created:

```ruby
class Restaurant < ApplicationRecord
end
```

That’s…pretty empty. We have a `Restaurant` class that inherits from `ApplicationRecord`, but nothing else. This represents a Restaurant record, but how does it know what columns are available? Rails will automatically inspect the table to see what columns are defined on it and make those columns available; no configuration is needed.

The generator also created a few test files in the `test/` directory, but we’ll ignore those for the sake of this tutorial.

Now let’s set up the model for a dish itself. As a shorthand for `rails generate`, you can just type `rails g`:

```bash
$ rails g model dish name:string rating:integer restaurant:references
```

You’ve seen a `string` column before, and you can probably guess what `integer` does, but what about `references`? This creates a foreign key column that references another model. By default the column name and the name of the other model are the same, in this case `restaurant`.

Go ahead and migrate the database again:

```bash
$ rails db:migrate
```

Then check the `app/models/dish.rb` file:

```ruby
class Dish < ApplicationRecord
  belongs_to :restaurant
end
```

There’s one difference in this generated file: using `references` *does* actually result in another line of code being added to the model, a call to `belongs_to`. This indicates that a `Dish` belongs to a `Restaurant`: that is, it has a foreign key pointing to it. This is a many-to-one relationship: many dishes can belong to one restaurant.

We can set up the reverse one-to-many relationship as well: the fact that a restauant has many dishes. Rails doesn’t do this for us: we need to do it manually. Add the following line to `restaurant.rb`:

```diff
 class Restaurant < ApplicationRecord
+  has_many :dishes
 end
```

Now that our models are set up, we can create some records. You could do it by hand, but Rails has the concept of a `seeds.rb` file, which allows you to “seed” your database with sample data. Let’s use that to set up some data. Replace the contents of `db/seeds.rb` with the following:

```ruby
sushi_place = Restaurant.create!(name: 'Sushi Place', address: '123 Main Street')
burger_place = Restaurant.create!(name: 'Burger Place', address: '456 Other Street')

sushi_place.dishes.create!(name: 'Volcano Roll', rating: 3)
sushi_place.dishes.create!(name: 'Salmon Nigiri', rating: 4)

burger_place.dishes.create!(name: 'Barbecue Burger', rating: 5)
burger_place.dishes.create!(name: 'Slider', rating: 3)
```

Run the seeds file to seed the database:

```sh
$ rails db:seed
```

Note that we can just pass the attributes to the `create!()` method by name. Notice, too, that we can access the `dishes` relationship for a given restaurant, and `create!` a record on that relationship—that way Rails knows what foreign key value to provide for the `restaurant` relationship.

## Setting Up the Web Service

Now that we’ve got our data all set, let’s set up JSONAPI::Resources (JR) so we can access it via a web service.

Ruby application dependencies are specified in the file `Gemfile` at the root of the project. Add the following line anywhere in that file other than inside a “group”:

```ruby
gem 'jsonapi-resources'
```

Then run the following command in the terminal:

```bash
$ bundle install
```

`bundle` is the command for Bundler, a Ruby library that handles dependencies. `bundle install` will ensure all the dependencies specified in your `Gemfile` are installed. It will record the exact version installed in `Gemfile.lock`.

To set up a JR web service, first we need to create a “resource”, which represents a model in an end-user-facing way. Run the following commands:

```bash
$ rails g jsonapi:resource restaurant
$ rails g jsonapi:resource dish
```

JR hooks in to Rails’ command line to add commands to generate resource files. First take a look at `app/resources/restaurant_resource.rb`:

```ruby
class RestaurantResource < JSONAPI::Resource
end
```

Once again, a pretty straightforward file. This time we’ll have to configure it, because JR doesn’t want to make any assumptions about the data we want to expose to end users; we need to explicitly tell it. Inside the `class` declaration, add these lines:

```ruby
attributes :name, :address

has_many :dishes
```

As you can probably guess, this means the `name` and `address` attributes, and `dishes` relationship, will be exposed to the end user. Add the following to `dish_resource.rb`:

```ruby
attributes :name, :rating

has_one :restaurant
```

Notice that while we used `belongs_to` in the model, in the resource JR uses `has_one` instead.

Now that our resources are set, we need to create controllers that handle the HTTP requests for restaurants and dishes. JR provides a generator that will give us controllers set for use with JR:

```bash
$ rails g jsonapi:controller restaurant
$ rails g jsonapi:controller dish
```

Open `app/controllers/restaurant_controller.rb` and you’ll see:

```ruby
class RestaurantsController < JSONAPI::ResourceController
end
```

The controller inherits from `JSONAPI::ResourceController`, which provides it with almost everything it needs; there’s just one Rails security feature we need to turn off. By default Rails enables an authenticity token feature that prevents Cross-Site Request Forgery attacks. This works when you use Rails to render forms on the server, but for APIs it won’t work, so we need to turn it off. We can do so by adding the following line inside each of the two controller classes:

```ruby
skip_before_action :verify_authenticity_token
```

The last piece of the puzzle is hooking up the routes. Open `routes.rb` and you’ll see the following:

```ruby
Rails.application.routes.draw do
  # For details on the DSL available within this file, see http://guides.rubyonrails.org/routing.html
end
```

This is where all the routes for your app are configured. As you might guess, JR provides helpers that will set up routes the way JR needs. Add the following inside the `do` block:

```ruby
jsonapi_resources :restaurants
jsonapi_resources :dishes
```

This will set up all necessary routes. For example, for restaurants, the following main routes are created:

- GET /restaurants — lists all the restaurants
- POST /restaurants — creates a new restaurant
- GET /restaurants/:id — gets one restaurant
- PATCH or PUT /restaurants/:id — updates a restaurant
- DELETE /restaurants/:id — deletes a restaurant

That’s a lot we’ve gotten without having to write almost any code!

## Trying It Out

Now let’s give it a try. Start the Rails server with `rails server`, or `rails s` for short:

```bash
$ rails s
```

Visit `http://localhost:3000/restaurants/1` in your browser. You should see something like the following:

```json
{
  "data": {
    "id": "1",
    "type": "restaurants",
    "links": {
      "self": "http://localhost:3000/restaurants/1"
    },
    "attributes": {
      "name": "Sushi Place",
      "address": "123 Main Street"
    },
    "relationships": {
      "dishes": {
        "links": {
          "self": "http://localhost:3000/restaurants/1/relationships/dishes",
          "related": "http://localhost:3000/restaurants/1/dishes"
        }
      }
    }
  }
}
```

If you’re using [Firefox][firefox] you should see the JSON data nicely formatted. If your browser doesn’t automatically format JSON, you may be able to find a browser extension to do so. For example, for Chrome you can use [JSONView][jsonview].

This is a JSON:API response for a single record. Let’s talk about what’s going on here:

- The top-level `data` property contains the main data for the response. In this case it’s one record; it can also be an array.
- The record contains an `id` property giving the record’s publicly-exposed ID, which by default is the database integer ID. But JSON:API IDs are always exposed as strings, to allow for the possibility of slugs or UUIDs.
- Even if you can infer the type of the record from context, JSON:API records always have a `type` field recording which type they are. In some contexts, records of different types will be intermixed in an array, so this keeps them distinct.
- `attributes` is an object containing all the attributes we exposed. They are nested instead of directly on the record to avoid colliding with other standard JSON:API properties like `type`.
- `relationships` provides data on the relationships for this record. In this case, the record has a `dishes` relationship. Two `links` are provided to get data related to that relationship:
	- The `self` link conceptually provides the relationships themselves, which is to say just the IDs of the related records
	- The `related` link provides the full related records.

Try visiting the `related` link, `http://localhost:3000/restaurants/1/dishes`, in the browser. You’ll see the following:

```json
{
  "data": [
    {
      "id": "1",
      "type": "dishes",
      "links": {
        "self": "http://localhost:3000/dishes/1"
      },
      "attributes": {
        "name": "Volcano Roll",
        "rating": 3
      },
      "relationships": {
        "restaurant": {
          "links": {
            "self": "http://localhost:3000/dishes/1/relationships/restaurant",
            "related": "http://localhost:3000/dishes/1/restaurant"
          }
        }
      }
    },
    {
      "id": "2",
      "type": "dishes",
      "links": {
        "self": "http://localhost:3000/dishes/2"
      },
      "attributes": {
        "name": "Salmon Nigiri",
        "rating": 4
      },
      "relationships": {
        "restaurant": {
          "links": {
            "self": "http://localhost:3000/dishes/2/relationships/restaurant",
            "related": "http://localhost:3000/dishes/2/restaurant"
          }
        }
      }
    }
  ]
}
```

Note that this time the `data` is an array of two records. Each of them also has their own `relationships` getting back to the `restaurants` associated with the record. These relationships are where JR really shines. Instead of having to manually build routes, controllers, and queries for all of these relationships, JR exposes them for you. And because it uses the standard JSON:API format, there are prebuilt client tools that can save you the same kind of code on the frontend!

Next, let’s take a look at the restaurants list view. Visit `http://localhost:3000/restaurants` and you’ll see all the records returned.

Next, let’s try creating a record. We won’t be able to do this in the browser; we’ll need a more sophisticated web service client to do so. One good option is [Postman][postman]—download it and start it up.

You can use Postman for GET requests as well: set up a GET request to `http://localhost:3000/restaurants` and see how it displays the same data as the browser.

Next, let’s create a POST request to the same URL, `http://localhost:3000/restaurants`. Go to the Headers tab and enter key “Content-Type” and value “application/vnd.api+json”—this is the content type JSON:API requires.

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
    "id": "3",
    "type": "restaurants",
    "links": {
      "self": "http://localhost:3000/restaurants/3"
    },
    "attributes": {
      "name": "Spaghetti Place",
      "address": "789 Third Street"
    },
    "relationships": {
      "dishes": {
        "links": {
          "self": "http://localhost:3000/restaurants/3/relationships/dishes",
          "related": "http://localhost:3000/restaurants/3/dishes"
        }
      }
    }
  }
}
```

Our new record is created and the data is returned to us!

Let’s see how we can create related data as well. To add a new dish associated with restaurant 1, `POST` to `http://localhost:3000/dishes`:

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

- Make a `PUT` request to `http://localhost:3000/restaurants/3`, passing in updated `attributes`.
- Make a `DELETE` request to `http://localhost:3000/restaurants/3` with no body to delete the record.

## There’s More
We’ve seen a ton of help JSONAPI::Resources has provided us: the ability to create, read, update, and delete records, including record relationships. But it offers a lot more too! It automatically exposes Rails validation errors, allows you to request only a subset of the fields you need, allows you to include related records in the response, as well as sorting, filtering, and pagination. To learn more, check out [the JSONAPI\:\:Resources Guide][jr].

Now that you have a JSON:API backend, you should try connecting to it from the frontend. Choose a tutorial from the [How to JSON:API home page](/)!

[firefox]: https://getfirefox.com
[install-rails]: https://guides.rubyonrails.org/getting_started.html
[jr]: http://jsonapi-resources.com/v0.9/guide/resources.html
[jsonapi-implementations]: https://jsonapi.org/implementations/
[jsonview]: https://chrome.google.com/webstore/detail/jsonview/chklaanhfefbnpoihckbnefhakgolnmc
[postman]: https://www.getpostman.com/
[sqlite]: https://www.sqlite.org/index.html
