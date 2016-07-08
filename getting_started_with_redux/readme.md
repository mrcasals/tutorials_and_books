# Getting started with Redux
Notes from the course on Egghead.io by Dan Abramov.

https://egghead.io/courses/getting-started-with-redux

## Instructions for the example code
Run this in your terminal:

```
npm run build
```

And open the `index.html` file.

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

    const visibilityFilter = (
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

## Todos: toggling a todo
Now we want to toggle a todo. We need to edit the `li` element in our React
code:

    <li key={todo.id}
        onClick={() => {
          store.dispatch({
            type: 'TOGGLE_TODO',
            id: todo.id
          });
        }}
        style={{
          textDecoration:
            todo.completed ?
              'line-through' :
              'none'
        }}>
      {todo.text}
    </li>

## Todos: Filtering todos
First we need a component to switch between filters:

    const FilterLink = ({
      filter,
      children
    }) => {
      return (
        <a href="#"
          onClick={e => {
            e.preventDefault();
            store.dispatch({
              type: 'SET_VISIBILITY_FILTER',
              filter
            });
          }}
        >
          {children}
        </a>
      );
    };

Then, in our `TodoApp` component, beneath the `ul` element, we show the
filters:

    </ul>
    <p>
      Show:
      {' '}
      <FilterLink
        filter='SHOW_ALL'
      >
        All
      </FilterLink>
      {' '}
      <FilterLink
        filter='SHOW_ACTIVE'
      >
        Active
      </FilterLink>
      {' '}
      <FilterLink
        filter='SHOW_COMPLETED'
      >
        Completed
      </FilterLink>
    </p>

And we need a way to get the todos by the current active filter:

    const getVisibleTodos = (
      todos,
      filter
    ) => {
      switch (filter) {
        case 'SHOW_ALL':
          return todos;
        case 'SHOW_COMPLETED';
          return todos.filter(
            t => t.completed
          );
        case 'SHOW_ACTIVE';
          return todos.filter(
            t => !t.completed
          );
      }
    };

Then we need to use this function in our `render` method:

    render() {
      const visibleTodos = getVisibleTodos(
        this.props.todos,
        this.props.visibilityFilter
      );

      ...

      </button>
      <ul>
        {visibleTodos.map( // we need to use the const
        )}
    }

And we need to pass the filter as a prop to the `TodoApp` component:

    const render = () => {
      ReactDOM.render(
        <TodoApp
          {...store.getState()}
        />,
        document.getElementById('root')
      );
    };

And it works! See the working example! But we don't actually have any feedback
on which filter is active, so let's add this. First of all, we deconstruct the
props in the `render` method:

    render() {
      const {
        todos,
        visibilityfilter
      } = this.props;
      const visibleTodos = getVisibleTodos(
        todos,
        visibilityFilter
      );

Then we pass the `visibilityFilter` to each `FilterLink` element, so that we
can style them if it's the active one:


    <p>
      Show:
      {' '}
      <FilterLink
        filter='SHOW_ALL'
        currentFilter={visibilityFilter}
      >
        All
      </FilterLink>
      {' '}
      <FilterLink
        filter='SHOW_ACTIVE'
        currentFilter={visibilityFilter}
      >
        Active
      </FilterLink>
      {' '}
      <FilterLink
        filter='SHOW_COMPLETED'
        currentFilter={visibilityFilter}
      >
        Completed
      </FilterLink>
    </p>

And finally, we update the definition of the `FilterLink` element:

    const FilterLink = ({
      filter,
      currentFilter,
      children
    }) => {
      if (filter === currentFilter) {
        return <span>{children}</span>;
      }

      return (
        <a href="#"
          onClick={e => {
            e.preventDefault();
            store.dispatch({
              type: 'SET_VISIBILITY_FILTER',
              filter
            });
          }}
        >
          {children}
        </a>
      );
    };

## Todos: Extracting presentational Components (Todo, TodoList)
We declare the `Todo` component as a function:

    const Todo = ({
      onClick,
      completed,
      text
    }) => (
      <li onClick={onClick}
          style={{
            textDecoration:
              completed ?
                'line-through' :
                'none'
          }}>
        {text}
      </li>
    );

We have removed the `key` attribute, as it's only used in array rendering.
Also, we remove any behavior from the component, so that it can be specified by
an upper level and this we we have a **presentational component**: a component
that only cares about styles. We do something similar with a `TodoList`
component:

    const TodoList = ({
      todos,
      onTodoClick
    }) => (
      <ul>
        {todos.map(todo =>
          <Todo
            key={todo.id}
            {...todo}
            onClick={() => onTodoClick(todo.id)}
          />
        )}
      </ul>
    );

Now we have two presentational components, but we need one that actually
specifies the behavior. This one is called a **container component** to pass
the data from the store. In this case, the top level `TodoApp` component acts
as a container component.

Finally, we need to replace the `ul` form the `TodoApp` `render` method with a
call to our new `TodoList` component:

    <TodoList
      todos={visibleTodos}
      onTodoClick={id =>
        store.dispatch({
          type: 'TOGGLE_TODO',
          id
        })
      } />

## Todos: Extract more presentational components (AddTodo, Footer, FilterLink)
We start extracting the `AddTodo` component:

    const AddTodo = ({
      onAddCLick
    }) => {
      let input;

      return (
        <div>
          <input ref={node =>
            input = node
          } />
          <button onClick={() => {
            onAddClick(input.value);
            input.value = '';
          }}>
            Add button
          </button>
        </div>
      );
    };

    <AddTodo
      onAddClick={text =>
        store.dispatch({
          type: 'ADD_TODO',
          id: nextTodoId++,
          text
        })
      }
    />

Next is the `Footer`, that contains all the filters. On the `FilterLink`, we
replace the `store.dispatch` call by a call to a function passed in the props:

    const FilterLink = ({
      filter,
      currentFilter,
      children,
      onClick
    }) => {
      if (filter === currentFilter) {
        return <span>{children}</span>;
      }

      return (
        <a href="#"
          onClick={onClick(filter)}
        >
          {children}
        </a>
      );
    };

    const Footer = ({
      visibilityFilter,
      onFilterClick
    }) => {
      <p>
        Show:
        {' '}
        <FilterLink
          filter='SHOW_ALL'
          currentFilter={visibilityFilter}
          onClick={onFilterClick}
        >
          All
        </FilterLink>
        {' '}
        <FilterLink
          filter='SHOW_ACTIVE'
          currentFilter={visibilityFilter}
          onClick={onFilterClick}
        >
          Active
        </FilterLink>
        {' '}
        <FilterLink
          filter='SHOW_COMPLETED'
          currentFilter={visibilityFilter}
          onClick={onFilterClick}
        >
          Completed
        </FilterLink>
      </p>
    };

    <Footer
      visibilityFilter={visibilityFilter}
      onFilterClick={ filter =>
        store.dispatch({
          type: 'SET_VISIBILITY_FILTER',
          filter
        })
      }
    />

Finally, we can refactor our `TodoApp` component into a function component, as
it will be easier to read.

## Todos: Extract container components (FilterLink)
See the `FilterLink` component. It need the `currentFilter` in order to style
itself. This means the `Footer` need to know the `visibilityFilter`, so that it
can send it to its children. The same happens with the `onFilterClick`
property. This breaks encapsulation, so we will refactor the `Footer`
component.

    const Footer = () => {
      <p>
        Show:
        {' '}
        <FilterLink
          filter='SHOW_ALL'
        >
          All
        </FilterLink>
        {' '}
        <FilterLink
          filter='SHOW_ACTIVE'
        >
          Active
        </FilterLink>
        {' '}
        <FilterLink
          filter='SHOW_COMPLETED'
        >
          Completed
        </FilterLink>
      </p>
    };

    <Footer />

Looking at the `FilterLink`, we see that it is still bound to the behavior, so
we change it to:

    const Link = ({
      active,
      children,
      onClick
    }) => {
      if (active) {
        return <span>{children}</span>;
      }

      return (
        <a
          href="#"
          onClick={e => {
            e.preventDefault();
            onClick();
          }}
        >
          {children}
        </a>
      );
    };

    class FilterLink extends Component {
      componentDidMount() {
        this.unsubscribe = store.subscribe(() =>
          this.forceUpdate()
        )
      }

      componentWillUnmount() {
        this.unsubscribe()
      }

      render() {
        const props = this.props;
        const state = store.getState();

        return (
        <Link
          active={
            props.filter ===
            state.visibilityFilter
          }
          onClick={() =>
            store.dispatch({
              type: 'SET_VISIBILITY_FILTER',
              filter: props.filter
            })
          }
        >
          {props.children}
        </Link>
        )
      }
    }

This `Link` does not know anything about the behavior, it only renders the link
depending on whether it is active or not. `FilterLink` will subscribe to the
`store` and calculate the props and actions needed by the `Link` component to
work properly.

## Todos: Extract container components (VisibleTodoList, AddTodo)
Following the work that we did with the `Footer` component, we will refactor
the `TodoList` component. It is a presentational component, but we're setting
its state from the top element of our view hierarchy. We don't want that, so we
will create another container component that will connect the presentation view
to the store:

    class VisibleTodoList extends Component {
      componentDidMount() {
        this.unsubscribe = store.subscribe(() =>
          this.forceUpdate()
        )
      }

      componentWillUnmount() {
        this.unsubscribe()
      }

      render() {
        const props = this.props;
        const state = store.getState();

        return (
          <TodoList
            todos={
              getVisibleTodos(
                state.todos,
                state.visibilityFilter
              )
            }
            onTodoClick={id =>
              store.dispatch({
                type: 'TOGGLE_TODO',
                id
              })
            }
          />
        )
      }
    }

    <VisibleTodoList />

In the last section we made `AddTodo` into a presentational component. Now we
will backtrack on this, because there isn't a lot of presentation or behavior
here, and it's easier to keep everything together in this case:

    const AddTodo = () => {
      let input;

      return (
        <div>
          <input ref={node => {
            input = node;
          }} />
          <button onClick={() => {
              store.dispatch({
                type: 'ADD_TODO',
                id: nextTodoId++,
                text: input.value
              })
            input.value = '';
          }}>
            Add Todo
          </button>
        </div>
      );
    };

    <AddTodo />

Thanks to that, we can remove all props from `TodoApp` because none of its
elements need the store sent from their parent. Also, this enables us to just
render the `TodoApp` component once, as it will not need to update itself (its
children do, and they do it automatically now):

    const TodoApp = () => (
      <div>
        <AddTodo />
        <VisibleTodoList />
        <Footer />
      </div>
    );

    ReactDOM.render(
      <TodoApp />,
      document.getElementById('root')
    );

## Todos: Passing the store down explicitly via props
We are currently relying in a global variable `store`, but this does not scale
in real world applications with multiple files. Also, it makes components hard
to test. A way to solve this could be passing down the store ecplicitly via
props to every container component and read it from the props, but this is
inconvenient as we need to modify every compoentnt in our code. This would look
like this:

    const TodoApp = ({ store }) => (
      <div>
        <AddTodo store={store} />
        <VisibleTodoList store={store} />
        <Footer store={store} />
      </div>
    );

    ReactDOM.render(
      <TodoApp store={createStore(todoApp)} />,
      document.getElementById('root')
    );

## Todos: Passing the store down implicitly via context
We start writing a `Provider` component that will just render its children:

    class Provider extends Component {
      getChildContext() {
        return {
          store: this.props.store
        }
      }

      render() {
        return this.props.children;
      }
    }
    Provider.childContextTypes = {
      store: React.PropTypes.object
    };

    ReactDOM.render(
      <Provider store={createStore(todoApp)}>
        <TodoApp />
      </Provider>,
      document.getElementById('root')
    );

We need to specify the `Provider.childContextTypes` part, otherwise it won't be
passed to the component children.

Now we need to change how our get the store. Instead of getting it from the
props, they will be getting it from the React context:

    // For class components
    const { store } = this.context;

    // For function components
    const AddTodo = (props, { store }) => {
      ...
    }

Also, for each component that receives the store we need to specify it.
Otherwise, the component will not receive it.

    VisibleTodoList.contextTypes = {
      store: React.PropTypes.object
    };

## Todos: Use the Provider from react-redux
Import the JS library in your HTML and use this:

    const { Provider } = ReactRedux;

## Todos: Generate containers with `connect` from ReactRedux (VisibleTodoList)
Our current `VisibleTodoList` component is part of a pattern of components that
only set the behavior of a single presentational component. It sets some props
and callbacks and delegates them to the presentational component `TodoList`.
This is part of a pattern and can be extracted, so let's do it!

We need to things now: an object that defines the props for the presentational
component, and another one that defines its callbacks:

    // It receives automaticalle the state from the `store`
    const mapStateToProps = (state) => {
      return {
        todos: getVisibleTodos(
          state.todos,
          state.visibilityFilter
        )
      };
    };

    // It receives automatically the `dispatch()` from the store
    const mapDispatchToProps = (dispatch) => {
      return {
        onTodoClick: (id) => {
          dispatch({
            type: 'TOGGLE_TODO',
            id
          })
        }
      };
    };

Now we can generate automatically the same `VisibleTodoList` just using the
`connect()` method from React Redux:

    const { connect } = ReactRedux;
    const VisibleTodoList = connect(
      mapStateToProps,
      mapDispatchToProps
    )(TodoList);

It will automatically subscribe to our store.

## Todos: Generate containers with `connect` from ReactRedux (AddTodo)
Now we can do the same from the previous lesson, but applying it to the
`AddTodo` component.

    let AddTodo = ({ dispatch }) => {
      ...
      // we need to remove any reference to `store` and only leave the
      // `dispatch` part
    }
    // we remove the `contextTypes` part, as we don't need it anymore
    // `connect` will deal with it automatically
    // we don't need to subscribe to the state, so we pass `null`
    // we don't do anything with the dispatch, so we pass `null` too
    // `connect(null, null) === connect()`
    AddTodo = connect()(AddTodo);

## Todos: Generate containers with `connect` from ReactRedux (FilterLink)
Now we'll refactor the `FilterLink` component. We need to write both functions,
which need access to `props`:

    const mapStateToLinkProps = (state, ownProps) => {
      return {
        active: ownProps.filter === state.visibilityFilter
      };
    };
    const mapDispatchToLinkProps = (dispatch, ownProps) => {
      return {
        onClick: ()=> {
          store.dispatch({
            type: 'SET_VISIBILITY_FILTER',
            filter: ownProps.filter
          })
        }
      };
    };
    const FilterLink = connect(
      mapStateToLinkProps,
      mapDispatchToLinkProps,
    )(Link);
