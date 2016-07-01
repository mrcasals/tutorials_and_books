# Getting started with Redux
Notes from the course on Egghead.io by Dan Abramov.

https://egghead.io/courses/getting-started-with-redux

## Intro
State is saved in an object.

Actions are used to change the state. Must be an object with a `type` field
that should be a string. It can have multiple fields with data related to the.
action (for example, the index of the counter we are incrementing) This way,
the state can be reproducable easily. Any data that gets into our application
gets there by actions, either if it comes form a network request or by user
input.

Pure vs Impure functions:
  * Pure functions are purely functional: no side-effects, and they do not
    modify elements of an array, but instead they return a new array.
  * Impure functions may cause network connections, override values that are
    passed to them, etc.

Useful pure functions:
* `Object.assign({}, old_object, modifications)` (only in ES6)
* `{...old_object, field: new_Value}` (splat operator)
* `Array.slice` and `Array.concat`

State mutations need to be described by a pure function, that takes the
previous state and the action being dispatched and returns the next state.
This is called **reducer**.

## First example: counter

    // This is the reducer
    // ES6 version
    const counter = (state = 0, action) => {
      switch (action.type){
        case 'INCREMENT':
          return state + 1;
        case 'DECREMENT':
          return state - 1;
        default:
          return state;
      }
    }

    const { crateStore } = Redux;
    const store = createStore(counter);

    console.log(store.getState()); // 0

    store.dispatch({ type: 'INCREMENT' });
    console.log(store.getState()); // 1

Example with the same `counter` function and Redux, but changing the HTML:

    const render = () => {
      document.body.innerText = store.getState();
    };

    store.subscribe(render);
    render(); // to render the initial state

    document.addEventListener('click', () => {
      store.dispatch({ type: 'INCREMENT' });
    });

This is how the `createStore` function works:

    const createStore = (reducer) => {
      let state;
      let listeners = [];

      const getState = () => state;

      const dispatch = (action) => {
        state = reducer(state, action);
        listeners.forEach(listener => listener());
      }

      const subscribe = (listener) => {
        listeners.push(listener);
        return () => {
          listeners = listeners.filter(l => l !== listener);
        };
      }

      // dummy action to get the reducer to return the initial value
      dispatch({});

      return { getState, dispatch, subscribe };
    };

Now, instead of updating the counter in the HTML manually, we can use React:

    const Counter = ({
      value,
      onIncrement,
      onDecrement,
    }) => (
      <div>
        <h1>{value}</h1>
        <button onClick={onIncrement}>+</button>
        <button onClick={onDecrement}>-</button>
      </div>
    );

    // new `render` version
    // requires a `<div id="root"></div>` element
    const render = () {
      ReactDOM.render(
        <Counter
          value={Store.getState()}
          onIncrement={() =>
            store.dispatch({
              type: 'INCREMENT'
            });
          }
          onDecrement={() =>
            store.dispatch({
              type: 'DECREMENT'
            });
          }
        />,
        document.getElementById('root');
      );
    };

## Second example: todos

    const todos = (state = [], action) => {
      switch (action.type) {
        case 'ADD_TODO':
          return [
            ...state,
            {
              id: action.id,
              text: action.text,
              completed: false
            }
          ];
        case 'TOGGLE_TODO':
          return state.map(todo => {
            if (todo.id !== action.id) {
              return todo;
            }

            return {
              ...todo,
              completed: !todo.completed
            }
          });
        default:
          return state;
      }
    }

We're starting to see that this reducer deals with both the collection and
individual items, so we'll split it in two reducers. This is called **reducer
composition** and helps you decoupling and isolating the functionalities of the
reducer. This is also easier to test.

    const todo = (state, action) => {
      switch (action.type) {
        case 'ADD_TODO':
          return {
            id: action.id,
            text: action.text,
            completed: false
          };
        case 'TOGGLE_TODO':
          if (state.id !== action.id) {
            return state;
          }

          return {
            ...state,
            completed: !state.completed
          };
        default:
          return state;
      }
    };

    const todos = (state = [], action) => {
      switch (action.type) {
        case 'ADD_TODO':
          return [
            ...state,
            todo(undefined, action)
          ];
        case 'TOGGLE_TODO':
          return state.map(t => todo(t, action));
        default:
          return state;
      }
    }

What if we want to store more information? Let's say, a filter that lets the
user choose which todos are visible: completed, incompleted, etc. We can just
create another reducer that wraps our `todos` reducer, and wrap our `todos`
array in an object that will hold all the state:

    const visibilityfilter = (
      state = 'SHOW_ALL',
      action
    ) => {
      switch (action.type) {
        case 'SET_VISIBILITY_FILTER':
          return action.filter;
        default:
          return state;
      }
    };

    const todoApp = (state = {}, action) => {
      return {
        todos: todos(
          state.todos, action
        ),
        visibilityFilter: visibilityFilter(
          state.visibilityFilter,
          action
        );
      };
    }

    const { createStore } = Redux;
    const store = createStore(todoApp);

This pattern is so common that most Redux implementations have this by default,
so you can replace the `todoApp` definition by:

    const { combineReducers } = Redux;
    const todoApp = combineReducers({
      todos: todos,
      visibilityfilter: visibilityFilter
    });

Note that the keys are the fields of the state object, and values are the
reducers it shouldc all to update the corresponding state fields. So it's
important to **name your reducers after the state keys they manage**, so you
can make the code even shorter:

    const { combineReducers } = Redux;
    const todoApp = combineReducers({
      todos,
      visibilityFilter
    });

This is how the `combineReducers` function works:

    const combineReducers = (reducers) => {
      return (state = {}, action) => {
        return Object.keys(reducers).reduce(
          (nextState, key) => {
            nextState[key] = reducers[key](
              state[key],
              action
            );
            return nextState;
          },
          {}
        );
      };
    };

## Todos: adding the view layout
Ok, so now we have all the reducers needed to build our todo app. Let's build
the view layer with React:

    const { Component } = React;

    let nextTodoId = 0;
    class TodoApp extends Component {
      render() {
        return (
          <div>
            <input ref={node =>
              this.input = node
            } />
            <button onClick={() => {
              store.dispatch({
                type: 'ADD_TODO',
                text: this.input.value,
                id: nextTodoId++
              });
              this.input.value = '';
            }}>
              Add button
            </button>
            <ul>
              {this.props.todos.map(todo =>
                <li key={todo.id}>
                  {todo.text}
                </li>
              )}
            </ul>
          </div>
        );
      }
    }

    const render = () => {
      ReactDOM.render(
        <TodoApp
          todos={store.getState().todos}
        />,
        document.getElementById('root')
      );
    };

    store.subscribe(render);
    render();