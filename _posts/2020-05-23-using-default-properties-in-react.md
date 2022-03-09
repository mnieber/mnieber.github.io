---
layout: post
title: 'Using default properties in React'
date: 2020-05-23 16:21:59 +0100
categories: react typescript
---

## Introduction

When passing properties in React application one usually chooses between so-called prop drilling and React contexts. Both have their cons and pros. Prop drilling means passing the properties down from the parent to the child component. It has the advantage that the parent component can control the values that are used by the child component but can lead to very verbose code. The opposite is true when contexts are used: concise code but no control over the properties of the child component. In this blog post, I will describe a third approach based on Typescript that combines the first two: either the property value is received from the parent component or else the value comes from a DefaultPropsContext. I will refer to this as the "default properties" approach. The source code that implements this approach is available on
[github](https://github.com/mnieber/react-default-props-context) and can be installed via
[npm](https://www.npmjs.com/package/react-default-props-context). In a
[follow-up post](https://mnieber.github.io/react/2020/05/26/inserting-facets-into-react-components.html) I will describe how you can use default properties in data containers.

**Note**: the "default properties" approach is designed to be _compatible_ with MobX. The code is completely independent from MobX though, it only requires React.

## The withDefaultProps function

In the "default properties" approach a component declares two types which - by convention - are called PropsT and DefaultPropsT. The component receives a list of properties `props` that has been enriched by merging in the default properties:

```
import { withDefaultProps, FC } from "react-default-props-context";

type PropsT = {
  name: string,
};

type DefaultPropsT = {
  color: string,
}

const MyComponent = withDefaultProps<PropsT, DefaultPropsT>(
  (props: PropsT & DefaultPropsT) => {

  // The color value comes either from p.color or from a DefaultPropsContext.
  return <text color={props.color}>Hello</text>;
}
```

The `withDefaultProps` function is a higher order function that adds the support for default properties. It creates a new properties object (that is passed into the wrapped component) that has a special lookup function. This lookup function will first try to resolve the property using the input argument of MyComponent. If unsuccessfull it will resolve the property by looking for a getter function in the nearest DefaultPropsContext. If still unsuccessfull then `undefined` is returned.

At this point, we need to clarify a few things:

- how are the default property values provided?
- why are default properties based on getter functions and not just on values?
- how should a property be passed in when we don't want to use the default value in the child component?

To answer these questions, take a look at another example:

```
import { observer } from "mobx-react";
import { DefaultPropsContext } from "react-default-props-context";

const MyFrame = observer(() => {
  const foo = getFoo();

  const defaultProps = {
    color: () => "red",
    bar: () => foo.bar,
  };

  return (
    <DefaultPropsContext.Provider value={defaultProps}>
      <MyComponent name="example" color="green"/>
    </DefaultPropsContext.Provider>
  )
})
```

The DefaultPropsContext provides the dictionary of default property values to the  `withDefaultProps` function. To override the default `color` we can set this property
explicitly on `MyComponent`. You will get the usual Typescript warnings if you are trying to set a property that does not exist (as a regular property or as a default property).
Now let's talk about the getter functions in the the `defaultProps` object. Since functions are more flexible than values this adds a
bit of additional complexity and power. The real reason though for having them has to do with MobX. If we were to copy the `foo.bar`
value directly into `defaultProps` then MobX would notice that we are referencing it. That would mean that `MyFrame` (which is the
component that creates the `defaultProps` object) might be rendered more often than necessary. By using a getter function we avoid referencing `foo.bar`.

## Using nested contexts

One of the features of `DefaultPropsContext` is that it allows nesting. Any nested `DefaultPropsContext` instance should extend and
override the default properties set by any parent `DefaultPropsContext` instances. You can code this manually by merging dictionaries
but a more convenient way is to use the `NestedDefaultPropsProvider` component. In the example below, we add an additional instance
of `MyComponent` with name "exampleInner" that uses an updated and extended set of default properties (where the color is "blue" and
there is an additional `baz` property).

```
import { observer } from "mobx-react";
import { NestedDefaultPropsProvider } from "react-default-props-context";

const MyFrame = observer(() => {
  const foo = getFoo();

  const defaultProps = {
    color: () => "red",
    bar: () => foo.bar,
  };

  const defaultPropsInner = {
    color: () => "blue",
    baz: () => foo.baz,
  };

  return (
    <NestedDefaultPropsProvider value={defaultProps}>
      <MyComponent name="example"/>
      <NestedDefaultPropsProvider value={defaultPropsInner}>
        <MyComponent name="exampleInner/>
      </NestedDefaultPropsProvider>
    </NestedDefaultPropsProvider>
  )
})
```

## What happens when a default property is overwritten?

In one of the examples above we used `color="green"` to override the default value of `color`. This has the effect of (automatically) inserting a `NestedDefaultPropsProvider` that provides the new value. This means that also children of the receiving component see the overridden value!

## Discussion

The benefit of using default properties as described is that it avoids prop drilling, while still allowing you to customize the property
values that a child component uses. By using nested DefaultPropsContext instances, you have a lot of flexibitily in deciding which parts
of the render tree see which default values.
This flexibility comes at a price though: default properties can originate from _any_ DefaultPropsContext instance higher up in the tree.
In this sense, it is different from most other React contexts. To put it in another way: when a component declares that it accepts a
default property, it doesn't care where this property comes from, as long as it's provided by a DefaultPropsContext. If it tries to use a
value that no DefaultPropsContext provides, then it will receive `undefined` (Typescript cannot help us with a warning there).
In my experience, the benefits easily outweight the drawbacks. `DefaultPropsContext` allows you to get information to the place where it is
required, without the need to jump through hoops. This results in shorter and more readable code, that is easier to refactor. So if prop
drilling in your code gets out of hand, I would recommend to give this a try.
