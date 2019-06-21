---
layout: tutorial
title: Building a JSON:API Frontend with Ember Data
logo: /images/ember.svg
logo_alt: Ember.js logo
---

[Ember.js][ember] includes a built-in data layer library, Ember Data, that is targeted at connecting to JSON:API web services.

To try it out, let's create a webapp for rating dishes at restaurants. We'll call it "Opinion Ate".

Create a new Ember app using [Ember CLI][ember-cli]:

```sh
$ npm install -g ember-cli
$ ember new --no-welcome opinion-ate
$ cd opinion-ate
```

The JSON:API web service we'll be connecting to is [sandboxapi.reststate.org](https://sandboxapi.reststate.org/), a free service that allows you to create an account so you can write data as well as read it. Sign up for an account there.

Next, we need to get a token to authenticate with. We aren't going to build a login form as part of this tutorial. Instead, use a web service client app like [Postman](https://www.getpostman.com/) to send the following request:

```http
POST https://sandboxapi.reststate.org/oauth/token

grant_type=password
username=you@yourodmain.com
password=yourpassword
```

You'll receive back a response like:

```json
{
    "access_token": "Hhd07mqAY1QlhoinAcKMB5zlmRiatjOh5Ainh90yWPI",
    "token_type": "bearer",
    "expires_in": 7200,
    "created_at": 1531855327
}
```

Let's set up Ember Data with that access token to handle the web service connection. Generate an Ember Data adapter for your application:

```sh
$ ember generate adapter application
```

This creates an `app/adapters/application.js` file. Edit it to add the following:

```diff
 import DS from 'ember-data';

+const accessToken = 'PASTE YOUR ACTUAL TOKEN HERE';

 export default DS.JSONAPIAdapter.extend({
+  host: 'https://sandboxapi.reststate.org',
+
+  init() {
+    this._super(...arguments);
+
+    this.set('headers', {
+      'Authorization': `Bearer ${accessToken}`,
+    });
+  },
});
```

This will add the authorization header when the adapter is created.

Now, we need to set up models for each type of resource we want to create. Generate these with Ember CLI. As a shorthand for `ember generate`, you can just type `ember g`:

```sh
$ ember g model restaurant
$ ember g model dish
```

We need to edit each model file to declare the attributes and relationships it has. First edit `app/models/dish.js`;

```diff
 import DS from 'ember-data';
-const { Model } = DS;
+const { Model, attr, belongsTo } = DS;

 export default Model.extend({
+  name: attr(),
+  rating: attr(),
+  restauant: belongsTo('restaurant'),
 });
```

Then edit `app/models/restaurant.js`:

```diff
 import DS from 'ember-data';
-const { Model } = DS;
+const { Model, attr, hasMany } = DS;

 export default Model.extend({
+  name: attr(),
+  dishes: hasMany('dish'),
 });
```

The argument to `belongsTo()` and `hasMany()` is the name of the model the field relates to.

That's all we have to do to set up our data layer! Now let's put it to use.

Let's display a list of the restaurants. In Ember, data loading typically happens in the route, so that Ember can intelligently handle routing as the data loads. Generate a route file for the index route, the root of the site:

```sh
$ ember g route index
```

Then add a model hook to the route:

```diff
 import Route from '@ember/routing/route';

 export default Route.extend({
+  model() {
+    return this.store.findAll('restaurant');
+  },
 });
```

`this.store` provides access to the Ember Data store, and is automatically available to routes. We call `findAll()` to load all records, passing the record name of `'restaurant'` to it.

Now let's create a component to render these restaurants:

```sh
$ ember g component RestaurantList
```

Open the generated `app/templates/components/restaurant-list.hbs` and enter the following:

```handlebars
{% raw %}<ul>
  {{#each @restaurants as |restaurant|}}
    <li>
      {{restaurant.name}}
    </li>
  {{/each}}
</ul>{% endraw %}
```

Now include the component in the index route's template, passing the restaurants to it:

```handlebars
{% raw %}<RestaurantList @restaurants={{this.model}} />{% endraw %}
```

Start the app:

```sh
$ ember server
```

Go to `http://localhost:4200` and you'll see some sample restaurants that were created by default for you when you signed up for a Sandbox API account.

Now that we've set up reading our data, let's see how we can write data. Let's allow the user to create a new restaurant.

We'll create a component to hold our new restaurant form:

```sh
$ ember g component NewRestaurantForm
```

Open the generated file `app/templates/components/new-restaurant-form.hbs` and add a simple form:

```handlebars
{% raw %}<form onSubmit={{action this.handleCreate}}>
  <div>
    Name:
    <Input type="text" @value={{this.name}} />
  </div>
  <div>
    Address:
    <Input type="text" @value={{this.address}} />
  </div>
  <button>Create</button>
</form>{% endraw %}
```

Now open the component's JavaScript file, `app/components/new-restaurant-form.js`, and add the following:

```diff
 import Component from '@ember/component';
+import { inject as service } from '@ember/service';

 export default Component.extend({
+  store: service(),

+  name: '',
+  address: '',

+  async handleCreate(event) {
+    event.preventDefault();
+
+    const post = this.store.createRecord('restaurant', {
+      name: this.name,
+      address: this.address,
+    });
+    await post.save();
+
+    this.setProperties({
+      name: '',
+      address: '',
+    });
+  },
 });
```

Components don't have access to the Ember Data `store` automatically, so we need to use `service()` to inject the `store` service into the component. From there, we can use the store just as we have in the route.

Now, add the `NewRestaurantForm` component to the index route's template:

```diff
{% raw %}+<NewRestaurantForm />
 <RestaurantList @restaurants={{this.model}} />{% endraw %}
```

Reload the app and you should be able to submit a new restaurant, and it should appear in the list right away. This is because Ember Data automatically adds it to the local store of restaurants; you don't need to do that manually.

Next, let's make a way to delete restaurants. Add a delete button to each list item:

```diff
{% raw %} <ul>
   {{#each @restaurants as |restaurant|}}
     <li>
       {{restaurant.name}}
+      <button
+        type="button"
+        onClick={{action this.deleteRestaurant restaurant}}
+      >
+        Delete
+      </button>
     </li>
   {{/each}}
 </ul>{% endraw %}
```

Implement `deleteRestaurant()` in `app/components/restaurant-list.js`:

```diff
 import Component from '@ember/component';

 export default Component.extend({
+  deleteRestaurant(restaurant) {
+    restaurant.destroyRecord();
+  }
 });
```

Try it out and you can delete records from your list. They're removed from the server, from your local Ember Data store, and from the screen.

Let's wrap things up by showing how you can load related data: the dishes for each restaurant.

Generate a new route for a restaurant detail page:

```sh
$ ember g route restaurant
```

Edit the route in `app/router.js` to take a parameter for the restaurant ID:

```diff
 Router.map(function() {
-  this.route('restaurant');
+  this.route('restaurant', { path: 'restaurant/:id' });
 });
```

In the route's JavaScript file `app/routes/restaurant.js`, add a model hook:

```diff
 export default Route.extend({
+  model({ id }) {
+    return this.store.findRecord('restaurant', id, {
+      include: 'dishes',
+    });
+  },
 });
```

Note that we pass an options object, specifying `include: 'dishes'`. This will request the related dish records to be returned in the response, so we don't have to make multiple requests.

This time let's render the list of records directly in the route's template, `app/templates/restaurant.hbs`:

```handlebars
{% raw %}<h1>{{this.model.name}}</h1>

<ul>
  {{#each this.model.dishes as |dish|}}
    <li>
      {{dish.name}}
      -
      {{dish.rating}} stars
    </li>
  {{/each}}
</ul>{% endraw %}
```

Finally, let's link each restaurant in the RestaurantList to its detail page:

```diff
{% raw %} <li>
-  {{restaurant.name}}
+  <LinkTo
+    @route="restaurant"
+    @model={{restaurant.id}}
+  >
+    {{restaurant.name}}
+  </LinkTo>
   <button
     type="button"{% endraw %}
```

Reload the app and click a link to go to a restauant detail page. You should see the dishes related to that restauant.

With that, our tutorial is complete. Notice how much functionality we got without needing to write any custom store code! JSON:API's conventions allow us to use a library like Ember Data to focus on our application and not on managing data.

Now that you have a JSON:API frontend, you should try creating your own backend to power it. Choose a tutorial from the [How to JSON:API home page](/)!

## More Options

Instead of Ember Data, you can try:

- [ember-orbit](https://github.com/orbitjs/ember-orbit), a more advanced client including offline storage and synchronization

[ember]: https://emberjs.com
