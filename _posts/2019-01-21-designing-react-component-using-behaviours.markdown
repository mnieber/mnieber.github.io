---
layout: post
title:  "Designing a React component using behaviours"
date:   2019-01-21 11:56:59 +0100
categories: react
---
Introduction
------------

Good software engineers know that writing software is to a large extent about separation of concerns, also known as modularization. It's easy to understand a well designed module because you can ignore the concerns that are handled by other modules. As a fan of React, I noticed that many of my components were hard to understand because they were mixing two concerns: a) the construction of an html element, and b) the behaviour of that html element when events are fired.

To make things worse, my event handlers themselves were hard to understand. A single click handler could be updating `state.highlightedItem`, `state.isEditing`, and `state.displayedItems`. When this local state is also mutated in other event handlers, the interactions between the handlers get pretty hairy. In this article, I will describe an approach for capturing the behaviour of a React component in functions, leading to a more modular design.

Global outline
--------------

Following the tradition of using a TodoApp example, I will implement a `TodoList` component with the following requirements:

1. the user can highlight one of the existing todo's.
2. the user can enable editing of the highlighted todo. The changes can be saved or discarded.
3. a new todo can be inserted right below the currently highlighted one. The new todo should become highlighted and editable.
4. if the user cancels the new todo, then the previous todo list and highlight should be restored and editing should be disabled.
5. if the user is editing a new todo and highlights a different todo in the list, then the new todo should be discarded.

These requirements will be implemented in a `TodoList` component with the following signature:

    {% highlight javascript %}
    export type TodoListPropsT = {
      todos: Array<TodoT>,
      highlightedTodoId: UUID,
      actSetHighlightedTodoId: Function,
      actInsertTodos: Function,
      actUpdateTodos: Function,
    };

    function TodoList(props: TodoListPropsT) {
        // ... implementation goes here
    }
    {% endhighlight %}

As you can see, the properties are described with a `Flow` type. As an aside, I am using the `act` prefix for functions that are actions on the Redux store. Moreover, punning is used in this article to shorten `{foo: foo, bar: bar}` to `{foo, bar}`.

Behaviours
----------

I define a behaviour to be a collection of variables and functions that capture one behavioural concern that the React component must deal with. For example, our third requirement refers to the concerns of creating a new todo and inserting a todo into the list. Therefore, our design will include a "insert todo" behaviour and a "new todo" behaviour that should be as separate as possible. Saving or discarding changes to a todo is captured in a "save todo" behaviour. Note that I did not try to capture *every* behaviour in a separate function. In two cases ("highlight todo" and "enable editing") I decided it was simpler to manage this state directly in `TodoList`.

Designing the "Insert todo" behaviour
-------------------------------------

Let's focus first on inserting a todo. The following `Flow` type describes the "insert todo" behaviour:

    {% highlight javascript %}
    type InsertTodoBvrT - {|
      prepare: Function,
      finalize: Function,
      preview: Array<TodoT>
    |};
    {% endhighlight %}

Insertion is performed in two steps: first `prepare(precedingTodoId, todo)` is called to tentatively insert the todo into the list, at the location given by `precedingTodoId`. Then, `finalize` is called to either cancel or confirm the insertion. The `preview` variable contains the list of todo's, which may contain a todo that was tentatively inserted with `prepare`.

An instance of `InsertTodoBvrT` is created by calling `useInsertTodo`:

    {% highlight javascript %}
    export function useInsertTodo(
      todos: Array<TodoT>,
      actInsertTodos: Function,
    ): InsertTodoBvrT {
        // Implementation goes here...
        return {prepare, finalize, preview}
    }
    {% endhighlight %}

You can look up the implementation of `useInsertTodo` in the [accompanying repository](https://github.com/mnieber/behaviours_example), but for the moment, let's see how it can be used:

    {% highlight javascript %}
    function tryItOut(
      todos: Array<TodoT>,
      actInsertTodos: Function,
    ) {
        const insertTodoBvr = useInsertTodo(todos, actInsertTodos);
        console.log("Current todos:", insertTodoBvr.preview);

        // Create a stand-alone todo object
        const newTodo = _createTodo();

        // Tentatively insert the new todo below the todo at index 1
        insertTodoBvr.prepare(todos[1].id, newTodo);
        console.log("Current todos:", insertTodoBvr.preview);

        // Cancel the insertion (the new todo will be removed from the preview)
        insertTodoBvr.finalize(/* cancel */ true);
        console.log("Current todos:", insertTodoBvr.preview);
    }
    {% endhighlight %}


Designing the "New todo" behaviour
----------------------------------

The "New todo" behaviour should orchestrate everything that happens when the user clicks the "new todo" button. Note that this includes inserting the new todo into the list. Of course, insertion is a separate concern. Therefore, the "New todo" behaviour will use an instance of the "Insert todo" behaviour so that it can be agnostic of the details around insertion. The `NewTodoBvrT` type looks like this:

    {% highlight javascript %}
    type NewTodoBvrT = {|
      addNewTodo: Function,
      finalize: Function,
      newTodo: ?TodoT,
      setHighlightedTodoId: Function,
    |};
    {% endhighlight %}

The `addNewTodo` function tentatively inserts a new todo below the currently highlighted one. The `finalize` function either cancels or confirms the insertion of the new todo. The `newTodo` variable holds the new todo. Finally, the `setHighlightedTodoId` function is a wrapper around the normal `setHighlightedTodoId` that automatically cancels the new todo if the highlight moves elsewhere.

The signature of `useNewTodo` is:

    {% highlight javascript %}
    export function useNewTodo(
      highlightedTodoId: UUID,
      setHighlightedTodoId: Function,
      insertTodoBvr: InsertTodoBvrT,
      setIsEditing: Function,
    ): NewTodoBvrT {
        // Implementation goes here...
        return {addNewTodo, finalize, newTodo, setHighlightedTodoId};
    }
    {% endhighlight %}

It can be used as follows:

    {% highlight javascript %}
    function tryItOut(
      todos: Array<TodoT>,
      actInsertTodos: Function,
    ) {
        const insertTodoBvr = useInsertTodo(todos, actInsertTodos);

        let highlightedTodoId = todos[1].id;
        const setHighlightedTodoId = id => {highlightedTodoId = id};
        const setIsEditing = x => {/* dummy */};

        const newTodoBvr = useNewTodo(
            highlightedTodoId,
            setHighlightedTodoId,
            insertTodoBvr,
            setIsEditing,
        );

        newTodoBvr.addNewTodo();
        console.log(newTodoBvr.newTodo, insertTodoBvr.preview, todos);

        // Cancel the new todo by moving the highlight
        newTodoBvr.setHighlightedTodoId(todos[2].id);
        console.log(newTodoBvr.newTodo, insertTodoBvr.preview, todos);

        // Again, add a new todo, and this time confirm it
        newTodoBvr.addNewTodo();
        newTodoBvr.finalize(/* cancel */ false);
        console.log(newTodoBvr.newTodo, insertTodoBvr.preview, todos);
    }
    {% endhighlight %}

You may ask yourself: we passed in a `highlightedTodoId` to `useNewTodo`, so what happens when we later call `setHighlightedTodoId`? Indeed, this creates an inconsistency because `newTodoBvr` now contains a stale value. This can be corrected by recreating `newTodoBvr` with the latest value of `highlightedTodoId`. Fortunately, this is exactly what happpens when we use the behaviours in the `TodoList` component, because `TodoList` is re-rendered when the highlight changes.

Designing the "Save todo" behaviour
-----------------------------------

Finally, here is the code for for `SaveTodoBvrT`:

    {% highlight javascript %}
    type SaveTodoBvrT = {|
      saveTodo: Function,
      discardChanges: Function,
    |};
    {% endhighlight %}

The `saveTodo` function accepts a todo id and a patch. For example, to update the second todo you can call `saveTodo(todos[1].id, {name: 'Buy almond milk'})`. Some extra work must be done though when saving a new todo that was tentatively inserted into the list: `saveTodo` must call `newTodoBvr.finalize(/* cancel */ false)` to confirm the insertion. Since `saveTodo` doesn't know whether it's saving a new todo or an existing one, `newTodoBvr.finalize` should be a no-op if no new todo was created. Similarly, `discardChanges` should call `newTodoBvr.finalize(/* cancel */ true)` to undo the tentative insertion of a new todo.

As can be expected, the `useSaveTodo` function takes the `newTodoBvr` as an argument:

    {% highlight javascript %}
    export function useSaveTodo(
      todos: Array<TodoT>,
      newTodoBvr: NewTodoBvrT,
      setIsEditing: Function,
      actUpdateTodos: Function,
    ): SaveTodoBvrT {
        // Implementation goes here...
        return {saveTodo, discardChanges};
    }
    {% endhighlight %}

It can be used as follows:

    {% highlight javascript %}
    function tryItOut(
      todos: Array<TodoT>,
      actInsertTodos: Function,
    ) {
        const insertTodoBvr = useInsertTodo(todos, actInsertTodos);

        let highlightedTodoId = todos[1].id;
        const setHighlightedTodoId = id => {highlightedTodoId = id};
        const setIsEditing = x => {/* dummy */};

        const newTodoBvr = useNewTodo(
            highlightedTodoId,
            setHighlightedTodoId,
            insertTodoBvr,
            setIsEditing,
        );

        const saveTodoBvr = useSaveTodo(
            todos,
            newTodoBvr,
            setIsEditing,
            actUpdateTodos
        );

        // Update an existing todo
        saveTodoBvr.saveTodo(todos[1].id, {name: 'Foo'});

        // Save a new todo
        newTodoBvr.addNewTodo();
        saveTodoBvr.saveTodo(newTodoBvr.newTodo.id, {name: 'Bar'});
        console.log("Todo's:", todos);

        // Cancel a new todo
        newTodoBvr.addNewTodo();
        saveTodoBvr.discardChanges();
        console.log("Todo's:", todos);
    }
    {% endhighlight %}

Implementing the behaviours
---------------------------

The full implementation can be found in the [accompanying repository](https://github.com/mnieber/behaviours_example), but to illustrate a typical case, I will show the implementation of `useNewTodo`. Note that a React hook is used to store the new todo in the local state of `useNewTodo`, which allows us to properly isolate all code related to insertion.

    {% highlight javascript %}
    export function useNewTodo(
      highlightedTodoId: UUID,
      setHighlightedTodoId: Function,
      insertTodoBvr: InsertTodoBvrT,
      setIsEditing: Function,
    ): NewTodoBvrT {
      const [newTodo, setNewTodo] = React.useState(null);

      // Store a new todo in the local state
      function addNewTodo() {
        const newTodo = _createNewTodo();
        setNewTodo(newTodo);
        setHighlightedTodoId(newTodo.id);
        insertTodoBvr.prepare(highlightedTodoId, newTodo);
        setIsEditing(true);
      }

      // Remove new todo from the local state
      function finalize(isCancel: boolean) {
        setIsEditing(false);
        const prevHighlightedTodoId = insertTodoBvr.finalize(isCancel);
        if (newTodo && isCancel) {
          setHighlightedTodoId(prevHighlightedTodoId);
        }
        setNewTodo(null);
      }

      // Wrapper that cancels the new todo if the highlight moves elsewhere
      function setHighlightedTodoIdExt(id: UUID) {
        if (newTodo && id != newTodo.id) {
          finalize(/* isCancel */ true);
        }
        setHighlightedTodoId(id);
      }

      return {
        newTodo,
        addNewTodo,
        finalize,
        setHighlightedTodoId: setHighlightedTodoIdExt
      };
    }
    {% endhighlight %}

Testing the behaviours
----------------------

Since the behaviours are nicely separated from the React component, testing them is quite easy. The only complicating factor is that we need to create the behaviours inside a render function, since they rely on React hooks. Therefore, I'm using a `TestComponent` that creates the behaviours in a `sandbox` object which can be accessed from the test:

    {% highlight javascript %}
    const sandbox = {};

    function TestComponent({
      todos,
      highlightedTodoId,
      setHighlightedTodoId,
      setIsEditing,
      actInsertTodos,
      actUpdateTodos,
    }) {
      sandbox.insertTodoBvr = useInsertTodo(
        todos,
        actInsertTodos
      )

      sandbox.newTodoBvr = useNewTodo(
        highlightedTodoId,
        setHighlightedTodoId,
        sandbox.insertTodoBvr,
        setIsEditing,
      );

      sandbox.saveTodoBvr = useSaveTodo(
        sandbox.insertTodoBvr.preview,
        sandbox.newTodoBvr,
        setIsEditing,
        actUpdateTodos
      )

      return [];
    }

    import * as data from 'todo/tests/data'
    import TestRenderer from "react-test-renderer";
    // Other imports go here...

    test('test useInsertTodo', function (t) {
      const testComponent = TestRenderer.create(<TestComponent
        todos={data.todos}
        actInsertTodos={actInsertTodos}
        setHighlightedTodoId={()=>{}}
        setIsEditingEnabled={()=>{}}
        highlightedTodoId={""}
        actUpdateTodos={()=>{}}
      />);

      t.deepEqual(
        sandbox.insertTodoBvr.preview, data.todos,
        "Initially, the preview is just the list of todo's"
      );

      // More tests follow...
    };
    {% endhighlight %}

Using the behaviours in TodoList
--------------------------------

The creation of the behaviours in `TodoList` looks similar to the `TestComponent` code above. The behaviours can be directly linked to click events. For example, the click event of `btnNewTodo` can be directly connected to `newTodoBvr.addNewTodo()`.

It's important to note that the behaviours are recreated each time that `TodoList` is rendered. This ensures that each behaviour has up-to-date parameter values. For example, `newTodoBvr` always receives the latest `highlightedTodoId` so that new todo's are properly inserted below the highlighted one.

Discussion
----------

Some salient properties of the proposed approach are:

1. The behaviours are tightly coupled using very concrete code. In a more loosely coupled design, `useInsertTodo` might fire an event such as `onNewTodoCreated(todo)` which can be used to insert the new todo in the list. However, in our case `useNewTodo` directly calls `insertTodoBvr.prepare(highlightedTodoId, newTodo)`. I believe this direct style enhances the readability and makes it easier to reason about the code (in the loosely coupled design it's hard to predict the exact effect of firing `onNewTodoCreated`).
2. The achieved modularization relies on React hooks to maintain a local state in a function.
3. Readability is improved a lot by using `Flow` to document the types.
4. It's quite easy to test.

In my experience, separating the behaviours of a component into functions forces the developer to think more deeply about the interactions between them. Before using behaviours I had local event handlers with access to *all* the state, and the code for creating items and saving them tended to become mixed up. In the proposed approach, each behaviour has its own local state, and interactions only happen via interfaces, as it should be.
