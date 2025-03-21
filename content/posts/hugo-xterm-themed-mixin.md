+++
title = "SCSS mixin to apply themed colors in web UI design"
description = """How to apply themed colors to web UI components using SCSS
mixin as implemented in hugo-xterm theme."""
date = "2024-04-27T11:52:51+0530"
author = "Mani Kumar"
categories = ["web-development"]
tags = ["scss", "theme", "colors"]
+++

One of the most interesting parts of developing hugo-xterm theme was to
support multiple themed colors and allow them to be changed dynamically by
selecting the theme on dropdown menu.

To achieve this we use a SCSS mixin to provide colors for each theme per UI
component. It can also be achieved using CSS variables and `var()` function
but I have used the SCSS mixin method as it is closer to SCSS/SASS than CSS in
terms of readability.

For this let's follow the simplified steps below:

Define theme specific colors as SCSS variables
----------------------------------------------

Using this naming convention `<component>--<theme>` provide all the theme
specific colors values as SCSS variables example:

```scss {file="_variables.scss"}
/* Can also be RGB, Hex or HSL value */
$accent--light: lightblue;
$background--light: lighten($accent--light, 90%);
$text--light: lighten($accent--light, 40%),
/* more light theme component color variables */

$accent--dark: darkblue;
$background--dark: darken($accent--dark, 80%);
$text--light: darken($accent--light, 10%),
/* more dark theme component color variables */

/* more themes component color variables */
```

Define collection of themes as SCSS map
---------------------------------------

Define all the themed colors in `themes` map so that they can be accessed by
iterating in `themed()` mixin. Each themed color is a pair of component and
its value.

```scss {file="_themes.scss"}
/* Themes */
$themes: (
  light: (
    $accent: $accent--light,
    $background: $background--light,
    $text: $text--light,
    /* more components */
  ),
  dark: (
    $accent: $accent--dark,
    $background: $background--dark,
    $text: $text--dark,
    /* more components */
   ),
  /* more themes */
);
```

Use SCSS global variable to hold current theme
----------------------------------------------

When we iterate through each item in `themes` map we get the theme name i.e.
`light, dark` and its map object that contains the theme specific
component-value pairs i.e. `$accent: $accent--light, $background:
$background--dark,`.

This map object is stored in `$current-theme` global variable at the start of
the iteration and reset at the end so that next theme can be applied to the UI
components.

It is a global variable because it needs to be accessed in both `themed()`
mixin and its helper function `t()` that extracts the component-value pairs.

```scss {file="_themes.scss"}
$current-theme: null;
```

SCSS mixin to apply the themes
------------------------------

First we set a default theme e.g. "light" so that it is applied when nothing
is selected. Additionally we can set a theme for print media.

Next we remove this default theme from the `themes` map as it is already
applied to components. Then we iterate through each remaining themes in
`themes` map and extract component-value pairs using the helper function
`t()`.

```scss {file="_themes.scss"}
@mixin themed() {
  $current-theme: map-get($themes, "light") !global;
  & {
    @content;
  }

  @media print {
   & {
      @content;
    }
  }

  $themes-local: map-remove($themes, "light");
  @each $theme-name, $theme-map in $themes-local {
    .theme--#{$theme-name} & {
      $current-theme: $theme-map !global;
      @content;
      $current-theme: null !global;
    }
  }
}
```

Here `&` is the parent component for which we are applying the themed colors
e.g. `h1` and `@content` is the placeholder for all the properties of this
component which will be replaced by these properties after SCSS compiles to
CSS.

These themes are defined under the theme specific CSS classes with this naming
convention `.theme--<theme-name>` so that they don't clash with each other and
helps to uniquely identify & apply a theme dynamically in JavaScript.

### Example usage of this mixin

To apply all the themed colors to UI component `h1` we use the mixin as below:

```scss {file="_main.scss"}
h1 {
  @include themed {
    background: t($accent);
  }
}
```

It compiles to:

```css {file="main.min.css"}
.theme--light h1 /* some component */ {
  /* component property: CSS theme specific value */
  background: #000;
}

.theme--dark h1 /* some component */ {
  /* component property: CSS theme specific value */
  background: #fff;
}
```

### SCSS helper function `t()` to extract CSS values

To simplify extracting the CSS value of a component in current theme we use
this helper function.

```scss {file="_themes.scss"}
@function t($key) {
  @return map-get($current-theme, $key);
}
```

JavaScript logic to change the theme
------------------------------------

The current implementation in hugo-xterm lacks selecting the theme from a
dropdown menu instead it has theme toggle logic which can be modified as
below:

```javascript {file="theme.js"}
const themeDark = "theme--dark";
const themeLight = "theme--light";
const bodyClassList = document.body.classList;
const themeMenuDropDown = document.querySelector(".theme-menu-dropdown");

themeToggle.addEventListener("click", e => {
  e.stopPropagation();
  const newTheme = getThemeFromClick(e.target);
  if (newTheme !== currentTheme) {
      bodyClassList.remove(currentTheme);
      bodyClassList.add(newTheme);
      localStorage.setItem(preferTheme, newTheme);
  }
});
```

References
----------

This article and the implemented logic in hugo-xterm theme is based on below
article and its implementation.

* [How to Create a Dark Mode in Sass][a1]
* [kaitlinmctigue.github.io][g1]

[a1]: https://medium.com/@katiemctigue/how-to-create-a-dark-mode-in-sass-609f131a3995
[g1]: https://github.com/kaitlinmctigue/kaitlinmctigue.github.io
