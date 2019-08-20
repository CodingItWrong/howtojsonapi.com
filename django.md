---
layout: tutorial
title: Building a JSON:API Backend with Django REST Framework JSON API
logo: /images/django.svg
logo_alt: Django logo
---

[Django REST Framework JSON API][dja] is a library for creating JSON:API backends using the [Django framework][django], built on top of the [Django REST Framework][drf] library.

To try it out, let's create a web service for rating dishes at restaurants. We'll call it "Opinion Ate".

First, [install Python][install-python].

Create a new Python project with a virtual environment:

```bash
$ mkdir django_jsonapi
$ cd django_jsonapi
$ python3 -m venv env
$ source env/bin/activate
```

Install Django itself:

```bash
$ pip install django
```

Now create a Django project and app:

```bash
$ django-admin startproject django_jsonapi .  # Note the trailing '.' character
$ django-admin startapp opinion_ate
```

Update `django_jsonapi/settings.py` to indicate that `opinion_ate` is an installed app:

```diff
 INSTALLED_APPS = [
+    'opinion_ate.apps.OpinionAteConfig',
     'django.contrib.admin',
```

## Models

Django persists data to the database using classes called Models. Django REST Framework JSON API uses the same models, so to start building our app we’ll create models in the typical Django way.

Replace the contents of `opinion_ate/models.py` with the following:

```python
from django.db import models

class Restaurant(models.Model):
    name = models.CharField(max_length=200)
    address = models.CharField(max_length=200)

class Dish(models.Model):
    name = models.CharField(max_length=200)
    rating = models.IntegerField()
    restaurant = models.ForeignKey(Restaurant, on_delete=models.CASCADE)
```

This defines two new models, `Restaurant` and `Dish`. `Restaurant` has a `name` and `address` field on it, both character fields. `Dish` has a `name` character field, a `rating` integer, and a foreign key pointing to the related `Restaurant`.

Next, create a database migration to update the database to match our current model definitions:

```bash
$ python manage.py makemigrations opinion_ate
```

By default a new Django app is set up with a SQLite database, which is just a flat file. This is the simplest way to go for experimentation purposes. If you'd like to use another SQL database like Postgres or MySQL, follow the Django docs on [installing the appropriate database bindings][database].


Run the following command to run those migrations against the SQLite database:

```bash
$ python manage.py migrate
```

Now that our models are set up, we can create some records. You could do it by hand, but Django has the concept of fixture files, which allows you to import sample data into your database. Let’s use that to set up some data. Create an `opinion_ate/fixtures/` folder, then a `restaurants.json` file inside it with the following contents:

```json
[
  {
    "model": "opinion_ate.restaurant",
    "pk": 1,
    "fields": {
      "name": "Sushi Place",
      "address": "123 Main Street"
    }
  },
  {
    "model": "opinion_ate.dish",
    "pk": 1,
    "fields": {
      "name": "Volcano Roll",
      "rating": 3,
      "restaurant_id": 1
    }
  },
  {
    "model": "opinion_ate.dish",
    "pk": 2,
    "fields": {
      "name": "Salmon Nigiri",
      "rating": 4,
      "restaurant_id": 1
    }
  },
  {
    "model": "opinion_ate.restaurant",
    "pk": 2,
    "fields": {
      "name": "Burger Place",
      "address": "456 Other Street"
    }
  },
  {
    "model": "opinion_ate.dish",
    "pk": 3,
    "fields": {
      "name": "Barbecue Burger",
      "rating": 5,
      "restaurant_id": 2
    }
  },
  {
    "model": "opinion_ate.dish",
    "pk": 4,
    "fields": {
      "name": "Slider",
      "rating": 3,
      "restaurant_id": 2
    }
  }
]
```

Load the data from the fixture with the following command:

```sh
$ python manage.py loaddata restaurants
```

## Setting Up the Web Service

Now that we’ve got our data all set, let’s set up Django REST Framework JSON API (DJA) so we can access it via a web service.

Install the following dependencies:

```bash
$ pip install djangorestframework
$ pip install djangorestframework-jsonapi
$ pip install django-filter
```

Add `rest_framework` as an installed app for your project in `django_jsonapi/settings.py`:

```diff
 INSTALLED_APPS = [
     'opinion_ate.apps.OpinionAteConfig',
+    'rest_framework',
     'django.contrib.admin',
```

Next, configure Django REST Framework to use JSON API by pasting this big chunk of config into `django_jsonapi/settings.py`. This comes straight from the DJA docs, with one exception: we disable pagination here for the sake of simplicity.

```python
REST_FRAMEWORK = {
    'EXCEPTION_HANDLER': 'rest_framework_json_api.exceptions.exception_handler',
    'DEFAULT_PAGINATION_CLASS':
        'rest_framework_json_api.pagination.JsonApiPageNumberPagination',
    'DEFAULT_PARSER_CLASSES': (
        'rest_framework_json_api.parsers.JSONParser',
        'rest_framework.parsers.FormParser',
        'rest_framework.parsers.MultiPartParser'
    ),
    'DEFAULT_RENDERER_CLASSES': (
        'rest_framework_json_api.renderers.JSONRenderer',
        # If you're performance testing, you will want to use the browseable API
        # without forms, as the forms can generate their own queries.
        # If performance testing, enable:
        # 'example.utils.BrowsableAPIRendererWithoutForms',
        # Otherwise, to play around with the browseable API, enable:
        'rest_framework.renderers.BrowsableAPIRenderer'
    ),
    'DEFAULT_METADATA_CLASS': 'rest_framework_json_api.metadata.JSONAPIMetadata',
    'DEFAULT_FILTER_BACKENDS': (
        'rest_framework_json_api.filters.QueryParameterValidationFilter',
        'rest_framework_json_api.filters.OrderingFilter',
        'rest_framework_json_api.django_filters.DjangoFilterBackend',
        'rest_framework.filters.SearchFilter',
    ),
    'SEARCH_PARAM': 'filter[search]',
    'TEST_REQUEST_RENDERER_CLASSES': (
        'rest_framework_json_api.renderers.JSONRenderer',
    ),
    'TEST_REQUEST_DEFAULT_FORMAT': 'vnd.api+json'
}
```

To set up a DJA web service, first we need to create "serializers", which translate models to their end-user-facing format. Create an `opinion_ate/serializers.py` file, then add the following contents:

```python
from rest_framework_json_api import serializers
from opinion_ate.models import Restaurant, Dish

class RestaurantSerializer(serializers.HyperlinkedModelSerializer):
    class Meta:
        model = Restaurant
        fields = ('name', 'address', 'dish_set')

class DishSerializer(serializers.HyperlinkedModelSerializer):
    class Meta:
        model = Dish
        fields = ('name', 'rating', 'restaurant')
```

DJA doesn’t want to make any assumptions about the data we want to expose to end users; we need to explicitly tell it. The `fields` values we provided mean that those fields will be exposed to the end user.

But what's the `dish_set` field on `Restaurant`? This is the inverse of the `Dish.restaurant` foreign key; it's the set of dishes associated with a restaurant. We didn't have to declare this on the model; it's automatically available by virtue of a dish having a restaurant.

Now that our serializers are set, we need to create views that handle the HTTP requests for restaurants and dishes. DJA provides base viewset classes that will give us what we need with just a little configuration. Replace `opinion_ate/views.py` with the following:

```python
from opinion_ate.models import Restaurant, Dish
from opinion_ate.serializers import RestaurantSerializer, DishSerializer
from rest_framework import viewsets

class RestaurantViewSet(viewsets.ModelViewSet):
    queryset = Restaurant.objects.all()
    serializer_class = RestaurantSerializer

class DishViewSet(viewsets.ModelViewSet):
    queryset = Dish.objects.all()
    serializer_class = DishSerializer
```

The last piece of the puzzle is hooking up the URLs. Open `django_jsonapi/urls.py` and replace the contents with the following:

```python
from django.urls import include, path
from rest_framework import routers
from opinion_ate import views

router = routers.DefaultRouter()
router.register(r'restaurants', views.RestaurantViewSet)
router.register(r'dishes', views.DishViewSet)

urlpatterns = [
    path('', include(router.urls)),
]
```

This will hook up all necessary URLs. For example, for restaurants, the following main URLs are now available:

- GET /restaurants/ — lists all the restaurants
- POST /restaurants/ — creates a new restaurant
- GET /restaurants/:id — gets one restaurant
- PATCH /restaurants/:id — updates a restaurant
- DELETE /restaurants/:id — deletes a restaurant

That’s a lot we’ve gotten without having to write almost any code!

## Trying It Out

Now let’s give it a try. Start the Django server:

```bash
$ python manage.py runserver
```

Visit `http://localhost:8000/dishes/1` in your browser. You should see something like the following:

```json
{
  "data": {
    "type": "Dish",
    "id": "1",
    "attributes": {
      "name": "Volcano Roll",
      "rating": 3
    },
    "relationships": {
      "restaurant": {
        "data": {
          "type": "Restaurant",
          "id": "1"
        },
        "links": {
          "related": "http://localhost:8000/restaurants/1/"
        }
      }
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
- `relationships` provides data on the relationships for this record. In this case, the record has a `restaurant` relationship. We're given a "resource identifier object," containing the type and ID of the record, but not its attributes.

Try visiting `http://localhost:8000/restaurants/`, in the browser. You’ll see the following:

```json
{
  "data": [
    {
      "type": "Restaurant",
      "id": "1",
      "attributes": {
        "name": "Sushi Place",
        "address": "123 Main Street"
      },
      "relationships": {
        "dish_set": {
          "data": [
            {
              "type": "Dish",
              "id": "1"
            },
            {
              "type": "Dish",
              "id": "2"
            }
          ],
          "meta": {
            "count": 2
          }
        }
      }
    },
    {
      "type": "Restaurant",
      "id": "2",
      "attributes": {
        "name": "Burger Place",
        "address": "456 Other Street"
      },
      "relationships": {
        "dish_set": {
          "data": [
            {
              "type": "Dish",
              "id": "3"
            },
            {
              "type": "Dish",
              "id": "4"
            }
          ],
          "meta": {
            "count": 2
          }
        }
      }
    }
  ]
}
```

Note that this time the `data` is an array of two records. Each of them also has their own `relationships` getting back to the dishes associated with the restaurant. These relationships are where DJA really shines. Instead of having to manually build URLs, views, and queries for all of these relationships, DJA exposes them for you. And because it uses the standard JSON:API format, there are prebuilt client tools that can save you the same kind of code on the frontend!

Next, let’s try creating a record. We won’t be able to do this in the browser; we’ll need a more sophisticated web service client to do so. One good option is [Postman][postman]—download it and start it up.

You can use Postman for GET requests as well: set up a GET request to `http://localhost:8000/restaurants/` and see how it displays the same data as the browser.

Next, let’s create a POST request to the same URL, `http://localhost:8000/restaurants/`. Go to the Headers tab and enter key “Content-Type” and value “application/vnd.api+json”—this is the content type JSON:API requires.

Next, switch to the Body tab. Leave the dropdown as "Text"; if you change it to "JSON", Postman will change the "Content-Type" to "application/json", which our server won't accept. Enter the following:

```json
{
  "data": {
    "type": "Restaurant",
    "attributes": {
      "name": "Spaghetti Place",
      "address": "789 Third Street",
      "dish_set": []
    }
  }
}
```

Notice that we don't have to provide an `id` because we’re relying on the server to generate it. And we don’t have to provide the `relationships` or `links`, just the `attributes` we want to set on the new record.

Now that our request is set up, click Send and you should get a “201 Created” response, with the following body:

```json
{
  "data": {
    "type": "Restaurant",
    "id": "3",
    "attributes": {
      "name": "Spaghetti Place",
      "address": "789 Third Street"
    },
    "relationships": {
      "dish_set": {
        "data": [],
        "meta": {
          "count": 0
        }
      }
    }
  }
}
```

Our new record is created and the data is returned to us!

If you’d like to try out updating and deleting records:

- Make a `PATCH` request to `http://localhost:8000/restaurants/3/`, passing in updated `attributes`.
- Make a `DELETE` request to `http://localhost:8000/restaurants/3/` with no body to delete the record.

## There’s More
We’ve seen a ton of help Django REST Framework JSON API has provided us: the ability to create, read, update, and delete records, including record relationships. But it offers a lot more too! It allows you to request only a subset of the fields you need, allows you to include related records in the response, as well as sorting, filtering, and pagination. To learn more, check out [the DJA Guide][dja].

Now that you have a JSON:API backend, you should try connecting to it from the frontend. Choose a tutorial from the [How to JSON:API home page](/)!

[database]: https://docs.djangoproject.com/en/2.2/topics/install/#database-installation
[dja]: https://django-rest-framework-json-api.readthedocs.io/en/stable/index.html
[django]: https://www.djangoproject.com/
[drf]: https://www.django-rest-framework.org/
[firefox]: https://getfirefox.com
[install-python]: https://www.python.org/downloads/
[jsonview]: https://chrome.google.com/webstore/detail/jsonview/chklaanhfefbnpoihckbnefhakgolnmc
[postman]: https://www.getpostman.com/
[sqlite]: https://www.sqlite.org/index.html
