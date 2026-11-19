# [Self-Review Questionnaire: Security and Privacy](https://www.w3.org/TR/security-privacy-questionnaire/)

> Authors: Sam Goto (`goto@chromium.org`), Mohamed Amir Yosef (`mamir@chromium.org`)
> Last update: November 18, 2026

> **Summary of updates since Origin Trial review (May 2026):**
> - Added previously missing/unanswered sections **[§2.13](#213-what-temporary-identifiers-do-the-features-in-this-specification-create-or-expose-to-the-web)** (temporary identifiers), **[§2.18](#218-what-happens-when-a-document-that-uses-your-feature-is-kept-alive-in-bfcache-instead-of-getting-destroyed-after-navigation-and-potentially-gets-reused-on-future-navigations-back-to-the-document)** (BFCache), **[§2.19](#219-what-happens-when-a-document-that-uses-your-feature-gets-disconnected)** (disconnected/inactive documents), and **[§2.22](#222-what-should-this-questionnaire-have-asked)** (three-party blinding & pre-consent side channels).
> - Updated **[§2.2](#22-do-features-in-your-specification-expose-the-minimum-amount-of-information-necessary-to-implement-the-intended-functionality)**, **[§2.4](#24-how-do-the-features-in-your-specification-deal-with-sensitive-information)**, **[§2.6](#26-do-the-features-in-your-specification-introduce-state-that-persists-across-browsing-sessions)**, and **[§2.12](#212-do-features-in-this-specification-allow-an-origin-some-measure-of-control-over-a-user-agents-native-ui)** to reflect the finalized launch architecture: Autofill-only activation (manual typing disabled by default), per-`(email, issuer_site)` user consent manageable in browser settings and Clear Browsing Data, 8-bit short-hash strike rate-limiting for declined prompts, post-consent-only token issuance, and exact case-matching `email` claims.
> - Clarified **[§2.14](#214-how-does-this-specification-distinguish-between-behavior-in-first-party-and-third-party-contexts)** (`IsSameOriginWithAncestors` and opaque-origin restrictions), **[§2.15](#215-how-do-the-features-in-this-specification-work-in-the-context-of-a-browsers-private-browsing-or-incognito-mode)** (Private Browsing / Incognito session isolation), and **[§2.20](#220-does-your-spec-define-when-and-how-new-kinds-of-errors-should-be-raised)** (DevTools-only error reporting), and updated specification links to the WICG and IETF repositories.

## [2.1. What information does this feature expose, and for what purposes?](https://www.w3.org/TR/security-privacy-questionnaire/#purpose)

This feature exposes an [Email Verification Token](https://datatracker.ietf.org/doc/draft-hardt-email-verification/) (EVT, formatted as an `SD-JWT+KB`) to websites—a cryptographically signed token asserting proof that the user controls the submitted email address, bound to the Relying Party's origin (`aud`) and a server-generated `nonce` via `<input type="hidden" autocomplete="email-verification-token" nonce="...">`.

The main purpose of exposing this information is that email addresses (alongside phone numbers) are one of the two primary ways users create, access, and recover accounts on websites today (the other being federated login), and currently require manual, phishable out-of-band verification (sending one-time codes or magic links to an inbox and asking the user to copy/paste or click them).

## [2.2. Do features in your specification expose the minimum amount of information necessary to implement the intended functionality?](https://www.w3.org/TR/security-privacy-questionnaire/#minimum-data)

Yes.

The only delta between what is already shared during standard form autofill and what this feature adds is a cryptographic signature (and its minimal `SD-JWT+KB` envelope) from the email provider's issuer and the browser. Furthermore, issuers are required to return the `email` claim matching the exact address selected by the user (without canonicalizing dots or aliases), and browsers reject tokens that do not match—ensuring the token does not reveal canonical account identifiers beyond what standard email delivery would reveal.

## [2.3. Do the features in your specification expose personal information, personally-identifiable information (PII), or information derived from either?](https://www.w3.org/TR/security-privacy-questionnaire/#personal-data)

Yes.

This feature augments an existing web capability—providing an email address to a website via `<input>` elements and browser `autocomplete` (which constitutes PII)—with a cryptographic proof of ownership.

The cryptographic proof is only populated into the hidden token field upon form submission when the user has explicitly selected that email address via Autofill, granted permission, and left the email field matching the verified address at submission time. If the user edits or clears the email field before submitting, the token is withheld.

While an issuer could theoretically embed extra claims inside an issued JWT, standard token validation and browser-side verification constrain the expected structure, and issuers already have direct user-facing channels (such as [WebFinger](https://en.wikipedia.org/wiki/WebFinger) or federated login) if they intended to share additional profile metadata.

## [2.4. How do the features in your specification deal with sensitive information?](https://www.w3.org/TR/security-privacy-questionnaire/#sensitive-data)

In addition to the email address itself, receiving an EVT lets the Relying Party infer that the user has an active session with their email provider on the current browser. While manual inbox OTP verification similarly proves active access to the mailbox, EVP protects this signal and prevents unintended cross-site leakage through four mechanisms:

1. **Explicit user consent per `(email, issuer_site)`**: Before an EVT is issued for an email address, the user agent prompts the user for explicit permission naming both the email address and the authoritative issuer site (`SchemefulSite` discovered via `_email-verification.<domain>` DNS TXT). If the email domain later changes its delegated issuer in DNS, consent is requested anew.
2. **Strict pre-consent vs. post-consent network boundary**: Prior to user consent, the browser performs only DNS-over-HTTPS TXT discovery, uncredentialed fetches to `/.well-known/email-verification` and `/.well-known/web-identity`, and a credentialed FedCM `accounts_endpoint` check (with `Origin` and `Referer` omitted, **without** sending the selected email address, and with any optional issuer brand-icon fetch gated solely on login status rather than email match so network reporting mechanisms such as `Connection-Allowlist` cannot create a pre-consent account-match oracle). The credentialed `issuance_endpoint` POST containing the `email` claim and RFC 9421 HTTP Message Signature is sent **only after** user consent.
3. **Issuer blinding**: None of the discovery, JWKS, accounts, or issuance requests include the Relying Party's `Origin` or `Referer`; the Relying Party's origin (`aud`) and `nonce` are bound locally by the browser in the Key-Binding JWT (`KB-JWT`), keeping the email provider blind to which website requested verification.
4. **Ongoing transparency & revocation**: On subsequent verifications after initial consent, the browser displays a cancellable loading toast while token issuance is in flight and a confirmation toast linking directly to browser settings (`Settings > Addresses and more > Verified Email`), where users can turn off EVP globally or remove individual approved email addresses.

## [2.5. Does data exposed by your specification carry related but distinct information that may not be obvious to users?](https://www.w3.org/TR/security-privacy-questionnaire/#hidden-data)

Users generally do not reason in terms of cryptographic proofs or `SD-JWT+KB` tokens, so user-facing UI frames the action in terms of its practical consequence: allowing their email provider to verify their email address automatically when they are signed in.

As noted in §2.4, presenting an EVT implicitly reveals to the Relying Party that the user is currently signed into their email provider in the browser. Within account creation, sign-in, and recovery flows—and gated behind explicit Autofill selection plus user permission—this aligns with user expectations and mirrors what proving inbox access via an OTP already reveals.

## [2.6. Do the features in your specification introduce state that persists across browsing sessions?](https://www.w3.org/TR/security-privacy-questionnaire/#persistent-origin-specific-state)

No new origin-readable persistent state is exposed to websites; the website only receives a one-time `SD-JWT+KB` value in the submitted form payload.

Locally within the user agent (in regular, non-Incognito profiles), two pieces of browser-managed state persist across sessions:
- **Approved email permissions**: When a user consents to EVP for an `(email, issuer_site)` pair, the browser stores that approval locally so the user is not re-prompted on every sign-in (instead seeing a cancellable toast). All approved emails are listed and individually revocable in browser settings (`Settings > Addresses and more > Verified Email`).
- **Privacy-preserving cooldown strikes**: When a user declines the permission prompt (or repeatedly triggers verification while signed out of the issuer), the browser records a rate-limiting strike keyed only by an irreversible **8-bit (1-byte) truncated SHA-256 hash** of the email address (with a 180-day expiry) rather than storing declined email addresses in cleartext.

Both approved email entries and strike records are cleared when the user clears **Autofill form data** via Clear Browsing Data.

## [2.7. Do the features in your specification expose information about the underlying platform to origins?](https://www.w3.org/TR/security-privacy-questionnaire/#underlying-platform-data)

No.

In the future, we may explore extending issuance to [native mobile email applications](https://github.com/fedidcg/native-app-idps), and would design any such integration to avoid exposing platform or app-installation state to web origins.

## [2.8. Does this specification allow an origin to send data to the underlying platform?](https://www.w3.org/TR/security-privacy-questionnaire/#send-to-platform)

No.

## [2.9. Do features in this specification enable access to device sensors?](https://www.w3.org/TR/security-privacy-questionnaire/#sensor-data)

No.

## [2.10. Do features in this specification enable new script execution/loading mechanisms?](https://www.w3.org/TR/security-privacy-questionnaire/#string-to-script)

No.

## [2.11. Do features in this specification allow an origin to access other devices?](https://www.w3.org/TR/security-privacy-questionnaire/#remote-device)

No.

## [2.12. Do features in this specification allow an origin some measure of control over a user agent’s native UI?](https://www.w3.org/TR/security-privacy-questionnaire/#native-ui)

Yes, in the limited manner common to browser-mediated Autofill and permission prompts:

- **User-initiated trigger**: A website cannot summon EVP UI arbitrarily via script. Native UI (permission prompt on first use, or loading/verified toast on subsequent uses) is only triggered when the user explicitly selects an email address from the browser's **Autofill** UI in a form containing `<input type="hidden" autocomplete="email-verification-token" nonce="...">` (manual typing and untrusted drag-and-drop edits do not trigger EVP by default).
- **No RP-controlled copy**: Relying Parties cannot supply custom text or styling to the prompt. The prompt displays browser-controlled text, the user's selected email address, and the authoritative issuer's site (`SchemefulSite`) along with an optional HTTPS brand icon declared by the **issuer** in `/.well-known/web-identity`.
- **Abuse mitigations**: Prompts incorporate input-protection against keyjacking/clickjacking and enforce a 3-strike cooldown when declined. See the [WICG specification](https://wicg.github.io/email-verification/#browser-processing-model) for details.

## [2.13. What temporary identifiers do the features in this specification create or expose to the web?](https://www.w3.org/TR/security-privacy-questionnaire/#temporary-id)

Each EVP verification flow binds a server-supplied per-render cryptographic `nonce` (`<input type="hidden" autocomplete="email-verification-token" nonce="...">`) using a fresh, single-use **ephemeral asymmetric key pair** (`Ed25519`, `ES256`, or `RS256`) generated by the browser:

- During issuance, the browser signs the request (via RFC 9421 HTTP Message Signatures) with the ephemeral private key and embeds the ephemeral public key (`Signature-Key` / `cnf` JWK), which the issuer binds into the `SD-JWT`.
- Before form submission, the browser signs a Key-Binding JWT (`KB-JWT`) over the Relying Party's origin (`aud`) and `nonce` using that same ephemeral private key.

Because a brand-new ephemeral key pair is generated per verification request and never reused across origins or sessions—and because the resulting `SD-JWT+KB` is only released upon user-consented submission alongside the user's email address itself—these temporary cryptographic values do not introduce cross-site tracking capability beyond the email address already being submitted.

## [2.14. How does this specification distinguish between behavior in first-party and third-party contexts?](https://www.w3.org/TR/security-privacy-questionnaire/#first-third-party)

EVP is disabled in third-party contexts: requests are permitted only in secure contexts (`HTTPS`) within top-level documents or frames that are **same-origin with all of their ancestors** (`IsSameOriginWithAncestors`), and are rejected in cross-origin iframes, fenced frames, and opaque origins (such as `data:` URLs or sandboxed frames without `allow-same-origin`). Enabling delegation to cross-origin iframes via explicit Permissions Policy may be considered in future work.

Note that on the issuer network leg, the credentialed `accounts_endpoint` and `issuance_endpoint` requests deliberately attach the issuer's first-party cookies along with `Sec-Fetch-Dest: webidentity` / `Sec-Fetch-Dest: email-verification` (and omit `Origin`/`Referer`); issuers opt in explicitly by publishing `_email-verification.<domain>` DNS TXT records and `/.well-known/email-verification` metadata and validating `Sec-Fetch-Dest`.

## [2.15. How do the features in this specification work in the context of a browser’s Private Browsing or Incognito mode?](https://www.w3.org/TR/security-privacy-questionnaire/#private-browsing)

EVP respects full profile and storage isolation in Private Browsing / Incognito mode:

- **Cookie & session isolation**: Discovery of logged-in accounts and token issuance use the Incognito profile's isolated cookie jar and Login Status state. An issuer session in the user's regular profile is never used or leaked in Incognito; EVP only activates in Incognito if the user has explicitly signed in to their email provider within that Incognito session and selects an Autofill email entry.
- **No persistent state written to disk**: Any permission decision or strike state recorded during an Incognito session lives strictly in memory for that Incognito profile and is discarded when the Incognito session closes, without reading from or writing back to regular-profile EVP approvals in `chrome://settings` (unless shared Autofill profile suggestions themselves are selected by the user, in which case consent is still requested within the Incognito session and discarded on close).

## [2.16. Does this specification have both "Security Considerations" and "Privacy Considerations" sections?](https://www.w3.org/TR/security-privacy-questionnaire/#considerations)

Yes:
- **Frontend / Browser specification (WICG)**: [Privacy Considerations](https://wicg.github.io/email-verification/#privacy) and [Security Considerations](https://wicg.github.io/email-verification/#security) (plus extended discussion in the [WICG Explainer](https://github.com/WICG/email-verification#privacy-considerations)).
- **Backend / Issuance protocol specification (IETF)**: [Privacy Considerations and Security Considerations in `draft-hardt-email-verification`](https://datatracker.ietf.org/doc/draft-hardt-email-verification/).

## [2.17. Do features in your specification enable origins to downgrade default security protections?](https://www.w3.org/TR/security-privacy-questionnaire/#relaxed-sop)

No.

## [2.18. What happens when a document that uses your feature is kept alive in BFCache (instead of getting destroyed) after navigation, and potentially gets reused on future navigations back to the document?](https://www.w3.org/TR/security-privacy-questionnaire/#bfcache)

When a document navigates away (entering BFCache or otherwise), any in-flight verification check or token issuance request is immediately cancelled, weak callbacks are invalidated, and any open permission prompt or loading UI is dismissed. Frames that are not in the `kActive` lifecycle state cannot trigger UI or receive tokens.

If the document is later restored from BFCache, no background requests or prompts resume automatically. Any token that had already been delivered to the renderer before navigation remains bound to that document's origin (`aud`), `nonce`, and specific email field, and will only be populated on explicit form submission if the email input still contains the matching verified email address (with Relying Party single-use `nonce` and `iat`/`exp` validation preventing replay of stale tokens).

## [2.19. What happens when a document that uses your feature gets disconnected?](https://www.w3.org/TR/security-privacy-questionnaire/#non-fully-active)

Browser-side verification state is bound to the `RenderFrameHost` and `WebContents`. If a document or frame becomes non-fully-active, disconnected, navigated, or destroyed while discovery, permission prompting, or token issuance is in progress, the user agent aborts the flow, hides any active permission or toast UI, and discards any late network responses without delivering a token to the renderer.

## [2.20. Does your spec define when and how new kinds of errors should be raised?](https://www.w3.org/TR/security-privacy-questionnaire/#error-handling)

Protocol-level [issuance and discovery errors](https://datatracker.ietf.org/doc/draft-hardt-email-verification/) are **not** exposed to page scripts or DOM events, preventing Relying Parties from distinguishing between non-participating email domains, signed-out users, strike cooldowns, or user prompt declines.

Websites simply receive either a populated `SD-JWT+KB` token or an empty hidden field upon form submission, and fall back to standard email verification when no token is present. For developers debugging their own integrations, diagnostics are reported exclusively to browser DevTools (`InspectorIssue` / console) without exposing cross-origin user state to the page.

## [2.21. Does your feature allow sites to learn about the user’s use of assistive technology?](https://www.w3.org/TR/security-privacy-questionnaire/#accessibility-devices)

No.

## [2.22. What should this questionnaire have asked?](https://www.w3.org/TR/security-privacy-questionnaire/#missing-questions)

For browser-mediated three-party identity and verification protocols (involving a Relying Party, User Agent, and Issuer), two architectural questions are especially important:
1. **Does the Issuer learn which Relying Party the user is interacting with?** In EVP, the Issuer is kept blind: browser requests omit `Origin`/`Referer` and bind the Relying Party's origin (`aud`) and `nonce` locally via the browser's `KB-JWT`.
2. **Can pre-consent discovery or account-check fetches leak user login state to the Relying Party via network side channels?** In EVP, all network fetches occur in the browser process without exposing failure reasons to page script, pre-consent fetches avoid `email_matched`-dependent subresource requests that could trigger network reporting side channels (such as `Connection-Allowlist`), and token issuance occurs strictly after user consent.
