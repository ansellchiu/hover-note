# hover-note

A drop-in layer for leaving notes on elements of a running web page, readable and writable by both humans and agents.

## Language

**Annotation**:
A note pinned to one element of a page: an Anchor plus a Thread.
_Avoid_: pin, marker, feedback item

**Anchor**:
The data an Annotation keeps so it can find its element again after the page changes.
_Avoid_: selector, locator (those are kinds of Anchor data, not the concept)

**Thread**:
The ordered Comments on one Annotation, with a lifecycle (e.g. open, resolved).

**Comment**:
One message in a Thread, written by a human or an agent.
_Avoid_: note, reply (as the general term)

**Targeting**:
Picking which element an Annotation attaches to: hover on desktop, an inspect-mode equivalent on touch.
_Avoid_: selection (clashes with text selection)

**Backend**:
Whatever stores Annotations and runs the Thread lifecycle: files in the repo by default, or a remote store such as SitePing.
_Avoid_: storage, server (a Backend may be neither)
