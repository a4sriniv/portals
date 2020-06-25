# Portals

Portals enable seamless navigations between pages. In particular, we propose a new `<portal>` HTML element which enables a page to show another page as an inset, and then *activate* it to perform a seamless transition to a new state, where the formerly-inset page becomes the top-level document.

Portals encompass some of the use cases that iframes currently do, but with better security, privacy, and ergonomics properties. And via the activation operation, they enable new use cases like preloading or navigation transitions.

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
## Table of contents

- [Example](#example)
- [Use cases](#use-cases)
- [Objectives](#objectives)
- [Details](#details)
  - [Privacy threat model and restrictions](#privacy-threat-model-and-restrictions)
  - [Permissions and policies](#permissions-and-policies)
  - [Rendering](#rendering)
  - [Interactivity](#interactivity)
  - [Accessibility](#accessibility)
  - [Session history, navigation, and bfcache](#session-history-navigation-and-bfcache)
- [Summary of differences between portals and iframes](#summary-of-differences-between-portals-and-iframes)
- [Alternatives considered](#alternatives-considered)
  - [A new attribute on an existing element](#a-new-attribute-on-an-existing-element)
  - [TODO:](#todo)
- [Security and privacy considerations](#security-and-privacy-considerations)
- [Stakeholder feedback](#stakeholder-feedback)
- [Acknowledgments](#acknowledgments)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Example

A document can include a `<portal>` element, which renders the specified URL in a portal context:

```html
<portal id="myPortal" src="https://www.example.com/"></portal>
```

This is somewhat like an iframe, in that it provides a rendering of the specified document at `https://example.com/`. However, a portal is much more restricted than an iframe: user interaction with it [does not pass through to the portaled document](#interactivity), and when the portaled document is cross-origin, capabilities like storage and `postMessage()` communication are [prohibited](#privacy-threat-model-and-restrictions) while rendering in the portal context. This allows portals to be used as a more-restricted view onto another document when the full capabilities of an iframe are not needed.

In exchange for these restrictions, portals gain an additional capability that iframes do not have. A portal can be *activated*, which causes the embedding window to navigate, replacing its document with the portal:

```js
myPortal.activate();
```

_Note: [issue #174](https://github.com/WICG/portals/issues/174) discusses making this the default behavior when clicking on a portal, similar to a link._

At this point, the user will observe that their browser has navigated to `https://www.example.com/` (e.g., via changes to the URL bar contents and back/forward UI). Since `https://example.com/` was already loaded in the portal context, this navigation will occur seamlessly and instantly, without a network round-trip or document re-initialization.

For more advanced use cases, the `https://www.example.com/` document can react to activation, using the `portalactivate` event. It can use this event to adapt itself to its new context, and even to adopt its *predecessor* (the document which previously occupied the tab) into a new portal context.

```js
window.addEventListener('portalactivate', e => {
  document.body.classList.add('displayed-fully');
  document.requestStorageAccess().then(() => {
    document.getElementById('user').textContent = localStorage.getItem('current-user');
  });

  let predecessor = e.adoptPredecessor(document);
  console.assert(predecessor instanceof HTMLPortalElement);
  document.body.appendChild(predecessor);
});
```

## Use cases

_See the ["Key Scenarios"](./key-scenarios.md) document for more detail on each of these, including visualizations._

- **Pre-rendering**: by loading a page in a hidden portal, it is possible to pre-render the destination page. Then, activating the portal will instantly display the pre-rendered document in the same browser tab.

  This requires some care and cooperation from both sides to deal with the restrictions on portaled content, and the transition between portaled and activated states.

- **Navigation transitions**: pre-rendering opens the door for more elaborate transitions, by displaying the portal in some form, animating it (using resizing, translations, etc.) until it occupies the full viewport, then finally activating the portal to perform the instant navigation.

- **Aggregation**: multiple portals on the same page can be used to create more elaborate experiences, where the user chooses which portal to activate. This category of use cases includes cases like a news reader, a shopping site, an infinite scrolling list of articles, etc. By using a portal instead of (or in addition to) a link, the aggregated content has the opportunity to display a preview, and to benefit from pre-rendering and navigation transitions.

  Additionally, by using the ability to adopt the predecessor during the `portalactivate` event, more complicated interactions between the aggregator and the aggregated content can be built, such as retaining a portion of the shopping site or article-list UI in a portal even after navigating to an individual page.

- **"Better iframe"**: portals encompass some, but not all, of the use cases for iframes. And they do so in a way that is better for users, in terms of security and privacy. They also remove a lot of the legacy baggage and sharp edges that come with iframes, making them easier to use for web developers. So we anticipate many of the current cases for iframes, such as ads, being able to be replaced with portals.

  See [below](#summary-of-differences-between-portals-and-iframes) for a more detailed summary of the differences between portals and iframes.

## Objectives

Goals:

- Enable seamless navigations from a page showing a portal, to the portaled page

- Enable seamless navigations between pages of a portal-aware website

- Avoid the characteristics of iframes which have negative impacts on security, privacy, and performance

Non-goals:

- Built-in support for high-level navigation patterns, such as carousels or infinite lists. Portals provide a low-level building block for pre-rendering, which can be combined with the usual tools of HTML for creating navigation pattern UIs.

- Built-in support for portal-specific transition animations. Given that portals are represented by HTML elements, existing CSS mechanisms are enough to allow authors to create compelling navigation transitions.

- Subsume all the use cases of iframes. The use cases for portals overlap with those for iframes, but in exchange for the ability to be activated, portaled pages lose abilities like cross-origin communication, storage, and nontrivial interactivity. As such, portals are not suitable for use cases like embedded widgets.

- Allowing arbitrary unmodified web pages to be portaled. Cross-origin pages will need to adapt to work well when they are hosted in a portal.

## Details

The general idea of portals is summed up above: inset pre-rendering, activation, and predecessor adoption. These subsections go into more detail on important parts of how portals work.

### Privacy threat model and restrictions

The Portals design is intended to comply with the [W3C Target Privacy Threat Model](https://w3cping.github.io/privacy-threat-model/). This section discusses the aspects of that threat model that are particularly relevant to Portals and how the design satisfies them.

A portal can contain either a same-site or cross-site resource. Same-site portals don't present any privacy risks, but cross-site resources risk enabling [cross-site recognition](https://w3cping.github.io/privacy-threat-model/#model-cross-site-recognition) by creating a messaging channel across otherwise-partitioned domains. For simplicity, when a cross-site channel needs to be blocked, we also block it for same-site cross-origin portals. In some cases, marked below, we even block it for same-origin portals.

This design assumes that storage will be partitioned or blocked in line with the [storage partitioning proposal](https://github.com/privacycg/storage-partitioning), even though that is not the status quo in all browsers. Because portaled documents can be activated into a top-level browsing context, they (eventually) live in the first-party [storage shelf](https://storage.spec.whatwg.org/#storage-shelf) of their origin. This means we have to prevent communication with the surrounding document to the same extent we prevent it with a cross-site link opened in a new tab.

#### Channels that are blocked

* [`postMessage()`](https://developer.mozilla.org/en-US/docs/Web/API/Window/postMessage) isn't allowed between a cross-origin portal and its container.
* Blocked by preventing fetches within portals from using credentials until they're activated:
  * The sequence of portal loads: The source page can encode its user ID into the order in which a sequence of URLs are loaded into portals. To prevent the target from correlating this ID with its own user ID without a navigation, a document loaded into a cross-origin portal is fetched without credentials and doesn't have access to either first-party storage or multi-keyed storage. It has to use [`requestStorageAccess()`](https://developer.mozilla.org/en-US/docs/Web/API/Document/requestStorageAccess) (or another TBD mechanism) to get access when it's activated.
  * The site creates a portal, and the target site decides between a 204 and a real response based on the user's ID. Or the site delays the response by an amount of time that depends on the user's ID. Because the portal load is done without credentials, the site can't get its user ID in order to make this sort of decision.
* Side channels, further discussed in [Rendering](#rendering):
  * Portal Resize: JavaScript outside the portal could use a sequence of portal resizes to send a user ID, so a portal's content cannot observe any resizes after creation. If a portal is resized, that just rescales the view of the portal. For simplicity, this is the case for same-origin portals too.
  * Portal initial size: JavaScript outside the portal could use the initial size to send part of a user ID, so portals are always sized the same as the top-level tab that they'll be activated into and then scaled into the portal element. For simplicity, this is the case for same-origin portals too.
  * Intersection Observer: Won't give visibility information to script inside the portal, to avoid occlusion being used to send information.
  * [Page Visibility](https://developer.mozilla.org/en-US/docs/Web/API/Page_Visibility_API) and the timing of `requestAnimationFrame()` callbacks match the visibility of the top-level page, as in iframes, to prevent the page from encoding messages in visibility changes. However, this leads to the possibility that a portal and containing page could use the timing of user-caused visibility changes to join the user across site boundaries. Whether and how to prevent this is [still under discussion](https://github.com/WICG/portals/issues/193#issuecomment-639768218).

#### Channels that match navigation

* The URL of the activated portal and the referring URL are available to normal navigations to the same extent they're available to a portal. Solutions to link decoration will apply to both.


TODO:

- More side channels?
- Maybe this is also the place to discuss the fact that there's no sync access (even same-origin)?
- This section might benefit from being split in two.
- Note that design-discussions.md has discussion about hiding never-activated portals from their servers, which is a different sort of attack than the cross-site tracking we discuss here.

### Permissions and policies

TODO:

- Discuss default portaled state. You get no permissions; you inherit (how?) policies. Policies = CSP, document policies, feature policies, referrer policy, ... Anything else?
- Discuss how things change post-activation.
- Discuss practices and patterns for site authors, probably mostly from the point of view of someone being portaled that wants to do something permission-requiring once activated.
- Explicitly point out how activation makes this different from iframes in certain ways, but we still share a lot of the infrastructure.
- Briefly discuss permissions and policies that people might want to impose on portals, including some that aren't proposed yet (like blocking scripts or network access). Note that these will be developed independently, for iframes and portals alike.
- Somewhere we need to discuss our plans for `X-Frame-Options`, CSP `frame-ancestors`, and portal-specific variants of those? This might be the place for that?

### Rendering

Like iframes, portals can render their contents inline in another document. To ensure a smooth transition when activation occurs, and to limit the avenues for communication between the two documents, rendering generally occurs in the same way as it will when the portal is activated. This means that `document.visibilityState` and `document.hidden` will, like iframes, match the values in their host browsing context, even if they are offscreen. Similarly, `IntersectionObserver` will report intersections up to the portal contents viewport, but will assume that viewport is fully visible. Other behavior that depends on intersection with the viewport, such as lazy-loading images, behaves similarly.

`requestAnimationFrame` issues vsync callbacks as it would in the host document (in particular the host document should not control the frequency of animation updates), except that for performance reasons user agents may suspend or throttle callbacks to offscreen portals if the two documents are same-origin.

Since documents can detect when they are embedded in a portal, they may choose to suspend, limit or delay animations or other rendering activity that is not essential to prerendering. Authors may also style the document differently while in a portal, but if so they should take care to ensure that this doesn't make activation jarring (e.g. they may wish to animate elements in after activation, or reserve space for elements to avoid layout shift).

TODO:

- Discuss viewport size. Full viewport size at all times? Resizing OK or no? I'm sketchy on the plan here.
- Discuss practices and patterns for authors of portaling sites, e.g. how to create a prerender (with a `display: none` portal) or an animated transition
- Include more detailed samples of how authors would adapt for being in a portal and reacting to activation, including if applicable the resolution of https://github.com/WICG/portals/issues/3
- Maybe this is where we discuss fallback content for non-supporting browsers?

### Interactivity

Portals enable preloading, previewing and seamless transitions to another web page. They are expected to often be partially or fully offscreen, scaled, faded or otherwise styled in a way that makes them unnatural to interact with directly. Additionally, we expect many web pages to allow themselves to be loaded in a portal for the purposes of facilitating a seamless transition, but wish to mitigate certain kinds of threat (e.g. some forms of clickjacking) from an embedder who may not be fully trusted.

Therefore the portal content cannot be focused and does not receive input events. Instead, the `<portal>` element itself is focusable (similar to a button or link) and eligible to receive input events (such as clicks) in the host document. For instance, the host document may handle this click event to animate and activate the `<portal>` element and navigate to the target document.

When a portal is activated, any active pointers on the previous page are transferred to the newly activated page. The activated page receives a `touchstart`/`pointerdown` event for every active pointer, and `touchmove`/`pointermove` events for any subsequent pointer movement. The predecessor page will receive `touchcancel`/`pointercancel` events for pointers that were active. The user agent ends any active touch initiated gestures (eg: scrolls / swipes) in the predecessor page, and automatically starts new touch gestures in the activated page.  This enables touch gestures that trigger a portal activation to continue across portal activation. For example, if a portal is activated while a user is scrolling a page, the user can continue scrolling in the newly activated page without lifting their finger/stylus.

TODO:

- Did we end up adding a default click behavior like links?
- When/if we update this explainer to discuss resize limitations, comment on how that affects interactivity.
- Consider discussing how storage access limitations interact with interactivity.

### Accessibility

From an accessibility perspective, portals behave like a single activatable element (similar to a button). As discussed in the section above, the contents of portals are not interactive and don't receive input events and focus. As a result, the embedded contents of a portal are not exposed as elements in the accessibility tree.

Portals come with accessibility defaults right out of the box. Their default ARIA role is `"button"`, and they are therefore visible to screen-readers as a button by default. The portal element is also intended to be focusable and keyboard activatable in the same way as a button.

Portals also compute a default label from their embedded contents (by either using the title of the embedded page or concatenating all the visible text in the portal's viewport if the page doesn't have a title). This label can be overridden by authors using the `aria-label` attribute.

These defaults ensure that a portal can be accessed and described by assistive technology without any work from authors. Additionally, authors should add a click handler to activate the portal, even if it would otherwise be activated by some other gesture (e.g. a swipe), to ensure that assistive technology or keyboard users can activate the portal. ([#174](https://github.com/WICG/portals/issues/174) discusses adding this as default behavior.) Authors should use the `hidden` HTML attribute, or `display: none`, to hide portals that are meant to be hidden until activation time, e.g. portals that are only used for prerendering. (This will also hide them from the accessibility tree.)

Authors should respect the `prefers-reduced-motion` media query by conditionally disabling any animations used before/during portal activation. For CSS animations and transitions, this can be easily accomplished by overriding all animation durations with a short unnoticeable duration value when the media query is set. Animations triggered with the Web Animations API would have to be explicitly disabled in script by authors when the media query is set.

### Session history, navigation, and bfcache

TODO:

- One-sentence reiteration of how activation is a navigation.
- Talk about how session history is tricky.
- Talk about how bfcache is tricky.
- Outline our solution for these, in terms of observable effects for users (not in terms of spec primitives).
- Maybe compare/contrast with iframes. Is it helpful or confusing to talk about how portals are more like popups than iframes? (I.e. they are top-level browsing contexts in spec terms.)

## Summary of differences between portals and iframes

Portals are somewhat reminiscent of iframes, but are different in enough significant ways that we propose them as a new element.

From a user's perspective, a portal behaves more like a "super link" than an iframe. That is, it has the same [interactivity](#interactivity) and [accessibility](#accessibility) model of being a single activatable element, which will cause a navigation of the page they're currently viewing. It'll be fancier than a link, in that the portal might display a preview of the portaled content, and the navigation experience will be quicker (and potentially animated, if the site author so chooses). But the ways in which it is fancier will generally not remind users of iframes, i.e. of scrollable viewports into an independently-interactive piece of content hosted on another page.

From the perspective of implementers and specification authors, portals behave something like "popups that display inline". This is because they use the [top-level browsing context](https://html.spec.whatwg.org/#top-level-browsing-context) concept, instead of the [nested browsing context](https://html.spec.whatwg.org/#nested-browsing-context) concept. More specifically, [portal browsing contexts](https://wicg.github.io/portals/#portal-browsing-context) sit alongside [auxiliary browsing contexts](https://html.spec.whatwg.org/#auxiliary-browsing-context) (popups) as two distinct types of top-level browsing context, and much of the specification infrastructure is shared. This becomes even more true after activation, when the portal browsing context becomes just another tab.

Finally, the web developer dealing with a portal element's API sees the following differences from iframes:

- Even same-origin portals do not provide synchronous DOM access to the portaled `Window` or `Document` objects, whereas iframes give such access via `frame.contentWindow`/`frame.contentDocument`. This gives a more uniform isolation boundary for more predictable performance and security.

- Similarly, portaled `Window` objects are not accessible via accessors like `window.iframeName` or `window[0]`, and they cannot access related `Window` objects via `top` or `parent` (or `opener`).

- Navigations within a pre-activation portal do not affect session history. In contrast, navigating an iframe creates a new session history entry, and affects the resulting back button behavior.

- Portals cannot be made to navigate from the outside in the way iframes (or popups) can, via `window.open(url, iframeName)`.

- Portals can only load `http:` and `https:` URLs. This removes an entire category of confusing interactions regarding `about:blank`, `javascript:`, `blob:`, and `data:` URLs, as well as the `<iframe srcdoc="">` feature and its resulting `about:srcdoc` URLs. Notably, the portaled content will always have an origin derived from its URL, without any inheritance from the host document.

- Portals have a simplified permissioning model, compared to iframe's combination of `allow=""` + `allowfullscreen=""` + `allowpaymentrequest=""` + `sandbox=""`. See [above](#permissions-and-policies) for more on this. (TODO: include a summary when we know what it is. Just `allow=""`?)

TODO: summarize above sections that cause major differences, once they are written: privacy/restrictions, permissions/policies, maybe some more on session history. They may fit as bullets or they might need their own paragraph.

## Alternatives considered

### A new attribute on an existing element

It would be possible to design portals as an extension of an existing element. As discussed in the [summary of differences between portals and iframes](#summary-of-differences-between-portals-and-iframes), potential candidates would be `<iframe>` or `<a>`. So you could imagine something like

```html
<a href="https://example.com/portal-me" portal>Some text</a>
```

or

```html
<iframe src="https://example.com/portal-me" portal></iframe>
```

However, in both cases the new attribute would change the behavior of the element in ways that are problematic from the perspective of users, web developers, implementers, and specification writers.

For users, the biggest confusion would be the experience in browsers that do not support portals. Falling back to a link might work reasonably well, as long as the web developer specifically codes around the lack of activation behavior. Falling back to an iframe is likely to work poorly; portaled content operates under a very different security and privacy model than iframed content, and the resulting page would likely be broken.

Additionally, the behavioral differences outlined above would lead to extensive forks in the specification for these elements, to go down a new "portal path" whenever the attribute was present. This creates a maintenance burden for specification writers and implementers, and a confusing experience for web developers. The closest precedent we have for a single attribute causing such a dramatic change to behavior is `<input>`'s `type=""` attribute, which has been [a painful experience](https://html.spec.whatwg.org/#concept-input-apply). We would also have to define behavior for when the attribute is added or removed, including when such additions or removals happen during delicate phases of the element's lifecycle like parsing, navigation, interaction, or portal activation.

Finally, we believe that attempting to classify a portal as a "type of iframe" or "type of link" is pedagogically harmful. Although there is some overlap in use cases, a portal is a different piece of technology, and as such is best represented as its own element. It can thus generate its own documentation, developer guidance, and ecosystem discussion. This includes guidance both on how to use portals, as separate from iframes and links, and on how best to let your content be portaled, separately from letting it be framed or linked to.

### TODO:

- Other (historical?) solutions to prerendering
- Other (historical?) solutions to navigation transitions
- Adding activation ("promotion") to iframes ([text existed in explainer.md](https://github.com/WICG/portals/blob/d901563053af7d56a2cafee686ee89d651c49eb7/explainer.md#iframe-promotion) but was very implementer-focused)
- Using the fullscreen API ([text existed in explainer.md](https://github.com/WICG/portals/blob/d901563053af7d56a2cafee686ee89d651c49eb7/explainer.md#fullscreen-iframe) but was very implementer-focused)
- Allowing cross-origin communication and storage

## Security and privacy considerations

TODO:

- Can reference privacy threat model and restrictions above (or however that splits into sections when written)
- Eventually content here should be incorporated into the spec, but for now let's develop it in the explainer

See also the [W3C TAG Security and Privacy Questionnaire answers](./security-and-privacy-questionnaire.md).

## Stakeholder feedback

- W3C TAG: [w3ctag/design-reviews#331](https://github.com/w3ctag/design-reviews/issues/331)
- Browsers:
  - Safari: No feedback so far
  - Firefox: [mozilla/standards-positions#157](https://github.com/mozilla/standards-positions/issues/157)
  - Samsung: No feedback so far
  - UC: No feedback so far
  - Opera: No feedback so far
  - Edge: No feedback so far
- Web developers: TODO

## Acknowledgments

Thank you to Andrew Betts for his [promotable iframe proposal](https://discourse.wicg.io/t/proposal-for-promotable-iframe/2375), which inspired much of the thinking here.

Contributions and insights from:
Adithya Srinivasan,
Domenic Denicola,
Ian Clelland,
Jake Archibald,
Jeffrey Jasskin,
Jeremy Roman,
Kenji Baheux,
Kevin McNee,
Lucas Gadani,
Ojan Vafai,
Rick Byers, and
Yehuda Katz.
