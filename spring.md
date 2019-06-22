---
layout: tutorial
title: Building a JSON:API Backend with Crnk
logo: /images/spring.svg
logo_alt: Spring Framework logo
---

[Crnk][crnk] is a library for creating JSON:API backends in Java, and features integration with Spring Framework.

To try it out, let's create a web service for rating dishes at restaurants. We'll call it "Opinion Ate".

The easiest way to create a new Spring Boot project is with [Spring Initializr][initializr]. Go there and choose the following options:

- Project: Gradle Project
- Language: Java
- Spring Boot: (leave as default)
- Artifact: "opinion-ate"
- Dependencies:
  - Search for "web", then add "Spring Web Starter"
  - Search for "lombok", then add "Lombok"

Click "Generate the project"; this will download a zip file. Expand it.

This tutorial will assume you're using the free [IntelliJ IDEA CE][intellij] as your IDE.

Open IntelliJ and choose "Import Project". Open the project's `build.gradle` file. Leave all the settings as-is. When the project opens, IntelliJ will start a build automatically.

Your project's dependencies are configured in `build.gradle`. To add Crnk to your project, open it and make the following changes:

```diff
 repositories {
-    mavenCentral()
+    jcenter()
 }

+dependencyManagement {
+    imports {
+        mavenBom "io.crnk:crnk-bom:2.11.20190113153635"
+    }
+}

 dependencies {
     implementation 'org.springframework.boot:spring-boot-starter-web'
     compileOnly 'org.projectlombok:lombok'
     annotationProcessor 'org.projectlombok:lombok'
     testImplementation 'org.springframework.boot:spring-boot-starter-test'
+    compile 'io.crnk:crnk-setup-spring-boot2'
+    compile 'io.crnk:crnk-home'
 }
```

IntelliJ will let you know that your Gradle file has changed. Click "Import Changes" to download the appropriate dependencies.

## Resources
Crnk represents your data with classes annotated with `@JsonApiResource`. Let's create a resource class representing a restaurant.

In `src/main/java`, right-click `com.example.opinionate`, then choose New > Java Class. Name it "Restaurant". Replace its contents with the following:

```java
package com.example.opinionate;

import com.fasterxml.jackson.annotation.JsonProperty;
import io.crnk.core.resource.annotations.JsonApiId;
import io.crnk.core.resource.annotations.JsonApiResource;
import lombok.Getter;
import lombok.NoArgsConstructor;
import lombok.Setter;

@Getter
@Setter
@NoArgsConstructor
@JsonApiResource(type = "restaurants")
public class Restaurant {

    @JsonApiId
    private Long id;

    @JsonProperty
    private String name;

    @JsonProperty
    private String address;
}
```

The `@JsonApiResource` annotation indicates to Crnk that this class is a type of resource that should be available via our API. `type = "restaurants"` indicates the type, which will appear in the URL and in the `type` field of our data. Crnk will set up all necessary routes, specifically:

- GET /restaurants — lists all the restaurants
- POST /restaurants — creates a new restaurant
- GET /restaurants/:id — gets one restaurant
- PATCH /restaurants/:id — updates a restaurant
- DELETE /restaurants/:id — deletes a restaurant

That’s a lot we’ve gotten without having to write almost any code!

The Lombok annotations `@Getter`, `@Setter`, and `@NoArgsConstructor` will create the corresponding getters, setters, and no-arg constructor that we'll use to work with this class.

We also need a constructor with all of these fields. Right-click anywhere in your file then choose "Generate…". A modal will appear prompting you to "Choose Fields to Initialize by Constructor". Shift-click to select all the fields, then click OK. IntelliJ will add the following constructor:

```diff
     @JsonProperty
     private String address;
+
+    public Restaurant(Long id, String name, String address) {
+        this.id = id;
+        this.name = name;
+        this.address = address;
+    }
 }
```

Next we need a Repository class that will handle persisting our Restaurants. For the sake of this tutorial we'll just use an in-memory repository.

Create another new class in `com.example.opinionate` called "RestaurantRepository". Replace its contents with the following:

```java
package com.example.opinionate;

import io.crnk.core.queryspec.QuerySpec;
import io.crnk.core.repository.ResourceRepositoryBase;
import io.crnk.core.resource.list.ResourceList;
import org.springframework.stereotype.Component;

import java.util.HashMap;
import java.util.Map;
import java.util.concurrent.atomic.AtomicLong;

@Component
public class RestaurantRepository extends ResourceRepositoryBase<Restaurant, Long> {

    private static final AtomicLong ID_GENERATOR = new AtomicLong(3);

    private Map<Long, Restaurant> restaurants = new HashMap<>();

    public RestaurantRepository() {
        super(Restaurant.class);
        restaurants.put(
            (long) 1,
            new Restaurant((long) 1, "Sushi Place", "123 Main Street")
        );
        restaurants.put(
            (long) 2,
            new Restaurant((long) 2, "Burger Place", "456 Other Street")
        );
    }

    @Override
    public synchronized void delete(Long id) {
        restaurants.remove(id);
    }

    @Override
    public synchronized <S extends Restaurant> S save(S restaurant) {
        if (restaurant.getId() == null) {
            restaurant.setId(ID_GENERATOR.getAndIncrement());
        }
        restaurants.put(restaurant.getId(), restaurant);
        return restaurant;
    }

    @Override
    public synchronized ResourceList<Restaurant> findAll(QuerySpec querySpec) {
        return querySpec.apply(restaurants.values());
    }
}
```

Extending `ResourceRepositoryBase` indicates to Crnk that this is a repository to use for a certain resource type. By providing `<Restaurant, Long>` we indicate that this is the repository for `Restaurant` resources, and that the primary key is type `Long`.

Notice that in the constructor we set up a few test data records.

Believe it or not, with this, we're done our app!

## Trying It Out

Now let’s give it a try. Open the Gradle sidebar, then expand opinion-ate > Tasks > application. Right-click on "bootRun" and click Run.

Visit `http://localhost:8080/restaurants/1` in your browser. You should see something like the following:

```json
{
  "data": {
    "id": "1",
    "type": "restaurants",
    "attributes": {
      "address": "123 Main Street",
      "name": "Sushi Place"
    },
    "links": {
      "self": "http://localhost:8080/restaurants/1"
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

Next, let’s take a look at the restaurants list view. Visit `http://localhost:8080/restaurants`. Note that this time the `data` is an array of two records.

Next, let’s try creating a record. We won’t be able to do this in the browser; we’ll need a more sophisticated web service client to do so. One good option is [Postman][postman]—download it and start it up.

You can use Postman for GET requests as well: set up a GET request to `http://localhost:8080/restaurants` and see how it displays the same data as the browser.

Next, let’s create a POST request to the same URL, `http://localhost:8080/restaurants`. Go to the Headers tab and enter key “Content-Type” and value “application/vnd.api+json”—this is the content type JSON:API requires.

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

Notice that we don't have to provide an `id` because we’re relying on the server to generate it. And we don’t have to provide the `links`, just the `attributes` we want to set on the new record.

Now that our request is set up, click Send and you should get a “201 Created” response, with the following body:

```json
{
  "data": {
    "id": "3",
    "type": "restaurants",
    "attributes": {
      "address": "789 Third Street",
      "name": "Spaghetti Place"
    },
    "links": {
      "self": "http://localhost:8080/restaurants/3"
    }
  }
}
```

Our new record is created and the data is returned to us!

If you’d like to try out updating and deleting records:

- Make a `PUT` request to `http://localhost:8080/restaurants/3`, passing in updated `attributes`.
- Make a `DELETE` request to `http://localhost:8080/restaurants/3` with no body to delete the record.

## There’s More
We’ve seen a ton of help Crnk has provided us: the ability to create, read, update, and delete records. But it offers a lot more too! It automatically handles related records, allows you to request only a subset of the fields you need, allows you to include related records in the response, as well as sorting, filtering, and pagination. To learn more, check out [the Crnk Guide][crnk].

Now that you have a JSON:API backend, you should try connecting to it from the frontend. Choose a tutorial from the [How to JSON:API home page](/)!

*Special thanks to Harry Pritchett for his help with this tutorial!*

[crnk]: http://www.crnk.io/
[firefox]: https://getfirefox.com
[initializr]: https://start.spring.io/
[intellij]: https://www.jetbrains.com/idea/download/
[jsonview]: https://chrome.google.com/webstore/detail/jsonview/chklaanhfefbnpoihckbnefhakgolnmc
[postman]: https://www.getpostman.com/
