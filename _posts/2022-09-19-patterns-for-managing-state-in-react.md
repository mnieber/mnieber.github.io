# Introduction

To start with a quote from alexei.me: "State Management in Frontend is complicated and approaches are not yet settled." In this article I will outline some patterns that I use for state management in React. I like these patterns and they work well for my use-cases, but I haven't asked too many people if they would work for them too, so if you want to give feedback, please do. As a heads up, I will probable come back and edit this post several times, as I expect my ideas to change (in small or big ways) over time.

## On server state and UI state

I will be talking about server state and UI state. The former is the application information that is stored in the database. Server state management deals with querying the server, caching this information for using it in the various UI components, and sending mutation requests back to the server. We will also need logic for managing the UI state. This includes the contents of forms, selections, toggles that show and hide components, etc.

## Outline

As I explain the patterns, I will go from local to global. I will start with a pattern for capturing local state. Next, I will discuss how local state is filled with server data. Then, I will address how state can be shared among components. Finally, I will share some thoughts on adding behaviors to the UI state (with the aim of keeping the rendering components simple).

## Pattern 1: use high-level state objects in dumb React components

React components are functions that create the (shadow) DOM elements that are rendered in the browser. This task usually involves taking care of a lot of small details, such as setting margins, dealing with responsiveness, setting click handlers etc. This code tends to be relatively straightforward, but with a lot going on. If you mix this code with less-than-trivial state management code, then the result is usually not nicely readable. Therefore, I try to make each component agnostic of low level details related to state management.

One way to achieve this is to create high-level custom hooks that wrap around React's useState hook (or any other hooks). For example, imagine a delete button that requires the user to confirm a delete operation by typing a special word. This button could use the following state:

```
export const useLockedDeleteFlag = (unlockWord: string) => {
  const [askForTheUnlockWord, _setAskForTheUnlockWord] = React.useState(false);
  const [inputWord, setInputWord] = React.useState<string | null>(null);

  return {
    askForTheUnlockWord,
    // When the user presses delete, we ask for the unlock word
    setAskForTheUnlockWord: (x: boolean) => {
        setInputWord("");
        _setAskForTheUnlockWord(true);
    },
    isDeleteUnlocked: askForTheUnlockWord && inputWord === unlockWord,
    inputWord,
    // If this function is called with the correct word, then the delete
    // operation is unlocked
    setInputWord
  };
};
```

## Pattern 2: data fetching is triggered by the url

When a component must render some data, then it should either receive that data from a parent component (or React context) or fetch it from the server. I avoid the latter option because when different components request the same data then it can lead to overfetching. It can also lead to data inconsistencies between components that query the server at different times. Finally, giving a component the responsibility to fetch data that can clutter the code of that component.

In my approach, data fetching is triggered by the url. I look at the UrlRouter component to identify the group of components that need a particular dataset and wrap them in (what I call) a state-provider component. This state-provider fetches data from the server and stores it in a state object that is made available as a React context. Actually, I use a special kind of React context, but more about that later.

Note that the state-provider does not necessarily have to provide the orginal server state to its consumer components. It has the option of wrapping the server state in a high-level interface, that better fits the needs of the components.

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

And here is the example code for the state-provider. Note that it loads the data and then wraps it in a `ArtistState` object. If the clients require special logic for working with the server data, then this logic could be added to the `ArtistState` class.

```
export const ArtistStateContext = React.createContext<ArtistState | null>(null);

export const ArtistStateProvider = (props: React.PropsWithChildren<{}>) => {
  const artistStateQuery = useGetArtistStateQuery();
  const [artistState, setArtistState] = React.useState<ArtistState | null>(null);

  React.useEffect(() => {
    if (artistStateQuery.status === "success") {
      setArtistState(new ArtistState(
        artistStateQuery.data.artist.bio,
        artistStateQuery.data.artist.recordLabel,
      ));
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

The solution of pattern 2 is nice but it has a drawback since components are now coupled to the `ArtistStateContext`. This limits their reusability (we can only use them when our data is in a `ArtistStateContext`), and it also makes their data requirements less transparent (because we have to look inside the component function to find out which contexts it is using).

This can be improved by declaring all component properties upfront, and allowing the values for these properties to come from either the parent component or from a set of default properties. In most cases, the component doesn't know or care where the values really come from. The default properties are still stored in a React context, but this is transparent to the component.

Below, we see an `ArtistStateProvider` that declares two default properties, and a `ArtistBioView` that uses one of them (`artistBio`). It's important to note that all components that are nested inside the `ArtistStateProvider` can access the default properties, regardless of the level of nesting.

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

// An empty object `DefaultProps` object is used here as a "type". It can be
// converted to an actual type using `typeof DefaultProps`. This has the
// advantage that the type fields can be queried at run-time.
// Note that the type signatures for fields such as `artistBio` can be
// captured in a global look-up table. This guards against typos in the type
// fields, and reduces the number of import statements.

const stub = null as unknown;
const DefaultProps = {
  artistBio: stub as ArtistBioT;
  // or `...defaultProps.artistBio` if a global look-up table is used
};

export const ArtistBioView =
  withDefaultProps((props: PropsT & typeof DefaultProps) => {
    return (
      <div className={props.className}>
        {props.artistBio.text}
        Signed to <RecordLabelLogo />
      </div>
    );
  },
  DefaultProps
);
```

With this pattern, the `ArtistBioView` component can be used to show any artist. By default, it will show the biography of the "default" artist, but the parent component may set the `artistProfile` property to some other value (in that case, `withDefaultProps` automatically creates a new `NestedDefaultPropsProvider` so that also the children of `ArtistBioView` see that other value).

This pattern has proven very useful to me, but it's also contentious, as you can shoot yourself in the foot. The problem is that the parent component may explicitly set the `artistBio` property of the `ArtistBioView` component but leave the default value of `recordLabel` as it is. In this case, the `<RecordLabelLogo />` would show the wrong record label. Such a risk can be mitigated by always setting related properties together, using a typed "patch" object (so that forgetting one of the properties leads to a type error), but this only alleviates the problem rather than solving it.

So why do I still use this pattern?

a) it happens rarely that I override a default property. Usually the same objects are used everywhere in a particular branch of the rendering tree. I can only remember one occasion where the above mentioned problem bit me (at this point, I introduced the `patch` type to mitigate this risk in the future).

b) the benefits of more concise code (because of less prop drilling), that more clearly documents the data requirements, and has less coupling to particular contexts offers enough compensation for the increased risk and

c) I'm not writing banking applications (where mistakes are incredibly costly). And if I were, then I would use classical prop drilling instead of this approach.

## Pattern 4: use mutable shared UI state

In my experience, it often happens that local component state doesn't stay local. Initially, that flag that determines if a menu is open or closed is only needed by the menu component, but at some point other components may also want to get or set this flag. So, I usually do not hesitate to encapsulate UI state in an object that is shared between components.

In some cases, this shared UI state object becomes quite extensive. Take, for example, a song list in which the user can select, highlight (i.e. pick the current song), drag-and-drop and filter. These behaviours require state: we need to store which songs were selected, what position they are being dragged to, what keywords the list is filtered on, etc. These behaviours also interact. Some examples are:

- drag-and-drop should operate on the current selection,
- we cannot select songs that were hidden by the filter,
- when the filter is enabled, we may want to move the highlight to an item that is not filtered out.

Typically, what I do is create a state object that stores both the song list and all the behaviour-related state. Then I create a state-provider that provides the contents of this state to the consumer components. These components can then call `select(songId)`, `drop(targetSongId)`, `setFilter(keywords)`, etc. It turns out that the logic for handling selection, insertion, filtering, drag-and-drop, etc, can be written in a generic way (this is the topic of the `SkandhaJS` library I'm also working on). The example below shows how we can combine song-data with behaviors, map data between the behaviors, and set up interactions between them. This approach may seem overkill, but it has some advantages (especially when you add more and more behaviors):

- it keeps each behavior agnostic of all other behaviors (because the interaction between behaviors is determined in callback functions)
- it allows for behavior classes to be generic and therefore reusable. The interaction between behaviors can also be captured in generic functions (such as the `highlightIsCorrectedOnFilterChange` function used below)

```
import { Filtering } from 'skandha-facets/Filtering;
import { highlightIsCorrectedOnFilterChange } from 'skandha-facets/policies;
import { Selection, SelectionCbs, handleSelectItem } from 'skandha-facets/Selection;
import { Highlight } from 'skandha-facets/Highlight;

export class SongsState {
  songs = {
    data: new SongsData(),
    filtering: new Filtering<SongT>(),
    highlight: new Highlight<SongT>(),
    selection: new Selection<SongT>(),
  };

  constructor(props: PropsT) {
    // Map `data.songs` to `songs.filtering.inputItems`.
    // Map `filtering.filteredItems` to `data.songsDisplay`.
    // Map `data.songsDisplay` to `data.selectableIds`
    mapDataToProps(
      [[this.songs.filtering, 'inputItems'], () => this.songs.data.songs],
      [[this.songs.data, 'songsDisplay'], () => this.songs.filtering.filteredItems],
      [[this.songs.selection, 'selectableIds'], () => getIds(this.songs.data.songsDisplay)]
    );

    // Install a `selectItem` callback that handles the selection
    // and also updates the highlight.
    setCallbacks(this.songs.selection, {
      selectItem: {
        selectItem(this: SelectionCbs['selectItem']) {
          handleSelectItem(this.songs.selection, this.selectionParams);
          this.songs.highlight.id = this.selectionParams.itemId;
        },
      },
    } as SelectionCbs);

    // Install a `apply` callback that corrects the highlight after
    // the filter was applied.
    setCallbacks(this.songs.filtering, {
      apply: {
        exit(this: FilteringCbs['apply']) {
          highlightIsCorrectedOnFilterChange(
            this.songs.filtering,
            this.songs.highlight
          );
        },
      },
    } as FilteringCbs);
  }
}
```

When the shared state changes then components are re-rendered. Since I handle this with MobX, the shared state is mutable, and changes are synchronous. In general I like data to be immutable, but immutable UI state was giving me many headaches. For example, imagine that adding a song to the list triggers a rule that calculates if the user must upgrade to a paid account. This rule is complex, and checks other things besides the contents of the song list. With (synchronous) mutable state, I can add the song, run the rule and then - if necessary - remove the song and ask the user to upgrade. With immutable data, I would have to wait for the state change to happen in the future, and then run the rule. The problem is that by that time, the context in which the state change happened is not available anymore. To find out that the last added song must be removed again we'd have to reconstruct what happened in the past. This is just one example, but with immutable state I found myself running into this type of problem often.

## Conclusion

In this article I've illustrated some patterns that I use for state management in React. Of course, it reflects some of the trade-offs that I make. I prefer to avoid prop drilling, even if it introduces some risk of data inconsistencies. And I prefer mutable state, though I'm aware of the general advantages of immutable data.

What I think is sometimes overlooked in software development is that there are visible and hidden trade-offs. For example, testing makes applications more robust, but if there are many tests then the cost of changing the code increases (because the idea that you can change the code without having to update the tests is - in my experience - an illusion). This could mean that certain refactorings that improve robustness are never executed, because they have become too expensive. In other words, there is a hidden trade-off between code-size and robustness. This probably explains why programmers can debate forever on the best ways to develop software: everything is a trade-off, and most of these trade-offs are hard to pin down.

There can also be a trade-off between familiarity and effectiveness. React introduced some new approaches that were unfamiliar, such as the ability to write HTML in Javascript, the use of hooks to store state inside functions (as long as we unconditionally call all the hooks, in the same order) and the use of a shadow DOM. I propose that these things are arguably weird (at least initially) but not unintuitive or bad. Otherwise, few people would be using React. In that spirit, I hope you will approach these patterns with an open mind.

To conclude, I look forward to discussing these patterns with you, and to hearing what's good, bad and (perhaps) weird about them.
