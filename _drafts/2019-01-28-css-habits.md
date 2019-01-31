---
layout: post
title:  "CSS habits"
date:   2019-01-24 18:42:59 +0100
categories: CSS Tailwind BEM SMACSS
---
Introduction
------------

Few programmers would say that CSS is their favourite technology, but it has withstood the test of time fairly well, especially since (LESS)[http://lesscss.org/], (SASS)[https://sass-lang.com/] and more recently the (PostCSS)[https://postcss.org/] framework emerged as CSS extensions. Still, writing good CSS is hard. Although I am no expert on this topic, I've spent some time over the past years thinking and learning about strategies for creating CSS. In this article I will summarize my favourite solutions to the following challenges:

- how to keep the set of CSS rules relatively small and lean?
- how to reduce verbosity in CSS rules?
- how to name CSS classes?
- how to organize CSS rules?
- how to create layouts and responsive grids?
- how to integrate CSS into a webpage bundled by Webpack?


Lean CSS
--------

Writing CSS rules is harder when the HTML nesting structure is complex, so it pays off to keep the HTML structure as simple as possible. If you don't need that extra nested `div`: get rid of it.

With a simple enough HTML structure, one can usually find basic CSS rules that produce the correct effect, or almost the correct effect. In the latter case, getting it *exactly* right may require adding a bunch of additional rules. This tweaking tends to produce code that is much harder to read and maintain, or is even hacky. If this happens then I discuss with the graphical designer: would the design still be okay if this button appears slightly lower than originally intended? I understand that small changes *can* make a big visual difference, but in some cases it's not that important. Complecting the CSS carries a cost since predicting the combined effect of various special cases can become overwhelming, so it's not unreasonable to ask the designer for a compromise.

On a related note, I try to avoid adding a rule that corrects a previous rule. This means avoiding `!important`, but it also means asking yourself: why am I required here to move the button an extra 0.25 rems, why does it not "naturally" appear in the correct place? One can often find bugs and conceptual mistakes this way.


Compact CSS
-----------

Compact CSS rules are easier on the eyes, since there is less code to read. I've found (Tailwind)[https://tailwindcss.com] to be great for creating compact CSS. It allows you to build styles with a set of small utility classes which - by using the right prefixes - can be applied selectively depending on page width, hover, focus, etc. Tailwind can be inlined in HTML elements, or used in CSS files via the `@apply` operator:

  {% highlight html %}
  <div class="button--large hover:mx-2">
    Button with horizontal margin of 2 rems if hovered
  </div>
  {% endhighlight %}

  {% highlight css %}
  .button--large {
    @apply rounded; // Large buttons are rounded
  }
  {% endhighlight %}


When styling a HTML element, I make a distinction between intrinsic and incidental style properties. The intrinsic properties should be applied to any instance of the element, and therefore should be described in a CSS file (using `@apply` if possible). The incidental properties such as margins depend on a particular use-case and can often be inlined in the HTML.

Another way to achieve compact CSS is to use SASS mixins. As it's quite obvious how this removes duplication, I will not elaborate on it.

When it comes to limiting the total size of all the CSS files, I've never had to worry too much because the impact on the load time of the webpage has always been small. If I needed to tackle this problem I would follow Tailwind's advise and use (PurgeCSS)[https://www.purgecss.com/].

Naming
------

Since using (BEM)[http://getbem.com/introduction/] for naming, I have not looked back, it just works. A key principle of BEM is that elements are styled with rules starting with the element name. There is no "large" modifier that can be used for buttons *and* badges, instead one uses names such as "checkout-page__button--large" and "badge--large". This can lead to some duplication, but it helps a lot when you want to change a rule without surprising side effects. If you really want to avoid duplication, it's always possible to capture common rules in a SASS mixin.

A known limitation of CSS is the lack of namespaces. One can add a project-specific prefix to all rules, but this extra verbosity is annoying and looks ugly. There is also the option of using (CSS Modules)[https://www.triplet.fi/blog/practical-guide-to-react-and-css-modules/], but I'm uncomfortable with using patterns such as `composes: baseButton from "./base_button.css";` in my CSS, especially when SASS is already handling composition adequately. Moverover, I object to seeing weird identifiers in the CSS when I debug the webpage in a browser. I'm generally willing to adopt a radically different solution, but in this case I think the price outweighs the benefit.

In my recent projects (that were admittedly small) I have avoided the namespacing problem by relying only on Tailwind and custom CSS rules. This has an added benefit: when you start your CSS from scratch, there is no need to customize any rules provided by a framework such as Bootstrap. Avoiding such customizations means having less rules that correct a previous rule.


Organizing CSS rules
--------------------

The first thing to consider is that the website may have distinct parts. For example, a financial app may have a customer onboarding part and a banking part, which may look different. In that case, it makes sense to create two sets of CSS files. Of course, these two sets - although they are separate - will follow the same logical structure. Following the advice offered by the (SMACSS)[https://smacss.com/] framework, I divide the rules in the following layers:

- the **base** layer contains styles for basic elements such as `h1`. I consider Bootstrap elements such as `.card` to be basic elements too, so I would use the `base` layer to override them. Styling basic elements may seem a bad idea: what if the `h1` should look different in some part of the page? However, it has the benefit of emphasizing consistency: all `h1`'s *should* look the same in a consistent design. Moreover, the CSS becomes more compact if you avoid styling `h1` in various CSS classes. If you really need the `h1` to look different somewhere, you can still reset the style locally with ```{ all: unset; }```.
- the **layout** layer contains rules for laying out bigger elements, e.g. the division of the HTML page in header, body and footer.
- the **modules** layer contains the rules for smaller elements that are inserted into layouts. Typically a module will look the same regardless of which layout it's inserted into.

I don't follow the SMACSS guidelines for naming the CSS classes inside modules, nor do I use the state layer or themes layer. These areas are covered more adequately by using BEM and Tailwind utility classes.


Creating layouts and grids
--------------------------

Creating layouts used to require a potpourri of techniques until Flexbox provided a general purpose layout mechanism. I try to use it whenever possible, so that layouts follow the same patterns everywhere. Occasionally though, I will need `position: absolute`.

Tailwind claims that you can use it for responsive grids although it offers no special classes, and I agree. I even like their approach. If you want 3 columns on large screens, just create a `flex` layout with divs that use `lg:w-1/3` and you are done.

I'm still trying to find a good approach for ensuring that each `module` (in SMACSS's terminology) is properly positioned in a layout. I've tried baking margins into the module's CSS class so it consistently maintains the right distance, but this soon runs into problems because the amount of whitespace that looks good usually depends on the context. On the other hand, if you choose the amount of whitespace on an ad-hoc basis then it's harder to create a consistent look. It's also harder to change such a design globally, and introduce more space around all instances of a certain button. As a compromise, I've tried using modifiers such as `checkout-page__button--defaultPos` but this bloats the CSS and HTML.

In the end, I prefer the ad-hoc solution of inlining Tailwind utility classes (such as `ml-2` for a left-side margin) in the HTML. Even then, there are choices to be made. Should you try to make margins symmetrical or prefer left-side and top-side margins? So far, it seems that using left-side and top-side margins gives the best results.

Using a left-side margin or padding to separate items in a multi column layout has a problem though: items in the first column have no left-side neighbour and therefore don't need a margin. This can be remedied using the `nth-child` selector to remove the margin. However, it's not a perfect solution, as this selector cannot be inlined in the HTML via a utility class. In the end, if the logic for avoiding redundant whitespace becomes too complex, and the visual end-result is not affected that much, then I prefer to accept the redundancy.


Integrating CSS into the webpage/bundle
---------------------------------------

Since I use Webpack to bundle the website, the cleanest way to integrate CSS is with a loader:

  {% highlight javascript %}
    // Part of module: { rules: [ ... ] }
    {
      test: /\.css$/,
      use: [
        MiniCssExtractPlugin.loader,
        'css-loader',
      ]
    }
  {% endhighlight %}

The `css-loader` tells Webpack to insert the CSS directly into the bundle. The `MiniCssExtractPlugin.loader` is optional. It extracts the CSS from the bundle and copies it to the output directory; this helps with inspecting the CSS.

SCSS is integrated in a similar way, but uses two extra loaders: 'postcss-loader' (required for Tailwind's `@apply` operator) and 'sass-loader':

  {% highlight javascript %}
    // Part of module: { rules: [ ... ] }
    {
      test: /\.scss$/,
      use: [
        MiniCssExtractPlugin.loader,
        'css-loader',
        'postcss-loader',
        'sass-loader',
      ]
    }
  {% endhighlight %}

The `MiniCssExtractPlugin.loader` works in tandem with the `MiniCssExtractPlugin`. For some reason, using absolute paths in `MiniCssExtractPlugin` doesn't work. Instead, I use a path relative to the default webpack output directory:

  {% highlight javascript %}
  output: {
      path: '/srv/static/bundles',
      filename: "dev-[name]-[hash].js",
      chunkFilename: '[name].bundle.js',
  },
  plugins: [
    new MiniCssExtractPlugin({
      filename: "../css/[name].css",
      chunkFilename: "../css/[id].css"
    })
  ],
  {% endhighlight %}


Conclusion
----------

As I'm still learning about CSS, the advice in this article is quite basic. I advocate keeping the CSS as simple as possible, even if it means making small compromises to the visual design. Since getting the right style used to require a lot of tricks and hacks (think IE6), it's easy to think that hacky or messy CSS code is unavoidable, but I believe we should resist that thought. In some cases, by insisting on clean CSS I managed to throw away half the rules and get the same end-result (or close). In fact, CSS may be the area in the code base that requires the most rigorous weeding out of bad constructs, because that's the only way to keep some control over it.