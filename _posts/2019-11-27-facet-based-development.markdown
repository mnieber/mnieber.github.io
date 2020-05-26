---
layout: post
title:  "Facet based development with MobX"
date:   2019-11-27 16:21:59 +0100
categories: react
---
Introduction
------------

When I write a React application, I tend to implement quite a few things that already exist in my other React applications. Actually, this is how my React skills tend to evolve: I copy existing code and rework it to fit my new use-case. Usually, this reveals opportunities for factoring out use-case specific details and making the code's foundations more generic. This promotes code reuse but it often also makes the code more understandable. However, copy-and-paste coding has obvious drawbacks: maintaining a lot of somewhat similar versions of the same code does not scale well.
In this post, I will propose an approach - based on MobX - to make React code more reusable, without copy-and-paste. I will start off by reviewing the benefits of using layers. Then I will give a step-by-step explanation of how I structure the Domain Layer. In a follow-up blog post, I will talk about connecting this to the Data Layer and Presentation Layer. The source code that powers this approach is available in two github repositories: /home/maarten/projects/facet and /home/maarten/projects/facet-mobx.


Separating the data, domain and presentation layers
---------------------------------------------------

Using layers is a time-tested approach that leads to code that is easier to maintain and reason about. It also promotes code reuse because you can replace one of the layers and leave the other ones as they are. For example, let's consider the familiar use-case of displaying a list of Todo's with a "Filter" button that toggles between showing all todo's or only the ones that are not yet completed. Showing the button happens in the Presentation Layer, filtering the todo's in the Domain Layer and sending a PATCH request in the Data Layer.
Note that it's best to do the filtering in the Domain Layer because in that case we can replace the Presentation Layer without having to reimplement the filtering. Moreover, it often turns out that "local component state" needs to be exposed to other components at some point. Therefore, it's usually better to keep the Presentation layer dumb and move application state into the Domain Layer as much as possible.
It's not unusual to see the Presentation, Domain and Data layers combined in a single React component. The component is like a small application in that case. For the purpose of reuse, this is not a good approach because a new application:

- may have a command line interface instead of a graphical interface. Therefore, React components in the presentation layer should only visualize the current state and trigger callback functions that target the Domain Layer. This way, there is a clear flow from application state -> rendering -> processing of UX events -> new application state. When rendering and event processing are mixed within a React component, the code for that component tends to become hard to read very quickly.

- may have different rules for what happens when the filter is applied. For example, after applying the filter the highlighted todo may not be visible anymore. In that case, we need to decide in the Domain layer which todo is highlighted. Therefore, to facilitate reuse, we need an application-wide domain layer that does not hard code a fixed set of rules, but makes it easy to drop some of the rules and add some other ones.

- may store blog posts differently. Therefore, we require an application-wide Data Layer that offers an API for storing blog posts without revealing which technology (e.g. REST or GraphQL) is used to make requests to the back-end.

I will now describe how the Domain Layer in my application is designed, using the Todo List example introduced earlier.


Using Containers in the Domain Layer
------------------------------------

The Domain Layer stores all run-time data in so-called Containers. The SessionCtr stores all data related to the current user session, such as the User Profile. The TodoListCtr stores all loaded todo's, and the TodoCtr contains all data related to editing the current todo.

Each Container is composed of different Facets: class instances that relate to the different concerns that are being handled by the Container. For example, the Authentication Facet of the SessionCtr contains the logged in user-id and other authentication related data. The Profile Facet of the SessionCtr contains the currently loaded user profile and preferences. The `isFilterActive` flag mentioned above is stored in the Filtering Facet of the TodoListCtr.

We will start our Domain Layer by introducing a TodoListCtr that has a Storage Facet:

```
@facetClass class Storage {
  @input     itemById: { [UUID]: TodoT };
  @data      errorCodes: Array<String>;         // contains any IO errorCodes
  @operation loadItemsForUser(userId: UUID) {};
};

class TodoListCtr {
  @facet(Storage) storage: Storage;

  initFacets() {
    this.storage = initStorage();
  }

  initPolicies(dataApi) {
    const policies = [
      todosAreLoaded(dataApi)
    ];
    policies.forEach(policy => policy(this));
  }

  constructor(dataApi) {
    this.initFacets();
    this.initPolicies(dataApi);
  }
}
```

The idea behind the `initPolicies` function is that we want to avoid hard coding behaviour inside the Facet classes, because that would make it more difficult to reuse them. Instead, we regard the Facet class as an abstract interface that contains a minimal set of primitives for the Presentation Layer to work with. This allows us to use the same Facets in different applications, and still have application-specific behaviour. When React components in the Presentation Layer are generic enough, these can be reused as well, especially when they do not mix state rendering with state manipulation. Here is an example implementation for the `todosAreLoaded` Policy:

```
const todosAreLoaded = dataApi => ctr => {
  const storage = Storage.get(ctr);

  handle(storage, "loadItemsForUser", action(userId => {
    const response = dataApi.loadTodos(userId);
    storage.errorCodes = reponse.errorCodes;

    if (response.success) {
      storage.itemById = response.data;
    }
  }))
}

If you check the code for `initPolicies` you see that todosAreLoaded(dataApi) is added to the list of policies, and at the end of the function each policy is applied to the container. Since each Policy can potentially change the state of the Container, the order of the Policies is essential.
If the loading of Todo's should have side-effects, then you can create additional Policy functions that use `listen` instead of `handle`. Here is an example that uses a Logging Facet:

```
const loadingOfTodosIsLogged = ctr => {
  const storage = Storage.get(ctr);
  const logging = Logging.get(ctr);

  listenBefore(storage, "loadItemsForUser", action(userId => {
    logging.nrOfTodos = storage.itemById.length;
  }))

  listenAfter(storage, "loadItemsForUser", action(userId => {
    const prevNrOfTodos = logging.nrOfTodos;
    logging.nrOfTodos = storage.itemById.length;
    logging.log(`${logging.nrOfTodos - prevNrOfTodos} more todo's were loaded for user ${userId}`)
  }))
}
```

A key feature of this approach is that Policies depend on Facets that reside in the same container. This minimizes the amount of glue code that is needed to connect policies to their data sources. Also, since Policies do not depend on a specific Container type (as long as it offers the required Facets) it's possible to reuse them.


Routing data inside a Container
-------------------------------

We'll now add a Filtering Facet to our Container. It will filter the todo's in the Storage Facet:

```
@facetClass class Filtering {
  @input  filterableItems: { Array<TodoT> }
  @data   isFilterActive: boolean;
  @output filteredItems: { Array<TodoT> }
};

class TodoListCtr {
  @facet(Storage) storage: Storage;
  @facet(Filtering) filtering: Filtering;

  initFacets() {
    this.storage = initStorage();
    this.filtering = initFiltering();
  }

  initPolicies(dataApi) {
    const policies = [
      todosAreLoaded(dataApi),
      filteringActsOnStoredTodos,
    ];
    policies.forEach(policy => policy(this));
  }
}

const filteringActsOnStoredTodos = ctr => {
  reaction(
    () => Storage.get(ctr).itemById,
    itemById => {
      Filtering.get(ctr).filterableItems = Object.values(itemById);
    }
  );
}
```

The `reaction` function comes from MobX and ensures that the data mapping is applied whenever `ctr.storage.itemByIds` changes. Because the pattern for setting up data mappings tends to repeat itself, I'm using a helper function `mapData` that has the same effect:

```
const filteringActsOnStoredTodos = mapData(
  [Filtering, "filterableItems"], // the output value
  [Storage, "itemById"],          // the input value
  Object.values                   // the transformation function
);
```

Finally, we will add a `todosAreFiltered` Policy that does the actual filtering. This time we will use `mapDatas` to map several input values onto an output value:

```
const todosAreFiltered = mapDatas(
  [Filtering, "filteredItems"],      // the output value
  [
    [Filtering, "filterableItems"],  // the input values
    [Filtering, "isFilterActive"],
  ],
  (filterableItems, isFilterActive) => filter(
    item => !isFilterActive || item.isActive,
    filterableItems
  )
);
```

Discussion
----------

The Domain Layer is complex almost by definition, because it needs to reflect the different - often conflicting - rules that govern the behaviour of the application. By using Facets, this complexity is pushed into the Policy functions. This has the advantage that it keeps the mixing of concerns to a minimum, because a) the Facets remain simple and agnostic of each other and b) each Policy function will handle only one concern. The `mapDatas` function helps to create relationships between Facets based on loose coupling. Because Policy functions find all their dependencies inside the Container, they also require relatively little glue code.
What remains complex is to understand the combined effect of the different Policies, especially since the execution of a Policy may fire other Policies. This complexity is inherent in signal-driven designs. However, since the `initPolicies` contains the complete list of Policies, this complexity can be managed. Moreover, the facet library comes with a logging function that shows a nice representation of the Policy call-tree that includes the complete Container state before and after applying the Policy.
Finally, I have found that it's often intuitive to think about the application in terms of Containers and Facets. When the Presentation Layer needs to render some information, there is often a natural place for that information in one of the Facets. In my experience this means that despite the extra indirection that is introduced by the Policy functions, developing an application in this way can be quite fast.