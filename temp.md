## TL;DR

The proposed API will allow a website to capture an HTML element as a video stream. Only the target element and its descendant elements will be captured. Parent and sibling elements will not be captured, even if they draw over/under the target element.

```webidl
partial interface HTMLElement {
    MediaStream captureStream(optional double frameRequestRate);
};
```

In a way, this can be thought of as [HTMLCanvasElement.captureStream()](https://w3c.github.io/mediacapture-fromelement/#html-canvas-element-media-capture-extensions), expanded to any HTMLElement, with some additional security gating added to address the concern of leaking cross-origin content.


## Introduction

### State of the Art

The Web Platform currently offers the ability to screen-capture using [getDisplayMedia()](https://www.w3.org/TR/screen-capture/#mediadevices-additions). The resulting [MediaStream](https://www.w3.org/TR/mediacapture-streams/#dom-mediastream), composed of video and potentially also audio, can be manipulated locally and/or transmitted remotely. Some common use-cases include:
1. Video conferencing
2. Filing feedback
3. Storing the client-side rendering of a Web-app's audio/visual output

It is also possible to crop the resulting video using [Region Capture](https://w3c.github.io/mediacapture-region/). This is useful in multiple scenarios:
1. When video conferencing, cropping allows embedding the call in the same tab as captured content, without capturing the video conference itself.
2. When filing feedback, cropping helps remove irrelevant, and potentially private information, from the report.
3. When producing local recordings of an app's output, cropping lets the app remove its own UI elements if it deems them irrelevant to the captured content. For example, video-editing software can crop away the progress bar.

### Room for Improvement

The main technique available nowadays is to use getDisplayMedia() and Region Capture; in some ways, it's the only technique. One major downside is that this captures both occluding as well as occluded content.

Consider this example, where the progress bar is **occluding content** which is unintentionally captured:
<p align = "center">
<img src = "https://user-images.githubusercontent.com/22117736/196652116-713335ce-9ed8-4c50-95d7-3b03040c33ad.png">
</p>

Unintentionally capturing occluding content is usually the problem, but occluded content is also a concern if the target element has is partially transparent or has bevelled edges.

## Proposed Solution

Add a method along the lines of:

```webidl
partial interface HTMLElement {
    MediaStream captureStream(optional double frameRequestRate);
};
```

Initially, scope the discussion to a MediaStream with a single video MediaStreamTrack. Audio is an obvious follow-up, to be discussed later.

Successful invocations of this method are only possible from contexts that have:
1. Cross-origin isolation.
2. All captured content has opted in to being captured via a bespoke [Required Document Policy](https://wicg.github.io/document-policy/#required-document-policy). (That is, the target element and its descendants.)
3. Possibly some user prompt; to be discussed.

The second requirement could be tricky, as sibling elements could affect the shape of the target element, and so information could leak. As a start, we could go with requiring opt-in from all of the content in the page, and later explore if this requirement can be safely reduced in scope for elements whose rendering is unaffected by adjacent elements, perhaps via some CSS property or some other sandboxing property.

The necessity of a user prompt is debatable. Ideally, requirements 1 and 2 should ensure that any content captured, is either already known to the capturing page, or can be known to it via communication with embedded documents. (If those documents opted in to being captured, it is arguable that they would be willing to transmit such information.) However, some non-DOM information could leak, like link-purpling. It is therefore best to start with a required user prompt, and re-examine that requirement later.
