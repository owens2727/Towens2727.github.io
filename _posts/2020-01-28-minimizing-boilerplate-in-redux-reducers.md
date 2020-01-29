---
layout: post
title: Minimizing Boilerplate in Redux Reducers
description: Redux reducers handle updates to application state. Most reducers use common patterns that can be abstracted and re-used across multiple reducers. In this post, we introduce reducer creators, a means of minimizing reducer boilerblate and avoiding redudant reducer logic.
---

[Redux reducers](https://redux.js.org/basics/reducers/) handle updates to application state. Most reducers use common patterns that can be abstracted and re-used across multiple reducers. In this post, we examine *reducer creators* (also called *higher-order reducers* or *reducer factories*), a means of minimizing reducer boilerblate and avoiding redudant reducer logic.

This post explores the motivations behind the design patterns introduced by [Re-using Reducer Logic from the Redux](https://redux.js.org/recipes/structuring-reducers/reusing-reducer-logic). We'll also take things one step further by applying this design pattern to C.R.U.D. style resources fetched from and managed by a RESTful API.

### What are reducers?

> "Reducers specify how the application's state changes in response to actions sent to the store. Remember that actions only describe what happened, but don't describe how the application's state changes."
>
>  \- [Redux docs](https://redux.js.org/basics/reducers/)

Reducers are a critical part of the Redux architecture. Without them, we have no means of affecting the Redux store, our application's state.

Reducers receive actions dispatched from the app and decide how the application state should change in response. A simple reducer might look like:

```javascript
const todoList = (state = [], action) => {
  switch (action.type) {
    case 'ADD_TODO':
      return [...state, action.todo];
    case 'REMOVE_TODO':
      return state.filter(todo => todo.id !== action.todo.id);
    case 'CLEAR_TODOS':
      return [];
    default:
      return state;
  }
};
```

The above reducer helps store a list of to-do items. We set the default value with the `state = []` default function parameter, indicating that the list is empty to begin with. Actions of type `ADD_TODO`, `REMOVE_TODO`, and `CLEAR_TODOS` can update the state. All other action types will trigger the default case, where we simply leave the to-do list as is.

### Adding reducers can introduce redundancy

Let's say we wanted to track several to-do lists — one for today's tasks, one for later this week, and one for later this year. And for other reasons, we want to maintain these lists as seperate pieces of application state (rather than one list which can be filtered, perhaps a better solution for this contrived example).

We may end up with several reducers that look awfully similar:

```javascript
const todayTodoList = (state = [], action) => {
  switch (action.type) {
    case 'ADD_TODAY_TODO':
      return [...state, action.todo];
    case 'REMOVE_TODAY_TODO':
      return state.filter(todo => todo.id !== action.todo.id);
    case 'CLEAR_TODAY_TODOS':
      return [];
    default:
      return state;
  }
};

const weekTodoList = (state = [], action) => {
  switch (action.type) {
    case 'ADD_WEEK_TODO':
      return [...state, action.todo];
    case 'REMOVE_WEEK_TODO':
      return state.filter(todo => todo.id !== action.todo.id);
    case 'CLEAR_WEEK_TODOS':
      return [];
    default:
      return state;
  }
};

const yearTodoList = (state = [], action) => {
  // More duplicative reducer logic...
}
```

### Using reducer creators to eliminate redundancies

Even though the action types are different across the three reducers, we clearly have some redundant patterns that could be abstracted. If, down the line, we made updates to how to-do lists are handled, we'd have to update all three reducers, which is a recipe for disaster.

So how can we maintain these three independent lists while re-using common logic?

We introduce a function that *returns a reducer*. Since reducers are functions themselves, this is a function that returns a function.

*(note: lots of powerful Javascript patterns rely on fuctions that return functions. Some involve functions that return functions that return functions. Yuck. In these cases, it's often helpful to give certain function types a name that indicates what they're responsible for. Here, we have a **reducer creator** that returns a **reducer**.)*

```javascript
const todoListReducerCreator = () => {
  const todoList = (state = [], action) => {
    switch (action.type) {
      case 'ADD_TODO':
      // ...
    }
  }

  return todoList;
}
```

To make better use of syntax shorthands, we can re-write the above as:

```javascript
const todoListReducerCreator = () => (state = [], action) => {
  switch (action.type) {
    case 'ADD_TODO':
    // ...
  }
}
```

Calling `todoListReducerCreator()` will return a to-do list reducer that looks exactly like our simple, single to-do list.

So far, our factory isn't configurable — it takes no parameters and spits out the same reducer each time. To support multiple to-do lists, we need a factory that can churn out different types of to-do list reducers.


```javascript
const todoListReducerCreator = (dueBy) => (state = [], action) => {
  if (action.dueBy !== dueBy) {
    return state;
  }

  switch (action.type) {
    case 'ADD_TODO':
    // ...
  }
}

const todayTodoList = todoListReducerCreator('today')
const weekTodoList = todoListReducerCreator('week')
const yearTodoList = todoListReducerCreator('year')
```

Now, when we call `todoListReducerCreator('today')`, we're configuring the reducer with a `dueBy = 'today'` parameter, providing that reducer with a specific flavor.

We've also elected to keep the names of our action types general: `ADD_TODO`, `REMOVE_TODO`, `CLEAR_TODOS`, and so on. To ensure that dispatched actions impact the right to-do list, we additionally provide an `action.dueBy` field on actions that will target the desired to-do list reducer. The guard clause at the beginning of our reducer creator enforces this constraint. Without it, `ADD_TODO` would add each new item to *every* to-do list.

### Simple example of a reducer creator (list)

### More complicated resource reducer