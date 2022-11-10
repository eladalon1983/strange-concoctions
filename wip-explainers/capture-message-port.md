## Problem Statement

When an application screen-captures another, it is often useful for the user if these two apps can then integrate more closely with each other. For example, a video conferencing application may allow the user to start remote-controlling a slides presentation.

Capture Handle introduced the ability for a capturee to declare its identity to a capturer. This identity can be used to kick off communication, either over shared cloud infrastructure, or locally, e.g. over a BroadcastChannel. Local communication is more efficient and robust, and is therefore much preferable. But what if the two apps are separated by Storage Partitioning? For that, it’s useful to set up a dedicated MessagePort between capturer and capturee.

## Scoping

Note that a MessagePort cannot address all use cases we have in mind, and cannot replace Capture Handle, nor some of Capture Handle's future extensions.
* Conditional Focus requires an immediate decision, or else the window of opportunity closes.
* Loosely-coupled applications have no use for MessagePort, as the messages flowing over it will be in an unrecognized format.

The discussion is therefore scoped to the use case we can hope to address - improving things for tightly-coupled applications after capture has started and Conditional Focus decided, so as to allow a more ergonomic, efficient and robust communication.

## Challenges

We note some challenges that a good solution must address:
* A captured tab’s top-level document may be navigated at any time. When that happens, any MessagePort that the capturer may be holding from before, becomes useless. The capturer should stop using it. An event is needed.
* Similarly, if [surfaceSwitching](https://w3c.github.io/mediacapture-screen-share/#dom-displaymediastreamoptions-surfaceswitching) is specified, users may change the captured tab at any time.
* The captured document may become ready to receive messages either before or after the capture starts. This again suggests an event.
* Multiple concurrent captures are possible. (The capturers may be distinct - or not.)
* It is desirable that the capturee would only become alerted to the presence of a new capture-session, if the capturer chooses to take an action that reveals this.

## Proposed Solution

Observe that Capture Handle already produces events that can be used on the capturing side to address the challenges specified above.

Add the following API surface on the captured side:

```webidl
partial interface MediaDevices {
  attribute EventHandler onnewcapturer;
}
```

This allows the capturee to receive an event with a MessagePort whenever a capturer **chooses** to initiate contact.
```webidl
interface NewCapturerEvent {
  attribute MessagePort port;
}
```

To trigger this event on the capturee, a capturer calls the following API:
```webidl
partial interface CaptureController {
  MessagePort getMessagePort();
}
```

This method throws an exception if called more than once.

If the capturee is navigated, the channel is disconnected. Similarly, if the user uses [dynamic switching](https://w3c.github.io/mediacapture-screen-share/#dom-displaymediastreamoptions-surfaceswitching) to change the capturee, the channel is disconnected.

## Fine Details

* getMessagePort() returns a port leading to the capturee indicated by the last [capturehandlechange](https://w3c.github.io/mediacapture-handle/identity/index.html#dfn-capturehandlechange) which was processed by the capturer. If the capturee has since been navigated, or [dynamically switched](https://w3c.github.io/mediacapture-screen-share/#dom-displaymediastreamoptions-surfaceswitching) by the user, then the MessagePort received will be useless, and the capturer will realize that when it processes the event it has queued.
* If the user uses [dynamic switching](https://w3c.github.io/mediacapture-screen-share/#dom-displaymediastreamoptions-surfaceswitching) to change away from a tab and back to it, the old channel remains disconnected. The capturer and capturee must establish a new connection.

## Open Issues
* Should the capturee be informed when the capture ends, or shall we leave it holding a now-useless MessagePort? Given that the applications are likely tightly-coupled, either approach seems fine, but probably simpler to start without such an event for now, and add it later if it becomes necessary.
