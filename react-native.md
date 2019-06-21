---
layout: tutorial
title: Building a JSON:API App with React Native and Reststate/Mobx
logo: /images/react.svg
logo_alt: React logo
---

[Reststate/Mobx][reststate-mobx] is a library for creating frontends driven by JSON:API backends using the MobX data layer.

To try it out, let's create a React Native webapp for rating dishes at restaurants. We'll call it "Opinion Ate".

Create a new app using React Native CLI:

```sh
$ npm install -g react-native-cli
$ react-native init OpinionAteReactNative
```

Next, let's add our data layer dependencies:

```sh
$ yarn add @reststate/mobx mobx mobx-react axios
```

In addition to Reststate/Mobx, these include:
- [`mobx` and `mobx-react`](https://mobx.js.org/) - for reactivity in React apps, including React Native
- [`axios`](https://github.com/axios/axios) - a web service client

To demonstrate more of a realistic multi-screen application, let's add [React Navigation](https://reactnavigation.org/docs/en/getting-started.html) as well:

```sh
$ yarn add react-navigation react-native-gesture-handler
$ react-native link react-native-gesture-handler
```

Go ahead and start Metro bundler in one tab:

```sh
$ yarn start
```

And start the iOS app in another:

```sh
$ react-native run-ios
```

Next, we want to use `@reststate/mobx` to create stores for handling restaurants and dishes. The JSON:API web service we'll be connecting to is [sandboxapi.reststate.org](https://sandboxapi.reststate.org/), a free service that allows you to create an account so you can write data as well as read it. Sign up for an account there.

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

Let's set up an `axios` client with that access token to handle the web service connection. Create a `src` folder and a `src/api.js` file and add the following:

```javascript
import axios from 'axios';

const token = '[the token you received from the POST request above]';

const httpClient = axios.create({
  baseURL: 'https://sandboxapi.reststate.org/',
  headers: {
    'Content-Type': 'application/vnd.api+json',
    Authorization: `Bearer ${token}`,
  },
});
```

Now we'll create a `src/stores.js` module and create Reststate stores for restaurants and dishes:

```js
import { ResourceStore } from '@reststate/mobx';
import api from './api';

const restaurantStore = new ResourceStore({
  name: 'restaurants',
  httpClient: api,
});

const dishStore = new ResourceStore({
  name: 'dishes',
  httpClient: api,
});

export {
  restaurantStore,
  dishStore
};
```

That's all we have to do to set up our data layer! Now let's put it to use.

Let's set up the home screen to display a list of the restaurants.

First, replace the content of `App.js` with the following:

```jsx
import { createStackNavigator, createAppContainer } from 'react-navigation';
import RestaurantList from './src/RestaurantList';

const RootStack = createStackNavigator({
  RestaurantList,
});

export default createAppContainer(RootStack);
```

This sets up a `RestaurantList` component to be rendered as the first screen of our app. Now let's create that component.

Create `src/RestaurantList.js` and add the following content:

```jsx
{% raw %}import React, { Component } from 'react';
import { Button, FlatList, Text, View } from 'react-native';
import { observer } from 'mobx-react';
import { restaurantStore } from './stores';

class RestaurantList extends Component {
  static navigationOptions = {
    title: 'Restaurants',
  };

  componentDidMount() {
    restaurantStore.loadAll();
  }

  render() {
    return (
      <View>
        <FlatList
          data={restaurantStore.all().slice()}
          keyExtractor={(item) => item.id}
          renderItem={({ item: restaurant }) => (
            <View style={{
              flex: 1,
              flexDirection: 'row',
              justifyContent: 'space-between',
              alignItems: 'center',
            }}>
              <Text>{restaurant.attributes.name}</Text>
            </View>
          )}
        />
      </View>
    );
  }
}

export default observer(RestaurantList);{% endraw %}
```

Notice a few things:

- We call the `loadAll()` method on the `restaurantStore` object to load our data. We do this in the `componentDidMount` lifecycle method.
- In the `render` method, we acccess the restaurants using the `all()` method of the store. Then we call `slice()` on that collection to turn it into a new array. *When using FlatList with MobX, calling slice() is extremely important.* This ensures that the MobX collection is dereferenced synchronously while `render()` is being run, so MobX can track it as a dependency. (To learn more, read MobX's instructions on [Rendering ListViews in React Native](https://mobx.js.org/best/pitfalls.html#rendering-listviews-in-react-native).)
- The restaurant's ID is available as a property on the `restaurant` directly, but its name is under a `restaurants.attributes` object. This is the standard JSON:API resource object format, and to keep things simple `@reststate/mobx` exposes resources in the same format as JSON:API.
- We wrap the returned component in `mobx-react`'s `observer()` function. This is what will cause React Native to rerender the component when the list of restaurants updates--we'll see how later.

Reload the app in the simulator and you'll see some sample restaurants that were created by default for you when you signed up for a Sandbox API account.

A nice enhancement we could do would be to show the user when the data is loading from the server, or if it has errored out. To do this, let's check a few properties on the store in the template:

```diff
 render() {
+  if (restaurantStore.loading) {
+    return <Text>Loading…</Text>;
+  }
+  if (restaurantStore.error) {
+    return <Text>Error loading restaurants.</Text>;
+  }
   return (
     <div>
```

Now reload the app and you should briefly see the "Loading" message before the data loads. If you'd like to see the error message, change the `baseURL` in `api.js` to some incorrect URL, and the request to load the data will error out.

Now that we've set up reading our data, let's see how we can write data. Let's allow the user to create a new restaurant.

To do this, we'll create a `NewRestaurantForm` component. Create `src/NewRestaurantForm.js` and add the following typical React Native form. We'll handle actually creating the restaurant in a separate step:

```jsx
import React, { Component } from 'react';
import { Button, TextInput, View } from 'react-native';
import { restaurantStore } from './stores';

const initialState = {
  name: '',
  address: '',
};

export default class NewRestaurantForm extends Component {
  state = initialState;

  updateField = (field) => (text) => {
    this.setState({ [field]: text });
  }

  createRestaurant = async () => {
  }

  render() {
    const { name, address } = this.state;
    return (
      <View>
        <TextInput
          placeholder="Name"
          value={name}
          onChangeText={this.updateField('name')}
        />
        <TextInput
          placeholder="Address"
          value={address}
          onChangeText={this.updateField('address')}
        />
        <Button
          title="Create"
          onPress={this.createRestaurant}
        />
      </View>
    );
  }
}
```

Now let's hook this form up to our store:

```diff
 import { Button, TextInput, View } from 'react-native';
+import { restaurantStore } from './stores';

 const initialState = {
...
   createRestaurant = async () => {
+    const { name, address } = this.state;
+    await restaurantStore.create({
+      attributes: { name, address },
+    });
+
+    this.setState(initialState);
  }
```

Notice a few things:

- The object we pass to `restaurantStore.create()` follows the JSON:API resource object format: the attributes are under an `attributes` object. (If you know JSON:API, you may notice that we aren't passing a `type` property, though--`@reststate/mobx` can infer that from the fact that we're in the `restaurants` store.)
- We clear out the name and address after the `create` operation succeeds.

To use this form, we just need to add it to our `RestaurantList`:

```diff
 import { restaurantStore } from './stores';
+import NewRestaurantForm from './NewRestaurantForm';

 class RestaurantList extends Component {
...
     return (
       <View>
+        <NewRestaurantForm />
         <FlatList
```

Reload the app and you should be able to submit a new restaurant, and it should appear in the list right away. `@reststate/mobx` automatically adds it to the local store of restaurants; you don't need to do that manually.

We said earlier that wrapping `RestaurantList` in `mobx-react`'s `observer()` function allowed it to rerender when the list of restaurants changes. How does this work? `observer()` creates a higher-order component that watches to see which MobX data is accessed during the `render()` method. Then, when any of that data changes, MobX tells the components to rerender, displaying the updated data. This happens without us needing to explicitly declare the `render` method's dependencies! (To learn more about how MobX reactivity works, react about [MobX Concepts and Principles](https://mobx.js.org/intro/concepts.html).)

Next, let's make a way to delete restaurants. Add a delete button to each list item:

```diff
{% raw %} <View style={{
   flex: 1,
   flexDirection: 'row',
   justifyContent: 'space-between',
   alignItems: 'center',
 }}>
   <Text>{restaurant.attributes.name}</Text>
+  <Button
+    title="Delete"
+    onPress={() => restaurant.delete()}
+  />
 </View>{% endraw %}
```

This is all we need to do; the `restaurant` is a rich object with methods like `delete()` that will make the appropriate web service request and update the local store. Try it out and you can delete records from your list.

Let's wrap things up by showing how you can load related data: the dishes for each restaurant.

In `App.js`, add a new route to point to a restaurant detail component:

```diff
 import RestaurantList from './RestaurantList';
+import RestaurantDetail from './RestaurantDetail';

 const RootStack = createStackNavigator({
   RestaurantList,
+  RestaurantDetail,
 });
```

Create a new `src/RestuarantDetail.js` file for this component and start with the following:

```jsx
import React, { Component } from 'react';
import { FlatList, Text } from 'react-native';
import { observer } from 'mobx-react';
import { dishStore } from './stores';

class RestaurantDetail extends Component {
  componentDidMount() {
  }

  render() {
  }
}

export default observer(RestaurantDetail);
```

First let's retrieve the restaurant from the route and use it to set the title of the React Navigation screen:

```diff
 class RestaurantDetail extends Component {
+  static navigationOptions = ({ navigation }) => {
+    return {
+      title: navigation.getParam('restaurant').attributes.name,
+    };
+  };

   componentDidMount() {
```

Next, when the component mounts let's retrieve the related dishes from the store:

```diff
 componentDidMount() {
+  const restaurant = this.props.navigation.getParam('restaurant');
+  dishStore.loadRelated({ parent: restaurant });
 }
```

Now we access those dishes in the render method:

```diff
 render() {
+  const restaurant = this.props.navigation.getParam('restaurant');
+  const dishes = dishStore.related({ parent: restaurant });
+  return (
+    <FlatList
+      data={dishes.slice()}
+      keyExtractor={(item) => item.id}
+      renderItem={({ item: dish }) => (
+        <Text>{dish.attributes.name}</Text>
+      )}
+    />
+  );
}
```

Notice that we remembered to use `slice()` so MobX detects that we're accessing these records, for the sake of reactivity.

Finally, let's link each restaurant in the list to its detail screen:

```diff
{% raw %}
 <View style={{
   flex: 1,
   flexDirection: 'row',
   justifyContent: 'space-between',
   alignItems: 'center',
 }}>
-  <Text>{restaurant.attributes.name}</Text>
+  <Button
+    title={restaurant.attributes.name}
+    onPress={() => {
+      this.props.navigation.navigate('RestaurantDetail', {
+        restaurant,
+      })
+    }}
+  />
   <Button
     title="Delete"
     onPress={() => restaurant.delete()}
   />
 </View>{% endraw %}
```

Go back to the root of the app and tap a link to go to a restauant detail screen. You should see the dishes related to that restauant.

With that, our tutorial is complete. Notice how much functionality we got without needing to write any custom store code! JSON:API's conventions allow us to use a zero-configuration library like `@reststate/mobx` to focus on our application and not on managing data.

Now that you have a JSON:API frontend, you should try creating your own backend to power it. Choose a tutorial from the [How to JSON:API home page](/)!

## More Options

Instead of Reststate/Mobx, you can try:

- [redux-json-api](https://github.com/redux-json-api/redux-json-api), for use with Redux
- [react-orbitjs](https://orbitjs.com/), a more advanced client including offline storage and synchronization

[reststate-mobx]: https://mobx.reststate.org
