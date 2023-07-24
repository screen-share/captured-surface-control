# Captured Surface Control

## In a nutshell
We introduce a new Web API that allows Web applications to:
1. Produce wheel events in a captured tab or window.
2. Read/write the zoom level of a captured tab.

## Motivation
Nearly all video-conferencing Web applications offer their users the ability to share a browser tab, a native window, or screen. Many of these applications also show the local user a “preview tile” with a video of the captured [display surface](https://www.w3.org/TR/screen-capture/#dfn-display-surface).

All these applications suffer from the same drawback - if the user wishes to interact with a captured tab or window, the user must first switch to that surface, taking them away from the video-conferencing application. This presents a few issues:
1. Users are burdened by the need to jump around between the video-conferencing application and the captured surface.
2. Users can't see the captured application and the videos of remote users at the same time. (Modulo PiP; see below.)
3. Users are limited in their ability to interact with controls exposed by the video-conferencing application while they are away from it. Examples of such controls would be an embedded chat application, emoji reactions, knock-ins by users asking to join the call, multimedia controls, etc.
4. Local users who are presenting cannot delegate some limited control to remote participants. This leads to the all too familiar scenario, where remote users must beg the local user to change the slide or scroll a bit up/down.

## Terminology
A **"capturing application"** is an application which has called [getDisplayMedia()](https://www.w3.org/TR/screen-capture/#dom-mediadevices-getdisplaymedia), leading the browser to present a "media-picker" dialog to the user, from which the user will have chosen to capture a tab, a window or a screen. The surface selected by the user is the **"captured surface"**. If that surface was a tab, we call the Web application currently loaded in that tab the **"captured application"**. If the user's chosen surface surface is a window, we designate the associated native application as the **"captured application"**.

## General API shape

We extend [CaptureController](https://www.w3.org/TR/screen-capture/#capturecontroller) to support a limited set of low-risk actions.

### Scrolling

We define the following new action:

```webidl
dictionary CapturedWheelAction {
  int x = 0;
  int y = 0;
  int wheelDeltaX = 0;
  int wheelDeltaY = 0;
};

partial interface CaptureController {
  Promise<undefined> sendWheel(CapturedWheelAction action);
};
```

Using `sendWheel()`, a capturing application can deliver [wheel events](https://developer.mozilla.org/en-US/docs/Web/API/Element/wheel_event) of its chosen magnitude over coordinates of its choosing within a captured surface's viewport. The event is indistinguishable to the captured application from direct user interaction.

A reasonable way to use the API, assuming a `<video>` element called `'previewTile'` is employed by the capturing application, follows:
```js
const controller = new CaptureController();
const stream = await navigator.mediaDevices.getDisplayMedia({ controller });
const previewTile = document.getElementById('previewTile');  // <video>
previewTile.srcObject = stream;

// Perform a null-action so as to prompt the user for permission.
try {
  await controller.sendWheel({});
} catch (e) {
  return;  // Permission denied. Bail.
}

// Having obtained the user’s permission, we can now relay subsequent
// wheel events to the captured tab.
previewTile.addEventListener("wheel", (event) => {
  const [x, y] = translateCoordinates(event.offsetX, event.offsetY);
  controller.sendWheel({
    x, y, wheelDeltaX: event.wheelDeltaX, wheelDeltaY: event.wheelDeltaY});
});
```

When to ask for the user's permission, and how to educate users that scrolling over the preview-tile would scroll the captured surface, are left as exercises for the reader.

## Zooming

TODO

## Security and Privacy Concerns

TODO

## Common questions

### What about Picutre-in-Picture?

[Document Picutre-in-Picture](https://wicg.github.io/document-picture-in-picture/) presents an alternative partial solution to some of the problems addressed by Captured Surface Control. Given the different trade-offs chosen, some applications/users might prefer PiP, while other would prefer Captured Surface Control.

Benefits of PiP:
* No additional new APIs required.
* Captured surface fully interactive (as opposed to the limited set of actions afforded by Captured Surface Control).
* Captured surface available in its original resolution.

Benefits of CSC:
* All elements of the capturing application presented in their original size and are fully interactive. (Consider the challenges of fitting a chat box into the PiP.)
* Application has control of layout of remote participants videos and the captured-surface's preview tile.
* Local user can delegate some actions to remote users (through the mediation of the application).

It is expected that users intending to engage in significant interaction with the presented content would prefer PiP or other alternatives, while users who only need brief interactions to adjust scrolling and zooming would prefer CSC.

### What about Video Portal?

An alternative approach, currently labeled "Video Portal", has been discussed, for example in the [following slides deck](https://docs.google.com/presentation/d/1RIRPAg-M3pQYTFqL0rDGBIl8bQvLAzq122lWUF5JIy8/edit#slide=id.g1df86d70a44_0_25). This approach strikes yet *another* balance, giving control over the captured surface to the local user rather than to the application. On the one hand, this means that almost any action could be allowed; on the other, it means that all the power rests in the local user's hands, and cannot be delegated from the user to the application. We see this approach as a possible *complementary* approach that may be pursued separately from CSC.

## Uncommon questions

### What about blur?

Should a captured tab be unblurred before receiving `wheel` events? That's an open question for now.
