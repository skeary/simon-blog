---
title: "Extending Tailwind CSS with Vanilla Extract"
description: "Using Vanilla Extract with Tailwind CSS to support class variants and leverage the value of both frameworks."
date: "2022-05-31"
hero: "/images/tailwind-vanilla.jpg"
tags: ["css"]
layout: "../../layouts/BlogPostLayout.astro"
---

[Tailwind CSS](https://tailwindcss.com/) and [Vanilla Extra](https://vanilla-extract.style/) are two great CSS frameworks for styling React websites/applications. In many ways, the frameworks are alternative takes on how to approach effective CSS styling and there are a range of articles you can find on the pros/cons of each framework. With Tailwind a key benefit is the large range of well-thought out and amazingly documented utilitity CSS classes. In many cases you can style UI components by only using the provided class names:

```tsx
<button className="h-10 px-6 font-semibold rounded-md bg-black text-white" type="button">
  Click me
</button>
```

![Click me button](../../../public/images/tailwind-vanilla/click-me-btn.png)

On the other hand, Vanilla Extract provides none of these utility classes, but it perhaps does provide a more configurable and extendable approach for building your own styling framework. In particular, Vanilla Extract provides a simple to use approach for what they describe as [type-safe multi-variant](https://vanilla-extract.style/documentation/recipes-api/) styles, which is something not directly provided by Tailwind.

Rather than using one framework or the other, I've recently been experimenting with using the two together to leverage their relative benefits. This potentially provides the advantages of both frameworks while avoiding some of the potential drawbacks of each. Since I haven't seen this approach used elsewhere, I thought I'd document it in case it's useful to anyone else. In this article I explain how you could do this and why it might be worth considering.


## Tailwind

I think anyone who has used Tailwind appreciates the benefits of the very well-thought out utility classes it provides, and how they can really help developer productivity while keeping control of your CSS. Assuming you have your build environment set up correctly, Tailwind will produce optimal CSS files that only include the classes referenced, and your HTML elements will be rendered with the class string as it looks in the JSX. This means there is no runtime overhead by creating styles dynamically, which some other CSS-in-JS frameworks do, and the browser can process your styles using its highly efficient built-in [CSS parsing](https://twitter.com/slightlylate/status/1517173350467993600).

By using an abstracted sizing system (eg `px-6` for 6 "units" of left and right padding), Tailwind also gently pushes you to keep consistent spacing/sizing of things. You can still use the sizes inconsistently, but generally it seems relatively easy to be consistent. Of course, Tailwind is also very customizable and allows to change many things such as the spacing/sizing scales, if you need.

If you take the example of the button above, you obviously wouldn't want to repeat the same list of utility classes everywhere you want to style a button similarly! There are a number of simple approaches to keep your code DRY such as encapsulating the code within a React component:

```tsx
function Button(
  props: Omit<
    DetailedHTMLProps<
      ButtonHTMLAttributes<HTMLButtonElement>,
      HTMLButtonElement
    >,
    "className"
  >
) {
  return (
    <button
      {...props}
      className="h-10 px-6 font-semibold rounded-md bg-black text-white"
    />
  );
}
```

Another, perhaps even easier approach, is just to extract the className into a constant in an appropriate file and then import the constant wherever a button is needed:


```ts
// styles.ts
export const btnClass = 'h-10 px-6 font-semibold rounded-md bg-black text-white';
```

```tsx
import {btnClass} from 'buttonstyles';

<button className={btnClass} type="button">
  Click me
</button>
```

As a last example, you can use Tailwind's `@apply` directive in CSS to create your own classes, composed of the built-in utility classes:

```css
/* globals.css */
@tailwind base;
@tailwind components;
@tailwind utilities;

@layer components {
  .btn {
    @apply h-10 px-6 font-semibold rounded-md bg-black text-white;
  }
}
```

```tsx
<button className="btn" type="button">
  Click me
</button>
```

These options are all outlined in more detail in the [Tailwind docs](https://tailwindcss.com/docs/reusing-styles).

## Tailwind - A potential complication

The example above is just a simple styled button. Typically though, you probably want different variants of a button style based on its use. For example, the purpose/intent is one potential option - is the button for a primary, secondary or destructive action? You may also want different sizes of button depending on context. This is potentially where it can be a more complex to approach styling using Tailwind in a composable way. Below is a basic approach that works:

```ts
// buttonClass.ts
interface ButtonClassArgs {
  intent?: 'primary' | 'danger';
  size?: 'regular' | 'large'
}

function buttonClass({ intent = 'primary', size = 'regular' }: ButtonClassArgs) {
  const baseBtnClass = 'h-10 px-6 font-semibold rounded-md';
  const primaryBtnClass = ' bg-black text-white';
  const dangerBtnClass = ' bg-red-700 text-white';
  const regularBtnClass = '';
  const largeBtnClass = ' text-2xl';

  let btnClass = baseBtnClass;
  switch (intent) {
    case 'primary':
      btnClass += primaryBtnClass;
      break;
    case 'danger':
      btnClass += dangerBtnClass;
      break;
  }
  switch (size) {
    case 'regular':
      btnClass += regularBtnClass;
      break;
    case 'large':
      btnClass += largeBtnClass;
      break;
  }
  return btnClass;
}
```

The above code generates class strings for buttons with different intents (`primary` or `danger`) and with different size options (`regular` or `large`). It could then be used within a React component like so:

```tsx
<button
  className={buttonClass({ intent: 'danger', size: 'large'})}
  type="button"
>
  Danger!
</button>
```

It works but it doesn't feel particularly streamlined. You also have to make sure you don't try to be to clever with the [class string concatenation](https://medium.com/coding-at-dawn/the-tailwind-css-jit-mode-bug-that-only-happens-in-production-4f25ef009fe8) or in production things won't work.  Perhaps another approach is creating utility classes to with Tailwinds `@apply` directive to avoid the switch statements:


```css
/* globals.css */
@layer components {
  .btn-base {
    @apply h-10 px-6 font-semibold rounded-md;
  }
  .btn-primary {
    @apply bg-black text-white;
  }
  .btn-danger {
    @apply bg-red-700 text-white;
  }
  .btn-regular {
  }
  .btn-large {
    @apply text-2xl
  }
}
```


```ts
// buttonClass.ts
interface ButtonClassArgs {
  intent?: 'btn-primary' | 'btn-danger';
  size?: 'btn-regular' | 'btn-large'
}

function buttonClass({ intent = 'btn-primary', size = 'btn-regular' }: ButtonClassArgs) {
  return `btn-base ${intent} ${size}`;
}
```


```tsx
<button
  className={buttonClass({ intent: 'btn-danger', size: 'btn-large'})}
  type="button"
>
  Danger!
</button>

```

It still seems a bit too much boilerplate and the class names are now (a bit unfortunately) prefixed with "btn-". This isn't strictly needing in this case but might be a good idea to avoid potential class name clashing from using very generic class names (eg "primary").

## Potential solutions

It seems other developers have also wondered if there is a better way and have released some utility libraries to streamline creating variants:

- [classname-variants](https://www.npmjs.com/package/classname-variants)
- [cva](https://github.com/joe-bell/cva)

Both libraries work in a similar way, and essentially give you a streamlined way to generate a function like `buttonClass()` above. Below is an example using classname-variants:


```tsx
import { variants } from 'classname-variants';

const buttonClass = variants({
  base: 'h-10 px-6 font-semibold rounded-md',
  variants: {
    intent: {
      primary: 'bg-black text-white',
      danger: 'bg-red text-white',
    },
    size: {
      regular: 'text-base',
      large: 'text-2xl',
    },
  },
});
```

This is definitely less verbose, while giving you a nice typed interface. If you like this approach, I thikn both libraries are worth looking at and you could even roll your own implementation. Based on some experimentation, I came across another option - using the [Recipes API](https://vanilla-extract.style/documentation/recipes-api/) of the [Vanilla Extract](https://vanilla-extract.style/documentation/recipes-api/) library. While it wasn't intended for this use case, it essentially provides the same functionality, and perhaps has some nice benefits.

Using the Recipies API, the above would become:

```ts
// buttons.css.ts
import { recipe } from '@vanilla-extract/recipes';
export const buttonClass = recipe({
  base: ['h-10 px-6 font-semibold rounded-md'],
  variants: {
    intent: {
      primary: ['bg-black text-white'],
      danger: ['bg-red text-white'],
    },
    size: {
      regular: ['text-base'],
      large: ['text-2xl'],
    },
  },
  defaultVariants: {
    intent: 'primary',
    size: 'regular'
  }
});
```

While the syntax is very similar to the classname-variants syntax, there are probably a couple of things worth noting:

- You can see that rather than just strings, each value is an array. Within a `recipe()` call, as well as specifying a set of CSS class names, you can also build CSS styles in a similar manner to other CSS-in-JS libraries, defining individual style attributes. In fact, this is really the default way to do things if you were using Vanilla by itself. To accommodate the different ways to define styles, class name strings must be enclosed within an array.
- You generally must use `recipe()` (and other Vanilla Extract constructs) within files ending in `.css.ts`. Once your project is configured, these files will be evaluated at **build time** using the Vanilla Extract preprocessor. This allows Vanilla Extract to statically generate CSS classes and names and avoids some of the runtime overhead of other CSS-in-JS libraries.

As hinted in the first bullet point, the `recipe()` function provides a couple of other potential benefits compared to the alternatives I've seen. Firstly, and what I've found the most useful, is it provides a way to incorporate CSS properties/values with class names. For example, say you want to add custom box shadow to your buttons:


```ts
// buttons.css.ts
import { recipe } from '@vanilla-extract/recipes';
export const button = recipe({
  base: [
    'h-10 px-6 font-semibold rounded-md',
    {
      boxShadow: 'rgb(0 0 0 / 12%) 0px 6px 16px !important'
    }
  ],
  variants: {
    intent: {
      primary: ['bg-black text-white'],
      danger: ['bg-red text-white'],
    },
    size: {
      regular: ['text-base'],
      large: ['text-2xl'],
    },
  },
  defaultVariants: {
    intent: 'primary',
    size: 'regular'
  }
});
```

While you could achieve a similar effect by extracting the box shadow CSS into a new global class, putting it next to the other styles helps to keep related styling together.

With recipes you can also specify what Vanilla Extract calls `compoundVariants`. These allow you to specify certain styles or classes if a certain combination of variants (eg both danger and large) are specified. See the [docs](https://vanilla-extract.style/documentation/recipes-api/) for more info.

In addition to `recipe()`, Vanilla Extract also provides a simpler `style()` function which is similar but doesn't have variants functionality. This can sometimes also be a handy addition to Tailwind. Here's an example:

```ts
// button.css.ts
export const simpleButtonStyle = style([
  {
    boxShadow: 'rgb(0 0 0 / 12%) 0px 6px 16px !important'
  },
  'h-10 px-6 font-semibold rounded-md',
  'bg-black text-white'
]);
```

It's worth noting that Vanilla Extract is a popular, well-maintained library and has other functionality that may be of benefit.

## Conclusion

Hopefully this article is useful to anyone else wondering how to implement a class variants approach with Tailwind :) If you found it useful, you can let me know by commenting below. If you think it's a terrible idea or have a better approach, I'd also appreciate the feedack!


__Main image adapted from [Unsplash photo](https://unsplash.com/photos/TYA9f8hYEX4) by Jocelyn Morales__