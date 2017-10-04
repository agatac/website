---
title: "Glimmer.js Progress Report: October 2017"
author: Tom Dale
tags: Recent Posts
---

At EmberConf in March of this year, [we announced Glimmer.js][ann-glimmer-js], a library for
building modern UI components for the web. I wanted to give an update on what
we've been working on since then.

[ann-glimmer-js]: https://emberjs.com/blog/2017/04/05/emberconf-2017-state-of-the-union.html#toc_introducing-glimmer-js 

There were two primary motivations for releasing Glimmer.js as a standalone project:

1. We wanted people who aren't "all in" on Ember to have a way to incrementally
   adopt part of the framework.
2. We wanted a laboratory where we could freely run experiments on what a
   next-generation component library might look like, without creating churn for
   Ember users.

Because Ember's rendering layer is built on top of the shared Glimmer VM, our
intent has always been that successful experiments make their way upstream to
Ember users. And, once stabilized, we'd like Glimmer.js to be the
default component API for new Ember apps—but it's still too premature to set any timelines for that today.

## Unlocking Experimentation

While Ember is known for incorporating the best ideas from across the JavaScript
ecosystem, it's important that we contribute back our own innovations, too.
Refining new ideas takes time and, often, a few false starts. How do we
reconcile the need to experiment with Ember's vaunted stability guarantees?

One of the themes we discussed in our keynote was a new focus on _unlocking
experimentation_, that is, allowing people to easily try out and share unproven
ideas on top of the stable Ember core.

Unlocking experimentation doesn't just allow for _more_ ideas; it also leads to
_better_ ideas, because you can refine them before worrying about
backwards compatibility. With Glimmer.js, we wanted to eat our own experimental
dogfood.

We've wanted to overhaul the component API in Ember for some time now. But
because components play such a central role, we knew that we _had_ to have a
tangible implementation for people to play with before we could credibly ask
them to comment on an RFC. And we knew that having an implementation would
almost certainly shake out design issues that would lead to API changes.
Glimmer.js is our way to iterate on an implementation until we have something we
can feel confident submitting to the Ember RFC process.

Below, I'd like to share with you some updates on the work we've been doing since the initial release.

We've already received incredibly useful feedback from early adopters. Chad Hietala and I have also been on a team at
LinkedIn using Glimmer.js to build a production application.

[To quote DHH](http://david.heinemeierhansson.com/posts/6-why-theres-no-rails-inc):

> The best frameworks are in my opinion extracted, not envisioned. And the best
> way to extract is first to actually do.

If you notice that a lot of the recent work is performance-related, that is at
least partly due to our product's near-maniacal focus on mobile performance. We
are extremely excited about some of the recent breakthroughs we've made and have enjoyed proving
out some of the more esoteric ideas in a real app.

## What's New in Glimmer

### `<my-component />` → `<Component />`

One of the most eagerly-awaited features of Glimmer.js is "angle bracket
components," or components that you invoke `<like-this />` instead of
`{{like-this}}`. I personally really like this syntax because it visually
disambiguates components from the dynamic data that flows through them.
It also unifies the attribute syntax between HTML and components:

```hbs
{{!-- curly components --!}}
{{my-button title=title label=(t "Do Something")}}
{{!-- html --!}}
<button title={{title}} label={{t "Do Something"}}></button>
{{!-- angle bracket components --!}}
<my-button @title={{title}} @label={{t "Do Something"}} />
```

This syntax is how Glimmer.js works today. However, the more we discussed the
design and started using it in real projects, the more we believed that this
exact API was flawed and needed to be rethought.

When we introduced components to Ember, to align with the then-new [Custom
Elements spec][custom-elements], we required components to include a dash (`-`)
in their name.

[custom-elements]: https://www.w3.org/TR/custom-elements/#custom-element-conformance

Web Components have the problem of needing to disambiguate between built-in elements
and custom elements. What happens if I make a component called `<vr></vr>` and later
the browser adds a built-in Virtual Reality element with the same name?

The compromise was that custom elements must have a dash, keeping single-word
elements reserved for iterations of the HTML standard.

Ember components don't have the same problem, because they use `{{`/`}}` as
delimeters instead of `<`/`>`. At the time, we preemptively adopted this constraint because
we assumed Web Components were going to take the world by storm and at some
point we would need to migrate Ember components to Web Components.

As time has passed, though, it's become increasingly clear that the use cases
served by Web Components, wonderful as they are, do not have the full set of
functionality to replace everything that an Ember component (or React component,
etc.) needs to do.

Meanwhile, as we write more and more components, the two-word constraint only
becomes more grating. I hate the cognitive overhead of having to invent silly
names like `{{x-button}}` when a single word would be much better.

This confluence of factors puts us in a bit of a pickle:

1. We want to drop the annoying `dasherized-component` requirement.
2. We want to adopt `<angle-brackets>` syntax for components, but that puts us into
   the same compatibility trap Web Components was trying to avoid.
3. People are starting to want to use _actual_ Web Components, so how do we know
   if `<my-button>` means "create a custom element" or "create a Glimmer component"?

We've circled around different designs for *months*, and this topic has
dominated both our weekly calls and our in-person meetings, with various
proposals and counter-proposals.

At the most recent in-person meeting, we reached consensus on a proposal that
I'm really excited about.

How do you disambiguate between Glimmer components and HTML elements? Our
proposal is to borrow the same rule that React uses: *components always start
with a capital letter*.

Our above example turns into this:

```hbs
{{!-- new angle bracket components --!}}
<Button @title={{title}} @label={{t "Do Something"}} />
```

I love this for a few different reasons.

First, to me the capital letter makes components _really_ stand out in the
template, and we can improve syntax highlighting in editor plugins to make it
stand out even more. It also makes it clear when you're invoking a Web Component
or not, whereas the original Glimmer.js syntax was ambiguous.

Second, for better or worse, many people consider React to be an "industry
standard" and aligning component naming makes Glimmer templates feel that much
more familiar. (Although do note that we are just adopting the naming
convention, not JSX itself! While component invocation will look similar, JSX
will not work in a Glimmer template.)

This change also helps us solve the problem of "fragment" or "tagless" components, i.e.,
templates that don't have a single root element. We were always nervous about
the potential for confusion if you typed something like `<my-button />` in a
template, which looks like a custom element, and it sometimes had a root element
in the DOM and sometimes did not.

Today in Glimmer.js, it is a compile-time error if your component template
doesn't have a single root element. With `<Capital>` components, we will remove
this restriction and allow you to have whatever you want in your template.

One side-effect of this is that we will replace `this.element` (which today is a
reference to the component's root element) with `this.bounds.firstNode` and
`this.bounds.lastNode`, allowing you to traverse the range of DOM nodes
belonging to your component.

### Portals

We've made the built-in `{{in-element}}` helper public API. This helper will
render the block you pass to it inside a foreign element. (In React-land, this
is usually referred to as a portal and as of React 16 is included by default
in `react-dom`.)

### Binary Templates

It's crucial that web apps render instantly, or else users go elsewhere. When it
comes to improving web performance, one of the most frequent recommendations
you'll hear is to minimize the amount of total JavaScript in your app.

There are two reasons for this: not only does more JavaScript take longer to
download, just _parsing_ the JavaScript can become a noticeable bottleneck on
underpowered devices.

Complicating the advice to "use less JavaScript" is the fact that most modern
JavaScript libraries, including Angular, React, Vue and Svelte, compile
component templates (or JSX) to JavaScript that gets embedded in application
code. Without aggressive hand optimization, more templates means more
JavaScript.

Ember used to take a similar approach, compiling Handlebars templates into
JavaScript code that would first create and then update a component's DOM tree.

With Glimmer, however, we took a different approach. Instead of generating
JavaScript, we compile templates into a JSON data structure of "opcodes," or
rendering instructions. A small runtime evaluates these opcodes, translating them
into DOM creation, DOM updates, invocation of component hooks, etc.

Not only is [the JSON parser much faster than the full-blown JavaScript
parser](https://jsperf.com/json-parse-vs-eval-corrected/1), aggressively sharing
code in the Glimmer VM generates less on-device memory pressure and allows
JavaScript engines like V8 to more quickly generate JIT-optimized code. Best of
all, our compact JSON format is significantly smaller than the equivalent
compiled JavaScript. We received many reports of apps dropping 30-50% in total
(post-gzip!) size after upgrading to Ember 2.10, the first version to use this
JSON-based approach.

As exciting as this was, we knew that JSON was not the final word in compactly
and efficiently representing compiled templates.

At runtime, Glimmer VM would assemble each template's JSON "wire format" and
compile them into a final, internal representation that was just a large array
of 32-bit integers. After looking at traces of real-world Glimmer.js apps, we
knew we could improve boot times by precomputing this final compilation step at
build time.

Helpfully, browsers have become increasingly fluent at dealing with binary data,
largely driven by demanding multimedia use cases like audio, video, and 3D
graphics. And while JSON is fast to parse, as the old saw goes, no parse is faster
than no parse. What if we could serialize compiled templates into a binary format
that the VM could start executing _without a parse step_?

I'm no M. Night Shyamalan, so you've probably already guessed the ending here.
That's exactly what we've done. Recent versions of Glimmer VM include the
`@glimmer/bundle-compiler` package, our name for the compiler that produces a
binary "bundle" of all of your compiled templates.

We are planning to land support for binary templates as an opt-in in Glimmer.js
soon. (The feature is already landed in the low-level Glimmer VM but is not yet
exposed in a convenient way.)

One thing to note about the bundle compiler is that it requires knowing your
entire program statically at build time. The browser tends to be a pretty
dynamic environment, however, so Glimmer VM still supports "lazy compilation"
(i.e. compiling to JSON) as a first-class mode.

In the Ember ecosystem, apps and addons do very dynamic things (like register
components at runtime) which are incompatible with the bundle compiler. We want
to enable binary templates in Ember, but this is farther out because we will
need to figure out exactly what the constraints are and provide guidance for
app and addon authors.

In exchange for the (admittedly pretty incredible) performance benefits, binary
templates also introduce extra complexity.

Binary templates can't be inlined in HTML or JavaScript, so they must be fetched
as early as possible in the page lifecycle. No browser I tested yet supports
`<link rel="preload" as="fetch">`, which would allow a streaming HTML parser to
detect and fetch binary data very early in the page load. No tools or CDN know
what the heck a `.gbx` file is and require manual configuration. You probably
want H2 Push for these, but that's its own can of worms. Getting these optimally
deployed will probably suck for awhile, but I have faith that the Ember
community will do what it does best and rally around a set of shared, high
quality solutions for dealing with this.
