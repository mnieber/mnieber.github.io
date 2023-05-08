---
layout: post
title: 'A styling system based on inline styles and SCSS'
date: 2023-05-07 13:45:59 +0100
categories: programming css scss react
---

# A styling system based on inline styles and SCSS

Note that this post is still in draft. I would like to collect feedback so that I can improve it.

## Introduction: the problem with component libraries

Many developers opt for component libraries like MaterialUI as a foundation for their applications. Such libraries enable rapid user interface development while ensuring a consistent appearance. However, this convenience comes at the cost of limited customization options for the components. Consequently, the benefits of component libraries present a trade-off: either relinquish control over the customization of specific elements or resort to unconventional and hacky code to circumvent library constraints. Choosing limited customization hampers the potential of your UI, while settling for hacky code compromises code quality. This, in turn, indirectly affects the UI quality, as maintaining and extending the code becomes more challenging.

## Outline of this post

The problems with component libraries are explained in an excellent video by Theo from T3 (https://www.youtube.com/watch?v=CQuTF-bkOgc). Before proceeding, I highly recommend watching this video, as I'll be building on his advice to employ inline styles and create a custom component library for your application.

In the rest of this post, I will:

- Examine the benefits and limitations of inline styles;
- Explain why using inline styles doesn't conflict with creating a custom component library;
- Introduce my own strategy for managing styles.

I'll be discussing these topics within the context of React, as it is my preferred framework for building UIs. However, the concepts presented here are not exclusive to React and can be adapted for other frameworks as well.

## The benefits and drawbacks of inline styles

To understand the advantages and disadvantages of inline styles, it's helpful to examine the approach employed by TailwindUI. In TailwindUI, each component consists of an HTML code snippet with inline styles. To utilize a component, simply copy the snippet to your source file and modify it as required. This could result in multiple instances of the snippet in your source code. Here's an example of a snippet for an email form field:

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

This method offers significant flexibility, as every component instance can be tailored to suit its specific context. Some additional benefits include:

- Ease of locating an element's styles, as they are directly embedded in the HTML code;
- No need to devise suitable CSS class names;
- When you adjust the instance, there is no risk of breaking the appearance of other component instances.

Now, let's examine the impact of using inline styles on consistency. While copying from an HTML snippet does not enforce consistency, it encourages it since all form fields originate from the same template. However, maintaining consistency while altering inline styles requires discipline from the developer. Some other drawbacks include:

- When updating a component's appearance throughout the application, you may need to modify inline styles in multiple locations.
- Inline styles can appear cluttered and may not convey an intended purpose to the code reader.

Personally, I appreciate being able to examine a piece of code and confirm its correctness. This becomes challenging when faced with an extensive list of inline styles. Although a misplaced or missing style isn't catastrophic and noticeable UI issues will eventually be discovered, this situation can be psychologically stressful.

## Why inline styles and a custom component library are compatible

Utilizing inline styles doesn't restrict you from extracting frequently used styles or HTML structures into reusable components. When doing so, you trade some flexibility for increased consistency and reusability. However, it may seem counterintuitive to recommend against generic component libraries while building your own. The reason why this is not contradictory is that a custom component library is specifically tailored to your application's needs.

Unlike a generic component library, a custom one will inherently align with your application's appearance. Additionally, accommodating special cases becomes much simpler since you have full control over the code. For instance, if extracting a particular component proves difficult, you can choose to maintain it as a snippet. This flexibility in a custom component library allows you to strike the right balance between reusability and adaptability, ensuring a consistent and effective UI.

## My styling system

I will now explain how I base my own styling system on both inline styles, React components and SCSS.

## Breaking out intrinsic styles

As discussed earlier, inline styles are typically copied from a snippet and adjusted to suit the local context. However, there is often a subset of styles that remain unchanged. These can be referred to as the component's intrinsic styles. In my view, it's preferable to move these styles to a CSS class, reducing code clutter. Let's examine an example where this principle is applied. In this instance, the email-field code snippet has been transferred to a TypeScript function, and the intrinsic styles have been relocated to an SCSS file.

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

It's important to recognize that there are no rigid guidelines for determining which properties are intrinsic. For instance, if not all form labels share the same text color then the text-gray-900 style should remain inline. In most cases, the reader can intuitively discern why certain styles are inline while others are not. In less obvious situations, such as when a margin appears in the CSS file, it's helpful to include a comment explaining why the style is considered to be intrinsic.

## Ad-hoc properties and layouts

I refer to the properties that remain inline as ad-hoc properties. These typically involve the component's layout and geometry, including flex-box properties, margins, heights, widths, and paddings.

While ad-hoc properties can be adjusted to fit a specific context, it's good for consistency to limit the variation in these properties. For instance, if an email field has a vertical padding of 1.5, it's likely that the password field in the same form should also have this padding. To address this, I extract the ad-hoc properties and place them in a layout object.

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

As indicated by the comments, it's not mandatory to use all the keys in the layout object. For example, the `label` key is only utilized if the form-field includes a label. It's also possible to incorporate variations in the layout, as demonstrated below:

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

I typically include numerous comments in the layout object to clarify the intended use of each layout. Keep in mind that a developer might choose the wrong values from the layout, resulting in an inconsistent appearance. However, since the use of a layout object constrains the variation in ad-hoc styles, it still promotes greater consistency.

## Breaking out components

If we know that email fields should have the same HTML structure throughout the application, then we can extract this structure into an `EmailField` component function. This approach reduces our ability to make local adjustments, as changes to the component function will affect email fields across the entire application. However, utilizing CSS classes for intrinsic styles helps alleviate this loss of flexibility. For instance, we can target `FormField__Input` elements on an `AuthPage` and differentiate them from `FormField__Input` elements on the `UserProfilePage`.

Nonetheless, it's crucial to keep an eye on complexity. If achieving the desired appearance and behavior for a component in a specific context becomes challenging, then it might be more suitable to use dedicated component functions. For example, we could create an `AuthEmailField` component that uses an `AuthEmailField__Input` CSS class and a `UserProfileEmailField` component that uses a `UserProfileEmailField__Input` class. If `AuthEmailField` and `UserProfileEmailField` share duplicated logic, then this logic can be moved into reusable hooks.

## A repeating challenge

It should be clear now that the trade-off between consistency, code reuse and flexibility must be addressed in numerous contexts. We need to decide whether to extract specific HTML elements into components or leave them inline. Additionally, we must determine whether to use a single component (e.g., EmailFormField) applicable in multiple contexts or implement a different component function for each context (e.g., AuthEmailField and UserProfileEmailField). While this may seem daunting, there is a silver lining: since we aren't confined to a particular approach, we can select the most suitable solution for each context.

Initially, we can opt for inlining our styles, providing greater freedom and control. In specific cases, we may choose to extract certain aspects into CSS classes, layouts, and components. If we find the results unsatisfactory, we can always revert those changes. In my opinion, this approach is more favorable than utilizing a component library like Material UI, where all UI elements are restricted by default, and we might need to circumvent these constraints to achieve more flexibility.
