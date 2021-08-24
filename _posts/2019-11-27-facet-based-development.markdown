---
layout: post
title: 'Facet based development with MobX'
date: 2019-11-27 16:21:59 +0100
categories: react
---

## Introduction

When we program a React application we use existing components and libraries such as drop down buttons, state managers and HTTP clients. These components are fairly basic in the sense that they do not determine how the application behaves. For example, there is no reusable component to ensure that selected items also get highlighted, even though many applications need this kind of behaviour.
This article introduces my own approach for programming reusable behaviours. But first I will describe why two popular ways to make behaviours reusable have not been very successful. I then briefly explain two approaches that inspired my own approach: Template Methods and what I will call Smart Containers. From there I will outline the requirements for my approach, and explain how they are met. The source code for my library (called SkandhaJS) can be found [here](https://github.com/mnieber/skandha)
and [here](https://github.com/mnieber/skandha-mobx).

## Approaches that fail to enable reusable behaviour

### Reusing behaviour using frameworks

Frameworks offer not just components but also behaviours such as selection, highlighting, drag-and-drop, etc. They are considered to be useful but also limiting. The reason is that as long as you follow the patterns offered by the framework then things work fine, but customizing the behaviour usually amounts to breaking into a black box. Trying to go against the expectation of an existing module - which is what "breaking into something" is all about - tends to have unexpected consequences and leads to code that is hard to understand and maintain.

### Reusing behaviour using OOP

As Rich Hickey observed, Object Oriented Programmed (which, by the way, I do like) has failed to deliver on its promise of making
code reusable. Suppose you have a DoFoo class. You want to add a DoBar class and you see the potential to reuse DoFoo. Usually DoFoo will have code that is specifically there to support Foo and gets in the way of DoBar. You can try to make a common base-class that has only the "useful common bits" that serve both the purpose of DoFoo and DoBar. This will work if neither DoFoo nor DoBar needs to break into these common bits, otherwise you are back to square one. Moreover, it often happens that the base class is just an ad-hoc collection of "useful bits" that has no easily recognizable purpose. Such classes make a code-base hard to understand and maintain.

## Approaches that inspired SkandhaJS

### Template Method

Template Methods were successfully used in the Erlang language to create reusable behaviours. In this design pattern there is a
template that dictates the order of the main steps in a procedure, and callback functions that provide the implementation of these
steps. For example, in the pseudo code below the password is verified first, and forwarding the user to the next page comes after:

```
def goToProtectedPage():
    response = callbacks.verifyPassword();
    if (response.success) {
      callbacks.goToNextPage();
    }
```

This looks superfically similar to OOP where member functions can be overridden, but there is an important difference that has to do with granularity. Callback functions are fine grained: they can be mixed and matched. With OOP, every time you override a class you create a new type, without the possibility to match bits and pieces of these classes at will. Moreover, adding a type increases the complexity of the application. With a Template Method you are free to use any combination of callback functions, without the need to add types.

### Smart Containers

In this approach that I read about many years ago (in a white-paper that I unfortunately cannot find anymore), every module exists in a container. Each module can detect and use the other modules in the container. For example, a ShowPage module can detect a PasswordVerifier module in its container and use it before showing a page to the user. This allows you as a programmer to reuse behaviours by mixing and matching the right modules in your container. I don't claim that this is necessarily a good way to create applications, but I found the idea to be inspirational: modules can detect other modules in their direct neighborhood and adjust their behaviour accordingly.

## Outline of my approach

In this section I will outline my approach for creating reusable behaviours. Keep in mind that an application needs to be designed such that it can support reusable behaviours. In other words, you cannot take any existing application and put reusable behaviours on top. To allow for
reusable behaviours, I'm using the following application requirements:

- the application stores an Application State in its Application Layer which is separate from its Presentation Layer. The Application State contains information that is needed in several places in the application (e.g. the id of the currently selected document). On the other hand, if information is local to a visual component or its direct children then I may store it in the Presentation layer.

- the Presentation Layer is relatively dumb. It visualizes the Application State and responds to user actions by sending commands to the Application Layer. The effects of this action (which determines the behaviour of the application) is decided by the Application Layer.

- the Application State is stored in smart Containers, such as a Documents container and a Users container. I will explain below what "smart" means, but for now imagine that a Container with selection and highlight information can automatically synchronize the two.

- smart Containers contain reusable parts called Facets. You can think of Facets as the different concerns that the Container takes care of. For example, a Users Container may have a Selection Facet, a Highlight Facet and a Filtering Facet. In general we try to use abstraction in Facets to make them reusable (a User selection and a Document selection can use the same abstraction)

- Facets support operations that follow the Template Method pattern. For example, as a programmer you can customize the callbacks of the "selectItem" operation in the Selection Facet. We will see how combining different behaviours (e.g. after selecting an item in the Selection Facet, also highlight it in the Highlight Facet) is achieved via this callback mechanism.

## Using containers in the Application Layer

Let's now make these ideas concrete by looking at some code for the Users container. By the way, a complete example (and accompanying
sample application repository) can be found in the README of [SkandhaJS](https://github.com/mnieber/skandha).

```
class UsersCtr {
  @facet selection: Selection = new Selection();
  @facet highlight: Highlight = new Highlight();
  @facet filtering: Filtering = new Filtering();
  @facet inputs: Inputs = new Inputs();

  constructor() {
    registerCtr({ //                          [1]
      ctr: this,
      initCtr: () => {
        this._setCallbacks();
        this._installPolicies();
      }
    );
  }

  _setCallbacks() {
    const ctr = this;

    setCallbacks(this.selection, { //         [2,3]
      selectItem: {
        select(this: Selection_selectItem) {
          handleSelectItem(this.selectionParams);
          highlightFollowsSelection(ctr.selection, this.selectionParams);
        }
      ]
    })

    setCallbacks(this.highlight, {
      highlightItem: {
        highlightItem(this: Highlight_highlightItem) {
          handleHighlightItem(ctr.highlight, this.id)
        }
      }
    })

    // other actions omitted
  }

  _installPolicies() { //                     [4]
    mapData(
      [Selection, 'selectableIds'],
      [Inputs, 'userById'],
      getIds
    )(this);

    convertSelectedIdsToItems( //             [5]
      [Inputs, 'itemById']
    )(this);

    // other policies omitted
  }
};
```

Notes:

1. the `registerCtr` function ties the facet instances to the container. To get the container instance from
   the facet instance you can use `getCtr(facet)`.

2. the `setCallbacks` function installs callback functions for the operations of the `Selection` and `Highlight` facets.
   The `selectItem` callback function calls `handleSelectItem` to calculate the next selection. It also calls `highlightFollowsSelection` to ensure that selected items are also highlighted.

3. The `highlightFollowsSelection` function updates the highlighted item. It is implemented as follows:

   ```
       export function highlightFollowsSelection(
         this: Selection,
         { itemId, isShift, isCtrl }
       ) {
         const ctr = getCtr(this);
         Highlight.get(ctr).id = itemId;
       }
   ```

4. the `_installPolicies()` function sets up additional rules inside the container. Often these rules declare data mappings that route
   information from one facet to the other. In the example, we see that `Inputs.userById` is mapped onto `Selection.selectableIds`, using the `getIds` function to convert users into ids.

5. The second policy (`convertSelectedIdsToItems`) maps `this.selection.ids` onto `this.selection.items`
   using `this.inputs.itemById` as a lookup table (where `get` is a static member function of the facet class). It is implemented as:

```
    mapDatas(
      [Selection, "items"],
      [
        [Inputs, itemById],
        [Selection, "ids"],
      ],
      (itemById: any, ids: any) => lookUp(ids, itemById)
    )(this);
```

## A closer look at the Selection facet

Now that you have an impression of how a container is setup, let's look at one of the facets. The `Selection` facet is
implemented as follows:

```
class Selection_selectItem {
  selectionParams: SelectionParamsT;
  select() {}
}

export class Selection {
  @input selectableIds?: Array<any>;
  @observable @data ids: Array<any> = []; //                            [1]
  @observable @data anchorId: any;
  @output items?: Array<any>;

  @operation @host selectItem(selectionParams: SelectionParamsT) { //   [2-5]
    return (cbs: Selection_selectItem) => {
      const { itemId, isShift, isCtrl } = selectionParams;
      if (!this.selectableIds.includes(itemId)) {
        throw Error(`Not a selectable id: ${itemId}`);
      }
      cbs.select();
    }
  }

  static get = (ctr: any): Selection => ctr.selection;
}
```

Notes:

1. The `@observable` decorator comes from MobX. It allows clients to react automatically to changes in these members.

2. The `selectItem` function is an operation that verifies the `itemId` argument and then calls the "select" callback.
   The callbacks are installed via the `@host` decorator that comes from a library called [Aspiration](https://github.com/mnieber/aspiration) that deals with aspect oriented programming.

## Discussion

In the example we saw that highlighting is synchronized with selection even though the `Selection` facet has no knowledge of the `Highlight`
facet. This is a key property of the proposed approach: facets can interact without knowing about each other. These interactions are
orchestrated in the `_setCallbacks` and `_installPolicies` functions of the container. The `_setCallbacks` function installs callback
functions that have access to the entire container, which allows them to access and update multiple facets. The `_installPolicies` function
installs mapping functions (and possibly other interactions) that map data from one facet to the other. So although this is not truly a Smart
Container that magically detects the components that live inside it, the container still plays the role of a mediator and orchestrator.
This article only offers a small introduction, skipping over many of the details. This approach based on containers and facets is both
ambitious and experimental, but so far I've found that it produces good results, and I welcome you to try it out.
