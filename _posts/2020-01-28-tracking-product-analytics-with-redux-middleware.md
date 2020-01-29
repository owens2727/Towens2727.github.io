---
layout: post
title: Tracking Product Analytics with Redux Middleware
description: How to add product analytics tracking (like Google Analytics) into a React + Redux web app client using Redux middleware.
---

This post illustrates how to track product analytics for a web application by listening in on Redux actions from Redux middleware. Our goal is to observe when the user takes certain actions within the app and make external API calls to a product analytics tool with information about those actions.

For primers on these topics, see:
 - [Product analytics](https://mixpanel.com/topics/what-is-product-analytics/) — logging, aggregating, and exposing information about how users interact with your application
 - [Redux](https://redux.js.org/introduction/getting-started) — a framework for managing web client application state, often paired with React
 - [Redux Middleware](https://redux.js.org/advanced/middleware) — a configurable layer within Redux that triggers whenever a Redux action is dispatched

### Motivation: The Single Responsibility Principle

One of the highlights of working with React and Redux is their commitment to the **Single Responsibility Principle** — each module has exactly one job to do. React components render HTML. Redux actions define messages that the application can send to the Redux store. Redux reducers describe how the application state should change in response to actions. Everything has its place, and there are few side effects.

To introduce product analytics into this (hypothetically) pristine application, we need to make calls like:


```javascript
trackEvent({
  type: 'Checkout',
  itemsInCart: 5,
  costOfCart: 45.20,
  deviceType: 'mobile',
  userId: '19ddf5a2-110d-4172-a33c-c17ca479fee2'
})
```

The `trackEvent` call makes an external request to a product analytics tool (like Mixpanel or Google Analytics) that logs the payload and exposes it for further analysis. Within the product analytics tool, we can answer questions like *how likely is a user to checkout on mobile vs. web?*

`trackEvent` is by nature a side effect, and it'll feel out of place regardless of where call it from our React + Redux web client. So where should this call go?

 - It could live in **React components** within listeners like `onClick` or `onSubmit`, but this violates the single responsibility of react components: receiving props and returning HTML.
 - It could live in **Redux action creators**, but this violates the single responsibility of Redux action creators: defining the messages that the application can send to the Redux store.
 - It could live in **Redux reducers**, but this violates the single responsibility of Redux reducers: updating the application state in response to actions.

 None of these options are particularly appealing because they place an extra, unrelated burden on a part of the codebase that already has a job to do.


### Redux Middleware

Redux Middleware is a layer within Redux that triggers whenever a Redux action is dispatched. We register middleware with Redux on app initialization:

```javascript

import { createStore, combineReducers, applyMiddleware } from 'redux';
import rootReducer from './reducers';

// A module we'll explore and write later on...
import { analyticsMiddleware } from './middleware';


const todoApp = combineReducers(reducers);
const store = createStore(
  rootReducer,
  applyMiddleware(analyticsMiddleware)
);
```

Redux middleware can be used for an arbitrary number of purposes. Consider writing custom middleware for logic that:

1. should fire in response to every action
2. observes changes to the application, rather than causing or modifying them

This makes middleware a great place for logic that concerns logging, error handling, and analytics tracking.

*(note: contrary to the second point, Redux middlewares can in fact modify actions or even dispatch new ones, but that kind of power should be used with caution)*

### Dispatching Product Analytics Events in Redux Middleware

The skeleton of the product analytics middleware takes the form:

```javascript
const analyticsMiddleware = store => next => action => {
  const result = next(action);

  // We'll write our product analytics logic here...

  return result;
}
```

(*note: If this syntax is unfamiliar, check out [nested ES6 arrow functions](https://stackoverflow.com/a/32787782)*)

This function will fire on every action dispatched across the application. The middleware has access to the Redux store as well as the  action payload. It also receives `next`, a function that acts similar to `store.dispatch` in the context of middleware.

We'll now use the dispatched `action` to make `trackEvent` requests to our product analytics tool.

```javascript
const analyticsMiddleware = store => next => action => {
  const result = next(action);
  const state = store.getState();

  trackEventFromAction(action, state);

  return result;
}
```

`trackEventFromAction` is a function that takes the just-dispatched action, the current state, and sends an API call to our product analytics tool. It might look something like:

```javascript
const trackEventFromAction(action, state) {
  switch(action.type) {
    case 'USER_SIGN_UP':
      return trackEvent({
        type: 'Sign Up',
        userId: action.userId,
        signUpTimestamp: action.signUpTimestamp,
        // Likely include more properties here that are useful for
        // segmenting / filtering within your product analytics
        // tool.
      })
    case 'USER_LOG_IN':
      return trackEvent({
        type: 'Log In',
        // Copy over desired properties from the action to the
        // product analytics event
      })
    case 'ADD_ITEM_TO_CART':
      return trackEvent({
        type: 'Add Item to Cart',
        // ...
      })
    case 'CHECKOUT':
      return trackEvent({
        type: 'Checkout',
        itemsInCart: action.itemsInCart,
        costOfCart: action.costOfCart,
        deviceType: state.deviceType,
        userId: state.userId,
      })
    default:
      return null;
  }
}
```

We're now listening in on any action dispatched from any location in the app and making product analytics tracking requests in response. Using the switch statement, we can write custom logic for each action type, extracting information that is specific to that action type. We also have the option of calling `store.getState` to include additional state information (like the current duration of the user's session, for example) in the `trackEvent` payload.

### Benefits of Tracking Product Analytics in Middleware

So what've we gained through this approach?

**Tracking additional event types is easy.** We simply add another `case` to `switch(action.type)` to accomodate tracking on a new action type.

**All the product analytics logic lives in one place.** The single responsibility of the middleware is to listen to dispatched actions and track product analytics events accordingly.

**Actions are tracked regardless of where they fire from.** If a new component is added that fires an existing action type, we don't have to worry about copying and duplicating a `trackEvent` call into the new component, introducing redundant code.

### Shortcomings

**Issues in middleware are often hard to debug.** The middleware will fire on *every* action dispatch — that's a lot of chances for something to go wrong in a somewhat obscure part of the React + Redux architecture.

**Logic in middleware is easy to forget about.** With this setup, it's quite possible to add a new action type to the application and forget to update the middleware to log the new action. *"Middleware? More like Middle... where...?"*

**Only actions can trigger product analytics events.** In some cases, we may want to track an event that doesn't impact the Redux store. We'd have to create a new Redux action type that the middleware would use but the reducers would ultimately ignore.


### Wrapping Up

In this post, we explored how to add product analytics to a React + Redux client using Redux middleware. While it's just one approach to the problem, it does satisfy two chief objectives:

**The single responsibility principle** is satisfied — each module should has one and only one job.

We've stayed **D.R.Y. (don't repeat yourself)** — any potentially repetitive code is abstracted into reusable chunks.