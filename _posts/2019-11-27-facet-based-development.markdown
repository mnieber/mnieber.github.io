---
layout: post
title: "Facet based development with MobX"
date: 2019-11-27 16:21:59 +0100
categories: react
---

## Introduction

When we program a React application we use existing components and libraries such as drop down buttons, state managers and HTTP clients. These components are fairly basic in the sense that they do not determine how the application behaves. For example, there is no reusable component to ensure that selected items also get highlighted, even though many applications need this kind of behaviour.
This article describes why two popular ways to make behaviours reusable have not been very successful. It then briefly explains two approaches that
inspired my own approach: Template Methods and what I will call Smart Containers. From there I will outline the requirements for my approach, and
explain how they are met. The source code can be found [here](https://github.com/mnieber/facet) and
[here](https://github.com/mnieber/facet-mobx).

## Reusing behaviour using frameworks and OOP

Frameworks offer not just components but also behaviours such as selection, highlighting, drag-and-drop, etc. They are considered to be a good choice for less professional programmers but too limiting for more experienced people. The reason is that as long as you follow the patterns offered by the framework then things work fine, but customizing the behaviour usually amounts to breaking into a black box. Trying to go against the expectation of an existing module - which is what "breaking into something" is all about - tends to have unexpected consequences and leads to code that is hard to understand and maintain.

This is also why - as Rich Hickey observed - object oriented programmed has failed to deliver on its promise of making code reusable. Suppose you have a DoFoo class. Usually that class has code that is specifically there to support Foo and gets in the way if you want it to perform Bar. You can try to make a common base-class that has only the "useful common bits". This will work if neither DoFoo nor DoBar needs to break into these common bits, otherwise you are back to square one. Moreover, it often happens that the base class is just an ad-hoc collection of "useful bits" that has no easily recognizable purpose. Such classes make a code-base hard to understand and maintain.

## Template Method

Template Methods were successfully used in the Erlang language to create reusable behaviours. In this design pattern there is a template that
dictates the order of the main steps in a procedure, and callback functions that provide the implementation of these steps. For example, in the
pseudo code below the password is verified first, and forwarding the user to the next page comes after:

```
def goToProtectedPage():
    response = callbacks.verifyPassword();
    if (response.success) {
      callbacks.goToNextPage();
    }
```

This looks superfically similar to OOP where member functions can be overridden, but there is an important difference. With OOP, every time you override a class you create a new type. These new classes can not be mixed and matched at will. Moreover, adding a type increases the complexity of the application. With a Template Method you are free to use any combination of callback functions, without the need to add types.

## Smart Containers

In this approach that I read about many years ago (in a white-paper that I unfortunately cannot find anymore), every module exists in a container. Each module can detect and use the other modules in the container. For example, a ShowPage module can detect a PasswordVerifier module in its container and use it before showing a page to the user. This allows you as a programmer to reuse behaviours by mixing and matching the right modules in your container. I don't claim that this is necessarily a good way to create applications, but I found the idea to be inspirational.

## Outline of my approach

In this section I will outline my approach for creating reusable behaviours. Keep in mind that an application needs to be designed such that it can support reusable behaviours. In other words, you cannot take any existing application and put reusable behaviours on top. Therefore, the points below are also requirements for the application itself:

- the application stores an Application State in its Application Layer which is separate from its Presentation Layer. The Application State contains information that is needed in several places in the application (e.g. the id of the currently selected document). On the other hand, if information is local to a visual component or its direct children then I may store it in the Presentation layer.

- the Presentation Layer is relatively dumb. It visualizes the Application State and responds to user actions by sending commands to the Application Layer. For example, it may send a selectItem command when it detects a mouse click. The rule that causes selected items to also be highlighted is enforced in the Application Layer. This is true in general: behaviours are enforced in the Application Layer.

- the Application State is stored in smart Containers. For example, it may contain a Documents container and a Users container. I will explain below what "smart" means, but for now imagine that a Container with selection and highlight information can automatically synchronize the two.

- smart Containers contain reusable parts called Facets. You can think of Facets as the different concerns that the Container takes care of. For example, a Users Container may have a Selection Facet, a Highlight Facet and a Filtering Facet. The Selection Facet receives a list of selectable ids, and stores a list of the currently selected ones. Since the Selection Facet is only concerned with the ids, we can also use it in the Documents Container. We only need to provide a mapping between Documents and ids. In general we try to use generic concepts in Facets to make them reusable.

- Facets support operations that follow the Template Method pattern. For example, as a programmer you can customize the callbacks of the "selectItem" operation in the Selection Facet. We will see how combining different behaviours (e.g. after selecting an item in the Selection Facet, also highlight it in the Highlight Facet) is achieved via this callback mechanism.

## Using containers in the Application Layer

Let's now make these ideas concrete by looking at some pseudo code for the Users container.

```
class UsersCtr {
  @facet selection: Selection = new Selection();
  @facet highlight: Highlight = new Highlight();
  @facet filtering: Filtering = new Filtering();
  @facet inputs: Inputs = new Inputs();

  constructor() {
    registerFacets(this); //                  [1]
    this._installActions();
    this._installPolicies();
  }

  _installActions() {
    installActions(this.selection, { //       [2]
      selectItem: {
        select: [handleSelectItem],
        select_post: [highlightFollowsSelection]
      ]
    })

    installActions(this.highlight, {
      highlightItem: {
        highlightItem: [handleHighlightItem],
      }
    })

    // other actions omitted
  }

  _installPolicies() { //                     [3]
    mapData(
      [Inputs, userById],
      [Selection, selectableIds],
      getIds
    )(this);

    convertSelectedIdsToItems( //             [4]
      [Inputs, itemById]
    )(this);

    // other policies omitted
  }
};
```

Notes:

1. the `registerFacets` function ties the facet instances to the container. This makes it possible to get the container instance from
   the facet instance using `getCtr(facet)`

2. the `installActions` function installs callback functions for the "selectItem" operation of the `Selection` facet and for the
   "highlightItem" operation of the `Highlight` facet. The "selectItem" operation has two callbacks: `handleSelectItem` (which handles the
   "select" trigger) and `highlightFollowsSelection` (which is called after the "select" trigger was handled).

3. the `_installPolicies()` function sets up additional rules inside the container. Often these rules declare data mappings that route
   information from one facet to the other. The `mapData` function creates this mapping using MobX such that the output is updated
   automatically when the inputs change. In the example, we see that the 'userById' member of the `Inputs` facet is mapped onto the
   'selectableIds' member of the `Selection` facet, using the `getIds` function to convert users into ids.
4. The second policy creates a data mapping from `Selection.get(this).ids` to `Selection.get(this).items` using
   `Inputs.get(this).itemById` as a lookup table (where `get` is a static member function of the facet class). It is implemented as:

```
    mapDatas(
      [
        [Inputs, itemById],
        [Selection, "ids"],
      ],
      [Selection, "items"],
      (itemById: any, ids: any) => lookUp(ids, itemById)
    )(this);
```

## A closer look at the Selection facet

Now that you have an impression of how a container is setup, let's look at one of the facets. The `Selection` facet is
implemented as follows:

```
export class Selection {
  @input selectableIds?: Array<any>;
  @observable @data ids: Array<any> = []; //                [1]
  @observable @data anchorId: any;
  @output items?: Array<any>;

  @operation selectItem({ itemId, isShift, isCtrl }) { //   [2-5]
    if (!this.selectableIds.includes(itemId)) {
      throw Error(`Not a selectable id: ${itemId}`);
    }
    exec("select");
  }

  static get = (ctr: any): Selection => ctr.selection;
}
```

Notes:

1. The `@observable` decorator comes from MobX. It allows clients to react automatically to changes in these members.

2. The `selectItem` function is an operation that verifies the `itemId` argument and then calls the "select" action. The operation
   will search for an installed callback that has the "select" label. In our example, we installed an aray containing two callback
   functions: `handleSelectItem` and `highlightFollowsSelection`. Since `handleSelectItem` has the required label, it will be
   executed.

3. The operation will also execute any not-yet-executed callbacks that preceed `handleSelectItem` in the array of installed callback
   functions. This allows us to execute a function right before calling `handleSelectItem`. When the operation exits, any callbacks
   that are not yet executed will be called. In our case, this means that `highlightFollowsSelection` gets called at the end of the
   `selectItem` operation.

4. The `handleSelectItem` callback function updates the selection. All callback functions receive a reference to the facet instance
   (via the `this` pointer) as well as all the arguments that were passed into the operation function:

```
    export function handleSelectItem(
      this: Selection,
      { itemId, isShift, isCtrl }
    ) {
      // Implementation that sets this.ids and
      // this.anchorId omitted
    }
```

5. The `highlightFollowsSelection` function updates the highlighted item. As stated, it receives the same arguments as `handleSelectItem`.
   This time we need to look up and modify the `Highlight` facet that lives in the same container:

   ```
       export function highlightFollowsSelection(
         this: Selection,
         { itemId, isShift, isCtrl }
       ) {
         const ctr = getCtr(this);
         Highlight.get(ctr).id = itemId;
       }
   ```

## Discussion

In the example we saw that highlighting is synchronized with selection even though the `Selection` facet has no knowledge of the `Highlight`
facet. This is a key property of the proposed approach: facets can interact without knowing about each other. These interactions are
orchestrated in the `_installActions` and `_installPolicies` functions of the container. The `_installActions` function installs callback
functions that have access to the entire container, which allows them to access and update multiple facets. The `_installPolicies` function
installs mapping functions (and possibly other interactions) that map data from one facet to the other. So although this is not truly a Smart
Container that magically detects the components that live inside it, the container still plays the role of a mediator and orchestrator.
This article only offers a small introduction, skipping over many of the details. This approach based on containers and facets is both
ambitious and experimental, but so far I've found that it produces good results, and I welcome you to try it out.
