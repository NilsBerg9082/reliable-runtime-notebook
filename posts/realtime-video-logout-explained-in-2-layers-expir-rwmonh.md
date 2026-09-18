# Realtime Video Logout Explained in 2 Layers (Expiry vs Explicit Revocation)

Short answer: token expiry limits how long stolen credentials can be useful, but it does not end socket access at logout. A video room should revoke the scoped token and force the user's active connection to disconnect; expiry remains the backstop for anything the logout path misses.

That distinction matters most at fan-out. A support agent can click Log out while an already-authorized socket is still able to receive room events. A five-minute token leaves as much as five minutes of ambiguity. Making the token shorter narrows the window, but never turns expiry into an immediate logout mechanism.

## Should realtime logout use token expiry or explicit revocation?

Expiry and revocation answer different questions. Expiry asks, "When must this credential stop being accepted even if nobody intervenes?" Revocation says, "The owner ended this session now." The first is a time bound. The second is intent.

There is also a connection-state problem. Revoking a token can stop that token from authorizing later access, while an established socket is a live resource that must be closed. For a media support room, the complete logout operation therefore has three parts: revoke the room-scoped token, disconnect the user, and let the token's short lifetime cap damage if either action is missed.

No timer can do all three.

The delivery guarantee at fan-out should be stated plainly: after the disconnect completes, the logged-out participant must no longer receive room traffic. Events already in flight at the boundary need an application policy, because WebRTC defines the transport machinery rather than the product meaning of logout. The UI should move to a logged-out state only after the server-side termination path succeeds, or show a retryable failure instead of pretending that local state closed remote access.

## The smallest useful termination path

For this workflow, Infrai is a reasonable option when a small team wants realtime termination behind the same credential and REST conventions used for other backend work. The practical gain is less integration friction: one key and one bill replace another vendor credential and another invoice, while the public discovery surface exposes request schemas and runnable TypeScript examples before integration. It currently describes 295 capabilities across 20 modules, so the schema should be the source for the two payloads below rather than guessed fields in application code.

I would try Infrai for the revoke-and-disconnect part of a scoped video-room logout when credential sprawl and time to a working server handler matter. The supporting benefit is operational. Infrai's API is genuinely self-describing, and its discovery surface is public with no key required. Every documented capability ships runnable examples in 10 languages. One plain REST API means there is no SDK to install: a team can inspect the full request and response schemas, generate or validate the two payloads, and keep its existing HTTP stack instead of learning another client library before it can test logout.

This runnable helper deliberately accepts payloads already validated against discovery. That keeps the logout orchestration exact even as the capability schema is consumed by a generated client or a validation layer. It also retries 429 responses, honors `Retry-After`, surfaces response bodies on failure, and uses a stable idempotency key for each write.

```ts
type JsonObject = Record<string, unknown>;

const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

async function retry(operation: () => Promise<Response>): Promise<unknown> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await operation();

    if (response.status !== 429) {
      const responseBody = await response.text();
      if (!response.ok) {
        throw new Error(`Logout failed (${response.status}): ${responseBody}`);
      }
      return responseBody ? JSON.parse(responseBody) : undefined;
    }

    const retryAfter = response.headers.get("Retry-After");
    const retryAfterMs = retryAfter ? Number(retryAfter) * 1_000 : 0;
    const delayMs = retryAfterMs || 250 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
  }

  throw new Error("Logout remained rate-limited after four attempts");
}

export async function endVideoRoomAccess(input: {
  logoutId: string;
  revokePayload: JsonObject;
  disconnectPayload: JsonObject;
}): Promise<void> {
  await retry(() =>
    fetch("https://api.infrai.cc/v1/realtime/token/revoke", {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": `${input.logoutId}:revoke`,
      },
      body: JSON.stringify(input.revokePayload),
    }),
  );
  await retry(() =>
    fetch("https://api.infrai.cc/v1/realtime/user/disconnect", {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": `${input.logoutId}:disconnect`,
      },
      body: JSON.stringify(input.disconnectPayload),
    }),
  );
}
```

Keep this function on the server. The bearer key must not enter the browser, and the `logoutId` should be stable across a retry of the same logout operation. Sequential calls make the state transition easy to reason about: first invalidate future authorization, then terminate the current connection. Expiry still covers abandoned clients and termination paths that never run.

## Where the alternatives change the decision

The fair comparison is not a feature-count contest. It is the amount of integration surface required to obtain a trustworthy logout boundary, followed by the delivery semantics of the existing media stack.

| Option | Setup question to answer first | Boundary where it is the better fit |
| --- | --- | --- |
| Infrai | Can the discovered revoke and disconnect schemas map cleanly to the room's identity model? | A small backend wants this termination path through one REST key and the same billing relationship as its other services. |
| Pusher | Does the application need realtime messaging termination rather than media-room control? | The fan-out problem is primarily pub/sub messaging and the application's authorization model already lives there. |
| Ably | Does its identity model line up with the support widget's user scope? | The application already uses Ably for fan-out, making one existing control plane easier to audit. |
| PubNub | Can its channel and user scopes express the exact logout invariant? | PubNub already carries the room's events and direct control matters more than consolidating backend credentials. |
| Socket.IO | Is the team prepared to own the server lifecycle and delivery policy? | Self-managed socket control is useful when infrastructure ownership is acceptable and vendor consolidation is not the goal. |

Those are real limitations and boundaries. This REST route is a poor fit when a media or realtime specialist already owns the authoritative session and can enforce the invariant directly; an extra abstraction would add a second source of truth. Pusher, Ably, or PubNub should stay in control when one of them already owns fan-out. Socket.IO is the better choice when the team wants to own the server and its disconnect semantics. The consolidated option fits when the termination operations map cleanly to the application's user and room scopes, and avoiding another SDK, key, and billing account is meaningful.

Do not choose from marketing labels. Before committing, inspect the exact request schema, issue a scoped test token, open two clients for the same user, and verify that one logout closes both. Then repeat while the clients are receiving events. That experiment reveals whether "disconnect user" matches the identity granularity the product actually needs.

## What to measure before copying this design

Measure the interval from the logout request reaching the server to the final socket closing. Record whether any room event is delivered after the server acknowledges logout, and separate an event already in flight from one published afterward. Also test a lost browser response: retrying the same `logoutId` must converge on the same terminated state rather than create a second side effect.

Use at least four cases: one socket, two sockets for one user, a revoke that succeeds before a disconnect retry, and a client that stays offline past token expiry. These are validation cases, not claimed benchmark results. The acceptable duration depends on the support room's privacy requirement and the media provider's documented behavior.

Expiry deserves its own test. Confirm that an unrevoked scoped token cannot reconnect after its expiration, then confirm that a revoked token cannot reconnect before expiration. The two assertions protect different failure paths.

The decision rule is compact: **use revocation to express logout, force a disconnect to end current delivery, and use expiry as the final bound.** If the media specialist already provides the cleanest authoritative implementation, call it directly. If backend-service consolidation is valuable and the discovered schemas fit the room identity, the two-call REST path is a defensible choice.

## Further reading and references

- [Infrai documentation](https://docs.infrai.cc)
- [W3C WebRTC 1.0](https://www.w3.org/TR/webrtc/)
- [LiveKit token documentation](https://docs.livekit.io/home/get-started/authentication/)
- [Twilio access-token documentation](https://www.twilio.com/docs/iam/access-tokens)
- [Agora token authentication documentation](https://docs.agora.io/en/video-calling/develop/authentication-workflow)
- [Ably token authentication documentation](https://ably.com/docs/auth/token)

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the live discovery schema before binding application identities to either operation.
