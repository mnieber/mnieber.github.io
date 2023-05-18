---
layout: post
title: 'A session on CSS styles with ChatGPT'
date: 2023-05-17 09:45:59 +0100
categories: programming chatgpt css
---

In this long post, I'm showing a session I had with ChatGPT. I think it gives an interesting insight into how I use the tool, and how
I approach problem solving in general.

---

## 🔳 User

---

I'm creating a document that describes guidelines for managing CSS styles. My claim is that these guidelines will make it easier to maintain the style rules, and will help the programmer to have control over the styles. Can you read these guidelines and tell me if I am making a strong case? Please tell me if I'm suggesting certain benefits that will not be the case in practice.

Here is the document.

# CSS Guidelines

Writing CSS for an application is complicated for various reasons:

- It can be hard to predict which CSS rules will apply to an element.
- It can be hard to find the code of the CSS rules that apply to an element.
- It can be hard to understand the combined effect of multiple CSS rules.

To mitigate these problems, it's a good idea to use guidelines when writing CSS. In this document I will
propose guidelines. I'm assuming the context of a React application, but the guidelines can be used in other
frameworks as well. Before proposing the guidelines, I will introduce some terminology.

## Terminology

Ad-hoc styles: the styles of a React component that are assumed to vary among instances of that React component.
CSS component: A CSS class that is used to apply multiple CSS rules to a React component or component part.
CSS utility: A CSS class that is used to apply a single CSS rule to an element. For example, `small-font` is a utility class that applies the CSS rule `font-size: 12px;` to an element.
Intrinsic styles: the styles of a React component that are assumed to apply to every instance of that React component.

## Guidelines

### G1. BEM naming is used for CSS components.

### G2. We use css utility classes for ad-hoc styles.

Here is an example of how css utilities are used.

```css
.text-moss-green {
  color: #38a169;
}
```

```tsx
const EmailFormField = () => (
  <div className="EmailFormField FormField">
    {/* The text color is an ad-hoc style of the FormField__Label component. */}
    {/* For the EmailFormField, the label color happens to be moss green. */}
    <label className="FormField__Label text-moss-green">Email</label>
    <input className="FormField__Input" type="email" />
  </div>
);
```

### G3. We use an scss file for the intrinsic styles of a React component.

It's important to note that a certain CSS property may be part of the intrinsic styles of one React component, but part of the ad-hoc styles of another React component. For example, the `color` property may be part of the intrinsic styles of a `Button` component (which means that all buttons have the same color), but part of the ad-hoc styles of a `FormField` component (which means that every instance of a FormField can have a different color).

Here is an example of how intrinsic styles are used.

```scss
.FormField {
  font-size: 12px; // Intrinsic style
}

.FormField__Label {
  margin-bottom: 16px; // Intrinsic style
}

.FormField__Input {
  border: 1px solid black; // Intrinsic style
}
```

```tsx
const EmailFormField = () => (
  {/* Here we apply the FormField CSS component to the entire React component */}
  {/* The FormField CSS component adds the intrinsic styles for a FormField to the EmailFormField */}
  <div className="EmailFormField FormField">
    {/* The FormField__Label CSS component is used for the label part of an EmailFormField. */}
    <label className="FormField__Label text-moss-green">Email</label>
    <input className="FormField__Input" type="email" />
  </div>
);
```

### G4. We avoid overriding the CSS rules of a CSS component.

When we apply a CSS component to a React element, then we want to make it easy to understand which CSS rules apply to that React element.
Ideally, one of the following should be the case:

- The CSS component is defined in only one place.
- The CSS component is defined for different contexts. For every context, it is defined in only one place.

Here is an example of a CSS component where this guideline is followed.

```scss
.c-form-field {
  font-family: 'Roboto', sans-serif;
}

// There is no other rule that also targets .FormField. Therefore, by looking
// at this rule, we know which properties are part of a FormField.
.FormField {
  @apply c-form-field;

  font-size: 12px;
}
```

Here is another example, where the intrinsic styles of a `FormField` depend on the context.

```scss
.c-form-field {
  font-family: 'Roboto', sans-serif;
}

// This rule uniquely describes a FormField in the context of an AuthCard.
.AuthCard .FormField {
  @apply c-form-field;

  font-size: 12px;
}

// This rule uniquely describes a FormField in the context of a UserProfileCard.
.UserProfileCard .FormField {
  @apply c-form-field;

  font-size: 16px;
}
```

Notes:

- The programmer should ensure that the contexts in which a CSS component is used are not overlapping.
  For example, there should be no case of a .FormField that is in a .UserProfileCard that is inside a .AuthCard. If the contexts are
  overlapping, then it means that some CSS properties may be overridden.
- We should not use nesting to define a CSS component in a context. This way, it will be easier to find the places in the code where a CSS component is defined in a certain context.
  For example, to find out where the `.FormField` CSS component is defined in the context of an `AuthCard`, we can search for `.AuthCard .FormField` in the codebase.

---

## 🔳 ChatGPT

---

Your document presents clear, comprehensive guidelines for managing CSS styles and gives useful and illustrative examples. The benefits you claim, such as ease of maintaining style rules and a programmer's control over styles, are well supported by these guidelines. Here are the reasons why:

- BEM naming convention (G1): BEM (Block, Element, Modifier) is a popular naming convention that makes CSS code easier to read and understand. This makes maintenance easier, as it's straightforward to identify which elements the styles correspond to.

- Using CSS utility classes for ad-hoc styles (G2): This is a good practice that aids in reusability and separation of concerns. Each utility class has a specific purpose and is independent, which reduces the risk of unintended side effects when modifying styles.

- Using SCSS for intrinsic styles (G3): SCSS (Sassy CSS) allows the use of variables, nested rules, mixins, etc., which improves code organization and maintainability.

- Avoiding overriding CSS rules (G4): Overriding styles can cause confusion and unexpected behavior. Your guidelines promote a structured approach, preventing complex and unpredictable rule precedence issues that might be hard to debug.

However, there are a few aspects that could be improved for clarity or addressed for better CSS management:

- Specificity Guidelines: CSS specificity can be a tricky concept to handle in large applications. Some guidelines on how to manage CSS specificity would be beneficial.

- CSS Preprocessors: While you have mentioned SCSS, it might be useful to explain why using a preprocessor is beneficial in your guidelines.

- Commenting: You could introduce a standard way to comment CSS/SCSS to explain what blocks of styles do, which would enhance readability and maintainability.

- Modularity: Encouraging the splitting of CSS into smaller, reusable modules (each with a specific purpose) can make your CSS more manageable.

- State-based Design: In a dynamic application, it's likely that elements' styles will change based on their state. Guidelines on how to manage these state-based styles would be a good addition.

- Responsive Design: Guidelines on how to handle styles for different screen sizes and orientations are essential in the current multi-device environment.

Finally, remember that while guidelines are useful, they should be flexible enough to adapt to different projects' needs, as the best practices may vary depending on the specific requirements and constraints.

---

## 🔳 User

---

Thank you! Can you please elaborate on the following points:

1. I believe that the guideline for avoiding to override CSS rules (G4) will help to mitigate problems with CSS specificity. Do you think additional guidelines for dealing CSS specificity are still needed? If so, can you propose some guidelines?
2. Can you propose some guidelines for managing state-based styles?

---

## 🔳 ChatGPT

---

_Guidelines for CSS Specificity_

You're right that Guideline 4 (avoiding overriding CSS rules) does help to manage CSS specificity to an extent, as it discourages creating styles that compete with each other. However, specificity issues can still arise in other circumstances. For instance, if you use CSS selectors of different types (id, class, element, etc.) or when third-party libraries are involved. Here are some additional guidelines to better handle CSS specificity:

G5. Use Classes Instead of IDs: IDs carry a high specificity value which can lead to difficulties when you need to override these styles. Using class selectors, which have a lower specificity, can make the code more maintainable.
G6. Avoid Inline Styles: Inline styles have the highest specificity and can't be overridden by any other selector (except with !important). Avoid them to keep styles manageable.
G7. Be Wary of !important: Using !important is generally considered a bad practice, as it increases specificity to a very high level, making it difficult to override if necessary. It can create a domino effect where !important is used more and more, leading to a codebase that's hard to maintain.
Guidelines for Managing State-Based Styles

In a dynamic application, elements often change their appearance based on their state. You can manage this by creating styles specifically for these state changes. Here are some guidelines:

G8. Use Descriptive Modifier Classes: Similar to the BEM naming convention, you can create modifier classes that describe the state. For instance, if you have a button that's disabled, you could have a .button--disabled class. You can then toggle these classes in your JavaScript code as the state changes.
G9. Use CSS Pseudo-classes: For simpler state changes, like hover effects, you can use CSS pseudo-classes like :hover, :active, :focus, etc. These provide a straightforward way to style state changes without needing to alter the class list in your JavaScript code.
G10. Separate State Styles from Core Styles: Keep the styles that describe the state of an element separate from the styles that describe the element's base appearance. This will make it easier to understand and maintain the styles. For example:

```css
Copy code .button {
  /* core styles */
  padding: 10px 20px;
  font-size: 16px;
}

.button--disabled {
  /* state style */
  opacity: 0.5;
}
```

G11. Keep JavaScript and CSS States Synchronized\*\*: If you're using a JavaScript framework, ensure that changes in your JavaScript state are synchronized with your CSS. For instance, in a React application, use state and props to add or remove classes to your elements when the state changes.
These guidelines, in conjunction with the ones you've already written, should provide a comprehensive guide for managing CSS styles in a maintainable way.

---

## 🔳 User

---

I'm adding the following guideline, can you comment on whether the claimed benefits of this guideline will work well in practice?

### G5: Consider capturing ad-hoc styles in a layout object.

Even though ad-hoc properties may vary per component instance, it's still good to avoid having arbitrary variation in them. This will create a more consistent look.
We can achieve this by moving some of the ad-hoc properties to a layout object. Here is an example.

```tsx
export const layout = {
  FormField: {
    root: 'FormField flex flex-col',
    label: {
      root: 'FormField__Label mb-4',
      color: {
        mossGreen: 'text-moss-green',
        skyBlue: 'text-sky-blue',
      },
    },
    input: 'FormField__Input border border-gray-300 rounded',
  },
};
```

```tsx
const EmailFormField = () => {
  const l = layout.FormField;

  return (
    {/* Here we apply the FormField CSS component to the entire React component */}
    {/* The FormField CSS component adds the intrinsic styles for a FormField to the EmailFormField */}
    <div className={cn("EmailFormField", l.root)}>
      <label className={cn(l.label.root, l.label.color.green)}>Email</label>
      <input className={cn(l.input)} type="email" />
    </div>
  );
};
```

Notes:

- the colors field in layout.FormField.label.color offers a choice of ad-hoc colors for the label. By limiting these choices, we create more consistency, while still allowing
  the color to vary per component instance.

---

## 🔳 ChatGPT

---

The proposed G5 guideline offers an interesting and powerful approach to managing ad-hoc styles in a more structured and consistent way. The benefits of this approach are:

- Consistency: By defining possible variations of ad-hoc styles in a layout object, you limit the degree of freedom that might lead to inconsistency in your UI.

- Reusability and Maintainability: All styles are centralized in a layout object, which is reusable across different components. If a style needs to change, you only need to update it in one place.

- Readability: By extracting styles into a layout object, you also make your components more readable, as they won't be cluttered with numerous class names.

- Ease of Refactoring: If you need to change the design system in the future, having a centralized layout object will make the refactoring easier.

However, be aware of the potential trade-offs:

- Indirection: Developers need to look up the layout object to understand what styles are applied, which could create an extra cognitive load, especially for new developers or larger codebases.

- Complexity: The object structure might become complex and hard to manage if there are many nested layers of styles.

- Reduced Flexibility: While limiting options can increase consistency, it might also lead to a lack of flexibility in some situations. There might be cases where a custom style is needed, and the layout object approach could make such exceptions more difficult to handle.

A balanced approach would be to use this strategy where it makes sense—namely, for commonly used and shared components—while allowing for exceptions when necessary. Always consider the needs of the specific project and the team's preferences and familiarity with this approach.

---

## 🔳 User

---

Thank you for the feedback! Let's consider the scenario in which a FormField has been styled with scss files and a layout object.
The scss files contain the intrinsic styles, and the layout object contains the ad-hoc styles. The FormField can be used in either the
AuthCard context or the UserProfileCard context, as has been illustrated before:

```scss
.c-form-field {
  font-family: 'Roboto', sans-serif;
}

// This rule uniquely describes a FormField in the context of an AuthCard.
.AuthCard .FormField {
  @apply c-form-field;

  font-size: 12px;
}

// This rule uniquely describes a FormField in the context of a UserProfileCard.
.UserProfileCard .FormField {
  @apply c-form-field;

  font-size: 16px;
}
```

It's interesting to consider what should happen when we decide that the font-size must be an ad-hoc property instead
of an intrinsic property. How should we refactor the code to avoid breaking something? The challenge here is that previously
we relied on the cascading nature of style-sheets to correctly style the FormField depending on the context (either
.AuthCard or .UserProfileCard). This means we could create an element with the .FormField class without needing to know the
context of the element. Now that we want to make the font-size an ad-hoc property, we need to know the context of the element,
and apply a font-size of 12px or 16px accordingly.

I propose the following work-flow:

1. Find all instances of `FormField` in the typescript code (i.e. all .ts and .tsx files) and annotate them with a comment: `// ad-hoc: font-size`.
2. For every annotated instance, check if the context of the FormField is known. If it is known, then add the correct font-size as an ad-hoc style.
3. If the context is not known, then we must decide whether to parametrize the component (e.g. add a context attribute that can be either 'auth' or 'user-profile')
   or create a copy of the component for every context. In the latter case, we must name each copy after the context, e.g. AuthFormField and UserProfileFormField.
4. After step 3, the typescript compiler will tell us where we broke the code. In each place, we must ask ourselves the same question as in step 3:
   do we know the context, and if not: do we parametrize or create a copy of the component?
5. After fixing the code in step 4, we can remove the intrinsic style from the scss file, and remove the comments added in step 1.

Do you agree this is a sound analysis of the problem? Do you agree this is a proper way to handle the problem? Do you have any suggestions for improvement?

---

## 🔳 ChatGPT

---

Your approach to refactor the code when transitioning from intrinsic to ad-hoc styling is thoughtful and methodical. It takes into account the necessary steps to prevent any breaking changes and makes sure to fully understand the contexts where the FormField component is used. Here's some feedback on the individual steps and overall approach:

- Annotating instances of FormField: This step is a good way to keep track of all instances of the FormField component in your codebase that need to be refactored. It provides an overview of the task at hand.

- Checking context and adding ad-hoc styles: This step is crucial to correctly refactor the font-size property to be an ad-hoc style. Knowing the context will guide which value the font-size should take.

- Context Decision: The decision here depends heavily on the specific use-cases in your application. If the components vary greatly depending on the context, creating separate components might be more maintainable and straightforward. If the differences are minor, adding a context prop might be a better choice.

- Refactoring according to Typescript compiler feedback: This step ensures that your refactor doesn't break any existing functionality and shows the effectiveness of using a statically typed language.

- Removing intrinsic style: Cleaning up after the refactoring process is complete is essential to avoid confusion and maintain the cleanliness of your codebase.

Some additional thoughts and improvements:

- Consider automating the annotation process if your codebase is large. You can use tooling or scripting to search for instances of FormField and add comments accordingly.
  In step 2, adding the correct font-size as an ad-hoc style might be facilitated by extending the layout object with the font sizes for different contexts. This could potentially simplify the management of these styles.

- When refactoring the FormField component, consider using automated tests (if not already in place) to ensure functionality doesn't break. Unit tests, snapshot tests, and end-to-end tests can be very beneficial in large refactoring tasks.

- Lastly, make sure to communicate these changes to your team. If other developers are working on the same codebase, they should be aware of these changes and why they are made.
  Your approach overall is solid and systematic, addressing the key steps in the process and taking into consideration the decision points and potential pitfalls along the way. With the addition of the points above, you should be well-equipped to handle this transition.
