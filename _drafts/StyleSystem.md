# Addressing the CSS trade-off problem with inline styles

## The trade-off problem: consistency versus flexibility

In this first video, we'll take a look at a particular problem that developers face when creating a UI, which I will call the trade-off problem. The trade-off problem occurs when you try to create an application that looks both consistent and beautiful. To make the application look consistent, it helps to capture commonly used styles and layouts in building blocks that can be reused throughout the application. However, these building blocks introduce rigidity that tends to make the application less beautiful. This is because in a beautiful UI, elements need small adjustments that go against the structure that the building blocks provide. Therefore, there is a trade-off between consistency and reusability on the one hand, and flexibility and beauty on the other.

Note that I will discuss the trade-off problem in the context of React, since that's the framework I use to build UIs. However, the ideas presented here are not specific to React, they can be applied to other frameworks too.

## Different "solutions" to the trade-off problem

There is no real solution to the trade-off problem, but there are different approaches that each have their own strengths and weaknesses. These are very well explained in a video by Theo from T3 (https://www.youtube.com/watch?v=CQuTF-bkOgc). I recommend to check out this video first, since I'm building on the advice it gives, which is to create UI elements using inline styles. I will explain how inline styles address the trade-off problem, and how inline styles are used in my own approach for managing styles.

## Addressing the trade-off problem with inline styles

To understand the strengths and weaknesses of inline styles, it's insightful to take a look at the approach that is offered by the TailwindUI. In TailwindUI, every component is an HTML snippet that contains HTML elements with inline styles. To use a component, you simply copy it to your source file and adjust it as needed. For example, the following snippet shows an email form field:

```html
<div>
  <label for="email" class="block text-sm font-medium leading-6 text-gray-900"
    >Email address</label
  >
  <div class="mt-2">
    <input
      id="email"
      name="email"
      type="email"
      class="block w-full rounded-md border-0 py-1.5 text-gray-900 shadow-sm ring-1 ring-inset ring-gray-300 placeholder:text-gray-400 focus:ring-2 focus:ring-inset focus:ring-indigo-600 sm:text-sm sm:leading-6"
    />
  </div>
</div>
```

This approach offers a lot of flexibility. Every component instance can be adjusted so that it looks good in its particular context. When you adjust the instance, there is no risk of breaking the look of other component instances. Note that the use of inline styles doesn't prevent you from extracting commonly used styles or HTML structures into reusable building blocks. By doing so, we trade of some of the flexibility to have more consistency and reuse. This is what we will be doing later in this video.

Even though copying from a HTML doesn't enforce consistency, it still promotes it, because by using the same snippet for all your form fields you will create a consistent look. The drawback is that you need to be disciplined as a developer to maintain this consistency as you make changes to the inline styles. Also, when you want to change the look of a component everywhere in the application, then you need to update the inline styles in potentially many places.

## Breaking out intrinsic styles

We've seen above that inline styles are copied from a snippet and adjusted to fit the local context. However, there is usually a subset of inline styles that are never changed. I call these the intrinsic styles of the component. In my opinion, it's better to move these styles to a CSS class, so that the code becomes less noisy. If it's truly the case that we are not going to change these properties depending on the local context, then flexibility is not affected too much (in the worst case, we may have to make some of the properties inline again). In our example, this could look as follows:

```html
<div>
  <label for="email" className="FormField__Label">Email address</label>
  <div class="mt-2">
    <input
      id="email"
      name="email"
      type="email"
      className="FormField__Input py-1.5 w-full"
    />
  </div>
</div>
```

```scss
.FormFieldLabel {
  @apply block text-sm font-medium leading-6 text-gray-900;
}

.FormFieldInput {
  @apply block rounded-md border-0 text-gray-900 placeholder:text-gray-400 sm:text-sm sm:leading-6;
  @apply shadow-sm ring-1 ring-inset ring-gray-300 focus:ring-2 focus:ring-inset focus:ring-indigo-600;
}
```

It's important to note that there are no strict rules for picking the properties that are intrinsic. For example, if not all labels in a form should have the same text color then the `text-gray-900` style should remain inline.

### Ad-hoc properties

I use the term ad-hoc properties for those properties that remain inline. Usually, the ad-hoc properties deal with the layout of the component. Typical examples of ad-hoc properties are flex-box properties, margins, heights and widths and paddings (though as mentioned, for some components these properties may be intrinsic).

## Breaking out components

If we know that email fields should have the same HTML structure everywhere in the application, then we can extract the HTML structure into an `EmailField` component function. This reduces flexibility, because when the component function is changed then the email field will be updated everywhere in the application. In other words, we loose the ability to make small isolated changes. However, the fact that we used a CSS class to capture the intrinsic styles allows us to retain some flexibility. For example, we can have specific CSS rules to style a `FormField__Input` that appears on an `AuthPage`, and make it look different from a `FormField__Input` on the `UserProfilePage`. That said, when it becomes complicated to get the right look and behavior for `EmailField` in the authentication context, then it's probably better to create a dedicated `AuthEmailField` component, that uses a `AuthEmailField__Input` CSS class. If doing so means that the logic of the `EmailField` is duplicated, then consider moving that logic into reusable hooks.

## Breaking out layouts

Although ad-hoc properties are intended to be adjusted to suit a particular context, it's still good for consistency to limit the variation in these properties. For example, if the email field has a vertical paddding of 1.5, then probably the email field in the same form should also have this padding.

we want all fields in a form to have a consistent look, which means that the ad-hoc properties are the exception rather than the rule.

```

```
