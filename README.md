# Designing the CSSPseudoElement IDL Interface. Open questions.

**Abstract:** This document provides a number of questions for discussion on the design considerations for the CSSPseudoElement IDL interface, as specified in the CSS Pseudo Elements Module Level 4. It addresses concerns regarding object lifetime, support for various pseudo element types (tree-abiding, non-tree-abiding, parameterized, plural), existence checks, event handling, web compatibility.

## Table of Contents
- [1. Introduction.](#1-introduction)
- [2. Object Lifetime and Nature: Proxy vs. Direct Representation.](#2-object-lifetime-and-nature-proxy-vs-direct-representation)
  - [2.1. Existence Confusion.](#21-existence-confusion)
  - [2.2. Elements without pseudos.](#22-elements-without-pseudos)
- [3. Handling pseudo element variations.](#3-handling-pseudo-element-variations)
  - [3.1. Non-tree-abiding pseudo elements.](#31-non-tree-abiding-pseudo-elements)
  - [3.2 Parameterized pseudo elements.](#32-parameterized-pseudo-elements)
  - [3.3 Plurality: multiple pseudo elements of the same "kind".](#33-plurality-multiple-pseudo-elements-of-the-same-kind)
- [4. Event handling.](#4-event-handling)
  - [4.1. addEventListener on a Proxy/Handle.](#41-addeventlistener-on-a-proxyhandle)

## 1. Introduction.
The CSSPseudoElement IDL interface aims to provide web developers with a mechanism to interact with pseudo elements (e.g., `::before`, `::scroll-marker`, `::selection`) via JavaScript. This can be used to e.g. allow for querying their computed styles, using them in animations, event handling. The primary entry point is `Element.pseudo(type)`.

## 2. Object lifetime and nature: Proxy vs. Direct Representation.
A fundamental question is the nature of the `CSSPseudoElement` object returned by `Element.pseudo(type)`.

Current Spec Guidance says:
"Parse the `type` argument as a `<pseudo-element-selector>`, and let `type` be the result. If `type` is failure, return `null`. Otherwise, return the `CSSPseudoElement` object representing the pseudo element that would match the selector `type` with this as its originating element."

This implies that if the selector syntax for the pseudo element type is valid, an object is returned. It doesn't explicitly state that the pseudo element must be currently generated or visible. The requirement to support `getComputedStyle()` for a highlight pseudo (which might not always be "active," e.g., `::selection` when nothing is selected) further suggests the object can exist independently of the pseudo element's current rendering state.

The following table summarizes possible solutions to the problem to show why the currently specified approach is the best.

| Model                         | Pros                                                                 | Cons                                                                        | Implementation Notes                                                         | Alignment with Spec/Resolutions                                 |
|-------------------------------|----------------------------------------------------------------------|-----------------------------------------------------------------------------|------------------------------------------------------------------------------|-----------------------------------------------------------------|
| A: Cached, Persistent Handle (Current) | Stable identity (`===` works). Simpler GC via weak refs.             | Indirect existence check needed (e.g., `getComputedStyle`). Potential dev confusion. | Requires caching mechanism (element-associated, weak refs). Object state is mostly static. | Aligns best with "same object" rule and #3607 resolution.       |
| B: Live Identity (Object if Rendered) | Direct existence check (`!== null`). More intuitive "live" state?    | Unstable identity if pseudo appears/disappears. Potential object churn.     | Return value depends on current style/layout. Requires checks on pseudo generation. | Conflicts with the "same object" rule. Considered in #3607.   |
| C: New Object if Rendered     | Direct existence check (`!== null`). No caching needed.                | No stable identity (`===` fails). High object churn. Poor developer ergonomics. | Return value depends on style/layout. Creates a new object each time if rendered. | Conflicts strongly with the "same object" rule. Least favored. |

Model A is the currently specified one, and it likely won’t be changed, as it has a number of useful pros:
* **Stable Identity:** Developers can get and hold onto a reference. This avoids scenarios where `Element.pseudo(type) === Element.pseudo(type)` might be `false` if the object was created on demand based on rendering.
* **Consistent API:** Simplifies developer interaction as they don't need to constantly check for `null` just because a pseudo element might temporarily not be rendered (e.g., originating element is `display: none`, or `content: none` for `::before`).
* **Facilitates `getComputedStyle()`:** Aligns with the ability to call `getComputedStyle()` even if the pseudo element isn't "visible" (e.g., for highlight pseudos, or to check what styles would apply).
* **EventTarget Potential:** Allows `addEventListener` to be called on a stable object, even if the pseudo element isn't currently rendered (eventual event firing would depend on actual rendering).

But, nevertheless, it has a bunch of open questions, this explainer would like to address.

### 2.1. Existence confusion.
[CSSWG issue](https://github.com/w3c/csswg-drafts/issues/12158)

Developers might need a clear way to distinguish between having an object and the pseudo element actually being "live" or rendered?

**Possible Solutions:**

* `getComputedStyle()` inspection:
    * Check `getComputedStyle(pseudoElement).display !== 'none'`. For `::before`/`::after`, also check `getComputedStyle(pseudoElement).content !== 'none'`.
    * **Pros:** Uses existing mechanisms.
    * **Cons:** Indirect; might not cover all cases (e.g., a pseudo element that exists but has 0x0 dimensions and no visual content). For highlight pseudos, "display" might not be the relevant property. Sometimes depends on the originating element style.
* A dedicated `exists` property or `isGenerated()` method:
    * `pseudoElement.exists` (nullable boolean, read-only)
    * **Pros:** Explicit and clear, allows to specify the return value on a per pseudo element basis.
    * **Cons:** Adds new API surface. Defining "exists" precisely across all pseudo element types can be tricky (e.g., does `::selection` "exist" if nothing is selected but it could be styled? - potentially return `null` here).
* Layout Information:
    * Methods like a hypothetical `pseudoElement.getBoundingClientRects()` returning an empty list or specific values could indicate non-existence or non-rendering.
    * **Pros:** Provides practical information.
    * **Cons:** Might be expensive to compute just for an existence check.

**Recommendation:**
Go with `exists`, as it gives more control for each pseudo element while unifying an interface to get the needed information.
Otherwise: Relying on `getComputedStyle()` initially seems pragmatic, given the limited list of pseudos supported currently. If specific use cases demonstrate its inadequacy, a dedicated property/method could be considered. For highlight pseudos, "existence" might mean "has associated ranges/segments."

### 2.2. Elements without pseudos.
[CSSWG issue](https://github.com/w3c/csswg-drafts/issues/12159)

What should elements that can’t have pseudos (e.g. `<input>`) return from `pseudo()` function? `null` or `CSSPseudoElement` object with `parent` and `element` as `null`?

* Return `null`
    * **Pros:**
        * Direct Feedback: Immediately signals to the developer that this specific combination of element and pseudo element type is not valid or won't render.
        * Simpler "Existence" Check: `null` means it doesn't exist in this context; no further checks needed on a pseudo-object.
    * **Cons:**
        * Inconsistent API Signature: `Element.pseudo()` would return `null` for two different reasons (invalid pseudo selector string or valid selector on an incompatible element), making error handling more complex.
        * Less Introspection: Prevents developers from getting a handle to inspect why it doesn't render (e.g., to check default computed styles for such a conceptual pseudo).
* Return a `CSSPseudoElement` object (whose state/computed style then reflects non-generation)
    * **Pros:**
        * API Consistency: `Element.pseudo(validPseudoTypeString)` always returns a `CSSPseudoElement` object. `null` is reserved only for invalid/unrecognized type strings.
        * Separation of Concerns: `Element.pseudo()` provides the handle; `.exists` (potentially) on that handle reveals rendering state as per CSS rules.
        * Robust Introspection: Allows developers to get the object and check its computed style (e.g., `display: "none"`, `content: "none"`) to confirm and understand its non-rendered state.
        * Alignment with Highlight Pseudos: Consistent with `::selection` where an object can exist even if no selection is active.
        * Forward Compatibility: If CSS rules change, the API behavior for `Element.pseudo()` remains stable.
    * **Cons:**
        * Indirect "Existence" Check: Developers need to get the object and then inspect its computed styles(or potentially `.exists`, if it’ll be resolved) to confirm it's not actually rendered, rather than just checking for `null`.
        * Potentially "Useless" Objects: Creates an object that might not correspond to anything visible or affecting layout.

**Recommendation:**
Return a `CSSPseudoElement` Object. This approach is generally favored because it provides a more consistent and predictable API contract (`null` only for invalid pseudo element type strings), and the author can use e.g. `getComputedStyle` to determine whether the object exists. The discussion needs to be on if `parent` and `element` should be `null` in that case.

## 3. Handling pseudo element variations.

### 3.1. Non-tree-abiding pseudo elements.
[CSSWG issue](https://github.com/w3c/csswg-drafts/issues/12160)

Should we add support for non-tree-abiding pseudo elements? E.g. for a hypothetical `getBoundingClientRects()` for `::selection`? Should `CSSPseudoElement` be enriched with new methods or should new specific interfaces be added?

**Recommendation:**
Based on some discussions in web animations, non-tree-abiding pseudo elements should likely have a separate interface that is not inherited from `EventTarget`, as it’s not possible sometimes to fit it into e.g. a bubbling model of events.

### 3.2 Parameterized pseudo elements.
[CSSWG issue](https://github.com/w3c/csswg-drafts/issues/12161)

How to retrieve pseudo argument(s) for e.g. `::part(foo)`, `::view-transition-group(name)`, `::slotted(selector)`? Should it just be a part of `type` value or a new attribute e.g. `argument` should be added?

* Include parameter within the `type` attribute
    * **Pros:**
        * Requires no changes to the existing interface structure (no new attributes).
        * The complete identifier used to retrieve the pseudo element is available.
    * **Cons:**
        * Developers would need to manually parse the `type` string to separate the base pseudo element name (e.g., `::part`) from its argument (e.g., `foo`).
        * Less structured; mixes the type identifier with its arguments.
        * Could become cumbersome if future pseudo elements allow more complex argument syntax (e.g., multiple arguments, different data types).
* Add a new dedicated attribute (e.g., `argument` or `parameters`)
    * **Description**: Introduce a new nullable read-only attribute specifically for the parameter(s). This could be a single string (e.g., `argument: CSSOMString?`) or potentially a sequence if multiple parameters were ever supported (e.g., `parameters: sequence<CSSOMString>?`). For `element.pseudo('::part(foo)')`, this new attribute would return `"foo"`, while the `type` attribute would likely return just `"::part"`.
    * **Pros:**
        * Provides clean, structured access to the parameter value(s).
        * Separates the core type identifier (`type`) from its specific arguments.
        * More extensible and robust for handling potentially more complex parameterized pseudo elements in the future.
        * Easier for developers to work with, avoiding manual string parsing.
    * **Cons:**
        * Requires adding a new attribute to the `CSSPseudoElement` interface, increasing its complexity slightly.
        * Requires defining the behavior for non-parameterized pseudos (e.g., should the attribute be `null`, an empty string, or an empty sequence?).

**Recommendation:**
Add a new dedicated attribute. Providing direct access to the parameter via a dedicated attribute improves developer ergonomics by avoiding manual string parsing and clearly separates the pseudo element's fundamental type from its specific instance arguments, also it is somewhat future-proof.

### 3.3 Plurality: multiple pseudo elements of the same "kind".
[CSSWG issue](https://github.com/w3c/csswg-drafts/issues/12162)

This is a significant challenge for the current `Element.pseudo(type)` which implies a single object per type. Examples are `::view-transition-old(*)` / `::view-transition-new(*)` (where `*` could match multiple captured elements with the same tag name), `::column` pseudos and each can have distinct `::scroll-marker` inside.

* `Element.pseudo(type)` returns the first or a representative one.
    * **Pros:** Simple, fits the current IDL.
    * **Cons:** Incomplete, doesn't allow access to other instances.
* Introduce `Element.pseudos(type)` returning a `NodeList`-like collection (`CSSPseudoElementList`?).
    * **Pros:** Explicitly handles plurality. Aligns with DOM patterns like `querySelectorAll`.
    * **Cons:** Adds new API. How would items in the list be uniquely identified or ordered if they are not tree-abiding?
* Parameterize potentially ambiguous pseudos, and let the parameter distinguish them (e.g. make `::column(selector)`).
    * **Pros:** Uses an existing mechanism for some cases.
    * **Cons:** Not a general solution.

**Recommendation:**
We likely need an `Element.pseudos(type)` or a similar mechanism for pseudo elements that are inherently plural, as an author might want to add a click event listener to gather clicks statistics from every `::scroll-marker` nested into `::column`.

## 4. Event handling.
The Core Conflict: `event.target`
A fundamental aspect of current web platform behavior is that user interaction events (like `click`, `mouseover`, etc.) that physically occur over the area styled or occupied by a pseudo element are retargeted before dispatch. The `event.target` property of the dispatched event is set to the originating `Element`, not the pseudo element itself. Existing web content heavily relies on this behavior for event delegation and handling.

### 4.1. `addEventListener` on a Proxy/Handle.
[CSSWG issue](https://github.com/w3c/csswg-drafts/issues/12163)

Given that the CSSPseudoElement object acts as a persistent handle, attaching an event listener via addEventListener raises further questions. 

* What does it mean to listen for an event on a handle when the actual pseudo element might not be currently rendered (e.g., due to display: none)?
* Should the listener only become active when the pseudo element is actually rendered?
* How does this interact with the retargeting behavior?
* If event.target remains the Element, how does a listener on the CSSPseudoElement get invoked?
* Does it require a special dispatch phase?

This would require some heavy discussions, as it’s already known that authors want to be able to count clicks on ::scroll-marker pseudo elements for analytics purposes.

Potential solutions on example of `::scroll-marker`:

* No direct listening on pseudo element (Status Quo):
    * **Description**: Do not allow `addEventListener('click',...)` directly on the `CSSPseudoElement` for `::scroll-marker`. Clicks physically hitting the marker would continue to dispatch with `event.target` set to the originating `Element`. Developers would have to attach the listener to the `Element` and use coordinate-based hit-testing within that listener to infer if the click landed on the marker area.
    * **Pros:**
        * Requires no changes to the event dispatch model.
        * Zero web compatibility risk regarding `event.target`.
    * **Cons:**
        * Very poor developer ergonomics for this specific use case. Calculating marker positions and sizes to perform manual hit-testing is complex, brittle (breaks with style changes), and potentially slow.
        * Makes the `CSSPseudoElement` object less useful and doesn't leverage its `EventTarget` inheritance.
* Allow direct listening with special dispatch phase:
    * **Description**: Allow developers to call `scrollMarkerPseudoElement.addEventListener('click', counterFunction)`. When a click physically occurs on the scroll marker:
        * Special Phase: The browser's event system identifies the hit target as the `::scroll-marker` and invokes `counterFunction` (and any other listeners attached directly to this specific `CSSPseudoElement` handle).
        * Standard Phase: The event dispatch continues as normal, but crucially, `event.target` is set to the originating scrollable `Element`. The event then proceeds through the capture and bubble phases relative to the `Element`, without re-invoking `counterFunction` during this standard flow.
    * **Pros:**
        * Directly addresses the developer need: allows attaching listeners specifically to the pseudo-element of interest.
        * Preserves web compatibility by ensuring `event.target` remains the `Element` during the standard, observable event flow.
        * Provides a meaningful use for the `EventTarget` inheritance on `CSSPseudoElement`.
        * Much better developer ergonomics than manual coordinate checking.
    * **Cons:**
        * Requires implementing the modified event dispatch logic (the "special phase").
        * The event model becomes slightly more complex internally, though the developer-facing API (`addEventListener` on the pseudo) is intuitive.
* Introduce a new specific event type:
    * **Description**: Instead of using the standard `click` event, define a new event type like `pseudoElementClick` that only fires on the `CSSPseudoElement` handle for pseudo elements. Standard `click` events would continue to target the `Element` only.
    * **Pros:**
        * Avoids any ambiguity or modification related to the standard `click` event dispatch.
        * Clearly separates the interaction.
    * **Cons:**
        * Introduces a new event type for a very specific interaction, potentially leading to event type proliferation.
        * Developers might intuitively try to listen for `click` first and be confused why it doesn't work directly on the pseudo-element handle without the special dispatch phase.
        * Doesn't fully resolve the general question of how other standard events (like `mouseover`) should interact with pseudo-elements if direct listeners are desired.
* Add `event.pseudoElement` as in `KeyframeEffect` for Web Animations:
  * **Description**: Add `pseudoElement` parameter as either `CSSPseudoElement` or `string` to `event` which will be set if the event was fired from the pseudo element. Allowing calling `addEventListener`  on `CSSPseudoElement` can even be optional here.
  * **Pros:**
    * Avoids web compat issues.
    * Sidesteps pseudo lifetime/eventing complexity for the use case if `pseudoElement` is represented as `string`.
    * Proven feasible for Web Animations.
  * **Cons:**
    * Adding new API. 

**Recommendation:**

Add `event.pseudoElement` as in KeyframeEffect for Web Animations.

Justification: This option provides the best balance. It directly enables the desired developer use case (listening for clicks on the scroll marker) with good ergonomics (`addEventListener` on the pseudo element object). Crucially, it achieves this while adhering to the non-negotiable web compatibility requirement that `event.target` for standard events like `click` must remain the originating `Element`.
