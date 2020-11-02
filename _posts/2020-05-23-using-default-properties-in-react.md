---
layout: post
title: "Using default properties in React"
date: 2020-05-23 16:21:59 +0100
categories: react typescript
---

## Introduction

There are two standard ways in which React components receive properties:

- the component can receive a property directly from the parent component, or
- the component can receive a property from some kind of context. This context may be global variable, or it may be a Context.Provider component.

The first approach has the advantage that the parent component can control the value that is used by the child component. However, passing down all values by setting properties - so-called prop drilling - leads to very verbose code. The second approach avoids prop-drilling but makes it harder to customize the property value that goes into a component. In this blog post, I will describe a third approach based on Typescript that combines the first two: either the property value is received from the parent component or else the value comes from a context. I will refer to this as the "default properties" approach, where the default values come from the context. The source code that implements this approach is available on github: <url here>.

Note that special efforts where made to ensure that the "default properties" approach works well with MobX, as will be explained below. The code is completely independent from MobX though, it only requires React.

## The mergeDefaultProps function

When a component uses default properties then it declares two types which - by convention - are called PropsT and DefaultPropsT. The component receives a list of properties `p` and enriches this list by merging in the default properties. The result is stored as `props`:

```
import { useDefaultProps, FC } from "react-default-props-context";

type PropsT = {
  name: string,
};

type DefaultPropsT = {
  color: string,
}

const MyComponent: FC<PropsT, DefaultPropsT> = (p: PropsT) => {
  const props = useDefaultProps<PropsT, DefaultPropsT>(p);

  // The color value comes either from p.color or from a DefaultPropsContext.
  return <text color={props.color}>Hello</text>;
}
```

The `useDefaultProps` function creates a new object that has a special property lookup function. This lookup function will first try to resolve the property lookup using the members of `p` (the input argument of MyComponent). If the property is not found then it searches for a getter function in the nearest DefaultPropsContext which is invoked.

At this point, we need to clarify a few things:

- how are the default property values provided?
- why are we using a getter function and not just a value?
- what if we want to override the default property value?

To answer these questions, take a look at the example below:

```
import { observer } from "mobx-react";
import { useDefaultProps, FC } from "react-default-props-context";

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

We see that a DefaultPropsContext provides the dictionary of default property values. The
`useDefaultProps` function uses this context to construct a new (merged) properties object. We also see
that we can override the value for the default `color` property value by passing it into `MyComponent`.
Note that you will get the usual Typescript warnings if you are trying to set a property on `MyComponent`
that does not exist (as a regular property or a default property).

As stated above, the defaultProps contains getter functions rather than just values. Since functions are more flexible than values this adds a bit of additional complexity and power. The real reason though has to do with MobX. If we were to copy the `foo.bar` value directly into `defaultProps` then MobX would notice that we are referencing it. That would mean that MyFrame might be rerendered more often than necessary. By using a getter function we avoid referencing `foo.bar`.

One of the features of DefaultPropsContext is that it allows nesting. Any nested DefaultPropsContext instance should extend and override the default properties set by any parent DefaultPropsContext instances. You can code this manually but a more convenient way is to use the NestedDefaultPropsProvider component. In the example below, we add an additional instance of `MyComponent` with name "exampleInner"
that uses an updated and extended set of default properties, where the color is "blue" and there is an additional `baz` property.

```
import { observer } from "mobx-react";
import { useDefaultProps, FC } from "react-default-props-context";

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

## Discussion

The benefit of using default properties as described is that it avoids prop-drilling, why still allowing you to customize the property values that
a child component uses. By using nested DefaultPropsContext instances, you have a lot of flexibitily in deciding which parts of the render tree
see which default values.
This flexibility comes at a price though: default properties can originate from any DefaultPropsContext instance higher up in the tree. In this sense, it is different from most other React contexts. Another way to put it is that when a component declares that it accepts a default property, it doesn't care where this property comes from, as long as it's provided by a DefaultPropsContext. And if it tries to use a value that no DefaultPropsContext provides, then a run-time error will be generated (Typescript cannot help us there).
In my experience, the benefits easily outweight the drawbacks. DefaultPropsContext allows you to get information to the place where it is required, without the need to jump through hoops. This results in shorter and more readable code, that is easier to refactor. So if prop-drilling
in your code gets out of hand, I would recommend to give this a try.
