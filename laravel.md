---
layout: tutorial
title: Building a JSON:API Backend with Laravel JSON API
logo: /images/laravel.svg
logo_alt: Laravel logo
---

[Laravel JSON API][laravel-json-api] is a library for creating JSON:API backends using the Laravel application framework.

To try it out, let's create a web service for rating dishes at restaurants. We'll call it "Opinion Ate".

First, we'll need the following installed to use Laravel:

- PHP
- The `composer` command
- The `laravel` command
- A MySQL database

If you're new to Laravel, one easy way to get it up and running is to use [Laravel Homestead][homestead], a virtual machine pre-configured for Laravel development. This guide will assume you're using Homestead; if not, make the necessary adjustments.

Edit your `Homestead.yml` file to map a folder on your host machine to the Vagrant VM; this will make it easy to edit the files in your editor of choice:

```yaml
folders:
  - map: ~/code
    to: /home/vagrant/code
```

After making this change, within your `Homestead` folder run `vagrant provision` to apply this configuration.

Connect to Homestead with `vagrant ssh`, then create a new Laravel app:

```bash
$ cd code
$ laravel new opinion-ate
```

Then, back in your `Homestead.yml` file, configure this app to be shown at a certain domain name:

```yaml
sites:
  - map: opinion-ate.test
    to: /home/vagrant/code/opinion-ate/public
```

Make sure not to forget the `/public` on the end.

Run `vagrant provision` once more to apply this configuration. Finally, on your host machine, edit your `/etc/hosts` file to point the domain name you configured to your local machine

```text
192.168.10.10  opinion-ate.test
```

Let's make sure our app is up and running. Go to `http://opinion-ate.test` in a browser. You should see the “Laravel" page with links to "Documentation", "Laracasts", and other things.

One last bit of configuration: check the `.env` file to make sure your database connection is correct. With Homestead, update it to match the following:

```text
DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=homestead
DB_USERNAME=homestead
DB_PASSWORD=secret
```

## Models
Laravel persists data to the database using classes called Eloquent models. Laravel JSON API uses the same models, so to start building our app we’ll create models in the typical Laravel way.

First let’s create a model representing a restaurant. If you aren't already connected to Homestead, run `vagrant ssh`, go into the `opinion-ate` directory, and stay connected for the rest of the tutorial. Then, run the following command:

```bash
$ php artisan make:model Restaurant --migration
```

You’ll see output like the following:

```bash
Model created successfully.
Created Migration: 2020_09_20_122048_create_restaurants_table
```

The generator created a number of files; let’s take a look at a few of them. First, open the file in `database/migrations` that ends with `_create_restaurants_table.php` — the date on the file will be different than mine, showing the time you ran the command.

```php
class CreateRestaurantsTable extends Migration
{
    /**
     * Run the migrations.
     *
     * @return void
     */
    public function up()
    {
        Schema::create('restaurants', function (Blueprint $table) {
            $table->id();
            $table->timestamps();
        });
    }

    /**
     * Reverse the migrations.
     *
     * @return void
     */
    public function down()
    {
        Schema::dropIfExists('restaurants');
    }
}
```

This file contains a *migration*, a class that tells Laravel how to make a change to a database. `Schema::create()` will create the `restaurants` table. The function passed to `Schema::create()` will be passed the argument `$table`, representing a table. It's already set to create an `id` and `timestamps`. Let's add a few additional columns:

```diff
 Schema::create('restaurants', function (Blueprint $table) {
     $table->id();
+    $table->string('name');
+    $table->string('address');
     $table->timestamps();
 });
```

The `restaurants` table hasn’t actually been created yet; the migration file just records *how* to create it. You can run it on your computer, when a coworker pulls it down she can run it on hers, and you can run it on the production server as well. Run the migration now with this command:

```bash
$ php artisan migrate
```

If your database connection info is correct, you should see the following output, including a few other migrations Laravel creates by default:

```bash
Migration table created successfully.
Migrating: 2014_10_12_000000_create_users_table
Migrated:  2014_10_12_000000_create_users_table (225.35ms)
Migrating: 2014_10_12_100000_create_password_resets_table
Migrated:  2014_10_12_100000_create_password_resets_table (169.18ms)
Migrating: 2019_08_19_000000_create_failed_jobs_table
Migrated:  2019_08_19_000000_create_failed_jobs_table (128.76ms)
Migrating: 2020_09_20_122048_create_restaurants_table
Migrated:  2020_09_20_122048_create_restaurants_table (73.07ms)
```

Next let’s look at the `app/Models/Restaurant.php` file created:

```php
use Illuminate\Database\Eloquent\Model;

class Restaurant extends Model
{
    //
}
```

That’s…pretty empty. We have a `Restaurant` class that inherits from `Eloquent\Model`, but nothing else. This represents a Restaurant record, but how does it know what columns are available? Laravel will automatically inspect the table to see what columns are defined on it and make those columns available. We do need to add one bit of configuration: the fields that are allowed to be assigned-to by end users:

```diff
 class Restaurant extends Model
 {
-    //
+    protected $fillable = ['name', 'address'];
 }
```

Now let’s set up the model for a dish itself.

```bash
$ php artisan make:model Dish --migration
```

Add the following fields to the migration:

```diff
 Schema::create('dishes', function (Blueprint $table) {
     $table->id();
+    $table->string('name');
+    $table->integer('rating');
+    $table->bigInteger('restaurant_id');
     $table->timestamps();
 });
```

Why are we using `bigInteger` for `restaurant_id`? Primary keys are created with `bigIncrements()`, which creates a big integer column that automatically increments. To match data types, we create a column with the same big integer data type.

And in the model file, mark these fields as fillable:

```diff
 class Dish extends Model
 {
-    //
+    protected $fillable = ['name', 'rating', 'restaurant_id'];
 }
```

Go ahead and migrate the database again:

```bash
$ php artisan migrate
```

Our models will automatically detect the columns on them, but to use relationships we need to declare them. Add the following to `Restaurant.php`:

```diff
 class Restaurant extends Model
 {
     protected $fillable = ['name', 'address'];
+
+    public function dishes()
+    {
+        return $this->hasMany('App\Models\Dish');
+    }
}
```

This allows you to get to a restaurant's dishes. Let's make a way to get back from the dish to a restaurant too. Add the following to `Dish.php`:

```diff
 class Dish extends Model
 {
     protected $fillable = ['name', 'rating'];
+
+    public function restaurant()
+    {
+        return $this->belongsTo('App\Models\Restaurant');
+    }
}
```

Now that our models are set up, we can create some records. You could do it by hand, but Laravel has the concept of seeder files, which allow you to “seed” your database with sample data. Let’s use them to set up some data.

Generate a seeder file:

```sh
$ php artisan make:seeder RestaurantsTableSeeder
```

This will create the file `database/seeds/RestaurantsTableSeeder.php`. Add the following to it:

```diff
+use App\Restaurant;

 class RestaurantsTableSeeder extends Seeder
 {
     /**
      * Run the database seeds.
      *
      * @return void
      */
     public function run()
     {
+        $sushiPlace = Restaurant::create(['name' => 'Sushi Place', 'address' => '123 Main Street']);
+        $burgerPlace = Restaurant::create(['name' => 'Burger Place', 'address' => '456 Other Street']);
+
+        $sushiPlace->dishes()->createMany([
+            ['name' => 'Volcano Roll', 'rating' => 3],
+            ['name' => 'Salmon Nigiri', 'rating' => 4],
+        ]);
+
+        $burgerPlace->dishes()->createMany([
+            ['name' => 'Barbecue Burger', 'rating' => 5],
+            ['name' => 'Slider', 'rating' => 3],
+        ]);
     }
}
```

Note that we can just pass the attributes to the `::create()` method in an array. Notice, too, that we can access the `dishes()` relationship for a given restaurant, then `createMany()` records on that relationship—that way Laravel knows what foreign key value to provide for the `restaurant` relationship.

Next, call that seeder file from the main `DatabaseSeeder.php`:

```diff
 class DatabaseSeeder extends Seeder
 {
     /**
      * Seed the application's database.
      *
      * @return void
      */
     public function run()
     {
-        // $this->call(UsersTableSeeder::class);
+        $this->call(RestaurantsTableSeeder::class);
     }
 }
```

Run the seed command to seed the database:

```sh
$ php artisan db:seed
```

## Setting Up the Web Service

Now that we’ve got our data all set, let’s set up Laravel JSON API (LJA) so we can access it via a web service.

Add LJA and its associated testing library to your project's dependencies using Composer:

```php
$ composer require cloudcreativity/laravel-json-api
$ composer require --dev cloudcreativity/json-api-testing
```

Next, we need to make a few configuration tweaks. By default, Laravel adds an `/api/` prefix to API routes, but LJA also has functionality to add prefixes like `/api/v1/` to your API routes. To prevent these from conflicting, let's remove Laravel's default `/api/` prefix set up in `app/Providers/RouteServiceProvider.php`:

```diff
 protected function mapApiRoutes()
 {
-    Route::prefix('api')
-         ->middleware('api')
+    Route::middleware('api')
          ->namespace($this->namespace)
          ->group(base_path('routes/api.php'));
 }
```

The LJA guides also describe setting up JSON-specific exception handling, but leaving this off for now can help us easily see any stack traces we get as we're getting the API set up.

LJA can host multiple APIs in the same Laravel application, so first we need to generate the default API:

```sh
$ php artisan make:json-api
```

This creates a file `config/json-api-default.php`. We just have to configure one thing in there: which models are available as resources in the API. Find the `'resources'` key and make the following change:

```diff
     'resources' => [
-        'posts' => \App\Post::class,
+        'restaurants' => \App\Restaurant::class,
+        'dishes' => \App\Dish::class,
     ],
```

For each of these resources, we need to create a "schema", a class that instructs LJA which attributes and relationships to expose. Generate a schema for each of our two models:

```sh
$ php artisan make:json-api:schema Restaurants
$ php artisan make:json-api:schema Dishes
```

These commands together create a directory `app/JsonApi/` with a folder under them for each of our two resources. Each folder has a `Schema.php` file. Open `app/JsonApi/Restaurants/Schema.php` and find the function `getAttributes()`. Add our string attributes to it:

```diff
 public function getAttributes($resource)
 {
     return [
+        'name' => $resource->name,
+        'address' => $resource->address,
         'created-at' => $resource->created_at->toAtomString(),
         'updated-at' => $resource->updated_at->toAtomString(),
    ];
}
```

Note that `created-at` and `updated-at` are exposed automatically. `toAtomString()` is a method on datetime objects created with the Carbon library that Laravel uses by default; it formats the dates.

We also want to expose the `dishes` relationship on a restaurant. Add the following function to the `Schema` class:

```php
public function getRelationships($resource, $isPrimary, array $includeRelationships)
{
    return [
        'dishes' => [
            self::SHOW_SELF => true,
            self::SHOW_RELATED => true,
        ]
    ];
}
```

Now make analogous changes to the `Dishes/Schema.php` file:

```diff
 class Schema extends SchemaProvider
 {
...
     public function getAttributes($resource)
     {
         return [
+            'name' => $resource->name,
+            'rating' => $resource->rating,
             'created-at' => $resource->created_at->toAtomString(),
             'updated-at' => $resource->updated_at->toAtomString(),
         ];
     }
+
+    public function getRelationships($resource, $isPrimary, array $includeRelationships)
+    {
+        return [
+            'restaurant' => [
+                self::SHOW_SELF => true,
+                self::SHOW_RELATED => true,
+            ]
+        ];
+    }
}
```

We need to generate one more type of class for each of our models: an adapter. LJA uses the adapter to find how to query the relationships for each model type. Generate an adapter for each model:

```bash
$ php artisan make:json-api:adapter Restaurants
$ php artisan make:json-api:adapter Dishes
```

Under our `app/JsonApi` folders for each model, this creates an `Adapter.php` class. First open `Restaurants/Adapter.php` and add a `dishes()` function to the `Adapter` class:

```php
protected function dishes()
{
    return $this->hasMany();
}
```

This indicates to LJA that `dishes` is a has-many relationship.

Add an analogous function to `Dishes/Adapter.php`

```php
protected function restaurant()
{
    return $this->hasOne();
}
```

The last piece of the puzzle is hooking up the routes. Open `routes/api.php` and you’ll see the following:

```php
Route::middleware('auth:api')->get('/user', function (Request $request) {
    return $request->user();
});
```

This is where all the routes for your app are configured. LJA provides a function that will set up routes the way LJA needs. Add the following at the end of the file:

```php
JsonApi::register('default')->routes(function ($api) {
    $api->resource('restaurants')->relationships(function ($relations) {
        $relations->hasMany('dishes');
    });
    $api->resource('dishes')->relationships(function ($relations) {
        $relations->hasOne('restaurant');
    });
});
```

Note that we specify both the main resources and the relationships available on them. This will set up all necessary routes. For example, for restaurants, the following main routes are created:

- GET /restaurants — lists all the restaurants
- POST /restaurants — creates a new restaurant
- GET /restaurants/:id — gets one restaurant
- PATCH /restaurants/:id — updates a restaurant
- DELETE /restaurants/:id — deletes a restaurant

That’s a lot we’ve gotten without having to write almost any code!

## Trying It Out

Now let’s give it a try.

Visit `http://opinion-ate.test/api/v1/restaurants/1` in your browser. You should see something like the following:

```json
{
  "data": {
    "type": "restaurants",
    "id": "1",
    "attributes": {
      "name": "Sushi Place",
      "address": "123 Main Street",
      "created-at": "2019-06-05T11:44:08+00:00",
      "updated-at": "2019-06-05T11:44:08+00:00"
    },
    "relationships": {
      "dishes": {
        "links": {
          "self": "http://opinion-ate.test/api/v1/restaurants/1/relationships/dishes",
          "related": "http://opinion-ate.test/api/v1/restaurants/1/dishes"
        }
      }
    },
    "links": {
      "self": "http://opinion-ate.test/api/v1/restaurants/1"
    }
  }
}
```

If you’re using [Firefox][firefox] you should see the JSON data nicely formatted. If your browser doesn’t automatically format JSON, you may be able to find a browser extension to do so. For example, for Chrome you can use [JSONView][jsonview].

This is a JSON:API response for a single record. Let’s talk about what’s going on here:

- The top-level `data` property contains the main data for the response. In this case it’s one record; it can also be an array.
- Even if you can infer the type of the record from context, JSON:API records always have a `type` field recording which type they are. In some contexts, records of different types will be intermixed in an array, so this keeps them distinct.
- The record contains an `id` property giving the record’s publicly-exposed ID, which by default is the database integer ID. But JSON:API IDs are always exposed as strings, to allow for the possibility of slugs or UUIDs.
- `attributes` is an object containing all the attributes we exposed. They are nested instead of directly on the record to avoid colliding with other standard JSON:API properties like `type`.
- `relationships` provides data on the relationships for this record. In this case, the record has a `dishes` relationship. Two `links` are provided to get data related to that relationship:
	- The `self` link conceptually provides the relationships themselves, which is to say just the IDs of the related records
	- The `related` link provides the full related records.

Try visiting the `related` link, `http://opinion-ate.test/api/v1/restaurants/1/dishes`, in the browser. You’ll see the following:

```json
{
  "data": [
    {
      "type": "dishes",
      "id": "1",
      "attributes": {
        "name": "Volcano Roll",
        "rating": 3,
        "created-at": "2019-06-05T11:44:24+00:00",
        "updated-at": "2019-06-05T11:44:24+00:00"
      },
      "relationships": {
        "restaurant": {
          "links": {
            "self": "http://opinion-ate.test/api/v1/dishes/1/relationships/restaurant",
            "related": "http://opinion-ate.test/api/v1/dishes/1/restaurant"
          }
        }
      },
      "links": {
        "self": "http://opinion-ate.test/api/v1/dishes/1"
      }
    },
    {
      "type": "dishes",
      "id": "2",
      "attributes": {
        "name": "Salmon Nigiri",
        "rating": 4,
        "created-at": "2019-06-05T11:44:24+00:00",
        "updated-at": "2019-06-05T11:44:24+00:00"
      },
      "relationships": {
        "restaurant": {
          "links": {
            "self": "http://opinion-ate.test/api/v1/dishes/2/relationships/restaurant",
            "related": "http://opinion-ate.test/api/v1/dishes/2/restaurant"
          }
        }
      },
      "links": {
        "self": "http://opinion-ate.test/api/v1/dishes/2"
      }
    }
  ]
}
```

Note that this time the `data` is an array of two records. Each of them also has their own `relationships` getting back to the `restaurants` associated with the record. These relationships are where LJA really shines. Instead of having to manually build routes, controllers, and queries for all of these relationships, LJA exposes them for you. And because it uses the standard JSON:API format, there are prebuilt client tools that can save you the same kind of code on the frontend!

Next, let’s take a look at the restaurants list view. Visit `http://opinion-ate.test/api/v1/restaurants` and you’ll see all the records returned.

Next, let’s try creating a record. We won’t be able to do this in the browser; we’ll need a more sophisticated web service client to do so. One good option is [Postman][postman]—download it and start it up.

You can use Postman for GET requests as well: set up a GET request to `http://opinion-ate.test/api/v1/restaurants` and see how it displays the same data as the browser.

Next, let’s create a POST request to the same URL, `http://opinion-ate.test/api/v1/restaurants`. Go to the Headers tab and enter key “Content-Type” and value “application/vnd.api+json”—this is the content type JSON:API requires.

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
    "type": "restaurants",
    "id": "3",
    "attributes": {
      "name": "Spaghetti Place",
      "address": "789 Third Street",
      "created-at": "2019-06-09T00:07:50+00:00",
      "updated-at": "2019-06-09T00:07:50+00:00"
    },
    "relationships": {
      "dishes": {
        "links": {
          "self": "http://opinion-ate.test/api/v1/restaurants/3/relationships/dishes",
          "related": "http://opinion-ate.test/api/v1/restaurants/3/dishes"
        }
      }
    },
    "links": {
      "self": "http://opinion-ate.test/api/v1/restaurants/3"
    }
  }
}
```

Our new record is created and the data is returned to us!

If you’d like to try out updating and deleting records:

- Make a `PATCH` request to `http://opinion-ate.test/api/v1/restaurants/3`, passing in updated `attributes`.
- Make a `DELETE` request to `http://opinion-ate.test/api/v1/restaurants/3` with no body to delete the record.

## There’s More
We’ve seen a ton of help Laravel JSON API has provided us: the ability to create, read, update, and delete records, including record relationships. But it offers a lot more too! It allows you to configure Laravel validators to run, allows you to request only a subset of the fields you need, allows you to include related records in the response, as well as sorting, filtering, and pagination. To learn more, check out [the Laravel JSON API Docs][laravel-json-api].

Now that you have a JSON:API backend, you should try connecting to it from the frontend. Choose a tutorial from the [How to JSON:API home page](/)!

[firefox]: https://getfirefox.com
[homestead]: https://laravel.com/docs/5.8/homestead
[laravel-json-api]: https://laravel-json-api.readthedocs.io/en/latest/
[jsonview]: https://chrome.google.com/webstore/detail/jsonview/chklaanhfefbnpoihckbnefhakgolnmc
[postman]: https://www.getpostman.com/
