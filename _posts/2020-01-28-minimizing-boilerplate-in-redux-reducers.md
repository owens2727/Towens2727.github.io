---
layout: post
title: Minimizing Boilerplate in Redux Reducers
description: An exploration of reducer creators — functions that return reducers — which can abstract potentially redundant and repeated logic across similar reducers.
---

[Redux reducers](https://redux.js.org/basics/reducers/) handle updates to application state. Most reducers use common patterns that can be abstracted and re-used across multiple reducers. In this post, we examine *reducer creators* (also called *higher-order reducers* or *reducer factories*), a means of minimizing reducer boilerplate and avoiding redundant reducer logic.

This post explores the motivations behind the design patterns introduced by [Reusing Reducer Logic from the Redux](https://redux.js.org/recipes/structuring-reducers/reusing-reducer-logic). We'll also take things one step further by applying this design pattern to C.R.U.D. style resources fetched from and managed by a RESTful API.

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

So how can we maintain these three independent lists while reusing common logic?

We introduce a function that *returns a reducer*. Since reducers are functions themselves, this is a function that returns a function.

*(note: lots of powerful Javascript patterns rely on functions that return functions. Some involve functions that return functions that return functions. Yuck. In these cases, it's often helpful to give certain function types a name that indicates what they're responsible for. Here, we have a **reducer creator** that returns a **reducer**.)*

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

To make better use of syntax shorthands, we can rewrite the above as:

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

### Applying reducer creators to CRUD Resources

Many React + Redux applications interact with resources stored on a back-end server and accessed via an API. The client can *create, read, update, and destroy* these resources — actions commonly referenced by the acronym CRUD.

We'll extend our to-do list example to consider an application where to-do items are stored on a persistent back-end.

Following Redux best practices for [Redux async actions](https://redux.js.org/advanced/async-actions/) and resources we:
 - store resources as a map (object) from the resource ID to the full resource object
 - update the store when back-end requests complete (dispatching `_SUCCESS` actions)
 - never mutate the state

```javascript
import { omit } from 'lodash';

const resourceReducerCreator = resourceType => (
  (state = {}, action) => {
    if (action.resourceType !== resourceType) {
      return state;
    }

    switch (action.type) {
      case 'CREATE_RESOURCE_SUCCESS':
      case 'READ_RESOURCE_SUCCESS':
      case 'UPDATE_RESOURCE_SUCCESS':
        return { ...state, [action.resource.id]: action.resource };
      case 'DESTROY_RESOURCE_SUCCESS':
        return omit(state, [action.resource.id]);
      default:
        return state;
    }
  }
);

const todos = resourceReducerCreator('todos');
```

*(note: Javascript switch cases fall through until they hit a `return` or `break` statement. We take advantage of this cascading syntax to reuse logic for reading, creating, and updating resources.)*

Here `resourceReducerCreator` is a reducer creator that returns reducers that can manage resources. We call it with the name of the resource (`'todos'`) — this prevents cross-contamination of actions that concern a different type of resource.

Create, read, and update operations all add the resource to the store, replacing an existing copy of that resource if necessary. The destroy operation removes the resource from the `id -> resource` map.

If we wanted to allow for sub-tasks on each to-do item, we could easily introduce a new resource reducer with:

```javascript
const subtasks = resourceReducerCreator('todos');
```

Entire applications can spring up quickly with minimal additional reducer logic:

```javascript
const users = resourceReducerCreator('users');
const notes = resourceReducerCreator('notes');
const priorities = resourceReducerCreator('priorities');
const bookmarks = resourceReducerCreator('bookmarks');
```

### Wrapping Up

In this post, we looked at one pattern for minimizing repetitive logic in reducers. There are plenty of other strategies for [reducing boilerplate in Redux applications](https://redux.js.org/recipes/reducing-boilerplate). Boilerplate is often a nicer way of saying "redundant code," and redundant code can lead to nasty bugs and slower development. Be wary of code sections that have different variable names but redundant control flows (ifs, elses, loops, etc.). Even though these sections aren't precise *copies* of each other, the control flow itself can often be abstracted.