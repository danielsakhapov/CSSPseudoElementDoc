# Designing the CSSPseudoElement IDL Interface. Open questions.

**Abstract:** This document provides a number of questions for discussion on the design considerations for the [CSSPseudoElement IDL interface](https://drafts.csswg.org/css-pseudo-4/#CSSPseudoElement-interface), as specified in the CSS Pseudo Elements Module Level 4. It addresses concerns regarding object lifetime, support for various pseudo element types (tree-abiding, non-tree-abiding, parameterized, plural), existence checks, event handling, web compatibility.

## Table of Contents
- [1. Introduction.](#1-introduction)
- [2. CSSPseudoElement Interface and Use Cases.](#2-csspseudoelement-interface-and-use-cases)
- [3. Future Extensions.](#3-future-extensions)
  - [3.1. Expanding supported pseudo-element types.](#31-expanding-supported-pseudo-element-types)
  - [3.2. Geometry methods.](#32-geometry-methods)
  - [3.3. Computed style access.](#33-computed-style-access)
  - [3.4. Web Animations API.](#34-web-animations-api)
- [4. Object Lifetime and Nature: Proxy vs. Direct Representation.](#4-object-lifetime-and-nature-proxy-vs-direct-representation)
  - [4.1. Existence Confusion.](#41-existence-confusion)
  - [4.2. Elements without pseudos.](#42-elements-without-pseudos)
- [5. Handling pseudo element variations.](#5-handling-pseudo-element-variations)
  - [5.1. Non-tree-abiding pseudo elements.](#51-non-tree-abiding-pseudo-elements)
  - [5.2 Parameterized pseudo elements.](#52-parameterized-pseudo-elements)
  - [5.3 Plurality: multiple pseudo elements of the same "kind".](#53-plurality-multiple-pseudo-elements-of-the-same-kind)
- [6. Event handling.](#6-event-handling)
  - [6.1. addEventListener on a Proxy/Handle.](#61-addeventlistener-on-a-proxyhandle)

## 1. Introduction.
[The CSSPseudoElement IDL interface](https://drafts.csswg.org/css-pseudo-4/#CSSPseudoElement-interface) aims to provide web developers with a mechanism to interact with pseudo elements (e.g., `::before`, `::scroll-marker`) via JavaScript. This can be used to e.g. allow for querying their computed styles, using them in animations, event handling. The primary entry point is `Element.pseudo(type)`.

## 2. CSSPseudoElement Interface and Use Cases.

`CSSPseudoElement` is the JavaScript interface for representing a pseudo-element. It is returned from `Element.pseudo(type)`, where `type` is currently one of: `::after`, `::before`, `::marker`. It acts as a proxy object — unlike a CSS pseudo-element, a `CSSPseudoElement` handle always exists once obtained, regardless of whether the pseudo-element is currently rendered.

### Interface overview

`CSSPseudoElement` exposes the following attributes and methods:

- `type` — a string representing the type of the pseudo-element (e.g. `"::before"`).
- `element` — the ultimate originating element of the pseudo-element.
- `parent` — the immediate originating element: either an `Element` or a `CSSPseudoElement` for nested pseudo-elements.
- `pseudo(type)` — a method to retrieve nested pseudo-elements.
- `selectorText` — *(resolved in [#12161](https://github.com/w3c/csswg-drafts/issues/12161))* the full, normalized selector string identifying this pseudo-element (e.g. `"::scroll-button(left)"`).

### Use cases

Some scenarios where `CSSPseudoElement` (together with `event.pseudoTarget` and other future extensions) unlocks new capabilities:

**Event-driven interactions:**

- `::scroll-marker` analytics — track which carousel pages are most visited by listening for clicks on each scroll marker.
```js
document.querySelectorAll('.carousel-item').forEach(item => {
  item.addEventListener('click', (event) => {
    if (event.pseudoTarget?.type === '::scroll-marker') {
      analytics.track('scroll-marker-click', { item: item.id });
    }
  });
});
```

- `::scroll-button` custom behaviour — detect when a scroll button is clicked to implement infinite-loop detection or custom scroll-speed logic.
```js
carousel.addEventListener('click', (event) => {
  if (event.pseudoTarget?.selectorText === '::scroll-button(inline-end)') {
    if (isAtEnd(carousel)) startLoop(carousel);
  }
});
```

- `::view-transition-*` interaction — cancel an in-progress view transition when its outgoing snapshot is clicked.
```js
document.addEventListener('click', (event) => {
  if (event.pseudoTarget?.type === '::view-transition-old') {
    document.startViewTransition(() => {}).skipTransition();
  }
});
```

- `::marker` fold/unfold — collapse a list item when its marker bullet is clicked.
```js
listItem.addEventListener('click', (event) => {
  if (event.pseudoTarget?.type === '::marker') {
    listItem.classList.toggle('collapsed');
  }
});
```

- `::backdrop` dismiss — close a dialog when its backdrop is clicked, without interfering with clicks inside the dialog content.
```js
dialog.addEventListener('click', (event) => {
  if (event.pseudoTarget?.type === '::backdrop') {
    dialog.close();
  }
});
```

**Animation and geometry:**

- Geometry-aware view transition — once geometry methods land (see [§3.2](#32-geometry-methods)), read the bounding rect of `::view-transition-old` to set a custom animation origin.
```js
document.addEventListener('transitionstart', (event) => {
  if (event.pseudoTarget?.type === '::view-transition-old') {
    const rect = event.pseudoTarget.getBoundingClientRect();
    setTransformOrigin(rect);
  }
});
```

- Web Animations API on a pseudo-element — animate a `::before` decoration directly (see [§3.4](#34-web-animations-api) for the proposed first-class handle support).
```js
// Today, string-based:
el.animate([{ opacity: 0 }, { opacity: 1 }], {
  pseudoElement: '::before',
  duration: 300,
});
```

**Testing and tooling:**

- Assert pseudo-element is rendered — verify that a `::scroll-marker` is generated for a list item.
```js
const marker = listItem.pseudo('::scroll-marker');
// If .exists lands (see §4.1):
assert_true(marker.exists, '::scroll-marker should be rendered');
```

- Polyfill guard — check whether a pseudo-element is natively rendered before activating a JS polyfill.

To support these cases, selected event types are extended with a `pseudoTarget` property, which is either a `CSSPseudoElement` (if the interaction occurred on a pseudo-element) or `null`.

This enables precise information about the event origin — not just that `event.target` (the ultimate originating element) was interacted with, but that specifically e.g. its `::after` was the hit target. Importantly, `event.target` itself is **unchanged**; the event simply carries additional pseudo-element context.

The following event types expose `pseudoTarget`:
- `UIEvent` (e.g. `click`, `keydown`, `focus`)
- `AnimationEvent`
- `TransitionEvent`

> **Note:** `mouseover`, `mouseout`, `mouseenter`, `mouseleave`, and their `pointer*` counterparts are **not yet supported** due to web compatibility concerns.

## 3. Future Extensions.

This section tracks capabilities that are planned or under active discussion for `CSSPseudoElement` but not yet specified or implemented.

### 3.1. Expanding supported pseudo-element types.
[CSSWG issue](https://github.com/w3c/csswg-drafts/issues/13346)

Currently `Element.pseudo(type)` is only specified for `::after`, `::before`, and `::marker`. Expanding the allow-list to other tree-abiding pseudo-elements is the most direct path to more use cases.

**Candidates under discussion:**

| Pseudo-element | Notes |
|---|---|
| `::scroll-marker` | Primary motivating use case for `event.pseudoTarget`. Pending explicit addition ([#13346](https://github.com/w3c/csswg-drafts/issues/13346)). |
| `::scroll-button(*)` | Same rationale as `::scroll-marker`; parameterised, so requires `selectorText` (already resolved, but needs more thought). |
| `::scroll-marker-group` | Tree-abiding container; useful to check existence and get geometry. |
| `::column` | Plural pseudo-element — requires `pseudoAll()` or `PseudoElementObserver` to be useful. |
| `::view-transition-*` | High-value target for geometry access; plural and parameterised. |
| `::backdrop` | Useful for click-to-dismiss patterns on dialogs and popovers. |
| `::interest-hint` | Useful for some additional logic on interaction. |

### 3.2. Geometry methods.
[CSSWG issue](https://github.com/w3c/csswg-drafts/issues/12373)

The CSSOM View spec already includes `CSSPseudoElement` in the `GeometryUtils` mixin, but `getBoundingClientRect()` and `getClientRects()` are not yet surfaced on the interface in practice. Also, the `GeometryUtils` are not quite
finalized yet.

Adding these would enable:
- Positioning custom tooltips or overlays relative to a pseudo-element.
- Computing keyframe values for coordinated `::view-transition-*` animations.
- Measuring `::scroll-marker` positions for custom scroll-progress indicators.
- Detecting whether a `::before` used as a decorative element is within the viewport.

**Proposed addition to `CSSPseudoElement`:**
```webidl
partial interface CSSPseudoElement {
  DOMRectList getClientRects();
  [NewObject] DOMRect getBoundingClientRect();
};
```

### 3.3. Computed style access.

Computed style for pseudo-elements is only accessible today via the **string-based** second parameter of `getComputedStyle`:

```js
// This is the only specified way today:
getComputedStyle(el, '::before').getPropertyValue('content');
```

There is no specified way to pass a `CSSPseudoElement` object as the first argument to `getComputedStyle`. The CSSOM spec defines `getComputedStyle(Element elt, optional CSSOMString? pseudoElt)` — the first parameter must be an `Element`. Future work is about making computed style access ergonomic through the `CSSPseudoElement` handle itself:

- **Object-based access** — `getComputedStyle(el.pseudo('::before'))` passing the handle as the first argument. This is not yet specified.
- **`.exists` / `PseudoElementObserver`** — see [§4.1](#41-existence-confusion) for the open question of how to expose render state.
- **Typed OM** — `el.pseudo('::before').computedStyleMap()` would give access to the CSS Typed OM for pseudo-elements, enabling structured reads and writes without string parsing.

### 3.4. Web Animations API.

The Web Animations API already accepts a pseudo-element target via `KeyframeEffect`'s `pseudoElement` option (as a string). `CSSPseudoElement` should become a first-class target:

```js
// Today (string-based):
new KeyframeEffect(el, keyframes, { pseudoElement: '::before' });

// Future (CSSPseudoElement handle):
const before = el.pseudo('::before');
before.animate(keyframes, options);
// or:
new KeyframeEffect(before, keyframes, options);
```

This would make the handle consistent with how it is used in the events API (`event.pseudoTarget`) and geometry API, reducing the need for two parallel pseudo-addressing mechanisms.

## 4. Object Lifetime and Nature: Proxy vs. Direct Representation.

`Element.pseudo(type)` returns a cached, persistent `CSSPseudoElement` handle for any valid type string — regardless of whether the pseudo element is currently rendered. This stable-identity model provides `===` equality across calls, consistent `getComputedStyle()` behaviour, and a stable target for event handling. The open questions below arise from this design.

### 4.1. Existence Confusion.
[CSSWG issue](https://github.com/w3c/csswg-drafts/issues/12158)

Since `CSSPseudoElement` always exists as an object, developers need a way to determine whether the underlying pseudo element is actually rendered.

**Options:**
* **`getComputedStyle()` inspection** — check `display !== 'none'` / `content !== 'none'`. Indirect, and forces a style computation.
* **`pseudoElement.exists` (nullable boolean)** — returns `true` / `false` / `null` on a per-pseudo basis. `null` for pseudo elements where the notion of "existence" is privacy-sensitive (e.g. `::spelling-error`, `::grammar-error`) or indeterminate.
* **Async `PseudoElementObserver`** — proposed by @noamr as an alternative that avoids forcing synchronous style/layout, similar to `ResizeObserver`, triggering during the style/layout loop.

**Discussion:** There are concerns that a synchronous `.exists` would force style/layout flushes and introduce subtle performance pitfalls. An async observer is preferred for tracking creation/removal of pseudo elements. However, a sync check may still be useful for specific point-in-time queries.

**Recommendation:**
Consider both: an async `PseudoElementObserver` for lifecycle tracking, and a carefully scoped sync `.exists` for point-in-time use cases. Privacy-sensitive pseudo elements should return `null`.

### 4.2. Elements without pseudos.
[CSSWG issue](https://github.com/w3c/csswg-drafts/issues/12159) — **Resolved**

**RESOLVED:** For a valid selector string, `Element.pseudo()` always returns a `CSSPseudoElement` object — even when called on an element that cannot generate that pseudo (e.g. `input.pseudo('::before')`). `null` is reserved exclusively for invalid/unrecognised type strings. [Spec PR](https://github.com/w3c/csswg-drafts/pull/12406).

## 5. Handling pseudo element variations.

### 5.1. Non-tree-abiding pseudo elements.
[CSSWG issue](https://github.com/w3c/csswg-drafts/issues/12160)

Should non-tree-abiding pseudo elements (e.g. `::selection`) be supported by `CSSPseudoElement`, or should they have a separate interface? This question was reopened after [§6.1](#61-addeventlistener-on-a-proxyhandle) resolved that `CSSPseudoElement` no longer inherits from `EventTarget`.

**Discussion:** A unified interface is prefered for author ergonomics. Mixin-based inheritance tree has been proposed — a `CSSPseudoElement` base class with opt-in mixins such as `CSSTreeAbidingProperties` (`.parent`, `.children`) and `CSSRangeBasedProperties` (for `::selection`), giving precise control over what each pseudo element type can and cannot do.

**Recommendation:**
Pursue a mixin-based approach. Leave `CSSPseudoElement` as a general base and apply mixins per pseudo element type to avoid over-promising capabilities.

### 5.2 Parameterized pseudo elements.
[CSSWG issue](https://github.com/w3c/csswg-drafts/issues/12161) — **Resolved**

How should the argument of a parameterized pseudo element (e.g. `::scroll-button(left)`) be exposed — embedded in `type`, or via a separate attribute?

**RESOLVED:** Add a **`selectorText`** property returning the full, normalized selector string (e.g. `"::scroll-button(left)"`). This gives developers clean, structured access without manual string parsing and round-trips correctly. The `type` attribute continues to return only the base pseudo element name. A [spec PR](https://github.com/w3c/csswg-drafts/pull/13169) is open.

### 5.3 Plurality: multiple pseudo elements of the same "kind".
[CSSWG issue](https://github.com/w3c/csswg-drafts/issues/12162)

`Element.pseudo(type)` assumes a single object per type, but some pseudo elements are inherently plural (e.g. `::column`, `::view-transition-group(*)`).

**Discussion:** The CSSWG initially resolved to add `element.pseudoAll()` returning a list of `CSSPseudoElement`. This was subsequently **reverted** pending more compelling use cases, due to unresolved questions around identity stability when the list grows/shrinks across layout changes, and the interaction with `addEventListener`.

**Recommendation:**
No `pseudoAll()` for now. Revisit once use cases and the identity model for plural pseudo elements are better understood. A `PseudoElementObserver` (see [§4.1](#41-existence-confusion)) may address some of the motivating scenarios.

## 6. Event handling.

Standard user interaction events fired over a pseudo element's area are retargeted to the originating `Element` before dispatch — `event.target` is always the element, never the pseudo element. Existing web content relies on this for event delegation.

### 6.1. `addEventListener` on a Proxy/Handle.
[CSSWG issue](https://github.com/w3c/csswg-drafts/issues/12163) — **Resolved**

The question was how to let developers react to interactions specifically on pseudo elements without breaking `event.target` web compatibility.

**Options considered:**
* Status quo — no direct listening; developers infer from hit coordinates. Poor ergonomics.
* Direct `addEventListener` on `CSSPseudoElement` with a special dispatch phase. Concerns raised about layering (annevk) and the complexity of `mouseover`/`mouseout` boundary events.
* New event types (`pseudoElementClick` etc.) — dismissed as leading to event proliferation.
* **`event.pseudoTarget`** — a new property on selected event types, set to the `CSSPseudoElement` if the interaction originated on a pseudo element, otherwise `null`. Modelled on `KeyframeEffect.pseudoElement` in Web Animations.

**RESOLVED:** `CSSPseudoElement` **does not inherit from `EventTarget`**. Instead, `event.pseudoTarget` is added to a selected, allow-listed set of event types. `event.target` is unchanged — it remains the originating element — so all existing code continues to work unmodified.

**Supported event types:**
- `UIEvent` (e.g. `click`, `keydown`, `focus`)
- `AnimationEvent`
- `TransitionEvent`

`mouseover`, `mouseout`, `mouseenter`, `mouseleave`, and `pointer*` boundary counterparts are **not supported** due to web compatibility risk (they would fire excessively as the pointer moves between the pseudo element and the originating element).
