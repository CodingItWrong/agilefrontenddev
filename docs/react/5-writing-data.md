---
title: 5 - Writing Data
---

# 5 - Writing Data

Now we're on to our next user-facing feature: adding a restaurant. This will give us a chance to go through another outside-in sequence starting from an end-to-end test.

Create a file `cypress/integration/managing-restaurants.spec.js` and add the following:

```js
describe('Managing Restaurants', () => {
  it('allows adding restaurants', () => {
    const restaurantId = 27;
    const restaurantName = 'Sushi Place';

    cy.server({force404: true});

    cy.route({
      method: 'GET',
      url: 'http://localhost:3333/restaurants',
      response: [],
    });

    cy.route({
      method: 'POST',
      url: 'http://localhost:3333/restaurants',
      response: {
        id: restaurantId,
        name: restaurantName,
      },
    }).as('addRestaurant');

    cy.visit('/');

    cy.contains('New Restaurant').click();

    cy.get('[placeholder="Name"]').type(restaurantName);
    cy.contains('Save Restaurant').click();

    cy.wait('@addRestaurant')
      .its('requestBody')
      .should('deep.equal', {
        name: restaurantName,
      });

    cy.contains(restaurantName);
  });
});
```

As in our previous E2E test, we are stubbing the GET request to load the restaurants—but this time we're returning an empty array, as we don't need any restaurants for the test.

We also stub a POST request, which is the request we'll use to create a restaurant. From it, we return an object that is the new restaurant that is created. We also chain a call to `.as()` on it to give it the name `addRestaurant`—we'll see why in a moment.

We visit the home page, and this time we do some interaction with the page:

- We find an element containing the text "New Restaurant" and click it
- We find an element with a placeholder of "Name" (so, presumably a text input), and we type a restaurant name into it.
- We find an element "Save Restaurant" and click it.

Next, we call `cy.wait()`. This waits for an HTTP request to be sent. We pass the name of the request we want to wait for, prepending an `@` to it. Specifically, we wait for our `addRestaurant` request to complete. Then we check that the restaurant name is correctly sent in the body of the request. It's not enough to stub out the request: we need to confirm our app is sending the *right* data to the server too.

Finally, we confirm that the restaurant name is shown on the page, showing that the restaurant has been added to the list.

Start Cypress with `yarn test:e2e`, then choose the managing restaurants test. It fails, showing the first bit of functionality we need to implement:

> CypressError: Timed out retrying: Expected to find content: 'New Restaurant' but never did.

The test fails trying to find a button titled "New Restaurant" to click. Where should this button live? We are intending that that button shows the Add Restaurant form. Since that form only has one element, we don't need to put it in a modal; we can just display it on the card above the list of restaurants.

What component should the "New Restaurant" button be in? We discussed earlier that RestaurantScreen would hold both the restaurant list and new restaurant form. It makes sense that RestaurantScreen would also hold the New Restaurant button, and would hide or show a NewRestaurantForm component when clicked.

Because of this, let's add the New Restaurant button to `RestaurantScreen`. Material-UI has a `Button` component for displaying buttons.

```diff
 import CardContent from '@material-ui/core/CardContent';
 import Typography from '@material-ui/core/Typography';
+import Button from '@material-ui/core/Button';
 import RestaurantList from './RestaurantList';
...
     <CardContent>
       <Typography variant="h5">Restaurants</Typography>
+      <Button variant="contained">New Restaurant</Button>
       <RestaurantList />
     </CardContent>
```

Rerun the E2E test and the New Restaurant button is found, and we get to a new error:

> CypressError: Timed out retrying: Expected to find element: '[placeholder="Name"]', but never found it.

We need a "Name" text input. That should live on the New Restaurant Form, so it's time to create that component. Create the file `src/components/NewRestaurantForm.js`, and add the following:

```js
import React from 'react';
import TextField from '@material-ui/core/TextField';

export const NewRestaurantForm = () => {
  return (
    <form>
      <TextField placeholder="Name" fullWidth variant="filled" />
    </form>
  );
};

export default NewRestaurantForm;
```

Note the use of Material-UI's `TextField` component. Also note that we're using the block form of the arrow function. We will need other statements in there besides the returned JSX.

The simplest way to get this to appear in the `RestaurantScreen` is to show it unconditionally. The E2E test doesn't say clicking the "New Restaurant" button is *needed* to show the form:

THINK ABOUT IF TEST NEEDS TO DRIVE CONDITION

```diff
 import RestaurantList from './RestaurantList';
+import NewRestaurantForm from './NewRestaurantForm';

 const RestaurantScreen = () => (
   <Card>
     <CardContent>
       <Typography variant="h5">Restaurants</Typography>
       <Button variant="contained">New Restaurant</Button>
+      <NewRestaurantForm />
       <RestaurantList />
```

Rerun the E2E tests and they should get past finding and typing into the Name input. The next error is:

> CypressError: Timed out retrying: Expected to find content: 'Save Restaurant' but never did.

To fix this error, we add a button to `NewRestaurantForm` but don't wire it up to anything yet:

```diff
 import TextField from '@material-ui/core/TextField';
+import Button from '@material-ui/core/Button';

 export const NewRestaurantForm = () => {
   return (
     <form>
       <TextField placeholder="Name" fullWidth variant="filled" />
+      <Button variant="contained" color="primary">
+        Save Restaurant
+      </Button>
     </form>
   );
```

Rerun the E2E tests and we get this failure:

> CypressError: Timed out retrying: cy.wait() timed out waiting 5000ms for the 1st request to the route: 'addRestaurant'. No request ever occurred.

So now we need to send the request is our backend service. This is missing logic, so we will want to step down to unit tests to add it. How will it work?

- The `NewRestarantForm` component will dispatch an asynchronous Redux action
- The action will call a function in our API client
- The API client will make an HTTP `POST` request

Starting from the outside as usual, we'll start with the `NewRestarantForm` component. We want to reproduce the failure from the E2E test at the unit level. We should specify, when you click the send button, it should dispatch an action to the store. Now, the E2E test failure didn't tell us that we need to send along the restaurant name entered in the form, but we can go ahead and specify that that should be passed to the store, too. Otherwise we would need to go back through our stack to pass it along.

Create the file `src/components/__tests__/NewRestaurantForm.spec.js` and start out by setting up the component and a mock function in a `beforeEach` block:

```js
import React from 'react';
import {render, fireEvent} from '@testing-library/react';
import {NewRestaurantForm} from '../NewRestaurantForm';

describe('NewRestaurantForm', () => {
  const restaurantName = 'Sushi Place';

  let createRestaurant;
  let context;

  beforeEach(() => {
    createRestaurant = jest.fn().mockName(‘createRestaurant’);
    context = render(<NewRestaurantForm createRestaurant={createRestaurant} />);
  });
});
```

Next, let's try to proactively organize our test file. Since we're taking the approach of having one expectation per test, it's likely that we will ultimately have multiple expectations for different situations. So let's group situations with a `describe` block with a `beforeEach`, even if there's currently only one expectation. Add the following:

```js
  describe('when filled in', () => {
    beforeEach(() => {
      const {getByPlaceholderText, getByText} = context;

      fireEvent.change(getByPlaceholderText('Name'), {
        target: {value: restaurantName},
      });
      fireEvent.submit(getByText('Save Restaurant'));
    });

    it('calls createRestaurant with the name', () => {
      expect(createRestaurant).toHaveBeenCalledWith(restaurantName);
    });
  });
```

WHY DOES SUBMITTING THE BUTTON WORK? CLICK BUTTON OR SUBMIT FORM

We describe the situation when the form is filled in. We enter a restaurant name into a text field, then click a submit button. Note that like in the Cypress test we find elements by their placeholder and title text. Also note that we find the form and trigger a submit event on it, rather than finding the submit button and triggering a click event on it. CONFIRM THE BUTTON DOESN"T WORK, AND FIND OUT WHY.

In `RestaurantList` we didn't pass any additional data with our action, so we just had to confirm that the action function was called. But here, we need to ensure the restaurant name is passed as an argument to the action function, so we need to use the `.toHaveBeenCalledWith()` matcher (CHECK THIS TERM). We pass one argument to it, confirming that the correct `restaurantName` is passed through.

Save the file and we get a failing test, as we expect:

```sh
  ● NewRestaurantForm › when filled in › calls createRestaurant with the name

    expect(createRestaurant).toHaveBeenCalledWith(...expected)

    Expected: "Sushi Place"

    Number of calls: 0

      25 |

      26 |     it('calls createRestaurant with the name', () => {
    > 27 |       expect(createRestaurant).toHaveBeenCalledWith(restaurantName);
         |                                ^
```

The test failure reports the function wasn't called at all. To write just enough production code to get past the current test failure, let's just call the function without any arguments:

```diff
 import Button from '@material-ui/core/Button';

-export const NewRestaurantForm = () => {
+export const NewRestaurantForm = ({createRestaurant}) => {
   return (
-    <form>
+    <form onSubmit={() => createRestaurant()}>
       <TextField placeholder="Name" fullWidth variant="filled" />
```

We set up an `onSubmit` prop for the form tag, passing an arrow function that calls `createRestaurant`. We don't just pass the `createRestaurant` function directly because that would result in passing the browser event object to `createRestaurant`, what we don't want. This way there are no arguments.

Save the file and the test failure has changed:

```sh
  ● NewRestaurantForm › when filled in › calls createRestaurant with the name

    expect(createRestaurant).toHaveBeenCalledWith(...expected)

    Expected: "Sushi Place"
    Received: called with 0 arguments

    Number of calls: 1


      25 |
      26 |     it('calls createRestaurant with the name', () => {
    > 27 |       expect(createRestaurant).toHaveBeenCalledWith(restaurantName);
         |                                ^
```

The function didn't receive the argument it expected: it wanted "Sushi Place", but it didn't receive any arguments. To pass the restaurant name, first we're going to need to set up a state item for the name:

```diff
-import React from 'react';
+import React, {useState} from 'react';
 import TextField from '@material-ui/core/TextField';
 import Button from '@material-ui/core/Button';

 export const NewRestaurantForm = ({createRestaurant}) => {
+  const [name, setName] = useState('');
+
   return (
```

Then, we'll make `TextField` a controlled component, reading its value from the `name` state item and writing changes back using `setName`:

```diff
   return (
     <form onSubmit={() => createRestaurant()}>
-      <TextField placeholder="Name" fullWidth variant="filled" />
+      <TextField
+        value={name}
+        onChange={e => setName(e.target.value)}
+        placeholder="Name"
+        fullWidth
+        variant="filled"
+      />
       <Button variant="contained" color="primary">
```

Finally, now that the entered text is stored in `name`, we'll pass that as the argument to `createRestaurant()`:

```diff
   return (
-    <form onSubmit={() => createRestaurant()}>
+    <form onSubmit={() => createRestaurant(name)}>
       <TextField
```

Save the file and the test passes.

We'll circle back to test-drive edge case functionality to the form later, but for now let's move on toward passing our E2E test by test-driving the store module. The restaurants module needs a `create` action that will make the appropriate call to the API, then insert the resulting record into the store. Let's write that test now. Below the "load action" group, add a "create action" group, and write a test to confirm the API is called:

```js
  describe('create action', () => {
    const newRestaurantName = 'Sushi Place';

    let api;
    let store;

    beforeEach(() => {
      api = {
        createRestaurant: jest.fn().mockName('createRestaurant'),
      };

      const initialState = {};

      store = createStore(
        restaurantsReducer,
        initialState,
        applyMiddleware(thunk.withExtraArgument(api)),
      );

      store.dispatch(createRestaurant(newRestaurantName));
    });

    it('saves the restaurant to the server', () => {
      expect(api.createRestaurant).toHaveBeenCalledWith(newRestaurantName);
    });
  });
```

We'll need to add a second expectation shortly so we go ahead and set up the test in a `beforeEach`.

We also need to import `createRestaurant`:

```diff
 import restaurantsReducer from '../restaurants/reducers';
-import {loadRestaurants} from '../restaurants/actions';
+import {loadRestaurants, createRestaurant} from '../restaurants/actions';

 describe('restaurants', () => {
```

The test fails because the API method was not called:

```sh
  ● restaurants › create action › saves the restaurant to the server

    expect(createRestaurant).toHaveBeenCalledWith(...expected)

    Expected: "Sushi Place"

    Number of calls: 0
```

We also get an error that the function doesn't exist:

```sh
    TypeError: (0 , _actions.createRestaurant) is not a function

      138 |       );
      139 |
    > 140 |       store.dispatch(createRestaurant(newRestaurantName));
          |                      ^
      141 |     });
```

Let's fix that error first. Export an empty function from `actions.js`:

```diff
 const recordLoadingError = () => ({type: RECORD_LOADING_ERROR});

+export const createRestaurant = () => {};
```

We get the error again that its return value isn't a valid thing to dispatch:

```sh
  ● restaurants › create action › saves the restaurant to the server

    Actions must be plain objects. Use custom middleware for async actions.

      138 |       );
      139 |
    > 140 |       store.dispatch(createRestaurant(newRestaurantName));
          |             ^
```

So let's have it return a function:

```diff
-export const createRestaurant = () => {};
+export const createRestaurant = () => () => {};
```

This fixes the error, so now we just get the expectation failure that `api.createRestaurant` wasn't called. Update the `createRestaurant` thunk to call it:

```diff
-export const createRestaurant = () => () => {};
+export const createRestaurant = () => (dispatch, getState, api) => {
+  api.createRestaurant();
+};
```

This changes the test failure. Now the method is called, but not with the right arguments:

```sh
● restaurants › create action › saves the restaurant to the server

    expect(createRestaurant).toHaveBeenCalledWith(...expected)

    Expected: "Sushi Place"
    Received: called with 0 arguments

    Number of calls: 1

      142 |
      143 |     it('saves the restaurant to the server', () => {
    > 144 |       expect(api.createRestaurant).toHaveBeenCalledWith(newRestaurantNam
e);

          |                                    ^
```

Our restaurant name is passed in as the first argument of the action, so we can pass it along to the API method:

```diff
-export const createRestaurant = () => (dispatch, getState, api) => {
+export const createRestaurant = name => (dispatch, getState, api) => {
-  api.createRestaurant();
+  api.createRestaurant(name);
 };
```

Save the file and the test passes. Now we need to specify one more thing that happens when the `create` action is dispatched: the returned restaurant from the API, including the ID that the API gives the record, is appended to the restaurant list in the state. To write that test, we're going to need to add a little to the setup as well:

```diff
   describe('create action', () => {
     const newRestaurantName = 'Sushi Place';
+    const existingRestaurant = {id: 1, name: 'Pizza Place'};
+    const responseRestaurant = {id: 2, name: newRestaurantName};

     let api;
     let store;

     beforeEach(() => {
       api = {
-        createRestaurant: jest.fn().mockName('createRestaurant'),
+        createRestaurant: jest
+          .fn()
+          .mockName('createRestaurant')
+          .mockResolvedValue(responseRestaurant),
       };

-      const initialState = {};
+      const initialState = {records: [existingRestaurant]};

       store = createStore(
```

This ensures the API call promise resolves, and provides a restaurant record for it. We also add a different restaurant to the pre-existing list of restaurants in the store. Save the file and the tests should still pass.

Now we're ready to specify that the returned restaurant is added to the store:

```js
    it('stores the returned restaurant in the store', () => {
      expect(store.getState().records).toEqual([
        existingRestaurant,
        responseRestaurant,
      ]);
    });
```

We ensure that the existing restaurant is still in the store, and the restaurant record returned from the server is added after it. Save the file and the test fails:

```sh
  ● restaurants › create action › stores the returned restaurant in the store

    expect(received).toEqual(expected) // deep equality

    - Expected
    + Received

      Array [
        Object {
          "id": 1,
          "name": "Pizza Place",
        },
    -   Object {
    -     "id": 2,
    -     "name": "Sushi Place",
    -   },
      ]

      152 |
      153 |     it('stores the returned restaurant in the store', () => {
    > 154 |       expect(store.getState().records).toEqual([
          |                                        ^
```

The store only contains the restaurant it was initialized with, not the new one the server returned. Let's update the action to handle the returned value:

```diff
 export const RECORD_LOADING_ERROR = 'RECORD_LOADING_ERROR';
+export const ADD_RESTAURANT = 'ADD_RESTAURANT';

 export const loadRestaurants = () => (dispatch, getState, api) => {
...
 export const createRestaurant = name => (dispatch, getState, api) => {
-  api.createRestaurant(name);
+  api.createRestaurant(name).then(record => dispatch(addRestaurant(record)));
 };
+
+const addRestaurant = record => ({
+  type: ADD_RESTAURANT,
+  record,
+});
```

After `createRestaurant()` resolves, we take the record the API returns to us and dispatch a new `addRestaurant()` action. Now let's respond to that action in the reducer:

```diff
   RECORD_LOADING_ERROR,
+  ADD_RESTAURANT,
 } from './actions';
...
 const records = (state = [], action) => {
   switch (action.type) {
     case STORE_RESTAURANTS:
       return action.records;
+    case ADD_RESTAURANT:
+      return [...state, action.record];
     default:
       return state;
   }
 };
```

When `ADD_RESTAURANT` is dispatched we set records to a new array including the previous array, plus the new record on the end.

Now our component and store should be set. Wire up the action to the form component by changing `NewRestaurantForm.js` to export a connected component instead:

```diff
 import React, {useState} from 'react';
+import {connect} from 'react-redux';
 import TextField from '@material-ui/core/TextField';
 import Button from '@material-ui/core/Button';
+import {createRestaurant} from '../store/restaurants/actions';

 export const NewRestaurantForm = ({createRestaurant}) => {
...
 };

-export default NewRestaurantForm;
+const mapStateToProps = null;
+const mapDispatchToProps = {createRestaurant};
+
+export default connect(mapStateToProps, mapDispatchToProps)(NewRestaurantForm);
```

Rerun the E2E test, and the API call isn't made. This is because the button in `NewRestaurantForm` isn't a submit button, so it's not submitting the form. Our test confirmed what submitting the form did, but it didn't confirm what clicking the button did, due to limitations with React Testing Library. That's what we have E2E tests for! To fix this, make the button a submit button:

```diff
         variant="filled"
         />
-      <Button variant="contained" color="primary">
+      <Button variant="contained" color="primary" type="submit">
         Save Restaurant
       </Button>
```

Now when we rerun the E2E test, we get a surprising addition in the test output. After the click:

- (FORM SUB) --submitting form--
- (PAGE LOAD) --page loaded--
- (NEW URL) http://localhost:3000/?
- (XHR STUB) GET 200 /restaurants

This sounds like the page is being reloaded, and it is. This is because by default HTML forms make their own request to the server when they're submitted, refreshing the page. This is because HTML forms predate using JavaScript to make HTTP requests. This reload restarts our frontend app, losing our progress. To prevent this from happening, we need to call the `preventDefault()` method on the event sent to the `onSubmit` event. We can do this by extracting a handler function:

```diff
 export const NewRestaurantForm = ({createRestaurant}) => {
   const [name, setName] = useState('');

+  const handleSubmit = e => {
+    e.preventDefault();
+    createRestaurant(name);
+  };
+
   return (
-    <form onSubmit={() => createRestaurant(name)}>
+    <form onSubmit={handleSubmit}>
       <TextField
```

Rerun the E2E test, and check the console:

```sh
TypeError: api.createRestaurant is not a function
```

Our component is successfully dispatching the action to the store, which is successfully calling `api.createRestaurant()`, but we haven't implemented it yet. Let's do that now. Remember, we don't unit test our API, so we can implement this method directly, driven by the E2E test. Let's start by fixing the immediate error by defining an empty `createRestaurant()` method:

```diff
 const api = {
   loadRestaurants() {
     return client.get('/restaurants').then(response => response.data);
   },
+  createRestaurant() {},
 };
```

Now we get another console error:

```sh
TypeError: Cannot read property 'then' of undefined
```

DON"T GET THIS IN REACT: But we also get a test failure after a few seconds:

> CypressError: Timed out retrying: cy.wait() timed out waiting 5000ms for the 1st request to the route: 'addRestaurant'. No request ever occurred.

We still aren't making the HTTP request that kicked off this whole sequence. Fixing this will move us forward better, so let's actually make the HTTP request in the API:

```diff
   },
-  createRestaurant() {},
+  createRestaurant() {
+    return client.post('/restaurants', {});
+  },
 };
```

Now the `POST` request is made, and we get an error on the assertion we made about the request's body:

> ASSERT expected {} to deeply equal { name: Sushi Place }

So we aren't passing the restaurant name in the `POST` body. That's easy to fix by passing it along from the argument to the method:

```diff
-  createRestaurant() {
+  createRestaurant(name) {
-    return client.post('/restaurants', {});
+    return client.post('/restaurants', {name});
  },
```

Cypress confirms we're sending the `POST` request to the server correctly, and we've finally moved on to the next E2E assertion failure:

> CypressError: Timed out retrying: Expected to find content: 'Sushi Place' but never did.

We aren't displaying the restaurant on the page. This is because we aren't yet returning it properly from the resolved value. The Axios promise resolves to the Axios response object, but we want to return a promise that resolves to the record. We can do this by getting the response body:

LOOK INTO WHY WE DIDN"T DO THIS BEFORE

```diff
   createRestaurant(name) {
-    return client.post('/restaurants', {name});
+    return client.post('/restaurants', {name}).then(response => response.data);
   },
```

Rerun the E2E test and it passes, and we see Sushi Place added to the restaurant list. Our feature is complete!

## Edge Cases
Now let's look into those edge cases:

* The form should clear out the text field after you save a restaurant
* If the form is submitted with an empty restaurant name, it should show a validation error, and not submit to the server
* If the save fails an error message should be shown, and the restaurant name should not be cleared

First, let's implement the form clearing out the text field after saving. In `NewRestaurantForm.spec.js`, add a new test:

```diff
     it('calls createRestaurant with the name', () => {
       expect(createRestaurant).toHaveBeenCalledWith(restaurantName);
     });
+
+    it('clears the name', () => {
+      const {getByPlaceholderText} = context;
+      expect(getByPlaceholderText('Name').value).toEqual('');
+    });
   });
```

Save the test, and we get a test failure confirming that the text field is not yet cleared:

```sh
  ● NewRestaurantForm › when filled in › clears the name

    expect(received).toEqual(expected) // deep equality

    Expected: ""
    Received: "Sushi Place"

      30 |     it('clears the name', () => {
      31 |       const {getByPlaceholderText} = context;
    > 32 |       expect(getByPlaceholderText('Name').value).toEqual('');
         |                                                  ^
```

Where in the component should we clear the text field? Well, we have another story that the name should _not_ be cleared if the web service call fails. If that's the case, then we should not clear the text field until the store action resolves successfully. Make this change in `NewRestaurantForm.js`:

```diff
     handleSave() {
-      this.createRestaurant(this.name);
+      this.createRestaurant(this.name).then(() => {
+        setName('');
+      });
     },
```

Save the file and the test fails, and we get a lot of scary console output. Let's handle one thing at a time. The easier one to fix is:

```sh
Error: Uncaught [TypeError: Cannot read property 'then' of undefined]
```

Our mocked `api.createRestaurant` doesn't return a promise; let's update it to return a resolved one:

```diff
     beforeEach(() => {
+      createRestaurant.mockResolvedValue();
+
       const {getByPlaceholderText, getByText} = context;
```

Save and the test now passes; what's left is a warning:

```sh
      Warning: An update to NewRestaurantForm inside a test was not wrapped in act(.
..).

      When testing, code that causes React state updates should be wrapped into act(
...):

      act(() => {
        /* fire events that update state */
      });

      /* assert on the output */

      This ensures that you're testing the behavior the user would see in the browse
r.
```

What exactly is going on here is complex and beyond the scope of this tutorial. Specifically in this case, the React component state is being updated asynchronously, and React is warning us that we may have timing issues.

In this specific case, the easiest fix is to use the `flush-promises` npm package. Add it to your project:

```sh
$ yarn add --dev flush-promises
```

Then add it to your test:

```diff
 import React from 'react';
-import {render, fireEvent} from '@testing-library/react';
+import {render, fireEvent, act} from '@testing-library/react';
+import flushPromises from 'flush-promises';
 import {NewRestaurantForm} from '../NewRestaurantForm';
...
       fireEvent.submit(getByText('Save Restaurant'));
+
+      return act(flushPromises);
     });
```

We call `act()` at the end of our `beforeEach` block, passing it the `flushPromises` function. This means that React will call that function and wait for it to resolve, responding to component changes that may have happened appropriately. We return the result of `act`, so that Jest will wait on *that* before running individual tests.

Save the file and our test finally passes cleanly!

Before we proceed, let's think about some refactoring. Look at the following statement from our test:

```js
fireEvent.change(getByPlaceholderText('Name'), {
	target: {value: restaurantName},
});
```

This is pretty verbose. There are a lot of details in there about the structure of browser events. Our test isn't really thinking in those terms; we just want to say that we fill in a field with a value. Let's make a helper function at the top of this test file to do that for us:

```js
const fillIn = (element, value) => fireEvent.change(element, {target: {value}});
```

Then we can replace our existing call:

```diff
     beforeEach(() => {
       createRestaurant.mockResolvedValue();

       const {getByPlaceholderText, getByText} = context;

-      fireEvent.change(getByPlaceholderText('Name'), {
-        target: {value: restaurantName},
-      });
+      fillIn(getByPlaceholderText('Name'), restaurantName);
       fireEvent.submit(getByText('Save Restaurant'));

       return act(flushPromises);
     });
```

This reads a lot more nicely: "fill in the field with placeholder "Name", with restaurantName". By contrast, `fireEvent.submit()` reads pretty simply as-is; we don't need to make a helper for it.

Now let's implement the validation error. Create a new `describe` block for this situation, below the "when filled in" describe block. We'll start with just one of the expectations, to confirm a validation error is shown:

```js
  describe('when empty', () => {
    beforeEach(() => {
      createRestaurant.mockResolvedValue();

      const {getByPlaceholderText, getByText} = context;
      fillIn(getByPlaceholderText('Name'), '');
      fireEvent.submit(getByText('Save Restaurant'));

      return act(flushPromises);
    });

    it('displays a validation error', () => {
      const {queryByText} = context;
      expect(queryByText('Name is required')).not.toBeNull();
    });
  });
```

We don't actually need the line that sets the value of the text field to the empty string, because right now it starts out empty. But explicitly adding that line make the intention of the test more clear. And that way, if we did decide in the future to start the form out with default text, we would be sure this test scenario still worked. It's a judgment call whether to add it or not.

Save the file and the test fails, because the error message is not found:

```sh
  ● NewRestaurantForm › when empty › displays a validation error

    expect(received).not.toBeNull()

    Received: null

      50 |     it('displays a validation error', () => {
      51 |       const {queryByText} = context;
    > 52 |       expect(queryByText('Name is required')).not.toBeNull();
         |                                                   ^
```

Let's fix this error in the simplest way possible by adding the error message unconditionally:

```diff
 import Button from '@material-ui/core/Button';
+import Alert from '@material-ui/lab/Alert';
 import {createRestaurant} from '../store/restaurants/actions';
...
   return (
     <form onSubmit={handleSubmit}>
+      <Alert severity="error">Name is required</Alert>
       <TextField
```

The tests pass. Now how can we write a test to drive out hiding that error message in other circumstances? Well, we can check that it's not shown when the form is initially mounted.

In preparation, let's move the error message text we're searching for to a constant directly under our top-level `describe`:

```diff
 describe('NewRestaurantForm', () => {
   const restaurantName = 'Sushi Place';
+  const requiredError = 'Name is required';
+
   let createRestaurant;
...
     it('displays a validation error', () => {
       const {queryByText} = context;
-      expect(queryByText('Name is required')).not.toBeNull();
+      expect(queryByText(requiredError)).not.toBeNull();
     });
```

Save and confirm the tests still pass.

Next, add a new `describe` above the "when filled in" one:

```js
  describe('initially', () => {
    it('does not display a validation error', () => {
      const {queryByText} = context;
      expect(queryByText(requiredError)).toBeNull();
    });
  });
```

The test fails because we are always showing the error right now:

```sh
  ● NewRestaurantForm › initially › does not display a validation error

    expect(received).toBeNull()

    Received: <div class="MuiAlert-message">Name is required</div>

      20 |     it('does not display a validation error', () => {
      21 |       const {queryByText} = context;
    > 22 |       expect(queryByText(requiredError)).toBeNull();
         |                                          ^
```

Time to add some logic around this error. We'll add state to indicate whether it should be shown:

```diff
 export const NewRestaurantForm = ({createRestaurant}) => {
   const [name, setName] = useState('');
+  const [validationError, setValidationError] = useState(false);

   const handleSubmit = e => {
...
   return (
     <form onSubmit={handleSubmit}>
-      <Alert severity="error">Name is required</Alert>
+      {validationError && <Alert severity="error">Name is required</Alert>}
       <TextField
```

Now, what logic should we use to set the `validationError` flag? Our tests just specify that initially the error is not shown, and after submitting an invalid form it's shown--that's all. The simplest logic to pass this test is to always show the validation error after saving:

```diff
   const handleSubmit = e => {
     e.preventDefault();
+    setValidationError(true);
     createRestaurant(name).then(() => {
```

Save the file and all tests pass.

It may feel obvious to you that this is not the correct final logic, so this should drive us to consider what test we are missing. What should behave differently? Well, when we submit a form with a name filled in, the validation error should not appear. Let's add that test to the "when filled in" `describe` block:

```js
it('does not display a validation error', () => {
  const {queryByText} = context;
  expect(queryByText(requiredError)).toBeNull();
});
```

We can pass this test by adding a conditional around setting the `validationError` flag:

```diff
   const handleSubmit = e => {
     e.preventDefault();
-    setValidationError(true);
+
+    if (!name) {
+      setValidationError(true);
+    }
+
     createRestaurant(name).then(() => {
```

Save the file and all tests pass.

Now, is there any other time we would want to hide or show the error message? Well, if the user submits an empty form, gets the error, then adds the missing name and submits it again, we would want the validation error cleared out. Let's create this scenario as another `describe` block, below the "when empty" one:

```js
  describe('when correcting a validation error', () => {
    beforeEach(() => {
      createRestaurant.mockResolvedValue();

      const {getByPlaceholderText, getByText} = context;
      fillIn(getByPlaceholderText('Name'), '');
      fireEvent.submit(getByText('Save Restaurant'));
      fillIn(getByPlaceholderText('Name'), restaurantName);
      fireEvent.submit(getByText('Save Restaurant'));

      return act(flushPromises);
    });

    it('clears the error message', () => {
      const {queryByText} = context;
      expect(queryByText(requiredError)).toBeNull();
    });
  });
```

Note that we repeat both sets of `beforeEach` steps from the other groups, submitting the empty form and then submitting the filled-in one. We want our unit tests to be independent, so they can be run without depending on the result of other tests. If this repeated code got too tedious we could extract it to helper functions that we could call in each `describe` block.

Save the test file and our new test fails:

```sh
  ● NewRestaurantForm › when correcting a validation error › clears the error messag
e

    expect(received).toBeNull()

    Received: <div class="MuiAlert-message">Name is required</div>

      80 |     it('clears the error message', () => {

      81 |       const {queryByText} = context;
    > 82 |       expect(queryByText(requiredError)).toBeNull();
         |                                          ^
```

We can fix this by clearing the `validationError` flag upon a successful submission:

```diff
     handleSave() {
       if (!this.name) {
         this.validationError = true;
+      } else {
+        this.validationError = false;
       }
```

Note that we aren't waiting for the web service to return to clear it out, the way we clear out the name field. We know right away that the form is valid, so we can clear it before the web service call is made.

Save and the tests pass. Now that we have an `each` branch to that conditional, let's invert the boolean to make it easier to read. Refactor it to:

```js
      if (this.name) {
        this.validationError = false;
      } else {
        this.validationError = true;
      }
```

Save and the tests should still pass.

Now we can handle the other expectation for when we submit an empty form: it should not dispatch the action to save the restaurant to the server. Add a new test in the "when empty" `describe` block:

```js
    it('does not call createRestaurant', () => {
      expect(createRestaurant).not.toHaveBeenCalled();
    });
```

We can fix this error by moving the call to `this.createRestaurant()` inside the true branch of the conditional:

```diff
     if (name) {
       setValidationError(false);
+      createRestaurant(name).then(() => {
+        setName('');
+      });
     } else {
       setValidationError(true);
     }
-
-    createRestaurant(name).then(() => {
-      setName('');
-    });
   };
```

Save the file and the test passes.

Our third exception case is when the web service call fails. We want to display an error message. We'll want to check for the message in a few different places, so let's set it up as a constant in the uppermost `describe` block:

```diff
 describe('NewRestaurantForm', () => {
   const requiredError = 'Name is required';
+  const serverError = 'The restaurant could not be saved. Please try again.';

   let createRestaurant;
```

Since this is a new situation, let's set this up as yet another new `describe` block:

```js
  describe('when the store action rejects', () => {
    beforeEach(() => {
      createRestaurant.mockRejectedValue();

      const {getByPlaceholderText, getByText} = context;

      fillIn(getByPlaceholderText('Name'), restaurantName);
      fireEvent.submit(getByText('Save Restaurant'));

      return act(flushPromises);
    });

    it('displays a server error', () => {
      const {queryByText} = context;
      expect(queryByText(serverError)).not.toBeNull();
    });
  });
```

This is the same as the successful submission case, but in the setup we call the `mockRejectedValue()` method of the mock function `restaurantsModule.actions.create`. This means that when this function is called, it will reject. In our case we don't actually care about what error it rejects with, so we don't have to provide a rejected value.

Save the file and we get an expectation failure:

```sh
  ● NewRestaurantForm › when the store action rejects › displays an error message

    expect(received).not.toBeNull()

    Received: null

      104 |       expect(queryByText(serverError)).not.toBeNull();
          |                                            ^
```

As usual, we'll first solve this by hard-coding the element into the component:

```diff
   return (
     <form onSubmit={handleSubmit}>
+      <Alert severity="error">
+        The restaurant could not be saved. Please try again.
+      </Alert>
       {validationError && <Alert severity="error">Name is required</Alert>}
```

Save and we get a bit of a strange error:

```sh
  ● NewRestaurantForm › when the store action rejects › displays an error message

    thrown: undefined

      100 |     });
      101 |
    > 102 |     it('displays an error message', () => {
          |     ^
```

The message isn't very helpful, but "thrown" is a clue. What's happening is that our call to `createRestaurants()` is rejecting, but we aren't handling it. Let's handle it with an empty `catch()` function, just to silence this warning; we'll add behavior to that `catch()` function momentarily.

```diff
     if (name) {
       setValidationError(false);
-      createRestaurant(name).then(() => {
-        setName('');
-      });
+      createRestaurant(name)
+        .then(() => {
+          setName('');
+        })
+        .catch(() => {});
     } else {
```

Save and the test passes. Now, when do we want that message to *not* show? For one thing, when the component initially mounts. Add another test to the "initially" describe block:

```js
    it('does not display a server error', () => {
      const {queryByText} = context;
      expect(queryByText(serverError)).toBeNull();
    });
```

We'll add another bit of state to track whether the error should show, starting hidden, and shown if the store action rejects:

```diff
 export const NewRestaurantForm = ({createRestaurant}) => {
   const [name, setName] = useState('');
   const [validationError, setValidationError] = useState(false);
+  const [serverError, setServerError] = useState(false);

   const handleSubmit = e => {
...
       createRestaurant(name)
         .then(() => {
           setName('');
         })
-        .catch(() => {});
+        .catch(() => {
+          setServerError(true);
+        });
     } else {
...
   return (
     <form onSubmit={handleSubmit}>
-      <Alert severity="error">
-        The restaurant could not be saved. Please try again.
-      </Alert>
+      {serverError && (
+        <Alert severity="error">
+          The restaurant could not be saved. Please try again.
+        </Alert>
+      )}
       {validationError && <Alert severity="error">Name is required</Alert>}
```

Save and the tests pass.

Let's also write a test to confirm that the server is not shown after the server request returns successfully. In the "when filled in" describe block, add an identical test:

```js
    it('does not display a server error', () => {
      const {queryByText} = context;
      expect(queryByText(serverError)).toBeNull();
    });
```

Save and the test passes. This is another instance where the test doesn't drive new behavior, but it's helpful for extra assurance that the code is behaving the way we expect.

We also want to hide the server error message each time we retry saving the form. This is a new situation, so let's create a new `describe` block for it:

```js
  describe('when retrying after a server error', () => {
    beforeEach(() => {
      createRestaurant.mockRejectedValueOnce().mockResolvedValueOnce();

      const {getByPlaceholderText, getByText} = context;
      fillIn(getByPlaceholderText('Name'), restaurantName);
      fireEvent.submit(getByText('Save Restaurant'));
      fireEvent.submit(getByText('Save Restaurant'));
      return act(flushPromises);
    });

    it('clears the server error', () => {
      const {queryByText} = context;
      expect(queryByText(serverError)).toBeNull();
    });
  });
```

SHOW PROBLEM FIRST?

We'll actually run into a problem clicking the submit button twice in a row, though. We want to wait for the first web request to return, _then_ send the second one. We can fix this by waiting for promises to flush after the first click, as well as after the second:

```diff
-    beforeEach(() => {
+    beforeEach(async () => {
       createRestaurant.mockRejectedValueOnce().mockResolvedValueOnce();

       const {getByPlaceholderText, getByText} = context;
       fillIn(getByPlaceholderText('Name'), restaurantName);
       fireEvent.submit(getByText('Save Restaurant'));
+      await act(flushPromises);
+
       fireEvent.submit(getByText('Save Restaurant'));
       return act(flushPromises);
    });
```

Note that we need to make the `beforeEach` function `async`, so we can `await` the call to `flushPromises()`. This ensures the results of the first click will complete before we start the second.

Save the file and you'll get the expected test failure:

```sh
  ● NewRestaurantForm › when retrying after a server error › clears the server error

    expect(received).toBeNull()

    Received: <div class="MuiAlert-message">The restaurant could not be saved. Pleas
e try again.</div>

      133 |     it('clears the server error', () => {
      134 |       const {queryByText} = context;
    > 135 |       expect(queryByText(serverError)).toBeNull();
          |                                        ^
```

We can make this test pass by just clearing the `serverError` when attempting to save:

```diff
     if (name) {
       setValidationError(false);
 +     setServerError(false);
       createRestaurant(name)
```

Save and the test should pass.

Now we have just one more test to make: that the restaurant name is not cleared when the server rejects. This should already be working because of how we implemented the code, but it would be frustrating for the user if they lost their data, so this is an especially important case to test. Add another expectation to the "when the store action rejects" `describe` block:

```js
    it('does not clear the name', () => {
      const {getByPlaceholderText} = context;
      expect(getByPlaceholderText('Name').value).toEqual(restaurantName);
    });
```

Save and the test passes, confirming that the user's data is safe.

That was a lot of edge cases, but we've added a lot of robustness to our form!

Imagine if we had tried to handle all of these cases in E2E tests. We either would have had a lot of slow tests, or else one long test that ran through an extremely long sequence. Instead, our E2E tests cover our main functionality, and our unit tests cover all the edge cases thoroughly.

Try running your E2E test and you get an error:

> Uncaught TypeError: Cannot read property 'then' of undefined

This is because of a mismatch between the contract of our component and async action:

- `NewRestaurantForm` expects its `createRestaurant` function prop to return a promise
- But our `createRestaurant` async action doesn't return the promise of the action it creates.

We can fix this by just returning:

```diff
 export const createRestaurant = name => (dispatch, getState, api) => {
-  api.createRestaurant(name).then(record => dispatch(addRestaurant(record)));
+  return api
+    .createRestaurant(name)
+    .then(record => dispatch(addRestaurant(record)));
 };
```

Rerun the E2E test and it passes.

REFACTOR FOR LAYOUT