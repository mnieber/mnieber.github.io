---
layout: post
title: 'Inserting facets into React components'
date: 2020-05-26 16:21:59 +0100
categories: react
---

## Introduction

In a previous [blog post](https://mnieber.github.io/react/2019/11/27/facet-based-development.html) I described how
I use Containers and Facets to structure the Application Layer for my React applications. In this post I will briefly
describe how I connect to these Containers in the Presentation Layer. The main idea is to use a CtrProvider component to create the containers and to use the useDefaultProps hook (described [here](https://mnieber.github.io/react/typescript/2020/05/23/using-default-properties-in-react.html)) to
access the Facets of these containers inside the component tree. In other words, this article brings together the
concepts of containers, facets and default properties.

## The CtrProvider component

The CtrProvider class is a helper component for creating container instances that takes three arguments:

- a `createCtr` function that is invoked only once to instantiate the container
- an `updateCtr` function sets up a MobX reaction that keeps the inputs of the container up-to-date
- a `getDefaultProps` function that extracts default properties from this container. These default properties are provided
  to the rest of the component tree using a `NestedDefaultPropsProvider`.

`CtrProvider` is implemented as follows:

```
import * as React from "react";
import { NestedDefaultPropsProvider } from "./NestedDefaultPropsProvider";

const ctrByKey: { [ctrKey: string]: any } = {};

type PropsT = React.PropsWithChildren<{
  createCtr: Function;
  destroyCtr: Function;
  updateCtr: Function;
  getDefaultProps: Function;
}>;

export const CtrProvider: React.FC<PropsT> = (props: PropsT) => {
  const [ctr] = React.useState(() => {
    return props.createCtr();
  });

  React.useEffect(() => {
    const cleanUpFunction = props.updateCtr ? props.updateCtr(ctr) : undefined;
    const unmount = () => {
      if (cleanUpFunction) cleanUpFunction();
      props.destroyCtr(ctr);
    };
    return unmount;
  });

  return (
    <NestedDefaultPropsProvider value={props.getDefaultProps(ctr)}>
      {props.children}
    </NestedDefaultPropsProvider>
  );
};
```

Notes:

1.  The `CtrProvider` uses a `NestedDefaultPropsProvider` to provide the default properties to its children. This means
    that when you nest `CtrProvider` instances, then their default properties will be merged.

In the example below, CtrProvider is used to provide a TodoListCtr instance:

```
export const TodoListCtrProvider = ({ children }) => {
  const createCtr = () => {
    const ctr = new TodoListCtr();

    // Do any actions that need to happen before using
    // the container.
    ctr.filtering.setIsFilterActive(true);

    return ctr;
  };

  const getDefaultProps = ctr => {
    return {
      todoListStorage: () => ctr.storage,
      todoListFiltering: () => ctr.filtering,
      todoListCtr: () => ctr,
    };
  };

  return (
    <CtrProvider
      createCtr={createCtr}
      getDefaultProps={getDefaultProps}
      children={children}
    />
  );
};

```

## Loading data into the Application Layer

Based on the current URL, the Url Router in the Presentation Layer determines which React component to render. However, the
Url Router also calls an operation in the Application Layer to load the data that the React component will present. For
example, the pseudo code below shows how the UrlRouter may load a list of todos:

```
if (url.matches("/:userId/posts")) {
    const urlParams = useParams();
    props.todoListStorage.loadTodosForUser(urlParams.userId);
}
```

## Discussion

The CtrProvider offers a clean way of inserting Containers and Facets into the component tree. These Facets are propagated down
the tree as default properties. When the Facets are received in a React component, I make sure that the data is already loaded by delegating the data fetching step to the UrlRouter.
