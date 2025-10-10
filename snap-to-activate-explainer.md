
We'd like to make it easier and more reliable for developers to build user experiences
where navigation can be done by moving parts of the UI
in a way that can correspond to scrolling and then scroll snapping.
This navigation should be able to change the URL
(so that the resulting state can be linked to and shared)
and [eventually](route-matching-explainer.md#declarative-patch-based-document-updates)
should be able to trigger loading of the resources needed
to display the new part of the UI.

## Use cases

Let's start by examining some use cases that this feature should address,
and thus make it simpler for developers to address these use cases well and declaratively.

### Panels or carousel within a single page app

One use case is a set of panels or a carousel within a single page app,
where swiping left or right moves to the adjacent panel
(and typically each panel fills the entire width of the UI).
It should be possible for navigating between panels to declaratively
change the document URL (without a new document load, like `pushState`)
to reflect the currently selected panel.
In many cases there are also either buttons/tabs at the top or bottom,
or arrows at the sides,
to move between these panels.

(This use case can also integrate with patching
and thus with lazy loading of the resources
needed for each panel.)

This use case is somewhat simpler since the activation here
is intended to be nondestructive.
(It does potentially cause resource loading,
but that's normal for exploring additional pieces of an app's UI.)

Here are two examples taken from native Android apps:

(INCLUDE VIDEO HERE from https://photos.app.goo.gl/9YonAy8GKg7vyQ1P6)

In this case, the Photos app allows swiping from side to side
to navigate between photos.
If this were on the Web,
it would be expected for the URL to change
when the UI moves to a different photo.
This allows the URL of a single photo to be shared.

(INCLUDE VIDEO HERE from https://photos.app.goo.gl/1ABe699gY39oiZmH9)

In this case, the maps app allows similar swiping from side-to-side
to navigate between successive trains departing from a single subway station
in the same direction.
The trains can also be selected by tapping on the train departure time
at the top of the UI.
In this particular case URLs are probably less critical
since the information in this particular UI is extremely transient,
but in general this sort of UI may want to have shareable URLs to represent each item.

### Pull to refresh

A slightly more advanced use case is pull-to-refresh.
This is a common UI pattern where scrolling a scrollable area
(often but not always a list)
to the top
and then continuing to scroll a little bit past the top
(where the scrolling goes slower than the user's finger
to provide the feeling of resistance or pulling)
causes the content to reload or to add new conent.
This requires having UI in the overscroll area (the area past the end)
of a scrollable container.
When activated, the UI in the overscroll area can trigger the refresh.

(This depends on a separate plan for rendering content in the overscroll area.)

In this use case,
snapping to and thus activating
an element that is in the overscroll area
would trigger the refresh.

This use case requires more care because the operation is destructive.
This means that this use case requires more careful consideration
of the implications for accessibility issues,
such as how it works with keyboard navigation,
how it integrates with assistive technology,
etc.

(INCLUDE VIDEO HERE from https://photos.app.goo.gl/JFuVWaXqnroUjeMH7)

### Swipe actions (like swipe to delete)

Another use case that is similar in many ways to pull-to-refresh
is swipe-to-delete.
Many user interfaces with lists of items that the user can delete
(for example, lists of messages)
allow swiping an individual item to the side to either delete an item
or remove it from the particular list that it is in (for example, to archive it).

This has many of the same concerns as pull-to-refresh:
it depends on interaction with the overscroll area,
and it is a destructive (perhaps more destructive than pull-to-refresh)
action that requires the same level care regarding accessibility issues.
(Many of the issues appear to be the same,
although some aspects may also be different.)

(INCLUDE VIDEO HERE from https://photos.app.goo.gl/N7D5k5WaDDCeoDQ27)

## Additional constraints

Some things worth thinking about when designing a solution for this are:

* An element, particularly one within a single-page app,
  may be activated in this way more than once in the lifetime of the page.

* Snap-to-activate should integrate with the work on declarative patching,
  since some of the use cases may involve
  activating substantial pieces of additional content.

* Snap-to-activate should integrate with declarative routing because
  (WRITE ME)

* While in many cases the snapping involved will be
  `scroll-snap-type: mandatory`,
  the design should probably consider `scroll-snap-type: proximity' as well.

## Possible directions


Options:

 - snap to an element with an ID changes the URL to that ID
 - mechanism tying an element to a separate link, snapping the element
   activates that link.

(Can both of these be tied to routemaps, or only the second?)


