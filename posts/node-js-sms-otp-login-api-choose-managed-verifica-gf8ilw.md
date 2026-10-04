# Node.js SMS OTP Login API: Choose Managed Verification over Custom Codes

TL;DR: Choose a managed verification flow over custom code generation when integration effort is the deciding constraint, but keep cooldowns, recipient suppression, and login policy in your own service. Build the code state machine yourself only when you must control delivery routing or authentication evidence end to end and can also own atomic consumption, abuse limits, and on-call diagnostics. A send receipt is not proof of delivery, and another resend is not a recovery plan.

Consider a bounded incident scenario in a developer tool: a user requests a login code, mistypes it twice, presses resend three times, and later corrects a phone number that cannot receive messages. The dangerous implementation treats each request as independent. It creates several valid codes, spends delivery capacity on a bad destination, and leaves the responder with a provider dashboard full of accepted sends but no answer to the first operational question: what page fired?

I would write the postmortem invariant this way: one login challenge has one server-owned state transition path, regardless of who generates or transports the code. That rule matters more than SDK convenience.

## How should a Node.js SMS OTP login API limit repeat sends?

The first failure is concurrency. Two requests can both observe that the cooldown has expired, both send, and both update state unless the decision is serialized. An in-process timer cannot enforce a cooldown across replicas, and a client-side disabled button cannot stop a direct request. The gate belongs in shared server-side storage.

The cooldown is security state.

The second failure is ambiguous identity. Rate limits tied only to an IP address punish offices and carrier gateways; limits tied only to a phone number let an attacker distribute attempts across many targets. A defensible policy evaluates several keys, such as account, normalized destination, challenge, and network source, while avoiding logs that expose the code itself. NIST SP 800-63B requires rate limiting when an authenticator output has less than 64 bits of entropy, requires an out-of-band secret to be accepted only once, and sets a maximum validity period of 10 minutes. Those are ceilings and invariants, not a recommendation to permit 10 minutes or wait until the broadest allowed attempt count.

Then delivery failure enters the loop. A permanent invalid-recipient result should suppress further automated sends to that normalized destination until a deliberate correction or reverification clears the suppression. A temporary delivery problem should have a bounded retry policy. Treating both as resend prompts converts a data-quality problem into an abuse and paging problem.

Retries need memory.

No dashboard fixes that.

## Managed verification or a custom state machine

A managed flow reduces the amount of security-sensitive code your team must integrate: code creation, expiry, and comparison can sit behind one verification operation. The trade-off is a wider external control boundary; your service still needs a local challenge identifier, authorization checks, cooldown enforcement, suppression state, and useful event correlation. If an outage occurs, responders need to distinguish a rejected local request from an accepted delivery request and a failed verification without relying on screenshots from a separate console.

A custom flow gives you control over message routing, data retention, and the precise evidence attached to each transition. It also makes every race yours. Hashing codes at rest, using cryptographically secure randomness, preventing replay, expiring challenges, limiting guesses, redacting telemetry, and handling concurrent resend and verify requests all become application responsibilities. That is a poor exchange when the primary goal is low integration effort.

| Decision condition | Managed verification | Custom codes |
|---|---|---|
| Small team, shared pager, ordinary login | Prefer it; keep policy local | Extra state and review burden |
| Required control of routing or evidence | May hide needed transitions | Prefer it if the team owns the full lifecycle |
| Multiple delivery channels | Confirm semantics channel by channel | One state model can coordinate channels |
| Strict portability requirement | Put a narrow adapter at the boundary | Avoid transport details in challenge logic |

The recommendation has a boundary. Do not choose managed verification merely because its happy-path sample is short. Choose it when its failure states can be mapped into your own stable result model, it exposes enough correlation data for incident response, and its retry semantics do not bypass your local cooldown. Otherwise, the integration is small only on launch day.

## Put the preventative decision in one transaction

The useful unit is not `sendSMS`; it is `BeginOrResendChallenge`. The following Go sketch leaves transport behind an interface and puts the decision beside durable state. The store implementation must make `Update` atomic for a normalized recipient, using a database transaction, conditional write, or equivalent serialization mechanism.

```go
package otp

import (
    "context"
    "errors"
    "time"
)

var (
    ErrCooldown   = errors.New("resend cooldown active")
    ErrSuppressed = errors.New("recipient suppressed")
)

type Challenge struct {
    ID            string
    Recipient     string
    NextSendAfter time.Time
    ExpiresAt     time.Time
}

type Store interface {
    Update(ctx context.Context, recipient string, fn func(*Challenge) error) (Challenge, error)
}

type Sender interface {
    StartVerification(ctx context.Context, challengeID, recipient string) error
}

type Suppressions interface {
    IsSuppressed(ctx context.Context, recipient string) (bool, error)
}

type Service struct {
    Store        Store
    Sender       Sender
    Suppressions Suppressions
    Cooldown     time.Duration
    Lifetime     time.Duration
    NewID        func() string
    Now          func() time.Time
}

func (s Service) BeginOrResend(ctx context.Context, recipient string) (Challenge, error) {
    blocked, err := s.Suppressions.IsSuppressed(ctx, recipient)
    if err != nil {
        return Challenge{}, err
    }
    if blocked {
        return Challenge{}, ErrSuppressed
    }

    now := s.Now()
    challenge, err := s.Store.Update(ctx, recipient, func(c *Challenge) error {
        if now.Before(c.NextSendAfter) {
            return ErrCooldown
        }
        c.ID = s.NewID()
        c.Recipient = recipient
        c.NextSendAfter = now.Add(s.Cooldown)
        c.ExpiresAt = now.Add(s.Lifetime)
        return nil
    })
    if err != nil {
        return Challenge{}, err
    }

    if err := s.Sender.StartVerification(ctx, challenge.ID, recipient); err != nil {
        return Challenge{}, err
    }
    return challenge, nil
}
```

This sketch intentionally does not roll back the cooldown when transport fails. Automatic rollback can turn a dependency outage into an unbounded resend storm. The user may see a bounded retry delay, which is a real availability cost; the alternative is letting repeated requests amplify the failing dependency. Record the delivery result against the challenge and make the retry decision from classified evidence, not from impatience.

Verification needs the same discipline. Compare through the verification boundary, increment failed-attempt state atomically, and consume the challenge in the transaction that establishes the authenticated session. A successful comparison followed by a separate, fallible `used = true` write leaves a replay window. Fast code is irrelevant there.

## Test the races, then test the page

A unit test that sends one code and verifies it once covers the path least likely to wake anyone. Run concurrent resend requests and assert that only one crosses the transport boundary. Race a valid verification against itself and assert that only one session is created. Advance a fake clock across the cooldown and expiry boundaries. Feed permanent invalid-recipient results into the suppression handler, then prove that another automated send is rejected before transport is called.

Test the collision.

The event trail should answer a short sequence without storing secrets: which challenge accepted the request, which policy rejected or allowed it, whether transport accepted it, whether a delivery-status event was correlated, and whether verification consumed the challenge. Use opaque challenge IDs and coarse result classes. Do not log plaintext codes, full phone numbers, or arbitrary provider payloads.

Alert on user harm or a broken control: sustained verification failure changes, suppression growth, transport rejection changes, or a mismatch between accepted challenges and downstream outcomes. A raw count of sends rising is context, not automatically a page. The responder should be able to move from the alert to correlated events and decide whether the fault is policy, storage, transport, or recipient data. If the page cannot name that boundary, it is unfinished.

For email fallback, keep authentication state separate from deliverability state. DMARC defines policy and reporting for domain-level email authentication; it does not tell an application that a mailbox is valid, nor does it replace bounce processing. A permanent mailbox failure can therefore suppress that email destination without consuming or silently rerouting the SMS challenge. Mixing those states makes recovery convenient to implement and hard to explain during an account-takeover review.

## Where this choice stops applying

Use a custom state machine when regulation, threat modeling, routing control, or evidence retention requires transitions the managed boundary cannot expose. The team must then budget for adversarial tests, storage consistency, key and secret handling, delivery callbacks, privacy review, and pager ownership. That is an operating commitment, not an SDK selection.

For a developer tool optimizing for integration effort, the default remains managed verification behind a narrow interface, with local cooldown and suppression policy. The final acceptance test is operational: one challenge, one consumable outcome, bounded retries, and enough evidence to identify the failed boundary without trusting a dashboard.

## Sources

- NIST, *Digital Identity Guidelines: Authentication and Lifecycle Management (SP 800-63B)*: https://pages.nist.gov/800-63-3/sp800-63b.html
- IETF, *RFC 7489: Domain-based Message Authentication, Reporting, and Conformance (DMARC)*: https://datatracker.ietf.org/doc/html/rfc7489
