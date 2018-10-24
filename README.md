# Explainer: Reducing `User-Agent` Granularity

Mike West, August 2018

_©2018, Google, Inc. All rights reserved._

(_Note: This isn't a proposal that's well thought out, and stamped solidly with the Google Seal of
Approval. It's a collection of interesting ideas for discussion, nothing more, nothing less._)

## A Problem

User agents identify themselves to servers as part of each HTTP request via the `User-Agent` header.
This header's value has grown in both length and complexity over the years; a complicated dance
between server-side sniffing to provide the right experience for the right devices on the one hand,
and client-side spoofing in order to bypass incorrect or inconvenient sniffing on the other. Chrome
on iOS, for instance, currently identifies itself as:

```http
User-Agent: Mozilla/5.0 (iPhone; CPU iPhone OS 12_0 like Mac OS X) AppleWebKit/605.1.15 (KHTML, like Gecko) CriOS/69.0.3497.105 Mobile/15E148 Safari/605.1
```

While Chrome on Android sends something more like:

```http
User-Agent: Mozilla/5.0 (Linux; Android 9; Pixel 2 XL Build/PPP3.180510.008) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/67.0.3396.87 Mobile Safari/537.36
```

There's a lot of entropy wrapped up in the UA string. This makes it an important part of
fingerprinting schemes of all sorts. Safari has [recently taken some steps][1] to reduce
the entropy of the user agent string, initially locking it to a single value, period, and
then [backing off a bit][2] due to developer feedback. This document proposes a mechanism
which might allow user agents generally to be a little more aggressive.

[1]: https://twitter.com/rmondello/status/943545865204989953?lang=en
[2]: https://bugs.webkit.org/show_bug.cgi?id=182629#c6

### Challenges 

From a developer's standpoint, the detail available in the UA string is valuable, and they
reasonably object to dropping it. The feedback to Safari's UA string freeze was right along these
lines, noting four broad categories of use:

1.  Brand and version information (e.g. "Chrome 69") allows websites to work around known bugs in
    specific releases that aren't otherwise detectable. For example, implementations of Content
    Security Policy have varied wildly between vendors, and it's difficult to know what policy to
    send in an HTTP response without knowing what browser is responsible for its parsing and
    execution.

2.  Developers will often negotiate what content to send based on the user agent and platform. Some
    application frameworks, for instance, will style an application on iOS differently from the same
    application on Android in order to match each platform's aesthetic and design patterns
    (<https://onsen.io/> was the first example in a quick skim of search results, but there are
    certainly others).

3.  Similarly to #1, OS revisions and architecture can be responsible for specific bugs which can
    be worked around in website's code, and narrowly useful for things like selecting appropriate
    executables for download (32 vs 64 bit, ARM vs Intel, etc).

4.  Sophisticated developers use model/make to tailor their sites to the capabilities of the
    device (e.g. [Facebook Year Class][3]) and to pinpoint performance bugs and regressions which
    sometimes are specific to model/make.

These are use cases that are interesting for us to support. Browsers certainly have bugs, and
developers certainly would be well-served by being able to work around them. We'll keep these
challenges in mind with the proposal that follows.

[3]: https://code.fb.com/android/year-class-a-classification-system-for-android/

## A Proposal

By default, servers should receive only the user agent's brand and major version number. Servers
can opt-into receiving information about minor versions, the underlying operating system's major
version, and details about the underlying architecture and device. The user agent can make
reasonable decisions about how to respond to sites' requests for more granular data. We might
accomplish this as follows:

1.  Browsers should deprecate the `User-Agent` string over time, initially locking bits of its
    value, and ramping up over time to lock the entire string to something generic for the device
    type. Chrome could perhaps send the following for mobile, regardless of the underlying device
    type:

    ```http
    Mozilla/5.0 (Linux; Android 9; Unspecified Device) AppleWebKit/537.36 (KHTML, like Gecko) Version/4.0 Chrome/71.1.2222.33 Mobile Safari/537.36
    ```

    And the following for desktop, similarly irrepspective of the underlying device:

    ```http
    Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/71.1.2222.33 Safari/537.36
    ```

    These strings would stay static, adding cruft to every request from now until forever.

    We can ratchet this deprecation over time, beginning by freezing the version numbers in the
    header, then removing platform and model information as developers migrate to the alternative
    mechanisms proposed below.

2.  Similarly, user agents would freeze the `navigator.appVersion`, `navigator.platform`,
    `navigator.productSub`, `navigator.vendor`, and `navigator.userAgent` attributes to
    appropriate values for the frozen `User-Agent` string.

3.  Chrome should introduce several new Client Hint header fields:

    1.  The `UA` header field represents the user agent's brand and major version. For example:

        ```http
        UA: "Chrome 73"
        ```

        Note: See the GREASE-like discussion below for how we could anticipate the inevitable lies
        which user agents might want to tell in this field.

    2.  The `UA-Platform` header field represents the platform's brand and major version. For example:

        ```http
        UA-Platform: "Win 10"
        ```

    3.  The `UA-Arch` header field represents the underlying architecture's instruction set and
        width. For example:

        ```http
        UA-Arch: "ARM64"
        ```

    4.  The `UA-Model` header field represents the user agent's underlying device model. For example:

        ```http
        UA-Model: "Pixel 2 XL"
        ```
4.  These client hints should also be exposed via JavaScript APIs, perhaps hanging off a new
    `navigator.getUserAgent()` method as something like:


    ```idl
    interface mixin NavigatorUA {
      [SecureContext] Promise<NavigatorUAData> getUserAgent();
    };
    Navigator includes NavigatorUA;

    interface NavigatorUAData {
      readonly attribute DOMString brand;          // "Chrome"
      readonly attribute DOMString version;        // "69"
      readonly attribute DOMString platform;       // "Win 10"
      readonly attribute DOMString architecture;   // "ARM64"
      readonly attribute DOMString model;          // ""
    };
    ```

    User agents can make intelligent decisions about what to reveal in each of these attributes.
    Top-level sites a user visits frequently (or installs!) might get more granular data than
    cross-origin, nested sites, for example. We could conceivably even inject a permission prompt
    between the site's request and the Promise's resolution, if we decided that was a reasonable
    approach.

User agents will attach the `UA` header to every secure outgoing request by default, with a value
that includes only the major version (e.g. "`Chrome 69`"). Servers can opt-into receiving more
detailed version information in the `UA` header, along with the other available Client Hints, by
delivering an `Accept-CH` header or `Accept-CH-Lifetime` header in the usual way.

Note the word "secure" in the paragraph above, and the `SecureContext` attribute in the IDL: these
client hints will not be delivered to plaintext endpoints. Non-secure HTTP will receive only the
locked `User-Agent` string, now and forever.

### For example...

A user agent's initial request to `https://example.com` will include the following request headers:

```http
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko)
            Chrome/71.1.2222.33 Safari/537.36
UA: "Chrome 74"
```

If a server delivers the following response header:

```http
Accept-CH: UA, UA-Platform, UA-Arch
```

Then subsequent requests to `https://example.com` will include the following request headers:

```http
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko)
            Chrome/71.1.2222.33 Safari/537.36
UA: "Chrome 74.0.3424.124"
UA-Platform: "macOS 12"
UA-Arch: "ARM64"
```

The user agent can make reasonable decisions about when to honor requests for detailed user agent
hints and first-parties can use `Feature-Policy` to decide which third-parties they'd like to
privilege with detailed user agent information. In any event, moving to this opt-in model means
that the extent of a site's usage can be monitored and evaluated.

# FAQ

## Do we really need to neuter the JavaScript interface too?

An excellent question! This proposal assumes that developers with access to JavaScript execution
do not need the user agent string in order to determine which resources to load and how they
ought to behave. They can examine other parts of the exposed API surface
(`WEBGL_debug_renderer_info`, for example). Developers who need this kind of information at
request-time could probably migrate to alternative mechanisms like Client Hints.

Our goal should eventually be to ratchet down on some of this granularity as well, and my intuition
is that we'll be able to do that more cleanly if we adjust the UA string in one fell swoop, and
then move on to the rest rather than doubling back at some point in the future.

## What about the compatibility hit we'll take from UA sniffing?

Locking the User-Agent string will lock in the existing behavior of UA-sniffing libraries. That
may break things in the future if we diverge significantly from today's behavior in some 
interesting way, but that doesn't seem like a risk unique to this proposal.

## Should the UA string be a set?

History has shown us that there are real incentives for user agents to lie about their branding
in order to thread the needle of sites' sniffing scripts. While I'm optimistic that we can reset
expectations around sniffing by freezing the thing that's sniffed-upon today, and creating a sane 
set of options for developers, it's likely that this is hopelessly naive. It's reasonable to
ponder what we should do to encourage sniffing in the right way, if we believe it's going to
happen one way or another.

One suggestion is to model `UA` as a set, rather than a single entry. For example, user agents
might encourage standardized processing of the `UA` string by randomly including additional,
intentionally incorrect, comma-separated entries with arbitrary ordering (similar conceptually
to TLS's [GREASE][4]). Chrome 73's `UA` value might then be `"Chrome 73", "NotBrowser 12"`, or
`"BrowsingIsFun Version 12b", "Chrome 73"`.

We'd reflect this value in the `NavigatorUAData.brand` attribute.

[4]: https://tools.ietf.org/html/draft-ietf-tls-grease-01

## As a concrete example, what UA string would Chrome on iOS send?

Chrome on iOS (as well as Edge on Android and iOS, Firefox on iOS, and every other non-Safari
browser on iOS) are interesting cases in which the underlying browser engine on a given platform
doesn't match the engine that the relevant browser built themselves. What should we do in these
cases?

I think we have a few options for the string:

1.  `"Chrome 73"`, which has the least entropy, but also sets poor expectations.
2.  `"CriOS 73"` (or `"Chrome on iOS 73"`, or similar) which is basically what's sent today, and categorizes the browser as distinct.
3.  `"CriOS 73", "Safari 12"`, which is interesting.
4.  `"Chrome 73", "Safari 12"`, which is more interesting.

I think I prefer the second.

(A more verbose alternative could add a `UA-Engine` header, containing values like `Blink`,
`EdgeHTML`, `Gecko`, or `WebKit`.)
