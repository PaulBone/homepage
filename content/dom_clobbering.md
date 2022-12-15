Title: DOM Clobbering
Author: Frederik Braun
Date: 2022-12-12

*This article [first appeared on the HTMLHell Advent Calendar 2022](https://www.htmhell.dev/adventcalendar/2022/12/).*

### Motivation

When thinking of HTML-related security bugs, people often think of script injection attacks, which is also known as [Cross-Site Scripting](https://en.wikipedia.org/wiki/Cross-site_scripting) (XSS). If an attacker is able to submit, modify or store content on your web page, they might include evil JavaScript code to modify the page or steal user information like cookies.
Most developers out there protect their websites against XSS by disallowing or controlling script execution.

However, this article is **NOT** about XSS.

This article is about an interesting attack that relies on HTML-only injections. The techniques here were [first described by Gareth Heyes in 2013](http://www.thespanner.co.uk/2013/05/16/dom-clobbering/), the topic gained additional attention due to a [recent research paper](https://publications.cispa.saarland/3756/) and its accompanying website [https://domclob.xyz/](https://domclob.xyz/) from Khodayari et al.

### Background

Before we dive into the vulnerability class of DOM Clobbering, we need to establish some background.

Have you ever noticed that whenever you include a HTML element with an `id` attribute, it becomes available as a global name (and a name on the `window` object)? So, for example

```html
<form id=freddy>
```

…will lead to `window.freddy` becoming a reference to that form element.

This is a legacy inheritance from the very old days of HTML and also known as *[named property access](https://html.spec.whatwg.org/multipage/nav-history-apis.html#named-access-on-the-window-object)* in the HTML spec. Studying the spec will inform us that the following elements will also appear on the `document` object: `embed`, `form`, `img`, and `object`.

Given that web browsers have to maintain backwards compatibility support for relatively old web pages, *named property access* is designed in such a way [that element references come before lookups of APIs](https://webidl.spec.whatwg.org/#legacy-platform-object-abstract-ops) and other new attributes on the `window` and `document` object.

### DOM Clobbering Attacks

In essence, this means that an attacker with the ability to inject HTML may mess with simple Web APIs - like the property `document.hidden`:

Imagine your website is trying to reduce or stop animations based on page visibility like so:

```js
if (!document.hidden) {
  startAnimations(); 
}
```

an injected HTML form element will be able to make this truthy, regardless of whether the page is really hidden:

With `<form id=hidden>` the value `document.hidden` would return a `HTMLFormElement` and therefore be considered truthy.

So, in essence, DOM Clobbering is an attack where injected HTML may confuse the existing JavaScript application and therefore change the application logic.

Another typical example of this vulnerability is where code is trying to fall back to a default config if some variable is set, like so:

```js
let config = defaultConfig || { /* some config here */};
```

If the `defaultConfig` name is clobbered, that other config is never going to be assigned correctly!

Let's look at a third, more complicated example: In this case, an attacker has injected two input elements with a `id` attribute of `childNodes` into an existing form `f` like here:

```html
<form id=f>
  <input id=childNodes>foo
  <input id=childNodes>bar
  <input type=text value="hidden-from-js">
</form>
```

These injected input elements will overwrite the `form` element's property and all following JavaScript code will **not** be able to iterate over `f.childNodes`!

Compare the clobbered property with the actual children in this screenshot below:

![Screenshot comparing devtools results for childNodes versus children property](/images/dom_clobbering.png)

Do you see the difference in length?

An attack vector like this one could be used to fool or confuse the website's JavaScript code when iterating over attacker-controlled DOM trees.

Just imagine what else could be overridden!

How would your web page behave if seemingly innocent function calls like `document.getElementById()` can fail with "Uncaught TypeError: `document.getElementById` is not a function" because the API has also been clobbered

### Prevention

The best way to prevent DOM Clobbering is to use a strong [Sanitizer](https://en.wikipedia.org/wiki/HTML_sanitization) library. A sanitizer typically goes through user-supplied HTML and only leaves the harmless bits - removing scripts, event handlers and other injections - like DOM Clobbering.

The author of this article recommends [DOMPurify](https://github.com/cure53/DOMPurify/). By default, DOMPurify will remove all clobbering properties. Using the option `SANITIZE_NAMED_PROPS: true` will instead prefix user-supplied identifiers with `user-content-`.

However, if you want to live on the edge you might also want to look at the upcoming [Sanitizer API](https://wicg.github.io/sanitizer-api/). The specification for the Sanitizer API is still in development, but Chrome and Firefox are already shipping a prototype and the [Sanitizer API Playground](https://sanitizer-api.dev/) has instructions on how to enable it.

In order to fix DOM Clobbering with the Sanitizer API, you need to configure the Sanitizer to disallow attributes of type `name` and `id`:

```js
const mySanitizer = new Sanitizer({
  blockAttributes: [
    {'name': 'id', elements: '*'},
    {'name': 'name', elements: '*'}
  ]
});
someElement.setHTML(input, {sanitizer: mySanitizer});
```

Good luck!

