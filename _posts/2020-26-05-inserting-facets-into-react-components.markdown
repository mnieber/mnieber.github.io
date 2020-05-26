---
layout: post
title:  "Facet based development"
date:   2020-05-26 16:21:59 +0100
categories: react
---
Introduction
------------

In a previous blog post I described how I use Containers and Facets to structure the Domain Layer for my React applications. In this post I will briefly describe how I connect to these Containers in the Presentation Layer. The main idea is to use a CtrProvider component to create the containers and to use mergeDefaultProps (described here: ) to pass the Facets of each container into the component tree.


The CtrProvider component
-------------------------

The CtrProvider class is a helper component for creating container instances that takes two arguments:

- a `createCtr` function that is invoked only once to instantiate the Container
- a `getDefaultProps` function that extracts default properties from this container. These default properties are provided
to the rest of the component tree using a NestedDefaultPropsProvider.

CtrProvider is implemented as follows:


```
type PropsT = {
  children: any,
  createCtr: Function,
  getDefaultProps: Function,
};

export const CtrProvider = observer((props: PropsT) => {
  const [ctr, setCtr] = React.useState(props.createCtr);
  return (
    <NestedDefaultPropsProvider value={props.getDefaultProps(ctr)}>
      {props.children}
    </NestedDefaultPropsProvider>
  );
});
```

In the example below, CtrProvider is used to provide a SessionCtr instance:

```
export const TodoListCtrProvider = ({ children }: PropsT) => {
  const createCtr = () => {
    const ctr = new TodoListCtr({
      dataApi: {
        loadTodos: (userId) => { /* load user id */ },
      },
    });

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

Loading data into the Domain Layer
----------------------------------

Based on the current URL, the Url Router in the Presentation Layer determines which React component to render. However, the Url Router also calls an operation in the Domain Layer to load the data that the React component will present. For example, the pseudo code below shows how the UrlRouter may load a list of todos:

```
if (url.matches("/:userId/posts")) {
    const urlParams = useParams();
    props.todoListStorage.loadTodosForUser(urlParams.userId);
}
```

Discussion
----------

The CtrProvider offers a clean way of inserting Containers and Facets into the component tree. These Facets are propagated down the tree as default properties. When the Facets are received in a React component, I make sure that the data is already loaded by delegating the data fetching step to the UrlRouter.
