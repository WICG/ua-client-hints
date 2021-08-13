# User Agent Client Hints

This repository hosts the User Agent Client Hints (UA-CH) specification.

UA-CH introduces the following [Client Hints](https://tools.ietf.org/html/rfc8942) HTTP headers and
a corresponding JavaScript API:

* `Sec-CH-UA-Arch`
* `Sec-CH-UA-Bitness`
* `Sec-CH-UA-Mobile`
* `Sec-CH-UA-Model`
* `Sec-CH-UA-Platform`
* `Sec-CH-UA-Platform-Version`
* `Sec-CH-UA`
* `Sec-CH-UA-Full-Version` (deprecated in favor of `Sec-CH-UA-Version-List`)
* `Sec-CH-UA-Version-List`


## Contributing

We welcome contributions in the form of new issues, comments on existing issues,
and pull requests. Before getting started, please read our
[Contributing Guide](CONTRIBUTING.md) and the [Code of Conduct](CODE_OF_CONDUCT.md).

# Explainer: Reducing `User-Agent` Granularity

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->

  - [A Problem](#a-problem)
    - [Challenges](#challenges)
  - [A Proposal](#a-proposal)
    - [For example...](#for-example)
- [Use-cases](#use-cases)
  - [Differential serving](#differential-serving)
    - [Based on browser features](#based-on-browser-features)
    - [Browser bug workaround](#browser-bug-workaround)
  - [Marketshare Analytics](#marketshare-analytics)
  - [Content adaptation](#content-adaptation)
    - [Browser based adaptation](#browser-based-adaptation)
    - [Mobile specific site](#mobile-specific-site)
    - [Low-powered devices](#low-powered-devices)
    - [OS specific styles](#os-specific-styles)
    - [OS integration](#os-integration)
    - [Browser and OS specific experiments](#browser-and-os-specific-experiments)
  - [User login notification](#user-login-notification)
  - [Download of appropriate binary executables](#download-of-appropriate-binary-executables)
  - [Conversion modeling](#conversion-modeling)
  - [Vulnerability filtering](#vulnerability-filtering)
  - [Logs and debugging](#logs-and-debugging)
  - [Fingerprinting](#fingerprinting)
    - [Spam filtering and bot detection](#spam-filtering-and-bot-detection)
    - [Persistent user tracking](#persistent-user-tracking)
  - [Blocking known bots and crawlers](#blocking-known-bots-and-crawlers)
- [FAQ](#faq)
  - [What user needs are solved by reducing the `User-Agent` header or adding User Agent Client Hints to the web platform?](#what-user-needs-are-solved-by-reducing-the-user-agent-header-or-adding-user-agent-client-hints-to-the-web-platform)
  - [Do we really need to neuter the JavaScript interface (i.e. `Navigator.userAgent`) too?](#do-we-really-need-to-neuter-the-javascript-interface-ie-navigatoruseragent-too)
  - [What about the compatibility hit we'll take from UA sniffing?](#what-about-the-compatibility-hit-well-take-from-ua-sniffing)
  - [Should the UA string really be a set?](#should-the-ua-string-really-be-a-set)
  - [As a concrete example, what UA string would Chrome on iOS send?](#as-a-concrete-example-what-ua-string-would-chrome-on-ios-send)
  - [Wait a minute, where is the Client Hints infrastructure specified?](#wait-a-minute-where-is-the-client-hints-infrastructure-specified)
  - [What's with the `Sec-CH-` prefix?](#whats-with-the-sec-ch--prefix)
  - [How does `Sec-CH-UA-Mobile` define "mobile"?](#how-does-sec-ch-ua-mobile-define-mobile)
  - [Aren’t we duplicating a lot of information already in the `User-Agent` header?](#arent-we-duplicating-a-lot-of-information-already-in-the-user-agent-header)
  - [Aren’t you adding a lot of new headers? Isn’t that going to bloat requests?](#arent-you-adding-a-lot-of-new-headers-isnt-that-going-to-bloat-requests)
- [Considered alternatives](#considered-alternatives)
  - [Freezing the UA string and reducing its information density without providing an alternative mechanism](#freezing-the-ua-string-and-reducing-its-information-density-without-providing-an-alternative-mechanism)
  - [Restructuring of the User-Agent string](#restructuring-of-the-user-agent-string)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

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

There's a lot of entropy wrapped up in the UA string that is sent to servers by default, for all
first- and third-party requests. This makes it an important part of fingerprinting schemes of all
sorts, as servers can [passively](https://w3c.github.io/fingerprinting-guidance/#dfn-passive-fingerprinting)
capture this information without the user agent’s, or more importantly the user’s, awareness or
ability to intervene or prevent such collection.

Safari has [recently taken some steps][1] to reduce the entropy of the user agent string, initially
locking it to a single value, period, and then [backing off a bit][2] due to developer feedback.
This document proposes a mechanism which might allow user agents generally to be a little more
aggressive.

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
    executables for download (32 vs 64 bit, ARM vs Intel, etc). Model information is likewise useful
    when bugs are limited to particular kinds of devices.

4.  Sophisticated developers use model/make to tailor their sites to the capabilities of the
    device (e.g. [Facebook Year Class][3]) and to pinpoint performance bugs and regressions which
    sometimes are specific to model/make.

These are use cases that are interesting for us to support. Browsers certainly have bugs, and
developers certainly would be well-served by being able to work around them. We'll keep these
challenges in mind with the proposal that follows.

[3]: https://code.fb.com/android/year-class-a-classification-system-for-android/

## A Proposal

By default, servers should receive only the user agent's brand, significant version number,
underlying platform name, and mobileness. Servers can opt-into receiving information about minor
versions, the underlying operating system's major version, and details about the underlying
architecture, bitness and device model. The user agent can make reasonable decisions about how to
respond to sites' requests for more granular data. Browsers might even offer user-controlled
settings to determine which hints or values should be allowed for some or all sites. We might
accomplish this as follows:

1.  Browsers should deprecate the `User-Agent` string over time, initially locking bits of its
    value, and ramping up over time to lock the entire string to something generic for the device
    type. Chrome could perhaps send the following for mobile, regardless of the underlying device
    type:

    ```http
    Mozilla/5.0 (Linux; Android 10; K) AppleWebKit/537.36 (KHTML, like Gecko) Version/4.0 Chrome/71.0.0.0 Mobile Safari/537.36
    ```

    And the following for desktop, similarly irrespective of the underlying device characteristics
    such as bitness or architecture:

    ```http
    Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/71.0.0.0 Safari/537.36
    ```

    These strings would stay static, adding cruft to every request from now until forever.

    We can ratchet this deprecation over time, beginning by freezing the version numbers in the
    header, then removing platform and model information as developers migrate to the alternative
    mechanisms proposed below.

2.  Similarly, user agents would freeze the `navigator.appVersion`, `navigator.platform`,
    `navigator.productSub`, `navigator.vendor`, and `navigator.userAgent` attributes to
    appropriate values for the frozen `User-Agent` string.

3.  Browsers should introduce several new Client Hint header fields:

    1.  The `Sec-CH-UA` header field represents the user agent's brand and significant version. For
        example:

        ```http
        Sec-CH-UA: "Chrome"; v="73", "Chromium"; v="73", "?Not:Your Browser"; v="11"
        ```

        Note: See the GREASE-like discussion below for how we could anticipate the inevitable lies
        which user agents might want to tell in this field and to learn more about the admittedly
        odd looking `"?Not:Your Browser"; v="11"`.

    1.  The `Sec-CH-UA-Arch` header field represents the underlying architecture's instruction set.
        For example:

        ```http
        Sec-CH-UA-Arch: "arm"
        ```

    1.  The `Sec-CH-UA-Bitness` header field represents the underlying architecture's bitness
        (i.e., the size in bits of an integer or memory address). This could be used to determine
        which binary to serve for downloads.

        For example:

        ```http
        Sec-CH-UA-Bitness: "64"
        ```

    1.  The `Sec-CH-UA-Mobile` header field represents whether the user agent should receive a
        specifically "mobile" UX.

        ```http
        Sec-CH-UA-Mobile: ?1
        ```

    1.  The `Sec-CH-UA-Model` header field represents the user agent's underlying device model. For
        example:

        ```http
        Sec-CH-UA-Model: "Pixel 2 XL"
        ```

    1.  The `Sec-CH-UA-Full-Version` header field represents the user agent's full version.

        ```http
        Sec-CH-UA-Full-Version: "73.1.2343B.TR"
        ```

        Advisement: `Sec-CH-UA-Full-Version` is deprecated and will be removed in the future.
        Developers should use `Sec-CH-UA-Version-List` instead.

    1.  The `Sec-CH-UA-Platform` header field represents the platform's brand and major version. For
        example:

        ```http
        Sec-CH-UA-Platform: "Windows"
        ```

    1.  The `Sec-CH-UA-Version-List` header field represents the full version for each brand in its
        brand list. For example:

        ```http
        Sec-CH-UA-Version-List: "Edge"; v="92.0.902.73", "Chromium"; v="92.0.4515.131", "?Not:Your Browser"; v="3.1.2.0"
        ```

4.  These client hints should also be exposed via JavaScript APIs via a new
    `navigator.userAgentData` attribute:


    ```idl
    dictionary NavigatorUABrandVersion {
      DOMString brand;    // "Google Chrome"
      DOMString version;  // "84"
    };

    dictionary UADataValues {
      FrozenArray&lt;NavigatorUABrandVersion&gt; brands; // [ {brand: "Google Chrome", version: "84"}, {brand: "Chromium", version: "84"} ]
      boolean mobile;             // true
      DOMString architecture;     // "arm"
      DOMString bitness;          // "64"
      DOMString model;            // "X644GTM"
      DOMString platform;         // "PhoneOS"
      DOMString platformVersion;  // "10A"
      DOMString uaFullVersion; // deprecated in favor of versionList
      FrozenArray&lt;NavigatorUABrandVersion&gt; versionList; // [ {brand: "Google Chrome", version: "84.0.4147.0"}, {brand: "Chromium", version: "84.0.4147"} ]
    };

    [Exposed=(Window,Worker)]
    interface NavigatorUAData {
      readonly attribute FrozenArray<NavigatorUABrandVersion> brands; // [ {brand: "Google Chrome", version: "84"}, {brand: "Chromium", version: "84"} ]
      readonly attribute boolean mobile; // false
      readonly attribute platform; // "PhoneOS"
      Promise<UADataValues> getHighEntropyValues(sequence<DOMString> hints); // [ "arm", "64", "X644GTM", "PhoneOS", "10A", "73.32.AGX.5" ]
    };

    interface mixin NavigatorUA {
      [SecureContext] readonly attribute NavigatorUAData userAgentData;
    };

    Navigator includes NavigatorUA;
    WorkerNavigator includes NavigatorUA;
    ```

    User agents can make intelligent decisions about what to reveal in each of these attributes.
    Top-level sites a user visits frequently (or installs!) might get more granular data than
    cross-origin, nested sites, for example. We could conceivably even inject a permission prompt
    between the site's request and the Promise's resolution, if we decided that was a reasonable
    approach.

User agents will attach the `Sec-CH-UA` header to every secure outgoing request by default, with a
value that includes only the significant version (e.g. `"Chrome"; v="69"`). They will also attach
`Sec-CH-UA-Mobile` and `Sec-CH-UA-Platform` headers by default. Servers can opt-into receiving more
detailed UA information using the other available Client Hints, by delivering an `Accept-CH` header
opt-in, that includes the information they are interested in.

Note the word "secure" in the paragraph above, and the `SecureContext` attribute in the IDL: these
client hints will not be delivered to plaintext endpoints. Non-secure HTTP will receive only the
reduced `User-Agent` string, now and forever.

### For example...

A user agent's initial request to `https://example.com` will include the following request headers:

```http
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko)
            Chrome/71.0.0.0 Safari/537.36
Sec-CH-UA: "Chrome"; v="74", ";Not)Your=Browser"; v="13"
Sec-CH-UA-Mobile: ?0
Sec-CH-UA-Platform: “Windows”
```

If a server delivers the following response header:

```http
Accept-CH: Sec-CH-UA-Version-List, Sec-CH-UA-Arch
```

Then subsequent requests to `https://example.com` will include the following request headers:

```http
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko)
            Chrome/71.0.0.0 Safari/537.36
Sec-CH-UA: "Chrome"; v="74", ";Not)Your=Browser"; v="13"
Sec-CH-UA-Mobile: ?0
Sec-CH-UA-Platform: "Windows"
Sec-CH-UA-Version-List: "Chrome"; v="74.0.3729.0", "Chromium"; v="74.0.3729.0", "?Not:Your Browser"; v="13.0.1.0"
Sec-CH-UA-Arch: "arm"
```

For use-cases where non-default hints are required on first request, the two
[Client Hints Reliability](https://github.com/WICG/client-hints-infrastructure/blob/main/reliability.md#client-hint-reliability)
mechanisms provide a means to do so. For example, an example using `Critical-CH` follows.

The initial request:

```http
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko)
            Chrome/71.0.0.0 Safari/537.36
Sec-CH-UA: "Chrome"; v="74", ";Not)Your=Browser"; v="13"
Sec-CH-UA-Mobile: ?0
Sec-CH-UA-Platform: “Windows”
```

The server responds that the `Sec-CH-UA-Version-List` is required on first-request in order to
deliver some optimized resource, for example:

```http
Accept-CH: Sec-CH-UA-Version-List
Critical-CH: Sec-CH-UA-Version-List
```

The client then retries the initial request with the requested hints:

```http
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko)
            Chrome/71.0.0.0 Safari/537.36
Sec-CH-UA: "Chrome"; v="74", ";Not)Your=Browser"; v="13"
Sec-CH-UA-Mobile: ?0
Sec-CH-UA-Platform: “Windows”
Sec-CH-UA-Version-List: "Chrome"; v="74.0.3729.0", "Chromium"; v="74.0.3729.0", "?Not:Your Browser"; v="13.0.1.0"
```

The user agent can make reasonable decisions about when to honor requests for detailed user agent
hints and first-parties can use `Permissions-Policy` to decide which third-parties they'd like to
privilege with detailed user agent information. In any event, moving to this opt-in model means
that the extent of a site's usage can be monitored and evaluated.

For developers that prefer using user agent information to make client-side decisions, they could
use the JavaScript API:

```javascript
  const uaData = navigator.userAgentData;
  const brands = uaData.brands;     // [ {brand: "Google Chrome", version: "84"}, {brand: "Chromium", version: "84"} ]
  const mobileness = uaData.mobile; // false
  const platform = uaData.platform; // “macOS”
  (async () => {
    // `getHighEntropyValues()` returns a Promise, so needs to be `await`ed on.
    const highEntropyValues = await uaData.getHighEntropyValues(
      ["platformVersion", "architecture", “bitness”, "model", "uaFullVersion"]);
    const platform = highEntropyValues.platform;               // "macOS"
    const platformVersion = highEntropyValues.platformVersion; //"10_15_4"
    const architecture = highEntropyValues.architecture;       // "x86"
    const bitness = highEntropyValues.bitness;                 // "64"
    const model = highEntropyValues.model;                     // ""
    const uaFullVersion = highEntropyValues.uaFullVersion;     // "84.0.4113.0"
  })();
```

# Use-cases

This section attempts to document the current uses for the `User-Agent` string,
and how similar functionality could be enabled using UA-CH.

## Differential serving

### Based on browser features
This use-case enables services like [polyfill.io](https://polyfill.io) to serve custom-tailored
polyfills to their users, without bloating up the experience of modern browser users.

Similarly, when serving Javascript to users, one can avoid transpilation (which can result in bloat
and inefficient code) for browsers that support the latest ES features that were used. Finally, when
serving images, some browsers don't update their `Accept` request headers, while in other cases
(*cough* WebP *cough*) the MIME type is not descriptive enough to distinguish between different
variants of the same format. In those cases, knowing the browser and its version can be critical to
serving the right image variant.

For that use case to work, the server needs to be aware of the browser and its meaningful version,
and map that to a list of available features. That enables it to know which polyfill or code variant
to serve.

Services that wish to do that using UA-CH will need to inspect the `Sec-CH-UA`
header, that is sent by default on every request, and modify their response
based on that.

### Browser bug workaround
Some browser versions have well-known bugs which require content to workaround them. Triggering
those bugs can result in browser crashes, content breakage and other issues, and those bugs are by
definition not something that can be feature detected. Therefore, content needs to avoid them
altogether for affected browser versions.

For that use case, servers need to be aware of the browser and its meaningful version, be aware of
browser bugs that impact them, and apply workarounds if the current browser version is impacted.

Services that wish to do that using UA-CH will need to inspect the `Sec-CH-UA` header, sent by
default on every request, and use it to modify their response.

## Marketshare Analytics
A browser's market share can be extremely important. Having visibility into a browser's usage can
encourage developers to test in that particular browser, ensuring fewer compatibility issues for
its users. On top of that, a browser's market share can have a direct impact on the browser vendors'
business goals, ensuring future development of the browser.

For market share analytics to work, the server needs to be aware of the browser and its meaningful
version, in order to be able to register them and find their relative market shares.

Sites that wish to provide market share analytics using UA-CH will need to inspect the `Sec-CH-UA`
header, that is sent by default on every request, and keep a record of it.

By design, looking at individual entries in the brands list makes it hard to distinguish between a
less-popular browser's truthful brand name and a more-popular browser's arbitrary GREASE. Since the
less-popular browser may include several popular brand names for compatibility purposes, its users
will likely be bucketed as using the more-popular one if this approach is taken, leading to
distorted views of usage share that favour already-popular browsers and with less-popular browsers
possibly never gaining any visibility.

Hence, for analytics purposes, it is better to treat the brands list as a unit, and compare it to
known lists of brands sent by the various (browser, version) pairs that are to be distinguished.
This will necessitate regular updates to the list of known lists of brands when new browser versions
are released or new browsers become popular, or else everything will get bucketed as an unknown
browser. However, as this doesn't break sites for users, failing closed for unknown browsers is
acceptable in this context.

Such a list of known lists of brands could be maintained centrally and used by many sites (as, e.g.,
browser feature support is maintained by caniuse and MDN, and consumed by many webmasters).

The specification recommends that browsers fix the brands list they send per version to make
counting usage shares simpler (and also to help with caching), so the known lists of brands can be a
simple list mapping from a set of brands to a (browser, version) pair.

## Content adaptation
Content adaptation is ensuring that users get content that's tailored to their needs. There are
[many dimensions to content adaptation](https://blog.yoav.ws/adapting_without_assumptions/) beyond
the UA string:
[viewport dimensions](https://developers.google.com/web/fundamentals/performance/optimizing-content-efficiency/client-hints),
[device memory](https://w3c.github.io/device-memory/),
[user preferences](https://wicg.github.io/savedata/) and more.  This sub-section covers content
adaptation needs that rely on information that is part of the current `User-Agent` string.

### Browser based adaptation
Some sites choose to serve slightly different content to different browsers. The reasons for that
vary. Some reasons are legitimate (e.g. wanting to serve different experiences to different browsers
due to their feature support). Other reasons are slightly less legitimate (e.g. warning users that
the site's developers haven't tested in their browser). And then there are reasons which are
outright wrong (e.g. Willingness to block certain browsers' users from accessing the site).

As browsers, we want to enable the former, while discouraging the latter.

### Mobile specific site
Many site owners serve different content between mobile and desktop sites. While responsive web
design has made it possible to serve multiple form factors using a single code base, there are still
cases where serving a mobile-specific version can be better adapted.

For those cases, serving mobile-specific sites to users on mobile devices can be helpful. For that
to work, the server needs to be aware, at HTML serving time, whether the user is on a mobile device
or not.

Sites that wish to serve mobile-specific sites using UA-CH can do that using the `Sec-CH-UA-Mobile`
headers that are sent by default on every request.

### Low-powered devices
Some sites serve different content to low powered devices that cannot deal with CPU intensive tasks,
large video and images, etc.  Such content adaptation typically uses the device model information
that's integrated in the current `User-Agent` string for that purpose, relying on server-side
databases to convert device models into memory, CPU power, and other categories on which they want
to split their content.

If the dimension on which the split is made is memory, the Device-Memory Client Hint can be used to
make that distinction.  Otherwise, with UA-CH, sites can still retrieve the device model by opting
in to the `Sec-CH-UA-Model` hint.

Both of these hints are not sent by default, so require some extra work.

Top-level origins will need to send `Accept-CH: Device-Memory, Sec-CH-UA-Model` headers with their
responses to opt-in to receiving those hints.  In cases where they absolutely need to perform that
adaptation on every navigation request, a redirect would be required here in the case where the
hints are not present in a browser that supports them. Alternatively, they might use Critical-CH to
have the client handle the additional request/response roundtrip.

Third-party origins that need to perform such adaptation would need
[delegation](https://github.com/WICG/client-hints-infrastructure#cross-origin-hint-delegation) from
the top-level origin. The top-level origin would need to opt-in using `Accept-CH`, as well as add
`Permissions-Policy` headers that delegate those hints to the third-party origin.

### OS specific styles
Some sites may wish to tailor their interfaces to match the user's OS. While progressive enhancement
is likely to be a better path here (e.g. through the application of different button styles using
script), there may be cases where folks would wish to deliver tailored inline styles based on the
platform and platform version.

Those cases are very similar to the case discussed above (in "Low-powered devices"), only with the
`Sec-CH-UA-Platform` and `Sec-CH-UA-Platform-Version` hints.

### OS integration
Similarly, some sites would want to change links to OS specific ones (e.g. [Android intent
links](https://developer.chrome.com/multidevice/android/intents)).  While, again, progressive
enhancement can be used to modify those links using script, rather than bake them into the HTML,
some sites may prefer server-side adaptation.

Again, like the "OS specific styles" case, they'd need to use the `Sec-CH-UA-Platform` and
`Sec-CH-UA-Platform-Version` hints to do so.

### Browser and OS specific experiments
Some servers may like to limit their multi variant experimentation to specific browsers, specific
platforms or specific versions of any of the above.  For experiments that are limited to browser
and version, those sites can use the `Sec-CH-UA` values sent by default on requests.  If they
require the platform and its version, they could use the default `Sec-CH-UA-Platform` hint but
would have to request the `Sec-CH-UA-Platform-Version` hint, or use client-side scripts to control
the experimentation.

## User login notification
Many sites, especially security sensitive ones, like to notify their users when a log-in from a new
device happens. That enables users to be aware of those logins, and take action in case it's not a
login that's done by them or on their behalf.

For those notifications to be meaningful, sites need to recognize and communicate the commercial
brand of the browser to the user. These messages often also include the platform and its version in
order to make sure the user knows which device is in question.

Since such messaging doesn't require any server-side adaptation, it's better for this case to use
the `userAgentData.getHighEntropyData()` method in order to retrieve the required information.

## Download of appropriate binary executables
Some sites are used to download binary executables of native applications, and need to be able to
propose the right binary to the user by default.  The right binary executable for the current user
depends on a few factors: their operating system, its version, its bitness, as well as their CPU
architecture.

In order to tackle that use case, download sites can opt-in to receive the `Sec-CH-UA-Platform`,
`Sec-CH-UA-Platform-Version`, `Sec-CH-UA-Architecture`, and `Sec-CH-UA-Bitness` hints (or query
them through the API), in order to ensure the right binary is offered to the user by default.

## Conversion modeling
Some machine learning models use various details from the `User-Agent` string in order to estimate
various things about users of those user agents.  Similar modeling would still be possible, but will
require explicit opt-in to collect the required bits of information.

## Vulnerability filtering
In some environments, proxy servers may be used to verify that the different users accessing
information are not doing so from obsolete devices that are potentially vulnerable to security
issues. While the browser and version information available from `Sec-CH-UA` can provide some
information, the browser and OS full version are often useful for that kind of analysis.

Such proxies would have to add a redirect step, or use one of the two
[Client Hint reliability mechanisms](https://github.com/WICG/client-hints-infrastructure/blob/main/reliability.md#client-hint-reliability)
that opts-in to getting the browser full version and the platform version in order to continue to
get access to those hints.

## Logs and debugging
Many services log the `User-Agent` string today and can use it in various ways when analyzing past
traffic or when trying to debug errors related to their service.  Those services will have to use
the lower entropy values available through `Sec-CH-UA` for logging purposes, or opt-in to receive
higher-entropy hints. The latter doesn't seem like something services should do just for forensic
purposes. On the other hand, when specific issues are encountered, it may make sense for those
services to opt-in to receive more details on the user agent, or use the
`userAgentData.getHighEntropyData()` API for that purpose.

## Fingerprinting

User fingerprinting is the practice of gathering multiple bits of user information from multiple
sources and intersecting them together to create a unique signature of the user, that would enable
to recognize them to be recognized later on, even if they clear state from their browsers (e.g. by
deleting cookies).

For those cases, the origin needs to gather as much entropy as possible, so it is likely to collect
all the hints.

### Spam filtering and bot detection
This is a case of fingerprinting that is not user-hostile, and therefore one we would like to
preserve.  With UA-CH this will be initially enabled by active collection of the various hints.

We hope that alternative methods or APIs will exist to address the spam filtering and bot detection
use cases in the future, as browsers may decide to intervene on behalf of their users by limiting
the collection of user-identifying entropy (e.g., the
[Privacy Budget](https://github.com/bslassey/privacy-budget) proposal).

### Persistent user tracking
This is a case of fingerprinting that this proposal *explicitly tries to make harder*.  Like the
case of "spam filtering", it would still be feasible to actively collect all the hints about the
user as bits of entropy. Unlike the above case, this is something that proposals such as the Privacy
Budget aim to prevent, without providing any alternative mechanisms for persistent user tracking.

## Blocking known bots and crawlers
Currently, the `User-Agent` string is often used as a brute-force way to block
known bots and crawlers.  There's a concern that moving "normal" traffic to
expose less entropy by default will also make it easier for bots to hide in the
crowd.  While there's some truth to that, that's not enough reason for making the crowd
be more personally identifiable.

Similar to the spam filtering case, there's hope that alternative methods would
be able to replace `User-Agent` string matching for this use-case.


# FAQ

## What user needs are solved by reducing the `User-Agent` header or adding User Agent Client Hints to the web platform?

As mentioned at the beginning of this explainer, we believe that the `User-Agent` string provides
too much passively-available entropy for actors wishing to track users via fingerprinting. By moving
the collection of this information to an active model, the user agent—either implicitly on the user’s
behalf, or explicitly through browser settings perhaps—can intervene and modify or refuse to provide
certain bits of information. This is a privacy win for users.

## Do we really need to neuter the JavaScript interface (i.e. `Navigator.userAgent`) too?

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

## Should the UA string really be a set?

Yes!

History has shown us that there are real incentives for user agents to lie
about their branding in order to thread the needle of sites' sniffing scripts,
and prevent their users from being blocked by UA-based allow/block lists.

Resetting expectations may help to prevent abuse of the UA string's brand in the
short term, but probably won't help in the long run.

Having `UA` be a set enables browsers to express their brand, as well as their
different equivalence sets. We hope that this enabled expressiveness will
enable sites to differentially serve based on capabilities while minimizing
wrongfully categorizing browsers in the wrong buckets.

Complementary to that, user agents might encourage standardized processing of
the `UA` string by randomly including additional, intentionally incorrect,
comma-separated entries with arbitrary ordering (similar conceptually to TLS's
[GREASE](https://tools.ietf.org/html/draft-ietf-tls-grease-01)).

Let's examine a few examples:
* In order to avoid sites from barring unknown browsers from their allow lists, Chrome 73 could send
  a UA set that includes an non-existent browser, and which varies once in a while.
  - `"Chrome"; v="73", ";Not=Browser"; v="12"`
* In order to enable equivalence classes based on Chromium versions, Chrome could add the rendering
  engine and its version to that.
  - `"Chrome"; v="73", ";Not=Browser"; v="12", "Chromium"; v="73"`
* Browsers based on Chromium may use a similar UA string, but use their own brand as part of the
  set, enabling sites to count them.
  - `"Awesome Browser"; v="60", "Chrome"; v="73", "Chromium"; v="73"`

We'd reflect this value in the `navigator.userAgentData.brands` attribute, which returns an array of
dictionaries containing brand and version.

## As a concrete example, what UA string would Chrome on iOS send?

Chrome on iOS (as well as Edge on Android and iOS, Firefox on iOS, and every other non-Safari
browser on iOS) are interesting cases in which the underlying browser engine on a given platform
doesn't match the engine that the relevant browser built themselves. What should we do in these
cases?

There are a few options for the string:

1.  `"Chrome"; v="73", ")Friendly-Browsing"; v="99"`, which has the least entropy, but also sets
    poor expectations.
2.  `"CriOS"; v="73", ")Friendly-Browsing"; v="99"` (or `"Chrome on iOS", v="73"`, or similar)
    which is basically what's sent today, and categorizes the browser as distinct.
3.  `"CriOS"; v="73", ")Friendly-Browsing"; v="99", "Safari"; v="12"`, which is interesting.
4.  `")Friendly-Browsing"; v="99", "Chrome"; v="73", "Safari"; v="12"`, which is more interesting,
    as it identifies as Chrome, rather than more cryptic CriOS.


## Wait a minute, where is the Client Hints infrastructure specified?

The infrastructure is specified as a separate
[document](https://wicg.github.io/client-hints-infrastructure), and integration with Fetch and HTML
happens there.

## What's with the `Sec-CH-` prefix?

Based on some discussion in [w3ctag/design-reviews#320](https://github.com/w3ctag/design-reviews/issues/320#issuecomment-435874298),
it seems reasonable to forbid access to these headers from JavaScript, and demarcate them as
browser-controlled client hints so they can be documented and included in requests without
triggering CORS preflights. A `Sec-CH-` prefix seems like a viable approach.

## How does `Sec-CH-UA-Mobile` define "mobile"?

This is a tough question. The motivation for the header is that a majority of user-agent header
sniffing is used by the server to decide if a "desktop" or "mobile" UX should be served. This is
currently implicitly defined in most modern browsers because they have two distinct UIs, a
"desktop" version (i.e. Windows, Mac OS, etc.) and a "mobile" version (i.e. Android, iOS). In
general, most browsers will also explicitly send "Mobile" in user-agent strings on "mobile"
platforms. As such, servers will usually use the platform or presence of this "mobile" identifier.
It's also worth pointing out that most modern browsers also have an explicit "request desktop site"
UI element in their mobile versions which should be honored. In a more general sense, though, a
"mobile" experience could be seen as a UX designed with smaller screens and touch-based interface
in mind.

## Aren’t we duplicating a lot of information already in the `User-Agent` header?

It’s true that for some period of time there will be redundancy between the `User-Agent` header and
the hints for platform, browser name and version, and mobileness. Over time, it may be possible to
further freeze the User-Agent header to a single static string as the web platform adopts to Client
Hints.

## Aren’t you adding a lot of new headers? Isn’t that going to bloat requests?

It’s true that we are adding multiple new headers per request. That’s quite a lot. But we don’t
expect every site to need all the hints for every request. For the most common requests, compression
technologies like [HPACK](https://www.rfc-editor.org/rfc/rfc7541.html) for HTTP/2 and
[QPACK](https://datatracker.ietf.org/doc/html/draft-ietf-quic-qpack) for HTTP/3 should help optimize
these requests. It’s also useful to keep in mind that by default User-Agent Client Hints are only
sent to top-level origins by default—explicit delegation is required to send them to third-party
origins or embedded frames. We feel the privacy wins and moving away from `User-Agent`’s legacy are
a worthwhile tradeoff here, at the expense of adding some new headers to requests.

# Considered alternatives
## Freezing the UA string and reducing its information density without providing an alternative mechanism
We've considered removing information from the UA string, similar to WebKit's
[attempt](https://webkit.org/blog/8042/release-notes-for-safari-technology-preview-46/#post-8042:~:text=Froze%20the%20user%2Dagent%20string%20to%20reduce,to%20prevent%20its%20use%20for%20fingerprinting).

We ruled out this alternative as it seems to leave many use-cases unaddressed and encourages covert
fingerprinting as a means to get at previously-removed information.

## Restructuring of the User-Agent string

Restructuring the User-Agent string could have addressed some of the compatibility concerns with
today's use of the User-Agent string, and potentially discourage its abuse.
At the same time, it would have incurred same or higher migration costs as this proposal. In
particular, attempts to restructure the string in-place would result in a lot of breakage of legacy
apps. On top of that, it wouldn’t have addressed any of the passive entropy concerns that are
motivating this proposal.
