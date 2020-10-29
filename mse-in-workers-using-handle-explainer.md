# Media Source Extensions: MSE-in-Workers Explainer

###### Author: Matthew Wolenetz, [Google Inc.](www.google.com) - December 17, 2018.  Last update October 29, 2020.

Please note this updated version no longer adds a `MediaSourceHandle` object.
This updated version also is limited to expanding MSE usage beyond the main
thread to just a `DedicatedWorker` context (not also a `SharedWorker` context.)
Further, this updated version drops the 'early-open' portion.
These were removed as major simplifications, and are not precluded from being
proposed again in future (especially the 'early-open' functionality.)

## tl;dr

We propose enabling Dedicated Web Workers to be able to create `MediaSource`
objects and `MediaSource` objectURLs. They may then `postMessage` or otherwise
communicate those objectURLs to the main thread for use in attaching to media
elements. The direct manipulation of the `MediaSource` and ancillary objects
like `SourceBuffer` is allowed only from the context that originally created the
`MediaSource` object. This allows both:

* improved access to, and
* improved performance of the [Media Source Extensions API](https://www.w3.org/TR/media-source/) (MSE).

We are incubating this idea initially in the WICG, with goal of producing an
experimental prototype implementation followed by specification in the next
version of MSE.

### Implementation status as of last update

* Chrome - implemented behind experimental flag as of 88.0.4300.0
* Firefox - interest expressed in F2Fs.
* Safari Technical Preview - interest expressed in F2Fs.
* Current experimental web-platform-test results:
  * Pending further prototyping, etc.

### Demo of Usage to Avoid "Buffering Jank" When There's Main Thread Contention

* See [the demo source and instructions](https://github.com/wolenetz/mse-in-workers-demo) in its
  repository.

## Background

Web authors have consistently requested that MSE be available from Worker
contexts: [MSE spec issue](https://github.com/w3c/media-source/issues/175).

Separately, we have strong suspicions that user experience
metrics, such as those measuring how long media playback takes to start, can be
impaired by MSE REC spec requiring implementations to delay operations like
`addSourceBuffer()` until after `MediaSource` is attached:
[Chromium's Tracking bug for MSE sourceopen latency improvement work](https://crbug.com/778082).

In the absence of such mechanisms, web authors are forced to await `MediaSource`
attachment completion on the main `Window` context, and perform all operations
on the MSE API also on the main `Window` context. Especially when the main
context is busy servicing other application logic, these restrictions can
significantly impair the user experience. For example, even if an app uses
Worker contexts to fetch media to feed to MSE, such feeding must still occur on
the main context, resulting in delayed media playback start and recurring
playback stalls in extreme cases when the playback reaches points in the
presentation timeline for which the app has been unable to buffer enough media.

Design consideration was given to alternative ideas that are not in scope of
this proposal:

* Upgrading the existing `MediaSource` object to be
  [Transferable](https://developer.mozilla.org/en-US/docs/Web/API/Transferable).
  However, `EventTargets` like `MediaSource` cannot be transferred. A viable,
  ergonomic way of overcoming this appears unlikely.
* Upgrading `HTMLMediaElement` (and its many related objects and APIs) to be
  usable on a Worker context. Though this might be a solution, the complexity of
  this approach led us to propose a simpler change scoped to just MSE.
* Providing a new, transferable object, such as a `MediaSourceHandle` in lieu of
  a `MediaSource` objectURL to enable attachment, for instance via the
  `srcObject` attribute on an `HTMLMediaElement`.
* Enabling usage of MSE API from `SharedWorker` or `ServiceWorker` contexts.
* Providing an explicit method, such as `MediaSource.open()` to enable
  applications to begin adding `SourceBuffer`s and starting initial
  `appendBuffer()` operations on them prior to the attachment of the
  `MediaSource` to an `HTMLMediaElement` completing.

Note that this proposal does not preclude separate or subsequent effort to
pursue these other ideas.

## Proposed Plan

This proposal is focused on enabling `DedicatedWorker` usage of MSE API to
improve application ability to buffer media for smoother playback despite a
potentially busy main `Window` context. It does not change the pre-existing
usability of the MSE API on the main `Window` context, for usage with
`MediaSource` objects created on that main context.

The REC version of [Media Source Extensions
API](https://www.w3.org/TR/media-source/) (MSE) allows access to the API only on
the main document `Window` context. Further, the MSE API requires a
`MediaSource` instance to first be "attached" to an `HTMLMediaElement` for that
element to be extended by MSE and for the `MediaSource` to become `open` and
allow further operations like `SourceBuffer` creation and manipulation. The
"attachment" is started by creating a `MediaSource` objectURL, and then setting
the `src` attribute of the `HTMLMediaElement` to be that objectURL.

To maintain backwards compatibility with that behavior, while affording improved
MSE API access and performance, this proposal adds feature-detectable
`MediaSource` constructor access to `DedicatedWorker` contexts. It also enables
`MediaSource` objectURL creation on a `DedicatedWorker` context, and enables
access to all of the MSE object model on `DedicatedWorker` context (for
instances constructed there).

To prevent leakage of access cross-context by the
app to the MSE object model, care will be taken especially to modify what the
`Audio,Video,TextTrack` reports in the extended-by-MSE `sourceBuffer` attribute
depending on which context is accessing the object: if not the same context as
where the `MediaSource` was constructed, then that field will return `null`.

## Open Questions

##### Can implementations of this proposed API significantly reduce the performance problems and eliminate the API access problems versus REC MSE?

###### Initial Thoughts:

This is a big part of why we're incubating. Of course, standardizing a more
ergonomic API with improved interoperability are also strong reasons for
incubation.


##### Why must we create the `MediaSource` objectURL on a `DedicatedWorker` context if we wish to use the underlying `MediaSource` on that worker context? Why not instead use a transferable object, like a `MediaSourceHandle`, constructed on the main `Window` context, and extracting something transferable from it to transfer to other contexts if needed there?

###### Initial Thoughts:

While this might allow some parallelization of attachment on the main document
`Window` context while the `DedicatedWorker` context is still
starting, we're not convinced yet that the additional complexity is worth it.
For example, using subworkers or other multi-worker arrangements communicating
directly via the Channel Messaging API to reduce the granularity of priority worker
startup might be a reasonable workaround. This is another good reason
to incubate.


##### Can detachment of `MediaSource` not automatically shutdown all SourceBuffers (and remove all their buffered media?)

###### Initial Thoughts:

Perhaps, but an orthogonal feature proposal to let an app explicitly request a
MediaSource object to behave differently on detach of it
would reduce the complexity of each proposal and let them be developed
in parallel or separately.
