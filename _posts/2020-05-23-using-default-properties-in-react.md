---
layout: post
title:  "Using default properties in React"
date:   2020-05-23 16:21:59 +0100
categories: react
---
Introduction
------------

There are two standard ways in which React components receive properties:
- the component can receive a property directly from the parent component, or
- the component can receive a property from some kind of context. This context may be a global Redux store, or it may be a Context.Provider component.

The first approach has the advantage that the parent component can control the value that is used by the child component. However, passing down all values by setting properties - so-called prop drilling - leads to very verbose code. The second approach avoids prop-drilling but makes it harder to customize the property value that goes into a component. In this blog post, I will describe a third approach that combines the first two: either the property value is received from the parent component or else the value comes from a context. I will refer to this as the "default properties" approach, where the default values come from the context. The source code that implements this approach is available on github:


The mergeDefaultProps function
------------------------------

When a component uses default properties then it declares two Flow types which - by convention - are called PropsT and DefaultPropsT. The approach does not depend on Flow, but it's easier to catch mistakes if you use it. The component receives a list of properties `p` and enriches this list by merging in the default properties. The result is stored as `props`:

```
type PropsT = {
  name: string,
  defaultProps: any,
};

type DefaultPropsT = {
  color: string,
}

const MyComponent(p: PropsT) {
  const props = mergeDefaultProps<PropsT & DefaultPropsT>(p);

  // The color value comes either from p.color or p.defaultProps.color
  const myText = <text color={props.color}>Hello</text>;

  // Use prop-drilling to pass props.defaultProps down to MyChildComponent.
  // MyChildComponent may refer to default properties that exist
  // in props.defaultProps but are not mentioned in DefaultPropsT.
  const myChildComponent = <MyChildComponent defaultProps={props.defaultProps}/>;

  return <div>{myText}{myChildComponent}</div>;
}```

The `mergeDefaultProps` function creates a new object that has a special property lookup function. This lookup function will first try to resolve the property lookup using the members of `p` (the input argument of MyComponent). If the property is not found, then it will use the property name to look up a getter function in `p.defaultProps`. If this getter function is found then it is invoked to produce the property value. The reason for using a function instead of storing values directly in `p.defaultProps` is that this plays better with MobX, as will be explained later.
Note that there is nothing special about `p.defaultProps`, it's just a dictionary that maps property names to getter functions. Each getter function takes no arguments and returns the property value. Flow will produce a warning if any property of `props` is accessed that does not exist in either PropsT or DefaultPropsT.


The DefaultPropsContext component
---------------------------------

It's common in a React application to distinguish between "smart" components that are connected to a data-store, and "dumb" components that only receive property values and are agnostic of the world around them. When using default properties, the smart component can be responsible for creating the 'defaultProps` dictionary that is passed into the dumb components that need it.
However, this scenario will not always work or be optimal. In some cases, the smart component is just a frame that is not directly responsible for rendering any child components. In that case, you can use a DefaultPropsContext to make the default properties available to any child component that needs it:

```
const SmartComponent = ({ children }) => {
  const foo = getFoo();
  const defaultProps = {
      color: () => "red",
      bar: () => foo.bar,
      // other properties...
  }

  return (
    <DefaultPropsContext.Provider value={defaultProps}>
      {children}
    </DefaultPropsContext.Provider>
  );
}
```

Those of you who are familiar with MobX will now be able to understand why getter functions are used in `defaultProps`. If we would copy the value of `foo.bar` directly into the defaultProps dictionary then MobX would detect that we are referencing `foo.bar`. This means that any change in `foo.bar` would cause SmartComponent to be re-rendered, which is not what we want.
One of the features of DefaultPropsContext is that it allows nesting. Any nested DefaultPropsContext instance should extend and override the default properties set by any parent DefaultPropsContext instances. You can code this manually but a more convenient way is to use the NestedDefaultPropsProvider component. In the example given below, if  AuthenticationRelatedDefaultProps and UserProfileRelatedDefaultProps both render a NestedDefaultPropsProvider then the combined set of default properties will be available to `{children}` via DefaultPropsContext.Consumer:

```
const SmartComponent = ({ children }) => {
  return (
    <AuthenticationRelatedDefaultProps>
        <UserProfileRelatedDefaultProps>
          {children}
        </UserProfileRelatedDefaultProps>
    </AuthenticationRelatedDefaultProps>
  );
}
```

To facilitate this pattern, there is also a `NestedDefaultPropsProvider` component that automatically merges its default properties

Discussion
----------

A benefit from using default properties as described is that it avoids a lot of verbosity. The reason is that React contexts are not a natural choice for dumb React components. Therefore, even when React context are used, there is often a lot of prop-drilling to pass properties down to dumb components. However, using prop-drilling to pass down several properties in a single `defaultProps` dictionary is a relatively light-weight option.
Moreover, using default properties allows for a lot of flexibility. Any parent component may decide to pass down a modified version of `defaultProps` to the sub-tree under that component.
Finally, receiving a single set of default properties is - in my opinion - more agile than querying one or more React contexts for those values. Take "defaultProps.color" as an example. If a child component needs a default color then it usually doesn't need to know if this color is provided by a ThemeContext or by a UserPreferencesContext. In that case, using `defaultProps.color` seems more appropriate than receiving the color through ThemeContext.Consumer or UserPreferencesContext.Consumer.
