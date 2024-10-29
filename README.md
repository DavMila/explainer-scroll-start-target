# Explainer: scroll-start-target

## Introduction
This is an explainer for `scroll-start-target`: a CSS property that
gives web authors the ability to set the initial scroll offset of a scrollable
container by indicating content that the container should be initially scrolled
to.

### Background
Web authors often wish to have scrollable containers scrolled away from the
default (0,0) position when the container first appears on screen. This could be,
for example, to reflect some state of the page which might make some of the
scrollable container's content more relevant than other content within the
container. In some other cases, websites would like to present users with content first,
and then have them scroll up to reveal an item, e.g. a search bar like in [iOS's Messages app](https://apple.stackexchange.com/questions/441349/how-to-pull-down-the-screen-to-reveal-the-search-bar-in-iphone-messages),
and in [Spotify](https://photos.app.goo.gl/ejDjf3KwVqiATcGj8).

On the web, one way to achieve this is via [fragment identifiers](https://developer.mozilla.org/en-US/docs/Web/API/Location/hash#:~:text=fragment%20identifier) in anchor links, but this method has the
constraint of being able to target only one item.

Another approach is to [create a temporary CSS animation](https://blog.kizu.dev/snappy-scroll-start/#the-scroll-start-workaround) which briefly modifies
the [`scroll-snap-align`](https://developer.mozilla.org/en-US/docs/Web/CSS/scroll-snap-align) property of the target item so as to cause a scroll to it:
```
.container {
  ...
  scroll-snap-type: x mandatory;
}

.target {
  ...
  animation: --snap 0.01s linear;
}

@keyframes --snap {
	from {
		scroll-snap-align: start;	
	}
}
```
This approach is rather brittle and would not work well in a scenario where the target
is to remain in view even through layout changes occurring as the page loads.

Another way is to use one of JavaScript's scrolling APIs, e.g. [scrollIntoView](https://developer.mozilla.org/en-US/docs/Web/API/Element/scrollIntoView), to
bring the relevant content into view after the page has been loaded. However, using
JavaScript to achieve the desired style of a page is often less than ideal
as it can result in degraded performance and can be prone to error. For example,
similar to [scroll anchoring](https://developer.mozilla.org/en-US/docs/Web/CSS/overflow-anchor/Guide_to_scroll_anchoring),
web authors would need to account for the dynamic way in which content is fetched and
loaded into a page such that changes in the layout of the page are necessary.
This means that: to achieve the intended effect of `scroll-start-target`, it may
not be enough to write:

```
window.onload = () => {
  ...
  const scroll_target = document.getElementById("scroll-target");
  scroll_target.scrollIntoView();
  ...
}
```
as the window `load` event will only fire once. The author will need to:
- detect the occurrence of layout changes that  result in their desired
  element not being scrolled into view, and
- invoke `scrollIntoView` again.

If using `scrollIntoView` to keep a particular element in view, the author would
also need to be mindful of interfering with a scrolling gesture from their user
i.e. if the user has scrolled away from the targeted content, `scroll-start-target`
no longer keeps the scroll container anchored to the content.

Additionally, the scrollable container in question may not appear on the page
until after the `load` event, e.g. until a user clicks a "Show Gallery" button
like in the gallery example below.

### Proposal
Introduce a CSS property called `scroll-start-target` which can be used to
indicate that the scroll container of the affected element (the `target`)
should, on its initial appearance/layout in the page, be scrolled to the `target`.

After the user agent detects an "explicit" scroll on the scroll container,
`scroll-start-target` isn't active anymore and the scroll container can be
freely scrolled. An explicit scroll would be any scroll arising from a user's gesture,
or a programmatic scroll API.

`scroll-start-target` is also not to interfere with scrolling due to a
[fragment identifiers](https://developer.mozilla.org/en-US/docs/Web/API/Location/hash#:~:text=fragment%20identifier) in a URL.

### Use Cases
- an image gallery that ought to start with the middle image in view,
- a page in which the author would like to first present content to the user and
  then have them scroll upwards to reveal a searchbar,
- a menu/list of options which starts scrolled to a user's previously-selected choices,
- a menu/list of options among which the author predicts some to be
  more relevant to the user than others due, perhaps, to prior user activity,
- a long page where the author wants to save the user the effort of scrolling to
  a particular section.

## Examples

### User Settings
In this [example](https://davmila.github.io/demo-scroll-start-target/drones/index.html)
when the "Maintenance Page" is loaded, the scroll container, when first displayed,
appears scrolled to the section corresponding to the selected drone.

### Gallery
In this [example](https://davmila.github.io/demo-scroll-start-target/gallery/index.html) the image gallery is not displayed until the "Show Gallery"
button is clicked after which the gallery is revealed and is scrolled to the
middle image.

## Syntax
The `scroll-start-target` property takes one parameter whose only valid values are:
- `auto`, indicating the element should be scrolled to, and
- `none`, the default value, indicating the element isn't to be scrolled to.

```
<style>
.scrollcontainer {
    overflow-x: scroll;
}
.target {
    scroll-start-target: [auto|none];
}
</style>
...
<div class="scrollcontainer">
    <div></div>
    <div></div>
    <div class="target"></div>
    <div></div>
    <div></div>
</div>
...
```

## Alternatives & Other Considerations

### Per-axis Parameters
One alternative option for `scroll-start-target`'s syntax would allow specifying
values for the block and inline axes separately:
```
<style>
...
div {
    scroll-start-target: <block> <inline>;
}
...
</style>
```
where `<block>` and `<inline>` could independently be `auto` or `none`.
By setting the bias on the block and inline axis independently of each other,
this syntax would make it possible for separate targets to be scrolled to in the
different axes such that neither target ends up within the scroll container's
scroll port which runs contrary to the purpose of `scroll-start-target`.

### Competing scroll-start-targets
It is possible for more than one element within the content of a scrollable
container to have its `scroll-start-target` set to `auto`, meaning the
scrollable container has to choose which `scroll-start-target` to scroll to.
Overall, user agents are to handle such scenarios by giving priority to the
element which comes first in [DOM order](https://dom.spec.whatwg.org/#concept-tree-order).

#### Competing `scroll-start-target`s & DOM-order
While priority is to be given to the `scroll-start-target` which comes first in
DOM order, there is some consideration for a scenario in which the author desires
a best effort attempt at bringing all the specified `scroll-start-target`s into
view. Specifically, where mutliple `scroll-start-target`s exist, user agents
are to scroll to each target in reverse-DOM-order, i.e. starting from the last
and working their way up to the first. At the moment, this is effectively the
same as only scrolling to the first in DOM order. However, these steps are
chosen in a way that will work well with a potential future css property that
allow a `nearest` scroll alignment similar to [scrollIntoView](https://developer.mozilla.org/en-US/docs/Web/API/Element/scrollIntoView)'s `nearest`
argument so that the reverse-DOM order procedure can be closer to "best effort."

As an example of a situation where a developer might want multiple `scroll-start-target`s
consider this [demo](https://davmila.github.io/demo-scroll-start-target/todo/index.html) where the "done" items in the TODO list
are all `scroll-start-target`s. When the user dismisses a done item, the next
one, by virtue of being a `scroll-start-target`, comes into view, making the process
of dismissing all done items very convenient for the user.
