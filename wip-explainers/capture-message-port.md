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
* The captured document may become ready to receive messages either before or after the capture starts. This again suggests that thhe capturer needs an event.
* Multiple concurrent captures are possible. (The capturers may be distinct - or not.)
* It is desirable that the capturee would only become alerted to the presence of a new capture-session, if the capturer chooses to take an action that reveals this.

## Proposed Solution

Observe that Capture Handle already produces events that can be used on the capturing side to address the challenges specified above.


Extend [CaptureHandleConfig](https://w3c.github.io/mediacapture-handle/identity/index.html#dom-capturehandleconfig) with an event handler:
```webidl
partial dictionary CaptureHandleConfig {
  EventHandler newCapturerEventHandler;
};
```

This allows the capturee to receive a dedicated event with a MessagePort whenever a capturer **chooses** to initiate contact.

```webidl
interface NewCapturerEvent {
  attribute Type type;  // "started" or "stopped"
  attribute MessagePort port;
}
```

A channel is established for the capturee when it gets a NewCapturerEvent with `type` set to "started". When the session ends, the capturee gets a new event with the very same port, but with `type` now set to "stopped".


To trigger the "started" event on the capturee, a capturer calls the following API:
```webidl
partial interface CaptureController {
  MessagePort getMessagePort();
}
```

To check if it makes sense to call getMessagePort(), the capturer must check `CaptureHandle.supportsMessagePort`.
```webidl
partial dictionary CaptureHandle {
  boolean supportsMessagePort;
};
```
The value of `CaptureHandle.supportsMessagePort` is determined by whether the capturee has set a handler or not.

The capturee may change the CaptureHandleConfig **without** breaking off existing channels.

The channel **is** broken if:
* The capture-session ends for whatever reason. (User-initiated or app-initiated.)
* The capturee is navigated.
* The user uses [dynamic switching](https://w3c.github.io/mediacapture-screen-share/#dom-displaymediastreamoptions-surfaceswitching) to change the captured surface.

We extend the [capturehandlechange](https://w3c.github.io/mediacapture-handle/identity/index.html#dfn-capturehandlechange) event to help the capturer distinguish non-channel-breaking events from channel-breaking events.
```webidl
interface CaptureHandleChangeEvent {
  attribute boolean messagePortInvalidated;
}
```

## Fine Details

* getMessagePort() throws if `!getCaptureHandle().supportsMessagePort`.
* getMessagePort() returns a port leading to the capturee indicated by the last [capturehandlechange](https://w3c.github.io/mediacapture-handle/identity/index.html#dfn-capturehandlechange) which was processed by the capturer. This MessagePort might already be useless, e.g. if the captured tab has been asynchronously navigated. This will be detected by the capturer when it processes the relevant event.
* If the user uses [dynamic switching](https://w3c.github.io/mediacapture-screen-share/#dom-displaymediastreamoptions-surfaceswitching) to change away from a tab and back to it, the old channel remains disconnected. The capturer and capturee may establish a new connection if they still want to talk.

## Open Issues
* Should the capturer be allowed to call getMessagePort() multiple times and establish multiple connections with the same capturee? That could potentially mislead the capturee as to how many capture-sessions there are. However, that seems like a niche concern, especially given that the apps are tightly-coupled.
