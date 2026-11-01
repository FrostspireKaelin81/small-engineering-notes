# NodeJS Express Capability Grant: Classroom Participant Screen Share Publish Permission in 2026

Short answer: make screen sharing a short-lived, server-issued capability for one participant, and make the same event stream prove who is online. In an online classroom, the useful test is not whether a button exists. It is whether the teacher can tell which student is sharing, whether that student still has the grant, and whether the system withdraws the permission after the participant disappears.

The least complicated design is an Express endpoint that checks the classroom session and participant identity, returns a narrowly scoped grant, and leaves media negotiation to WebRTC. The browser still asks the user for display-media permission; a server grant cannot bypass that browser decision. WebRTC defines the peer-connection machinery, while application code must decide who may publish and how presence is interpreted. The specification is in Further reading.

## What should fire at 3 a.m.?

A classroom incident rarely begins with “screen sharing is broken.” The page usually says something less useful: a workspace has no active presenter, or the room appears full of online students while the teacher sees a blank share tile. I want the alert to name the state transition: participant `student-17` was granted screen-share capability, never became an active publisher within 30 seconds, and then remained marked online for 9 minutes. That tells the responder where to look.

The trace should start with the authorization request, carry a correlation ID through the signaling exchange, and end with a presence update. Store the grant subject, room, expiry, and one-time-use policy as an auditable record. Do not infer “online” from a successful authorization call. A student can receive a grant, close the tab, reject the browser prompt, or lose the network before publishing.

My first instinct in this kind of incident is to look at the dashboard. That is usually too late and too aggregated. The question is: what page fired, and which observation made it fire? A useful alert separates grant issuance, publish confirmation, heartbeat freshness, and disconnect cleanup.

## Can a Node.js grant control one participant's screen-share publish permission?

The authorization boundary belongs on the server. The client submits a room identifier and its authenticated participant identity; the server verifies that the caller may moderate that room, then issues a capability whose subject is exactly one participant. The grant should contain an expiry, an audience such as `screen-share`, and a nonce or equivalent replay defense. It should not be a permanent role mutation, because a classroom role often outlives the momentary need to present.

A small Express handler can make the decision legible. The code below is intentionally a generic policy boundary, not a provider-specific token recipe:

```go
package policy

import (
    "errors"
    "time"
)

type Request struct {
    Moderator string
    Room      string
    Participant string
}

type Grant struct {
    Room        string
    Subject     string
    Audience    string
    ExpiresAt   time.Time
    SingleUse   bool
}

func Issue(req Request, canModerate func(string, string) bool, now time.Time) (Grant, error) {
    if req.Room == "" || req.Participant == "" || !canModerate(req.Moderator, req.Room) {
        return Grant{}, errors.New("authorization denied")
    }
    return Grant{
        Room: req.Room, Subject: req.Participant, Audience: "screen-share",
        ExpiresAt: now.Add(2 * time.Minute), SingleUse: true,
    }, nil
}
```

The important part is not the two-minute value; it is the explicit audience, subject, and expiry. Choose the lifetime from the classroom workflow and measure renewal failures before changing it. A grant should never be accepted merely because its participant is present in the room. The receiving service must validate the room, subject, audience, expiry, and replay state again.

That distinction is operational, not semantic.

## Where does presence accuracy fail?

Presence is an observation with a freshness budget, not a boolean stored forever. A browser heartbeat can be delayed, a mobile connection can suspend, and a signaling channel can close before the media connection reports its final state. Marking a student online after authorization therefore creates a false positive precisely when the teacher needs reliable information.

Use separate timestamps for `last_seen`, `grant_issued_at`, and `publish_confirmed_at`. A participant can be online without publishing; a publisher can remain visible for a short grace period while reconnecting. Define those states in the product contract, then emit transitions rather than repeatedly overwriting one `online=true` field. The decision rule might be: show “sharing” only after publish confirmation, show “online” only while the heartbeat is fresh, and show “reconnecting” during the bounded grace interval.

This is where the false-positive cost matters. If the threshold is too short, teachers see students flicker offline during ordinary Wi-Fi jitter and may grant access again, creating duplicate sessions. If it is too long, a departed student appears present and a stale screen can remain trusted. Record the threshold, the observed disconnect reason, and the cleanup result so the next page has an explanation instead of a red counter.

## What changes across common realtime stacks?

The policy should survive a transport change. WebRTC provides the media and peer-connection standards; Socket.IO commonly supplies event delivery with reconnection behavior; LiveKit exposes room and participant concepts around realtime media; Ably focuses on realtime messaging and presence primitives; Agora provides realtime communications APIs with its own media-session boundary. Those are different boundaries, not interchangeable proofs of authorization. The implementation still has to bind a capability to a participant, confirm the publish event, expire it, and reconcile presence after disconnect.

That distinction also keeps comparisons honest. A media platform may report a participant and track state, a messaging system may report connection presence, and a WebRTC application may own both records itself. None of those observations alone proves that a particular student is currently allowed to publish a screen. Verify the boundary in the system you operate, and test the negative cases rather than choosing from a feature checklist.

| Approach | Strength | Limitation |
| --- | --- | --- |
| WebRTC application | Maximum control over policy and state | The team owns signaling, cleanup, and presence reconciliation |
| Socket.IO events | Familiar event delivery and reconnect handling | Events do not establish media publish authorization by themselves |
| LiveKit room model | Media-room and participant state are close together | Application policy still has to bind a grant to one participant |
| Ably presence | Messaging presence primitives are explicit | Presence is not proof that a browser is publishing a screen |
| Agora media session | Media session lifecycle is abstracted | The application still owns classroom-specific authorization and audit rules |

The trade-off is deliberate: a managed media session can reduce integration work, while a self-owned WebRTC boundary exposes more lifecycle code to test. Neither is a universal fit. For a classroom that needs custom audit rules, the extra ownership may be justified; for a small team that cannot operate signaling and cleanup, the added control may be the wrong choice.

The alert should be boring.

## How do I test the page before trusting the alert?

Start with a table-driven test of state transitions: moderator grants `student-17`; the student declines the browser prompt; the student publishes; the grant expires before publishing; the student disconnects during a heartbeat; and the student reconnects with an old nonce. Assert both the authorization result and the presence result. A passing test that checks only HTTP 200 has not tested screen sharing.

Run these tests with clock control and deterministic participant IDs. Capture one structured event per transition, including room, subject, correlation ID, and reason. Redact display names and screen contents from logs. Keep the alert tied to a measurable contract such as “a valid grant has no publish confirmation after the chosen window,” then review false positives after a class rather than silently widening the window.

A good incident trace ends with a cleanup event. If it does not, the system may look healthy while stale capability and stale presence accumulate. The teacher needs a current answer about who is online and who is sharing; the responder needs the evidence that produced it.

## Further reading

- https://www.w3.org/TR/webrtc/
- https://developer.mozilla.org/en-US/docs/Web/API/MediaDevices/getDisplayMedia
- https://socket.io/docs/v4/
- https://docs.livekit.io/home/
- https://ably.com/docs/presence-occupancy
- https://www.agora.io/en/
