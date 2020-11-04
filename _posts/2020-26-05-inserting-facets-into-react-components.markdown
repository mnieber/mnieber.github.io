---
layout: post
title: "Facet based development"
date: 2020-05-26 16:21:59 +0100
categories: react
---

## Introduction

In a previous blog post I described how I use Containers and Facets to structure the Application Layer for my React applications. In this post I will briefly describe how I connect to these Containers in the Presentation Layer. The main idea is to use a CtrProvider component to create the containers and to use the useDefaultProps hook (described here: ) to pass the Facets of these containers into the component tree.

## The CtrProvider component

The CtrProvider class is a helper component for creating container instances that takes two arguments:

- a `createCtr` function that is invoked only once to instantiate the container
- an `updateCtr` function sets up a MobX reaction that keeps the inputs of the container up-to-date
- a `getDefaultProps` function that extracts default properties from this container. These default properties are provided
  to the rest of the component tree using a NestedDefaultPropsProvider.

CtrProvider is implemented as follows:

```
type PropsT = React.PropsWithChildren<{
  createCtr: Function,
  updateCtr: Function,
  getDefaultProps: Function,
}>;

export const CtrProvider: React.FC<PropsT> = observer((props: any) => {
  const [ctr] = React.useState(props.createCtr);

  React.useEffect(() => {
    if (props.updateCtr) {
      props.updateCtr(ctr);
    }
  });

  return (
    <NestedDefaultPropsProvider value={props.getDefaultProps(ctr)}>
      {props.children}
    </NestedDefaultPropsProvider>
  );
});
```

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

Based on the current URL, the Url Router in the Presentation Layer determines which React component to render. However, the Url Router also calls an operation in the Application Layer to load the data that the React component will present. For example, the pseudo code below shows how the UrlRouter may load a list of todos:

```
if (url.matches("/:userId/posts")) {
    const urlParams = useParams();
    props.todoListStorage.loadTodosForUser(urlParams.userId);
}
```

## Discussion

The CtrProvider offers a clean way of inserting Containers and Facets into the component tree. These Facets are propagated down the tree as default properties. When the Facets are received in a React component, I make sure that the data is already loaded by delegating the data fetching step to the UrlRouter.
