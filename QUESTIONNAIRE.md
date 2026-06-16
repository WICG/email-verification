# [Self-Review Questionnaire: Security and Privacy](https://www.w3.org/TR/security-privacy-questionnaire)

> Sam Goto (goto@chromium.org)
> Last update: May 20th, 2026

## [2.1 What information does this feature expose, and for what purposes?](https://www.w3.org/TR/security-privacy-questionnaire/#purpose)

This feature exposes an [Email Verification Token](https://dickhardt.github.io/email-verification/draft-hardt-email-verification.html#name-email-verification-token-ev) (EVT for short) to websites, a cryptographically signed token that asserts a proof of email ownership by the user carrying it.

The main purpose of exposing this information is because emails (and phone numbers) are 1 of the 2 main ways users create/access/recover accounts on websites in practice (the second one being social login), which require currently manual verification (via sending non-guessable codes to inboxes and having the user prove ownership by proving access to it).

## [2.2. Do features in your specification expose the minimum amount of information necessary to implement the intended functionality?](https://www.w3.org/TR/security-privacy-questionnaire/#minimum-data)

Yes.

The only delta between what's already being shared and what this feature adds is a cryptographic signature (and the envelope required) from the issuer and from the browser.

## [2.3. Do the features in your specification expose personal information, personally-identifiable information (PII), or information derived from either?](https://www.w3.org/TR/security-privacy-questionnaire/#personal-data)

Yes.

This feature auguments an already existing feature, namely, the ability to provide email addresses to websites (via `<input>` boxes and `autocomplete`), which does consistitute personally-identifiable information, with the addition of cryptographic proofs.

The cryptographic proof is only shared when the email autocomlete is also shared, so the delta in PII is the cryptographic proof.

It is the case that issuers could embed extra information in the cryptographic proof (e.g. generating signatures that contain extra information it), but we expect this to be the least convenient communication channel in comparison to alternatives (e.g. data that is already shared via [webfinger](https://en.wikipedia.org/wiki/WebFinger)).

## [2.4. How do the features in your specification deal with sensitive information?](https://www.w3.org/TR/security-privacy-questionnaire/#sensitive-data)

In addition to personally-identifiable information, one meaningful bit of information that also gets shared with the websites is that the user is logged in to their email provider.

The main way we deal with the extra bit of information is asking the user for [explicit permission](https://github.com/samuelgoto/email-verification-protocol/blob/main/README.md#permission-model). We are still iterating on how often and how granular (e.g. per email? per RP? per presentation? per email provider?) the permission is, but we do expect to have some.

That's not necessarily an "extra" bit of information considering the alternative (which is to prove ownership by proving access, and hence, proving that you are logged in to your email provider), but it could be an extra bit of information outside of the intended purpose of the feature (to verify email addresses).

For example, a malicious website could always ask for a verified email address outside of an email verification flow, and gather that the user is logged in or not to the email provider.

We can't come up with an actual practical example of when that might happen, but is worth noting that an "extra" bit of information is indeed provided compared to the status quo.

## [2.5. Does data exposed by your specification carry related but distinct information that may not be obvious to users?](https://www.w3.org/TR/security-privacy-questionnaire/#hidden-data)

Like we said in the section above, I think users can't really understand what a "cryptographic proof" is, or what an EVT is, so we do think that the information being shared has to be understood in terms of what consequences it might have for users.

For example, whether you are logged in to your email provider or not is "comprehensible" by users, and leaking that information while presenting an EVT might be non-obvious to them.

When used within the intended flows (account creation, access and recovery), outside of abuse, I think leaking that the user is "logged in" to their email provider won't be obvious, but acceptable (given (a) the benefits of automatically verifying email addresses and (b) the necessity to leak that too when verifying emails manually).

## [2.6. Do the features in your specification introduce state that persists across browsing sessions?](https://www.w3.org/TR/security-privacy-questionnaire/#persistent-origin-specific-state)

No.

The only data that gets stored is the explicit user permission to use the feature.

## [2.7. Do the features in your specification expose information about the underlying platform to origins?](https://www.w3.org/TR/security-privacy-questionnaire/#underlying-platform-data)

No.

We might extend the protocol to support [talking to native mobile applications](https://github.com/fedidcg/native-app-idps) (because email providers are often native apps, rather than web apps on mobile), but we think that can be done without exposing that information to websites.

## [2.8. Does this specification allow an origin to send data to the underlying platform?](https://www.w3.org/TR/security-privacy-questionnaire/#send-to-platform)

No.

## [2.9. Do features in this specification enable access to device sensors?](https://www.w3.org/TR/security-privacy-questionnaire/#sensor-data)

No.

## [2.10. Do features in this specification enable new script execution/loading mechanisms?](https://www.w3.org/TR/security-privacy-questionnaire/#string-to-script)

No.

## [2.11. Do features in this specification allow an origin to access other devices?](https://www.w3.org/TR/security-privacy-questionnaire/#remote-device)

No.

## [2.12. Do features in this specification allow an origin some measure of control over a user agent’s native UI?](https://www.w3.org/TR/security-privacy-questionnaire/#native-ui)

Yes.

We expect we'll need to introduce an explicit permission to the user before performing an automatic verification. While that's largely a choice each user agent has control of (in terms of the frequency and UX materialization), we do expect one to have.

We expect that the native UI would be triggered by user action (typing/selecting the email), and that the only customizable thing is the issuer site (and the email address from the user).

So, along the lines of any other permissioned API, we expect this one to give an origin the ability to trigger that permission programatically.

https://github.com/samuelgoto/email-verification-protocol#permission-model

## [2.14. How does this specification distinguish between behavior in first-party and third-party contexts?](https://www.w3.org/TR/security-privacy-questionnaire/#first-third-party)

We are disabling this feature in third party contexts (e.g. iframes). We might look into enabling it with the right permission policies, but we haven't gotten that far yet.

## [2.15. How do the features in this specification work in the context of a browser’s Private Browsing or Incognito mode?](https://www.w3.org/TR/security-privacy-questionnaire/#private-browsing)

The operation of this feature is indistiguishable between private browser / incognito mode and regular profiles.

## [2.16. Does this specification have both "Security Considerations" and "Privacy Considerations" sections?](https://www.w3.org/TR/security-privacy-questionnaire/#considerations)

[Yes](https://github.com/samuelgoto/email-verification-protocol#privacy-considerations) and [yes](https://github.com/samuelgoto/email-verification-protocol#security-considerations) for websites.

Also, [yes](https://dickhardt.github.io/email-verification/draft-hardt-email-verification.html#name-privacy-considerations) and [yes](https://dickhardt.github.io/email-verification/draft-hardt-email-verification.html#name-security-considerations) for issuers / email providers.

## [2.17. Do features in your specification enable origins to downgrade default security protections?](https://www.w3.org/TR/security-privacy-questionnaire/#relaxed-sop)

No.

## [2.18. What happens when a document that uses your feature is kept alive in BFCache (instead of getting destroyed) after navigation, and potentially gets reused on future navigations back to the document?](https://www.w3.org/TR/security-privacy-questionnaire/#bfcache)

## [2.19. What happens when a document that uses your feature gets disconnected?](https://www.w3.org/TR/security-privacy-questionnaire/#non-fully-active)

## [2.20. Does your spec define when and how new kinds of errors should be raised?](https://www.w3.org/TR/security-privacy-questionnaire/#error-handling)

There are a series of errors that can happen as a result of [issuing tokens](https://dickhardt.github.io/email-verification/draft-hardt-email-verification.html#name-error-responses), but that doesn't get exposed to websites.

Websites either get an EVT or don't in the form submission, and have to degrade gracefully (to the status quo) when the token isn't provided.

## [2.21. Does your feature allow sites to learn about the user’s use of assistive technology?](https://www.w3.org/TR/security-privacy-questionnaire/#accessibility-devices)

No.

## [2.22. What should this questionnaire have asked?](https://www.w3.org/TR/security-privacy-questionnaire/#missing-questions)


