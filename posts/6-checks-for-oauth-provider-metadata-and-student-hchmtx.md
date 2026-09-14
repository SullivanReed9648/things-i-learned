# 6 Checks for OAuth Provider Metadata and Student Identity Resolution

For an OAuth provider strategy, accept discovery simplicity only after proving a harder identity-resolution contract: every returning student must resolve to the same internal account without trusting a mutable email address.

Short answer: discovery should be boring and standards-based; identity resolution should be explicit, issuer-scoped, tested against collisions, and kept separate from session policy. For an edtech app that already supports email and password, this preserves a low-friction sign-in path without letting convenience silently weaken account ownership.

The data flow is small enough to describe in one breath. A learner selects an OAuth login, the app discovers the provider's authorization metadata, sends the browser through authorization with PKCE and state, validates the returned identity assertion, maps the pair of issuer and subject to a local user, and then creates its own application session. Password login reaches that same local-user boundary through a different verifier. The two entrances converge only after authentication.

## 1. What Should an OAuth Provider Strategy Test for Discovery and Identity Resolution?

Test metadata and identity as different failure domains. Discovery answers where to authorize, exchange a code, and find signing keys. OpenID Connect identity answers who authenticated, using the issuer (`iss`) and subject (`sub`) claims together. An email can help the interface, but it isn't a durable cross-provider primary key.

That separation matters in a school roster. Suppose `maya@district.example` first signs in with a district identity and later arrives through a second provider carrying the same address. Automatically joining those records treats matching text as proof of common ownership. Instead, create or locate an external identity by `(issuer, subject)`, attach it to exactly one local user, and require a separately authenticated linking ceremony before two identities converge. Don't make the callback improvise this decision.

**The core rule is simple:** authentication establishes an external identity; linking changes account ownership and deserves its own authorization check.

## 2. How Can You Test Identity Before Comparing Provider Features?

The following Python is deliberately a domain-layer example, not a provider SDK tutorial. It makes the invariant executable: the same issuer and subject return the same learner, a claimed email never causes an automatic merge, and an already-bound external identity can't be moved by an ordinary sign-in.

```python
from dataclasses import dataclass, field


class IdentityConflict(Exception):
    pass


@dataclass(frozen=True)
class VerifiedIdentity:
    issuer: str
    subject: str
    email: str | None
    email_verified: bool


@dataclass
class Learner:
    user_id: str
    password_email: str | None = None
    external_ids: set[tuple[str, str]] = field(default_factory=set)


class IdentityDirectory:
    def __init__(self) -> None:
        self.learners: dict[str, Learner] = {}
        self.external_index: dict[tuple[str, str], str] = {}

    def resolve_sign_in(self, identity: VerifiedIdentity) -> Learner | None:
        key = (identity.issuer, identity.subject)
        user_id = self.external_index.get(key)
        return self.learners.get(user_id) if user_id else None

    def link_after_reauthentication(
        self, user_id: str, identity: VerifiedIdentity
    ) -> Learner:
        key = (identity.issuer, identity.subject)
        existing_owner = self.external_index.get(key)
        if existing_owner is not None and existing_owner != user_id:
            raise IdentityConflict("external identity already belongs to another user")

        learner = self.learners[user_id]
        learner.external_ids.add(key)
        self.external_index[key] = user_id
        return learner


def exercise_contract() -> None:
    directory = IdentityDirectory()
    directory.learners["student_1042"] = Learner(
        user_id="student_1042",
        password_email="maya@district.example",
    )
    district_identity = VerifiedIdentity(
        issuer="https://login.district.example",
        subject="00u-7f91",
        email="maya@district.example",
        email_verified=True,
    )

    # A matching email still does not authorize an account merge.
    assert directory.resolve_sign_in(district_identity) is None
    directory.link_after_reauthentication("student_1042", district_identity)
    assert directory.resolve_sign_in(district_identity).user_id == "student_1042"


exercise_contract()
```

In production, `VerifiedIdentity` must be constructed only after protocol validation. For an OpenID Connect ID token, that includes verifying the signature and expected issuer, audience, and nonce; exact requirements depend on the flow and client type. OAuth alone is an authorization framework, so don't infer an authenticated person from an access token unless the provider defines a suitable identity protocol and your validation follows it. Keep that distinction visible in code review.

There is another useful negative test: two issuers may legally emit the same subject string. The composite keys stay different. Conversely, one issuer may give two different subject values for two client contexts, depending on its subject identifier policy, so an architecture that expects global portability should verify the provider's documented behavior during evaluation. I'm not sure any paper questionnaire can settle that for every tenant; a test tenant with captured, redacted claims is the evidence that resolves it.

## 3. Compare Six Checks With Evidence, Not a Feature Grid

1. Discoverability. Prefer published authorization-server or OpenID Provider metadata over endpoints copied into application settings. Validate the returned issuer exactly, restrict accepted schemes and hosts according to your trust configuration, cache metadata with a bounded refresh policy, and test signing-key rotation. Discovery reduces configuration work; it doesn't remove the need to decide which issuers are trusted.

2. Stable identity keys. Confirm that the provider supplies an issuer and subject with semantics your login protocol defines. Store both without lowercasing, trimming, or substituting email. The database should enforce uniqueness on the pair. This is a tiny schema choice with a huge blast radius.

3. Safe linking and recovery. Ask what happens when a learner already has a password account, changes schools, loses access to an institutional identity, or signs in through a guardian-managed address. Linking should require a fresh proof from the existing account or an equivalently strong recovery process. OWASP recommends generic authentication responses so an attacker can't use login and recovery screens to enumerate accounts; apply that rule to linking errors too.

4. Authorization-code defenses. Require exact redirect URI handling, `state` for request correlation and CSRF defenses, and PKCE where the applicable security profile calls for it. OAuth 2.0 Security Best Current Practice says clients should use authorization code flows protected by PKCE and should not use the implicit grant. A provider's attractive login button isn't compensation for weak protocol controls.

5. Session boundaries. Decide locally how long a learner session lasts, when sensitive changes demand reauthentication, and how sessions are revoked after credential changes. NIST SP 800-63B treats session management as distinct from the authentication event and describes both overall and inactivity timeouts. For a classroom quiz, repeated prompts create real friction; for changing a recovery address, an old browser session shouldn't be enough. The policy belongs to the action, not to whichever login button the learner picked.

6. Operational evidence. Instrument outcomes without logging authorization codes, tokens, passwords, or raw claims. Useful counters include discovery refresh success, invalid issuer, invalid audience, nonce mismatch, unknown external identity, link requested, and link rejected. Give each callback a correlation identifier and record the provider alias rather than secrets. Then run the same contract suite in CI against recorded, sanitized claim shapes so a notebook experiment becomes a production gate — with no model calls or token spend hiding in the authentication path.

The catch is that standards-based discovery doesn't guarantee identical provider behavior. Metadata can be syntactically valid while tenant configuration, subject policy, logout support, or claim availability differs. A managed broker may reduce integration work when a small team must support many enterprise connections; direct integrations may be a better fit when the team needs precise control over claims and incident handling; a self-hosted identity layer may suit organizations able to own patching, key protection, monitoring, and on-call work. None of those choices eliminates the local identity map.

## 4. How Should Password and OAuth Login Share a Session Boundary?

Email-and-password authentication needs its own controls: password hashing, rate limiting, secure recovery, breached-password screening where appropriate, and generic failure messages. It should produce the same internal authentication result as a resolved external identity: a local user ID, authentication time, method, and assurance context. Downstream course and billing code shouldn't receive a provider token or branch on provider-specific claims.

This is where session security and friction can be tuned without corrupting identity resolution. Use a normal session for routine learning activity, then require recent authentication for linking another identity, changing recovery factors, or exporting protected student data. Rotate the session identifier after authentication and privilege changes, set cookies with `Secure`, `HttpOnly`, and an appropriate `SameSite` policy, and invalidate server-side sessions according to the application's risk model.

Short paths are good.

But a short path isn't the same as automatic account merging. If the product team wants a one-click first login, create a new local account after a validated external identity and collect only the profile fields the app actually needs. If policy requires matching a school roster before access, represent “authenticated but not roster-authorized” as a distinct state. That gives support staff a comprehensible case to resolve without rewriting who the person is.

No silent merge.

## 5. Ship With an Operational Review, Then Re-Evaluate

Before launch, walk one learner through password signup, password recovery, first OAuth login, returning OAuth login, explicit account linking, attempted duplicate linking, institutional access loss, and session revocation. Confirm that logs explain each outcome without exposing credentials; alerts distinguish a configuration problem from an individual denial; support tools show local and external identifiers without allowing silent reassignment; and rollback can disable one provider while leaving password access and other providers intact. Run these cases after metadata or key changes as well as application releases.

The decision is ready when the team can show evidence for all six checks and explain its linking policy in one paragraph. If the app has only one controlled institutional issuer and no pre-existing accounts, the mapping can stay narrow. If users bring personal and school identities, share email addresses, or frequently cross organizations, spend more design effort on linking and recovery than on the discovery client. Discovery is configuration. Identity resolution is account ownership.

## References

- https://www.rfc-editor.org/rfc/rfc6749
- https://www.rfc-editor.org/rfc/rfc8414
- https://www.rfc-editor.org/rfc/rfc9700
- https://openid.net/specs/openid-connect-core-1_0.html
- https://openid.net/specs/openid-connect-discovery-1_0.html
- https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
- https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html
- https://pages.nist.gov/800-63-4/sp800-63b.html
