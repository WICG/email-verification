> Last update: May 19th, 2026
> 
> Authors: 
> - @samuelgoto
> - @dickhardt
>
> Status:
> - [x] [Intent to Prototype](https://groups.google.com/a/chromium.org/g/blink-dev/c/pWfWupaOtJw/m/MS6uaf_WAAAJ?utm_medium=email&utm_source=footer)
> - [x] [Intent to Experiment](https://groups.google.com/a/chromium.org/g/blink-dev/c/Vp1w1u6rYjE)

# Email Verification Tokens (EVTs)

TL;DR; This is a proposal to help users verify email addresses (e.g. during account creation, sign-in and account recovery) by providing cryptographic proof of ownership seamlessly rather than email OTPs manually.

# The Problem

As of 2025, 95% of the [50 most visited websites](https://en.wikipedia.org/wiki/List_of_most-visited_websites) support creating accounts and logging in with emails.

> federated login comes as a close second, available in 72% of them, unsurprisingly because that’s also another big source of verified email addresses

Out of those, 73% of them blocked the account creation process before users verified their emails. 26% of them started the flow with email acquisition and email verification before anything else (e.g. acquiring names or passwords/passkeys) and 47% of them performed email verification before even acquiring passwords (and, hence, passkeys).

That’s worrisome because email verification is currently done insecurely and inefficiently.

Currently, once an email is acquired (e.g. through an input box), websites generate, store and send a non-guessable code (or link which embeds the code, known as a “magic link”) to the user’s email inbox. The user proves ownership of the email address by proving they have access to the inbox by presenting the non-guessable code to the website (e.g. copying/pasting it in an input box or clicking on the magic link).

<!--
<img width="1152" height="864" alt="Email Verification Tokens (EVTs) (3)" src="https://github.com/user-attachments/assets/2a18ba73-38de-4251-aca6-0a72159791df" />
-->

Aside from the security concerns (i.e. phishing), this process is also cumbersome for users and inefficient for websites: (a) it relies on the delivery of emails (e.g. automated emails take a while to arrive and can and do often go into spam folders) and (b) switching the user’s context from the website to their email (web or native) app.

For websites, the inefficiencies caused by the friction of this process directly affect customer acquisition costs (aka CAC) in conversion funnels, which typically start from advertisement to drive users to discover their services and end in account creation.

In addition to account creation, email verification is also often used for account recovery (e.g. when users forget passwords), sign-in and second factor authentication (e.g. in sensitive logins such as banks or high stake actions, such as large purchases), affecting large parts of the the user’s account lifecycle.

# The Proposal

This proposal requires affecting change in 4 large and independent parts of the ecosystem: browsers, email providers, websites and users.

The idea is to reuse the email provider session (in the browser cookie jar or on [native apps](https://github.com/fedidcg/native-app-idps)) to issue an email verification token (EVT for short) that the browser can use to present to websites.

When these conditions are met (see activation considerations below), the proposal allows the website to skip the manual verification process, automatically gathering a cryptographically verified proof of ownership, and degrading gracefully to the status quo otherwise.

This proposal presupposes that the email provider has been set up ahead of time so that websites can request EVTs at run time.

The next sections go over each of these steps:

```
Step                                  Website              Browser              Issuer
                                         |                    |                    |
                                         |                    |                    |
1.1 Login Status                         |                    |<---- logged-in ----|
                                         |                    |                    |
2.1 EVT Request                          |------ nonce ------>|                    |
                                         |                    |                    |
3.1 Email Selection                      |         [Obtain email from user]        |
                                         |                    |                    |
3.2 Issuer Discovery                     |            [Discover issuer]            |
                                         |                    |                    |
3.3 Accounts Request                     |                    |<---- accounts -----|
                                         |                    |                    |
3.4 Permission                           |           [Obtain permission]           |
                                         |                    |                    |
3.5 EVT Issuance                         |                    |<------ EVT --------|
                                         |                    |                    |
3.6 KB Creation                          |              [Create KB-JWT]            |
                                         |                    |                    |
3.7 EVT Verification                     |              [Verify EVT+KB]            |
                                         |                    |                    |
3.8 Form submission                      |         [Await form submission]         |
                                         |                    |                    |
3.9 EVT Presentation                     |<----- EVT+KB ------|                    |
                                         |                    |                    |
4.1 EVT Verification              [Verify EVT+KB]             |                    |
                                         |                    |                    |
                                         +                    +                    +
```

Below, we'll go over each step of the process.

## 1.1 Login Status

First, everything starts with the user logging in to their email provider's issuer ahead of time. The issuer can notify the browser that they have a logged in user by calling the [Login Status API](https://w3c-fedid.github.io/login-status/):

```javascript
// When the user logs in to the IdP
navigator.login.setStatus("logged-in");
```

Alternatively, via HTTP headers:

```
Set-Login: logged-in
```

This bit is then later used in the [accounts request](#33-accounts-request) step.


> Calling the Login Status API also allows the browser to discover the [`login_url`](https://w3c-fedid.github.io/FedCM/#dom-identityproviderapiconfig-login_url), which allows the browser to know how to programaticaly log the user in. This feature is not yet supported and is noted as an [open question](#what-should-happen-when-users-are-logged-out-of-the-email-provider--issuer).

## 2.1 EVT Request

Second, websites explicitly and proactively add to their HTML forms an additional `<input>` element with (a) an `email-verification-token` in the `autocomplete` attribute and (b) a dynamically generated non-guessable code that is stored server-side in a newly introduced `nonce` attribute. 

For example:

```html
<input
  type="hidden"
  name="token"
  nonce="<?php generate_nonce() ?>"
  autocomplete="email-verification-token">
```

There are many other ways that we could expose this API to websites, which you can find in the alternatives considered and under consideration section below.

## 3.1 Email Selection

When the website exposes this extra `<input>` requesting an `autocomplete="email-verification-token"`, the browser observes when users select an email in an input box that belongs to the same form.

## 3.2 Issuer Discovery

When the user selects an email in the input box, the browser presupposes that the email provider exposes itself as an EVP-compatible provider ahead of time by implementing the [EVP protocol](https://dickhardt.github.io/email-verification/draft-hardt-email-verification.html).

<!--
<img width="1280" height="960" alt="Email Verification Tokens (EVTs) (4)" src="https://github.com/user-attachments/assets/eb27296a-19eb-4b45-9591-5f00dfd469aa" />
-->

The browser starts by [discovering the issuer](https://dickhardt.github.io/email-verification/draft-hardt-email-verification.html#name-issuer-discovery) associated with the email address  (which can, but doesn’t have to, be same-site with the email provider, e.g. allows [gmail.com](http://gmail.com) to delegate EVTs to [accounts.google.com](http://accounts.google.com)), via a DNS TXT record set ahead of time:

```
_email-verification.email-domain.example   TXT   iss=issuer.example
```

## 3.3 Accounts Request

Once the issuer is discovered, the browser checks if the user is [logged in](https://w3c-fedid.github.io/login-status/#logged-in) to the issuer. 

If not, it stops here.

If so, the browser fetches all of the accounts that the user is logged in to, to see if the user is logged in to the email that was selected on Step 3.1.

To do so, it starts by [fetching the `.well-known/web-identity` file](https://w3c-fedid.github.io/FedCM/#well-known-file), which allows it to learn two key things:

- The [`accounts endpoint`](https://w3c-fedid.github.io/FedCM/#dom-identityproviderwellknown-accounts_endpoint)
- The [`login_url`](https://w3c-fedid.github.io/FedCM/#dom-identityproviderwellknown-login_url)

For example:

```json
{
  "accounts_endpoint": "...",
  "login_url": "...",
}
```

The former is used by the browser to fetch the list of logged in accounts. 

For example:

```json
{
 "accounts": [{
   "email": "john_doe@idp.example",
  }, {
   "email": "johnny@idp.example",
  }]
}
```

The browser goes through every `account` and checks if the user is logged in to the `email` that was selected before. The browser makes a case-insensitive string equality check to see if they match.

> NOTE: possibly, in the future, we might want to deal with normalized emails, such as removing `.` or `+` from them. In the meantime, case insensitity goes a long way.

## 3.4 Permission

Once the browser has discovered the issuer and knows that the user is logged in to the issuer, it is pretty confident that can provide an EVT, so it can choose to ask the user for permission. Each browser implementation is responsible for making their own judgement based on their user’s expectations, so this specification isn’t opinionated about how the interface with the user materializes.

## 3.5 Issuance Request

Once the issuer is discovered, the [issuer metadata](https://dickhardt.github.io/email-verification/draft-hardt-email-verification.html#section-3.2) can be fetched:

```
https://{issuer}/.well-known/email-verification
```

The issuer metadata contains a reference to the `issuance_endpoint` `signing_alg_values_supported` which is then used to create an issuance request.

The browser generates an ephemeral asymmetric key pair, and constructs a `request_token` which is a signed JWT containing the ephemeral public key (`jwk` in header) and the request details (`email`, `aud` (issuer), and `iat` in payload).

The browser then sends this to the issuer's issuance endpoint using `application/x-www-form-urlencoded` content type, including the issuer's first-party cookies.

For example:

```http
POST /email-verification/issuance HTTP/1.1
Host: accounts.issuer.example
Cookie: session=...
Content-Type: application/x-www-form-urlencoded
Sec-Fetch-Dest: email-verification

request_token=eyJhbGciOiJFZERTQSIsImp3ayI6eyJrdHkiOiJPS1AiLCJjcnYiOiJFZDI1NTE5IiwieCI6IjExcUdBWWRrOUU2ejdtVDZyazZqMVFuWGI2cFlxNHY5d1hiNnBZcTR2OXcifX0.eyJhdWQiOiJpc3N1ZXIuZXhhbXBsZSIsImlhdCI6MTcyNDA4MzIwMCwiZW1haWwiOiJ1c2VyQGV4YW1wbGUuY29tIn0.signature...
```

The issuer verifies the request, checks the user's session, and returns an `issuance_token` containing a signed SD-JWT (with the `~` separator appended) in a JSON response:

```http
HTTP/1.1 200 OK
Content-Type: application/json
Set-Cookie: session=...; Secure; HttpOnly; SameSite=None

{"issuance_token":"eyJhbGciOiJFZERTQSIsImtpZCI6IjIwMjQtMDgtMTkiLCJ0eXAiOiJldnAtc2Qtand0In0...~"}
```

## 3.6 KB Creation

Once issued, the browser [binds the nonce](https://dickhardt.github.io/email-verification/draft-hardt-email-verification.html#name-kb-creation) and the audience to the EVT (binding both the origin of the website that the EVT is presented to as well as the dynamically generated nonce) and then.

## 3.7 EVT+KB Verification

Once the browser has a `EVT+KB` it performs a [verification](https://dickhardt.github.io/email-verification/draft-hardt-email-verification.html#name-token-verification) to check that everything is alright before it can present it.

## 3.8 Form submission

Once the browser has a verified `EVT+KB` it await for form submission before it presents the EVT.

## 3.9 EVT Presentation

Before the form gets submitted, the browser looks for an input with an `autocomplete="email-verification-token"` input field in the same `<form>` as the `<input type="email">` that was used and sets its `value` before the `onsubmit` event is fired.

## 4.1 EVT Verification

Upon form submission, because the browser has filled the value of the `<input>` element with the EVT, the website’s server gets that value in the form submission HTTP handler and is then able to skip the manual verification step by performing an automated [token verification step](https://dickhardt.github.io/email-verification/draft-hardt-email-verification.html#name-token-verification).

# Activation Considerations

There are orders of magnitude more users (say, \~billions) than websites (say, \~millions) than email providers (say, \~thousands) than browser vendors (say, \~tens), so our proposal focuses on pulling change in the reverse order: make browsers pull as much responsibility as possible, then email providers, then try to make things as lightweight as possible for websites and affect as little behavioral change (ideally, none) for users as possible.

By design, EVT allows websites to deploy it (a) without changing the user’s behavior (aside from accepting an extra permission prompt), (b) without requiring every browser to support EVT at the same time and (c) without relying on changing every email provider in the world in lock step.

Economically, because there is a cost involved in redeploying the website, we expect we’ll need a critical mass of email providers to support it before websites have any incentives to use it. For better or for worse, the current deployment of email providers follows a power law (e.g. \~10 email providers account for 50%+ of the market share), so we expect that incentivizing the head/torso of email providers might provide enough activation energy to bootstrap the flywheel. 

Once / if we reach that minimum activation energy, we expect websites to then move next. Because verifying email addresses directly lowers customer acquisition costs (CAC) in acquisition funnels, we believe the activation energy necessary might be small enough compared to the implementation costs.

# Relationship to other APIs

We are deliberately designing EVTs to be a composable building block that can be requested (e.g. through different frontends) and provided (e.g. through different backends) in various ways.

While this specific proposal focuses on requesting them through autocomplete, we expect EVTs may play a role in other ways the developer can make requests for the following related APIs:

## WebAuthn API

While not directly covered in the architecture described in this doc, we expect that the same UX (a browser mediated email selector) and architectural principles (e.g permission prompts) are going to apply to the Request User Info WebAuthn API [https://github.com/w3c/webauthn/blob/main/explainers/request-user-info.md](https://github.com/w3c/webauthn/blob/main/explainers/request-user-info.md) :

<img width="1152" height="864" alt="Email Verification Tokens (EVTs) (6)" src="https://github.com/user-attachments/assets/98c79248-5cb4-478d-9d2c-c8a80760bb99" />

Outside of the Passkey Creation API, some have raised whether EVT could diminish the role that passkeys will have going forward (as a first factor), and it is our opinion that (a) it is too soon to tell but if we had to make a prediction (b) it is unlikely that EVT is going to have any negative effect in our ability to move users away from passwords.

It is plausible that EVTs make passwords (and, hence, passkeys) altogether less useful by making login entirely done via (a progressively more efficient) email verification, and while we acknowledge that that’s a viable pattern currently deployed in small / niche websites, we believe that EVTs is just going to be an extra tool in the toolbox that will coexist symbiotically with passkeys.

## Federated Credentials API

A significant part of the deployment of FedCM is represented by email providers: most notably [google.com](http://google.com) (which is authoritative to [gmail.com](http://gmail.com) email addresses, accounts for \~45% of the email provider market share) [seznam.cz](http://seznam.cz), and [gmx.de](http://gmx.de) / [web.de](http://web.de) (account for a large portion of the market share in Europe).

That’s not entirely surprising, since social login competes with email verification for account creation (as noted above in the introduction).

This proposal relies on discovering email addresses from the autofill and could reconcile well with FedCM to discover and issue EVTs:

```html
// While logging into the email provider
<script>
  navigator.login.setStatus("logged-in", {
    accounts: [{
      firstName: "John",
      lastName: "Doe",
      email: "john@doe.com"
    }]
  });
  IdentityProvider.register("evp");
</script>

// While requesting in the RP, autofill gets augumented by email addresses that were
// declared in FedCM.
<input type="email" autocomplete="email">
<input type="hidden" nonce="1234" autocomplete="email-verification-token">
```

This could also be made to work with native applications through [https://github.com/fedidcg/native-app-idps](https://github.com/fedidcg/native-app-idps) 

## Digital Credentials API

We expect more and more government-issued identities to be made available on digital wallets, as well as IDs like corporate employment cards.

It is possible that we would want to allow users to pick verified email addresses from these IDs too, specifically in a way that conforms to EVT.

Much like EVTs are designed to augment WebAuthn and FedCM, we expect EVTs could be used inside of the DC API if the aggregator (currently, OSes) of digital credentials wanted to provide authoritative verified email addresses rather than derived (such as when “X verified Y”, such as “facebook/google” verifying “hotmail/yahoo” email addresses).

So, for example, if a website requested an EVT, and a native application (say, a digital wallet) had one registered at the OS level, the browser would be able to provide it through the autocomplete UX.

One immediate implementation challenge we face is that autofill UX is entirely done in the browser memory address space, whereas all of the digital credentials are aggregated at the operating system address space, so it is not clear how we’d augment them in autofill. One plausible answer is if we made the OS share the metadata with browsers, which is similar to how passkeys work on browsers on Android and iOS.

If/when we get to that point (where OSes share the metadata of the credentials with browsers), browsers could make them available in the EVT autocomplete.

It might be also be possible for the browser to talk to the wallets directly through the same mechanism FedCM is planning to use to talk to IdPs via [https://github.com/fedidcg/native-app-idps](https://github.com/fedidcg/native-app-idps), but that’s still too soon to tell if there is going to be better OS support for browsers.

# Open Questions

There are a few things that we are still unsure about:

## Can this handle directed email addresses?

We believe that it is possible to reconcile this proposal well with directed email addresses, but this is still an area of open investigation:

[https://dickhardt.github.io/email-verification/draft-hardt-email-verification.html\#name-private-email-addresses](https://dickhardt.github.io/email-verification/draft-hardt-email-verification.html#name-private-email-addresses)   

## What should happen when users are logged out of the email provider / issuer?

We think it is plausible that the browser might be able to help the user user login to the email provider / issuer inline in the flow. It is not quite clear to us yet how, but one of the intuitions is that we can offer a streamlined experience through passkeys:

[https://dickhardt.github.io/email-verification/draft-hardt-email-verification.html\#name-webauthn-authentication](https://dickhardt.github.io/email-verification/draft-hardt-email-verification.html#name-webauthn-authentication) 

## Is the permission prompt strictly necessary?

It is unclear to us right now what, if any, is the value of the permission prompt: from first principles, what does the website learn that it wouldn’t otherwise learn, and does that require user permission?

If we do choose to show a permission prompt, a few questions: 

- How often we want to show it to the user (e.g. Every time an email is selected? Once? Once per website?).  
- When do we show it? After we make credentialed requests? Before? If so, what happens when the user is logged out (have we made an offer to the user that we weren’t sure we would be able to keep?)?

## Could this also handle usernames?

Mike West and Dominic Battre both note that we could also tie this to `<input autocomplete="username">`, in addition to `<input autocomplete="email">`.

This system doesn't make any assertions that an email is delivered. You could take this a step further and claim that this system has little to do with "email", except insofar as an email address is used as an identifier for the provider to verify, and folks generally want email addresses today.

It seems to us that you could make the system quite a bit more generic by changing a few words, and introducing a mechanism by which an identifier could be tied to an issuer absent an explicit `@provider.example` in the string (e.g. another input field whose value is inferred from the identifier if possible, or set by the site for social sign in).

## Would this work with complex forms?

Mike West notes: what happens when forms have two email addresses?

That is, this example seems to assume that every form will have one and only one email address field, in which case it's trivial to find the \`type=email\` field and bind it to the hidden input field's `nonce` for the verification request. Some forms contain more than one email address, however. For example, there are  governmental forms that ask for both work and personal email, and I can imagine such entities wanting verification of those fields if it was easily possible (they don't ask for it today, or didn't in this specific case). How could that be supported? 

# Alternatives Considered

Email Verification is a broad area and there are multiple ways that we can accomplish this.

## Can't this be done in userland?

There are some aspects of this proposal that could have been done in userland, but not all of it.

First, implementing this in userland would require access to third party cookies, which gets blocked on a meaningful portion of the usage of the web.

Second, even with access to third party cookies, you’d need a neutral intermediator that could act as a holder between the issuer and the verifier. This was historically done with a user-trusted origin, such as [accountchooser.org](http://accountchooser.org) (for [OIDF’s Account Chooser](https://openid.net/wordpress-content/uploads/2011/12/ac-integration-spec.html#rfc.section.1)) and [persona.org](http://persona.org) (for [Mozilla’s BrowserID](https://en.wikipedia.org/wiki/Mozilla_Persona)).

We think part of the reason the problem of email verification hasn’t been solved yet is because this proposal relies on changing the browser.

## What if we activated email clients, rather than email providers?

One of the main forks on the road that we considered was whether to facilitate email providers providing an OTP or an EVT, for example, via augmentin WebOTP / `autocomplete="one-time-code"`. 

For example:

[https://github.com/samuelgoto/email-otp](https://github.com/samuelgoto/email-otp) 

We believe these aren’t mutually exclusive propositions, but are rather complimentary with different trade-offs.

The main reason an OTP-oriented approach is compelling is because it introduces another thin waist that we could buckle: email clients (IMAP/POP/SMTP).

There are orders of magnitude fewer email clients (e.g. notably Thunderbird, Spark and Apple Mail) than there are email providers, so if we could change \~tens of email clients to provide OTPs for \~thousands of email providers, that could also lead to a much lower activation energy.

OTPs still rely on email delivery and having an OTP page on websites, so while we expect this to be a reasonable topology, we expect EVTs to provide a better user experience (at the cost of having a higher activation energy).

Nonetheless, we think extending WebOTP / autocomplete \= “one-time-code” and coming up with conventions / integrations between email clients and browsers is not mutually exclusive with this proposal but are rather symbiotic.

## Website API

There are two big variations that we explored:

- JS Imperative APIs and   
- HTML Declarative APIs

### Have you looked at imperative APIs?

Within the JS Imperative API space, we explored augment three options:

- Extending the FedCM API  
- Extending the Digital Credentials API  
- Introducing a new credential type

We went over the relationship with FedCM and the DC API above, so refer to that to see how we reconcile the two, but the third one is still a valid formulation that we think could work:

```javascript
<script>
const {token} = await navigator.credentials.get({
  mediation: "conditional",
  email: {
    nonce: "1234"
  }
});
document.getElementByName("token").value = token;
</script>
<form action="signup.php" method="POST">
  <input type="email" name="email" autocomplete="email email-verification-protocol">
  <input type="hidden" name="token">
</form>
```

The pros are that:

- `nonce` goes into a Credential Manager API call, which sounds like the right place  
- `token` gets returned into a Credential Manager API call, which also sounds like the right place

The cons are:

- We’d like to release the token and resolve the promise upon form submission, rather than email selection, which makes “conditional” inconsistent with how WebAuthn works.  
- The developer is responsible for manually taken the value of the token and inserting into an input field (more control for developers but at the cost of being more work too)

### Have you looked at declarative APIs?

There are a series of variations of ways we could expose this as a HTML API. Here are a few that occurred to us and their trade-offs.

Our very first intuition was to create an EmailVerifiedEvent and dispatch it when the email address was selected:

```html
<input id="email"
       type="email"
       autocomplete="email"
       nonce="12345677890..random">
<script>
const input = document.getElementById('email')

input.addEventListener('emailverified', e => {
  // e.presentationToken is SD-JWT+KB
  console.log({
      presentationToken: e.presentationToken
  })
})
</script>
```

The main reason we moved away from this formulation was because we wanted to tie the EVT release with the form submission, and dispatching an event to coordinate with the onsubmit handler seemed unnecessary complex.

From there, our second intuition was to leave `nonce` in the `<input type=”email”>` and hard-code a special value that gets submitted into the form:

```html
<form action="signup.php" method="POST">
  <input type="email" name="email" autocomplete="email" nonce="1234">
</form>
```

   
Leading to a form submission where a “special” or “reserved” parameter (e.g. `email.__verification__`) would be passed upon form submission.

```
POST signup.php
email=foobar@gmail.com&email.__verification__=456
```

Having a `nonce` associated with an `<input type=”email”>` seemed awkward, so we looked into maybe introducing a new element type, say, `<nonce>`: 

```html
<form action="signup.php" method="POST">
  <input type="email" name="email" autocomplete="email" >
  <nonce for="email" value="1234"></nonce>
</form>
```

Introducing a `<nonce>` seemed like overkill, so we then considered “what if we create a new `<input>` type”, say `“verification”`?

```
 <input type="verification" for="email" nonce="1234" name="token">
```

That seemed like it would have all sorts of good properties, including that we could declare `nonce` and also `name`, solving both how you provide the `input` to the EVP algorithm with the `nonce` but also how EVP could “output” the result with the “name” property and how it is used in form submission, addressing the awkward “reserved” form submission parameter name.

`<input type="verification">` degrades gracefully to `<input type="text">` when the browser doesn’t know what it is, so we’d have to include a `style: display: none` in it if we wanted to make it invisible.

That’s when we realized that `<input type="hidden">` might actually be exactly what we were looking for, because it (a) is hidden, (b) participates in form submission with a name, (c) can already accept an `autocomplete` attribute that the browser could use to fill and (d) just needed to be extended to include `nonce`.

Hence, our latest proposal being:

```html
<form action="signup.php" method="POST">
  <input type="email" name="email" autocomplete="email">
  <!-- one-liner that can be added progressively to a form -->
  <input type="hidden" name="token" 
    nonce="<?php server_side_generate_nonce() ?>" 
    autocomplete="email-verification-token"
  >
</form>
```

We don’t think this is necessarily great, and would welcome suggestions on how to make this better, but it is the best proposal we heard so far (for the reasons described above).

# Privacy considerations

First, it seemed worth noting what this proposal does NOT do. 

This proposal does not change the fact that email addresses can be (and most often are):

- Global identifiers that are acquired by websites and sold to aggregators that can join the user’s online and offline activities (e.g. signing-up to a website with an email can be joined with creating an account with the same email at a chain clothing store).  
- Spam vectors that are acquired by websites and sold for remarketing purposes

By making email verification frictionless, we acknowledge that this proposal might perpetuate and sediment these practices more than they already are.

To address this at scale, it is important to understand why websites require email addresses to create accounts in the first place. We think there are three reasons: (a) it serves as a unique and memorable user identifier, (b) it serves as a reliable communication mechanism. The last property is important because a reliable communication mechanism can be used not only for (i) re-engagement (which is important in and of itself for websites), but (ii) as an account recovery mechanism (for account security).

We think it is plausible that WebAuthn might create a better (ii) account recovery mechanism, but we also think that there are always going to be websites that have a justifiable need to have a (i) re-engagement channel.

Thankfully, we think that browsers and email providers can provide email addresses that are directed / proxied, so that it can’t be used as a global identifier and spam can be controlled per website. See the Future Work section for more information on directed email addresses.

## Comparison to the Status Quo

Second, it seemed worth noting what this proposal DOES change in comparison to the status quo mechanism of email OTPs.

- First, the email provider is kept blind during presentation, so it no longer learns that the user is using specific websites. Obviously, email providers (a) are generally trusted by users and (b) learn as soon as the website sends an email to the user, but websites no longer have to at account creation (or ever) with this proposal.  
- Second, the website learns that the user is logged in to the email provider on this device, which is (slightly) more than they were learning with magic links (e.g. users can be logged in to their email inbox on other devices).  
- Third, the website gets a token signed by the issuer, which constitutes a communication channel between the email provider and the website. It is unclear how that would be abused given the current structure of incentives, but it seemed worth noting as distinct from email OTPs.  
- Fourth, because the issuance request sends the email that the user selected, the issuer learns that the user used an email address that does not necessarily match the email that the user was logged as. There are ways that we could address this, but it seemed like the costs would outweigh the benefits (users using public or shared computers occurred to us).

# Security considerations

There are various security considerations to be made.

Like the privacy considerations above, it is important to note how websites use email verification today and how this proposal might affect them.

First, it is important to note that this proposal doesn’t change the fact that email domains can (and do) change hands (e.g. a domain expiring and being bought by another operator), so they have to be used appropriately (e.g. in conjunction with other forms of authentication or for non sensitive operations).

Second, this proposal is weaker than OTPs in that they do NOT assert that the email was actually delivered to the user’s inbox, but rather that the user is logged in to the email app. In practice, we think this distinction isn’t meaningful (specially because delivery is also a function of uptime availability too), but it is worth noting that it is weaker than OTPs in that way.

Asides from that, here are a few considerations we are working through:

- Is the protocol safe?  
  - While the protocol described is using fairly well-established parts, when we put them together, have we introduced any additional attack? [https://dickhardt.github.io/email-verification/draft-hardt-email-verification.html](https://dickhardt.github.io/email-verification/draft-hardt-email-verification.html)   
  - Are the default crypto choices sensible?  
  - The proposal relies on DNS. Is DNS sufficiently secure if we resolve them server side?  
  - Should we include the `nonce` in the form submission?  
- Can EVTs be requested in subframes?  
  - Should we require permission policies for cross origin iframes?  
  - Should we disable this in fenced frames?  
  - Should we disable opaque origins?  
- Cookies  
  - SameSite=Strict  
  - Since we're sending verification requests in a first-party context, this creates a mechanism by which `SameSite=Strict` cookies are sent based on a user's activity on a third-party site. This is probably defensible, given the way in which the mechanism and endpoints involved are opt-in on the issuer's part, but we should mention it explicitly.  
- CORS  
  - It might be valuable to limit this verification mechanism's potential leakage of user state cross-origin by rejecting verification responses that contain `Access-Control-*` headers. (*User agents that limit third-party cookie access would mitigate this leak, but it's worth considering what we'd like the behavior to be for user agents that allow them. With CORS headers, this mechanism would provide a very straightforward user identification mechanism, especially in the ["issue all credentials" alternative discussed above](#heading=h.qd2yaibbowlw).*)  
  - Perhaps servers should return a signed "nope" response rather than an `authentication_required` error, which seems like it would mitigate some network-level distinctions that could be inferred without access to the RP's private key?  
- Client-side injection?  
  - Should we protect `nonce` and “value” from script? [https://bsky.app/profile/sgo.to/post/3mlbpzl3abc2y](https://bsky.app/profile/sgo.to/post/3mlbpzl3abc2y) 

