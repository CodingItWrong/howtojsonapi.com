---
title: Building a JSON:API Frontend with Ember Data
---

[Ember.js][ember] includes a built-in data layer library, Ember Data, that is targeted at connecting to JSON:API web services.

To try it out, let's create a webapp for browsing a video games database.

Create a new Ember app using [Ember CLI][ember-cli]:

```sh
$ npm install -g ember-cli
$ ember new --no-welcome video-games
$ cd video-games
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

Now, we need to set up models for each type of resource we want to create. Generate these with Ember CLI:

```sh
$ ember generate model system
$ ember generate model game
```

We need to edit each model file to declare the attributes and relationships it has. First edit `app/models/game.js`;

```diff
 import DS from 'ember-data';
-const { Model } = DS;
+const { Model, attr, belongsTo } = DS;

 export default Model.extend({
+  title: attr(),
+  year: attr(),
+  system: belongsTo('system'),
 });
```

Then edit `app/models/system.js`:

```diff
 import DS from 'ember-data';
-const { Model } = DS;
+const { Model, attr, hasMany } = DS;

 export default Model.extend({
+  name: attr(),
+  games: hasMany('game'),
 });
```

The argument to `belongsTo()` and `hasMany()` is the name of the model the field relates to.

That's all we have to do to set up our data layer! Now let's put it to use.

Let's display a list of the video game systems. In Ember, data loading typically happens in the route, so that Ember can intelligently handle routing as the data loads. Generate a route file for the index route, the root of the site:

```sh
$ ember generate route index
```

Then add a model hook to the route:

```diff
 import Route from '@ember/routing/route';

 export default Route.extend({
+  model() {
+    return this.store.findAll('system');
+  },
 });
```

`this.store` provides access to the Ember Data store, and is automatically available to routes. We call `findAll()` to load all records, passing the record name of `'system'` to it.

Now let's create a component to render these systems:

```sh
$ ember generate component SystemList
```

Open the generated `app/templates/components/system-list.hbs` and enter the following:

```handlebars
{% raw %}<ul>
  {{#each @systems as |system|}}
    <li>
      {{system.name}}
    </li>
  {{/each}}
</ul>{% endraw %}
```

Now include the component in the index route's template, passing the systems to it:

```handlebars
{% raw %}<SystemList @systems={{this.model}} />{% endraw %}
```

Go to `http://localhost:4200` and you should see the list of video game systems.

Now that we've set up reading our data, let's see how we can write data. Let's allow the user to create a new video game system.

We'll create a component to hold our new system form:

```sh
$ ember generate component NewSystemForm
```

Open the generated file `app/templates/components/new-system-form.hbs` and add a simple form:

```handlebars
{% raw %}<form onSubmit={{action this.handleCreate}}>
  <Input type="text" @value={{this.name}} />
  <button>Create</button>
</form>{% endraw %}
```

Now open the component's JavaScript file, `app/components/new-system-form.js`, and add the following:

```diff
 import Component from '@ember/component';
+import { inject as service } from '@ember/service';

 export default Component.extend({
+  store: service(),
+  name: '',
+  async handleCreate(event) {
+    event.preventDefault();
+    const post = this.store.createRecord('post', {
+      name: this.name,
+    });
+    await post.save();
+    this.name = '';
+  },
 });
```

Components don't have access to the Ember Data `store` automatically, so we need to use `service()` to inject the `store` service into the component. From there, we can use the store just as we have in the route.

Now, add the `NewSystemForm` component to the index route template:

```diff
{% raw %}+<NewSystemForm />
 <SystemList @systems={{this.model}} />{% endraw %}
```

Run the app and you should be able to submit a new system, and it should appear in the list right away. This is because Ember Data automatically adds it to the local store of systems; you don't need to do that manually.

Finally, let's make a way to delete systems. Add a delete button to each list item:

```diff
{% raw %} <ul>
   {{#each @systems as |system|}}
     <li>
       {{system.name}}
+      <button
+        type="button"
+        onClick={{action this.deleteSystem system}}
+      >
+        Delete
+      </button>
     </li>
   {{/each}}
 </ul>{% endraw %}
```

Implement `deleteSystem()` in `app/components/system-list.js`:

```diff
 import Component from '@ember/component';

 export default Component.extend({
+  deleteSystem(system) {
+    system.destroyRecord();
+  }
 });
```

Try it out and you can delete records from your list. They're removed from the server, from your local Ember Data store, and from the screen.

Let's wrap things up by showing how you can load related data: the video games for each system.

Generate a new route for a system detail page:

```sh
$ ember generate route system
```

Edit the route in `app/router.js` to take a parameter for the system ID:

```diff
 Router.map(function() {
-  this.route('system');
+  this.route('system', { path: 'system/:id' });
 });
```

In the route's JavaScript file, add a model hook:

```diff
 export default Route.extend({
+  model({ id }) {
+    return this.store.findRecord('system', id, {
+      include: 'games',
+    });
+  },
 });
```

Note that we pass an options object, specifying `include: 'games'`. This will request the related games records to be returned in the response, so we don't have to make multiple requests.

This time let's render the list of games directly in the route's template, `app/templates/system.hbs`:

```handlebars
{% raw %}<h1>{{this.model.name}}</h1>

<ul>
  {{#each this.model.games as |game|}}
    <li>
      {{game.title}}
      ({{game.year}})
    </li>
  {{/each}}
</ul>{% endraw %}
```

Finally, let's link each system in the SystemList to its detail page:

```diff
{% raw %} <li>
-  {{system.name}}
+  <LinkTo @route="system" @model={{system.id}}>{{system.name}}</LinkTo>
   <button
     type="button"{% endraw %}
```

Reload the app and click a link to go to a system detail page. You should see the games related to that system.

With that, our tutorial is complete. Notice how much functionality we got without needing to write any custom store code! JSON:API's conventions allow us to use a library like Ember Data to focus on our application and not on managing data.

Now that you have a JSON:API frontend, you should try creating your own backend to power it. Choose a tutorial from the [How to JSON:API home page](/)!

[ember]: https://emberjs.com
