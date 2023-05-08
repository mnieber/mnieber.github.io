---
layout: post
title: 'A styling system based on inline styles and SCSS'
date: 2023-05-07 13:45:59 +0100
categories: programming css scss react
---

# A styling system based on inline styles and SCSS

Note that this post is still in draft. I would like to collect feedback so that I can improve it.

## The trade-off problem: consistency versus flexibility

In this post, we'll take a look at a particular problem that developers face when creating a UI, which I will call the trade-off problem. The trade-off problem occurs when you try to create an application that looks both consistent and beautiful. To make the application look consistent, it helps to capture commonly used styles and layouts in building blocks that can be reused throughout the application. However, these building blocks introduce rigidity that makes it harder to adjust each UI element to its local context. Therefore, we see that there is a trade-off between consistency. reusability and DRY code on the one hand, and flexibility, beauty and isolated code on the other.

There is no real solution to the trade-off problem, but there are different approaches that each have their own strengths and weaknesses. These are very well explained in a video by Theo from T3 (https://www.youtube.com/watch?v=CQuTF-bkOgc), in which he advises to create UI elements using inline styles. I recommend to check out this video first, since I'm building on this advice. In the remainder of this post I will explain how inline styles address the trade-off problem, and how inline styles are used in my own approach for managing styles. I will discuss this in the context of React, since that's the framework I use to build UI's. However, the ideas presented here are not specific to React, they can be applied to other frameworks too.

## Addressing the trade-off problem with inline styles

To understand the strengths and weaknesses of inline styles, it's insightful to take a look at the approach that is offered by TailwindUI. In TailwindUI, every component is a code snippet that contains HTML elements with inline styles. To use a component, you simply copy the snippet to your source file and adjust it as needed. This means that your source code may contain multiple copies of the snippet. Here is an example of a snippet for an email form field:

```html
<div class="mb-6">
  <label
    for="email"
    class="mb-2 block text-sm font-medium leading-6 text-gray-900"
    >Email address</label
  >
  <div>
    <input
      id="email"
      name="email"
      type="email"
      class="block w-full rounded-md border-0 py-1.5 text-gray-900 shadow-sm ring-1 ring-inset ring-gray-300 placeholder:text-gray-400 focus:ring-2 focus:ring-inset focus:ring-indigo-600 sm:text-sm sm:leading-6"
    />
  </div>
</div>
```

This approach is very flexible, because every component instance can be adjusted so that it looks good in its particular context. Some other advantages are:

- you won't have to search for the styles of an element, since they are right there in the HTML code;
- there is no need to come up with good CSS class names;
- when you adjust the instance, there is no risk of breaking the look of other component instances.

Note that the use of inline styles doesn't prevent you from extracting commonly used styles or HTML structures into reusable building blocks. Whenever we do this, we trade of some of the flexibility to have more consistency and reuse. We'll come back to this approach later in this post.

Let's also consider the effect of using inline styles on consistency. Even though copying from a HTML snippet doesn't enforce consistency, it still promotes it, because by using the same snippet for all your form fields you will create a consistent look. The drawback is that you need to be disciplined as a developer to maintain this consistency as you make changes to the inline styles. Some other drawbacks are:

- when you want to change the look of a component everywhere in the application, then you need to update the inline styles in potentially many places.
- inline styles tend to look noisy, and they don't communicate an intent to the reader of the code.

Personally, I like being able to look at a piece of code, and confirm for myself that it looks correct. This is hard to do when you are looking at a big list of inline styles. Of course you could argue that a wrong or missing style is not the end of the world, and that if the result looks really wrong then you will notice this sooner or later, but psychologically, I find it creates some stress.

## Breaking out intrinsic styles

We've seen above that inline styles are copied from a snippet and adjusted to fit the local context. However, there is usually a subset of styles that are never changed. I call these the intrinsic styles of the component. In my opinion, it's better to move these styles to a CSS class, so that the code becomes less noisy. Let's look at an example where this is applied. In this example, the code snippet has been copied to a typescript function, and the intrinsic styles have been moved to a SCSS file.

```tsx
export const SignInForm = () => {
  return (
    <div className="SignInForm">
      {
        // Form title omitted...
      }
      <div className="FormField mb-6">
        <label for="email" className="FormField__Label mb-2">
          Email address
        </label>
        <div>
          <input
            id="email"
            name="email"
            type="email"
            className="FormField__Input py-1.5 w-full"
          />
        </div>
      </div>
      {
        // Other form elements such as the SubmitButton omitted...
      }
    </div>
  );
};
```

```scss
.FormField__Label {
  @apply block text-sm font-medium leading-6 text-gray-900;
}

.FormField__Input {
  @apply block rounded-md border-0 text-gray-900 placeholder:text-gray-400 sm:text-sm sm:leading-6;
  @apply shadow-sm ring-1 ring-inset ring-gray-300 focus:ring-2 focus:ring-inset focus:ring-indigo-600;
}
```

It's important to note that there are no strict rules for picking the properties that are intrinsic. For example, if not all labels in a form should have the same text color then the `text-gray-900` style should remain inline. In most cases, the reader can intuit why certain styles are inline and others are not. In less intuitive cases, for example when a margin appears in the CSS file, it's good to add a comment to explain why the style is considered to be intrinsic.

## Ad-hoc properties and layouts

I use the term ad-hoc properties for those properties that remain inline. Usually, the ad-hoc properties deal with the layout and geometry of the component, such as the flex-box properties, margins, heights, widths and paddings.

Although ad-hoc properties can be adjusted to suit a particular context, it's still good for consistency to limit the variation in these properties. For example, if the email field has a vertical padding of 1.5, then probably the password field in the same form should also have this padding. Therefore, I also break out the ad-hoc properties and put them in a layout object.

```tsx
import { classnames as cn } from 'classnames';

export const SignInForm = () => {
  return (
    <div className="SignInForm">
      {
        // Form title omitted...
      }
      <div className={cn(L.FormField.root)}>
        <label for="email" className={cn(L.FormField.label)}>
          Email address
        </label>
        <div>
          <input
            id="email"
            name="email"
            type="email"
            className={cn(L.FormField.input)}
          />
        </div>
      </div>
      {
        // Other form elements such as the SubmitButton omitted...
      }
    </div>
  );
};
```

```ts
export const L = {
  FormField: {
    // Use the root style in the top level element of the FormField
    root: 'FormField mb-6',
    // Use this style if the form-field contains a label
    label: 'FormField__Label mb-2',
    // Use this style if the form-field contains an input
    input: 'FormField__Input py-1.5 w-full',
  },
};
```

As can be seen from the comments, it's not mandatory to use all the keys in the layout object. For example, the `label` key is only used if the form-field has a label. It's also possible to include variations on the layout, as follows:

```tsx
import { classnames as cn } from 'classnames';

export const SignInForm = () => {
  return (
    <div className="SignInForm">
      <div
        className={cn(
          L.FormField.container.root,
          L.FormField.container.gap.big
        )}
      >
        <label
          for="email"
          className={cn(L.FormField.label.root, L.FormField.label.color.blue)}
        >
          Email address
        </label>
        {
          // Other elements stay the same...
        }
      </div>
    </div>
  );
};
```

```ts
export const L = {
  FormField: {
    // The container element is the top-level element that contains the entire form field
    container: {
      root: 'FormField',
      // Use one of the gap styles to create space below the form-field
      gap: {
        big: 'mb-6',
        small: 'mb-2',
      },
    },
    // Use this style if the form-field contains a label
    label: {
      root: 'FormField__Label mb-2',
      // Pick one color for the label
      color: {
        blue: 'text-blue-500',
        green: 'text-green-400',
      }
    }
    // Use this style if the form-field contains an input
    input: 'FormField__Input py-1.5 w-full',
  },
};
```

I usually add a lot of comments to the layout object, to make it clear how each layout is intended to be used. Since the use of a layout object limits the variation
in the ad-hoc styles, it's a step in the direction of greater consistency. However, it doesn't guarantee a consistent look, since a developer might pick the wrong
values from the layout. This is just another manifestation of the trade-off problem, which does not have a perfect solution.

## Breaking out components

If we know that email fields should have the same HTML structure everywhere in the application, then we can extract the HTML structure into an `EmailField` component function. This reduces our ability to make local changes, because when the component function is changed then the email field will be updated everywhere in the application. However, the fact that we are using CSS classes for intrinsic styles helps to mitigate this loss of flexibility. For example, we can target `FormField__Input` elements that appear on an `AuthPage`, and make them look different from `FormField__Input` elements on the `UserProfilePage`. We have to keep an eye on complexity though. When it becomes complicated to get the right look and behavior for a component in a particular context, then it's probably better to use dedicated component functions. For example, we could create a `AuthEmailField` component that uses a `AuthEmailField__Input` CSS class, and a `UserProfileEmailField` component that uses a `UserProfileEmailField__Input`. If `AuthEmailField` and `UserProfileEmailField` have duplicated logic, then this logic can be moved into reusable hooks.

## A repeating challenge

It becomes clear now that we have to solve the trade-off problem many times, in many different contexts. We must decide whether to extract certain HTML elements into components, or leave them inline. Also, we have to decide whether we want a single component (e.g. `EmailFormField`) that can be used in multiple contexts, or if we should use a different component function in each context (e.g. `AuthEmailField` and `UserProfileEmailField`). This seems like a headache, but there is also good news: since we are not locked into a particular approach, we can choose the best solution for each context. We can start out by inlining our styles, which gives us a lot of freedom and control. Then, in particular cases, we can decide to extract certain parts into CSS classes, layouts and components. If we're not happy then we can revert those changes. In my opinion, this is better than using a component library such as Material UI, where all UI elements are boxed in by default, and where we might be forced to break into these boxes if we want more flexibility.
