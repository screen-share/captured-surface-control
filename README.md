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

### Prelude: Permission Prompt

The first invocation of either `sendWheel()`, `getZoomLevel()` or `setZoomLevel()` on a given `CaptureController` object produces a [permission prompt](https://developer.mozilla.org/en-US/docs/Web/API/Permissions). If the user grants permission, all future invocations of these methods on that specific CaptureController object are allowed. It is for that reason that these three methods return a [Promise](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise); naturally, if the user refuses permission, the returned Promise is rejected.

Note that CaptureControllers are uniquely associated with a given capture-session, cannot be associated with another capture-session, and do not survive navigation.

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

We define the following new actions:

```webidl
partial interface CaptureController {
  int getMinZoomLevel();
  int getMaxZoomLevel();

  Promise<int> getZoomLevel();
  Promise<undefined> setZoomLevel(int zoomLevel);

  attribute EventHandler oncapturedzoomlevelchange;
};
```

**getMinZoomLevel() / getMaxZoomLevel()**  
These return the min/max zoom levels supported by the implementation, represented as percentages of the "default zoom-level", which is defined as 100%.

**getZoomLevel()**  
Returns the current zoom-level of the captured tab.

**setZoomLevel()**  
Given an integer value between the values returned by `getMinZoomLevel()` and `getMaxZoomLevel()`, sets the zoom level to that value.

One way to use these methods is as follows:
```js
const zoomIncreaseButton = document.getElementById('zoomInButton');
zoomIncreaseButton.addEventListener('click', async (event) => {
  const oldZoomLevel = await controller.getZoomLevel();
  const newZoomLevel = Math.min(oldZoomLevel + 10, controller.getMaxZoomLevel());
  controller.setZoomLevel(newZoomLevel);
});
````

**oncapturedzoomlevelchange**  
Users may change the zoom-level either through the capturing application, or through direct interaction with the captured tab. If the capturing application is displaying any user-facing controls and UX element, such as an indicator of the current zoom-level, or buttons to increase/decrease zoom, then the application will want to listen to such changes and reflect them. The `oncapturedzoomlevelchange` event handler supports that.

An example of reasonable use of the event handler is as follows:
```js
captureController.addEventListener('zoomLevelChange', (event) => {
  const controller = event.target;
  const zoomLevel = controller.getZoomLevel();

  // Update label.
  zoomLevelLabel.innerHTML = `${zoomLevel}%`;

  // Update controls.
  zoomResetButton.enabled = (zoomLevel != 100);
  zoomIncreaseButton.enabled = (zoomLevel < controller.getMaxZoomLevel());
  zoomDecreaseButton.enabled = (zoomLevel > controller.getMinZoomLevel());
});
```

## Security and Privacy Concerns

### Permission prompts
Permission prompts are currently used as mitigations for Web Platform capabilities which are arguably even riskier than those presented in this document - clipboard access, geolocation, mic- and camera-access, and most notably, screen-capture itself. It follows that, if the prompt can be clear enough for the user, it should be a sufficient mitigation for the risks associated with the API surfaces we introduce.

### Risks and mitigations
#### User confusion
To obtain initial permission to use the API, and to keep on using it, an application does not need to show the user a video representation of the surface under control via a `<video>` element. Even if it does show such a `<video>` element, it is not guaranteed that the `<video>` element is connected to a [MediaStreamTrack](https://developer.mozilla.org/en-US/docs/Web/API/MediaStreamTrack) derived from the surface being captured, or that the MediaStreamTrack being used is not manipulated in some ways, e.g. using Breakout Box.

A browser-mediated connection between the captured surface and the permission prompt is infeasible. Instead, we rely on careful phrasing of the permission prompt as the mitigation for these risks.

#### Access to pixels beyond those originally intended by the user
When a user chooses to share a tab or a window, it might be that they only intended to share with the app the content immediately visible. The APIs we introduce will allow the capturing application to gain access to yet more content. This is both the core risk of the API as well as its intended use.

We mitigate using a permission prompt. The downsides are as usual:
* Dialog fatigue and the risk of the user just approving so as to be done with it.
* Impaired usability of Web apps that now require additional toil of their users.
* Difficulty in communicating to the user precisely what permission they are asked to grant.
* Difficulty in communicating to the user why such a permission could be risky.

The downsides are all known, and should be considered against those associated with alternative approaches (e.g. Video Portal).

#### Access at a time not controlled by the user
Discussed below. (TODO: Link to the transient activation part.)

#### Side effects
Delivering wheel events might have effects other than scrolling the page. If we consider some modern dating applications, we note that “swiping right” could have far-reaching consequences, and might even culminate in matrimony. However, the author of this document argues that the permission prompt is sufficient here, as it was for other Web Platform capabilities.

#### Scrolling third-party iframes when self-capturing
Applications that are capturing their own tab can load arbitrary third-party content in iframes and scroll it, thereby either gaining access to new content of their choosing, or producing arbitrary side effects. The mitigation of the permission prompt is still presented as sufficient here, as is the unlikelihood of third-party content only being sensitive after scrolling. If necessary, in the future, it is possible to also remove the ability to control the current tab.

### Transient activation
If transient activation is not required before each individual invocation of the APIs introduced by this document, then once the initial permission is obtained, the API can be used at any time, possibly even while the user is away from the device and cannot observe the effects of their change.

We argue that requiring transient activation is not desirable, as it renders impossible some legitimate use-cases, without actually contributing significantly to security.

#### Legitimate use cases blocked by transient activation requirement
Consider the following legitimate use case:
1. The local user participates in a video-conferencing session and chooses to share a tab.
2. Through interaction with the browser’s permission prompt, the local user allows scrolling and zooming of the captured tab by the video-conferencing application.
3. Through interaction with the video-conferencing application, the user allows a specific remote participant to scroll the captured tab - when a remote user sends a request to scroll, the request gets translated by the local application to a sendWheel() invocation.
4. The remote user can now control a local presentation, removing the need for the remote user to repeatedly ask the local user - “next slide, please.” This is a major boon to such applications, where this is a critical user journey.

#### Insufficiency of transient activation requirement to protect the user
On the one hand, consider the following *potential attack*:
1. Obtain the user’s permission to capture another tab.
2. Use arbitrary user gestures to scroll the captured tab and change its zoom-level.

While executing the aforementioned attack, either avoid presenting a preview-tile to the user, or show a frozen preview-tile containing an older frame, hiding from the user the scrolling of the captured tab.

#### Conclusion
It is arguable that the legitimate use case described above is risky. However, we argue that it’s up to the application to only deploy it according to the local user’s genuine intentions.

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

### How does the zoom-level correspond to CSS zoom?

It is not currently clear that it does, but we're investigating the issue in light of [recent activity](https://github.com/w3c/csswg-drafts/issues/5623#issuecomment-1646125737) on that front.
