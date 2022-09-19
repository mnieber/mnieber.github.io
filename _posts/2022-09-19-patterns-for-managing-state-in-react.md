# Introduction

To start with a quote from alexei.me: "State Management in Frontend is complicated and approaches are not yet settled." In this article I will outline some patterns that I use for state management in React. My motivation for writing this down is that I would like to get feedback. I like these patterns and they work well for my use-cases, but I haven't asked too many people if they would work for them too. As a heads up, I will probable come back and edit this post several times, as I expect my ideas to change (in small or big ways) over time.

## On server state and UI state

I will be talking about server state and UI state. The former is the application information that is stored in the database. Server state management deals with querying the server, caching this information for using it in the various UI components, and sending mutation requests back to the server. We will also need logic for managing the UI state. This includes the contents of forms, selections, toggles that show and hide components, etc.

## Pattern 1: use high-level state objects in dumb React components

React components are functions that create the (shadow) DOM elements that are rendered in the browser. This task usually involves taking care of a lot of small details, such as setting margins, dealing with responsiveness, setting click handlers etc. This code tends to be relatively straightforward, but with a lot going on. If you mix this code with less-than-trivial state management code, then it's almost guaranteed that the result will not be nicely readable. Therefore, it's advisable to make the component agnostic of low level details related to state management.

One way to achieve this is to create high-level custom hooks that wrap around React's useState hook (or any other hooks). For example, imagine a delete button that requires the user to confirm by typing a special word. This button could use the following state:

```
export const useLockedDeleteFlag = (unlockWord: string) => {
  const [askForTheUnlockWord, _setAskForTheUnlockWord] = React.useState(false);
  const [inputWord, setInputWord] = React.useState<string | null>(null);

  return {
    askForTheUnlockWord,
    setAskForTheUnlockWord: (x: boolean) => {
        setInputWord("");
        _setAskForTheUnlockWord(true);
    },
    isDeleteUnlocked: askForTheUnlockWord && inputWord === unlockWord,
    inputWord,
    setInputWord
  };
};
```

In general, I also put the code that sends mutations to the server in state objects, and not in components. For example, the state object may have a `deleteUserAccount` function so that the component can be agnostic of how this works. In general, this leads to having more dumb components, and less smart ones. In turn, this helps testing.

## Pattern 2: data fetching is triggered by the url

When a component must render some data, then it should either receive that data from a parent component (or React context) or fetch it from the server. I avoid the latter option because when different components request the same data it can lead to overfetching. It can also lead to data inconsistencies between components that query the server at different time points. Finally, it adds a responsibility that can clutter the code of the component function.
Instead, I look at the UrlRouter component to identify the group of components that need a particular dataset and wrap them in (what I call) a state-provider component. This state-provider fetches data from the server and stores it in a state object that is made available as a React context. Actually, I use a special kind of React context, but more about that later. A benefit of this approach is that the state object can wrap the server state in a high-level interface, that better fits the needs of the components.

Below we see an example of two components wrapped in a state-provider component.

```
export const UrlRouter = () => {
  return (
    <Router history={history}>
      <Route path="/home"><WelcomeScreen /></Route>
      <Route path="/home/artist/:artistId">
        <ArtistStateProvider>
          <Route path="/home/artist/:artistId/bio">
            <ArtistBioView />
          </Route>
          <Route path="/home/artist/:artistId/recordLabel">
            <RecordLabelView />
          </Route>
        </ArtistStateProvider>
      </Route>
    </Router>
  );
};
```

And here is the example code for the state-provider.

```
export const ArtistStateContext = React.createContext<ArtistState | null>(null);

export const ArtistStateProvider = (props: React.PropsWithChildren<{}>) => {
  const artistStateQuery = useGetArtistStateQuery();
  const [artistState, setArtistState] = React.useState<ArtistState | null>(null);

  React.useEffect(() => {
    if (artistStateQuery.status === "success") {
      setArtistState(new ArtistState(artistStateQuery.data));
    }
  }, [artistStateQuery.status]);

  return (
    <ArtistStateContext.Provider value={artistState}>
      {props.children}
    </ArtistStateContext.Provider>
  )
};
```

## Pattern 3: use default properties

In my opinion, the solution of pattern 2 is nice but it has a drawback since components are now coupled to the `ArtistStateContext`. This limits their reusability, and it also makes their data requirements less transparent. Therefore, I created a new pattern in which all component properties are declared upfront, and where the values for these properties may come either from the parent component or from a context.

Below, we see an `ArtistStateProvider` that declares two default properties, and a `ArtistBioView` that uses these properties.

```
import { NestedDefaultPropsProvider } from 'react-default-props-context';

export const ArtistStateProvider = (props: React.PropsWithChildren<{}>) => {
  // The code for creating the ArtistState instance is the same as in pattern 2.
  // However, the return value is different, as shown below.

  const defaultProps = {
    artistBio: () => artistState.artist.bio,
    recordLabel: () => artistState.artist.recordLabel,
  }

  return (
    <NestedDefaultPropsProvider value={defaultProps}>
      {props.children}
    </NestedDefaultPropsProvider>
  )
};
```

```
import { withDefaultProps } from 'react-default-props-context';

type PropsT = {
  className?: any;
};

type DefaultPropsT = {
  artistBio: ArtistBioT;
};

export const ArtistBioView =
  withDefaultProps<PropsT, DefaultPropsT>((props: PropsT & DefaultPropsT) => {
    return (
      <div>
        {props.artistBio.text}
        Signed to <RecordLabelLogo />
      </div>
    );
  }
);
```

With this pattern, the `ArtistBioView` component can be used to show any artist. By default, it will show the biography of the "default" artist, but the parent component may set the `artistProfile` property to some other value (in that case, `withDefaultProps` automatically creates a new `NestedDefaultPropsProvider` so that also the children of `ArtistBioView` see that other value).
This pattern has proven very useful to me, but it's also contentious, as you can shoot yourself in the foot. The problem is that the parent component may explicitly set the `artistBio` property of the `ArtistBioView` component but leave the default value of `recordLabel` as it is. In this case, the `<RecordLabelLogo />` would show the wrong record label. Such a risk can be mitigated by always setting related properties together, using a typed "patch" object (so that forgetting one of the properties leads to a type error), but this only alleviates the problem rather than solving it.

So why still use this pattern? For me, the answer is that:

a) it happens rarely that I override a default property, because usually the same objects are used everywhere in a particular branch of the rendering tree. In fact, I can only recall one case I was bitten by the mentioned problem, and the mitigation of using a patch object would have caught that case.

b) the benefits of more concise code (because of less prop drilling), that more clearly documents the data requirements, and has less coupling to particular contexts offers enough compensation for the increased risk and

c) I'm not writing banking applications (where mistakes are incredibly costly). And if I were, then I would use classical prop drilling instead of this approach.

However, I do understand if you would make a different trade-off (and if you can illustrate why that trade-off would be different for you, please leave a comment).

## Pattern 4: use mutable shared UI state

In my experience, it often happens that local component state doesn't stay local. Initially, that flag that determines if a menu is open or closed is only needed by the menu component, but at some point other components may also want to get or set this flag. So, I usually do not hesitate to encapsulate UI state in an object that is shared between components.

In some cases, this shared UI state object becomes quite extensive. Take, for example, a song list in which the user can select, highlight (i.e. pick the current song), drag-and-drop and filter. These behaviours require state. For example, we need to store which songs were selected, what position they are being dragged to, what keywords the list is filtered on, etc. These behaviours also interact. For example, we can only select songs that have not been filtered out by the filter. Moreover, drag-and-drop should operate on the current selection, etc.
Typically, what I do is create a state object that stores both the song list and all the behaviour-related state. There is a state-provider for this state object, so that all components in the relevant part of the rendering tree can call `select(songId)`, `drop(targetSongId)`, `setFilter(keywords)`, etc.

I'm a fan of MobX so therefore I will typically add this state to a MobX-observable class instance. The use of MobX implies that state is mutable, and changes are synchronous. In general I like data to be immutable but when it comes to managing UI state, I noticed that immutable data was giving me many headaches. For example, imagine that adding a song to the list may trigger a rule that calculates if the user must upgrade to a paid account. This rule is complex, and checks other things besides the contents of the song list. With (synchronous) mutable state, I can just add the song, run the rule and then - if necessary - remove the song and ask the user to upgrade. With immutable data, I would have to wait for the state change to happen in the future, and then run the rule. The problem is that by that time, the context in which the state change happened is not available anymore. To find out that the last added song must be removed again we'd have to reconstruct what happened in the past. This is just one example, but with immutable state I found myself running into this type of problem often.

## Conclusion

In this article I've illustrated some patterns that I use for state management in React. Of course, it reflects some of the trade-offs that I make. I prefer to avoid prop drilling, even if it introduces some risk of data inconsistencies. And I prefer mutable state, though I'm aware of the general advantages of immutable data.

What I think is sometimes overlooked is that in software development there are hidden trade-offs. For example, testing makes applications more robust, but if there are many tests then the cost of changing the code increases (because the idea that you can change the code without having to update the tests is - in my experience - an illusion). This might mean that certain refactorings that improve robustness become too expensive and are never executed. In other words, there is a hidden trade-off between code-size and robustness. This probably explains why programmers can debate forever on the best ways to develop software: everything is a trade-off, and most of these trade-offs are hard to pin down.

There can also be a trade-off between familiarity and effectiveness. React introduced some new approaches that were unfamiliar, such as the ability to write HTML in Javascript, the use of hooks to store state inside functions (as long as we unconditionally call all the hooks, in the same order) and the use of a shadow DOM. I would argue that these things are arguably weird (at least initially) but not unintuitive or bad. Otherwise, few people would be using React. So if the patterns I presented seem weird, please keep an open mind.

To conclude, I look forward to discussing these patterns, and to hearing what's good, bad and (if you must) weird about them.
