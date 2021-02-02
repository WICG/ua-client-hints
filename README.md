# User Agent Client Hints

This repository hosts the User Agent Client Hints specification.

## Contributing

We welcome contributions in the form of new issues, comments on existing issues,
and pull requests. Before getting started, please read our
[Contributing Guide](CONTRIBUTING.md) and the [Code of Conduct](CODE_OF_CONDUCT.md).

# Explainer: Reducing `User-Agent` Granularity

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

By default, servers should receive only the user agent's brand and significant version number. Servers
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

    And the following for desktop, similarly irrespective of the underlying device:

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

3.  Browsers should introduce several new Client Hint header fields:

    1.  The `Sec-CH-UA` header field represents the user agent's brand and significant version. For example:

        ```http
        Sec-CH-UA: "Chrome"; v="73", "?Not:Your Browser"; v="11"
        ```

        Note: See the GREASE-like discussion below for how we could anticipate the inevitable lies
        which user agents might want to tell in this field and to learn more about the admittedly odd looking
        `"?Not:Your Browser"; v="11"`.

    2.  The `Sec-CH-UA-Platform` header field represents the platform's brand and major version. For example:

        ```http
        Sec-CH-UA-Platform: "Windows"
        ```

    3.  The `Sec-CH-UA-Arch` header field represents the underlying architecture's instruction set and
        width. For example:

        ```http
        Sec-CH-UA-Arch: "arm"
        ```

    4.  The `Sec-CH-UA-Model` header field represents the user agent's underlying device model. For example:

        ```http
        Sec-CH-UA-Model: "Pixel 2 XL"
        ```

    5.  The `Sec-CH-UA-Mobile` header field represents whether the user agent should receive a specifically "mobile"
        UX.

        ```http
        Sec-CH-UA-Mobile: ?1
        ```
    6.  The `Sec-CH-UA-Full-Version` header field represents the user agent's full version.

        ```http
        Sec-CH-UA-Full-Version: "73.1.2343B.TR"
        ```

4.  These client hints should also be exposed via JavaScript APIs, perhaps hanging off a new
    `navigator.userAgentData` attribute as something like:


    ```idl
    dictionary NavigatorUABrandVersion {
      DOMString brand;    // "Google Chrome"
      DOMString version;  // "84"
    };

    dictionary UADataValues {
      DOMString platform;         // "PhoneOS"
      DOMString platformVersion;  // "10A"
      DOMString architecture;     // "arm"
      DOMString model;            // "X644GTM"
      DOMString uaFullVersion;    // "73.32.AGX.5"
    };

    [Exposed=(Window,Worker)]
    interface NavigatorUAData {
      readonly attribute FrozenArray<NavigatorUABrandVersion> brands;              // [ {brand: "Google Chrome", version: "84"}, {brand: "Chromium", version: "84"} ]
      readonly attribute boolean mobile;                                                 // false
      Promise<UADataValues> getHighEntropyValues(sequence<DOMString> hints); // { "PhoneOS", "10A", "arm", "X644GTM", "73.32.AGX.5" }
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
`Sec-CH-UA-Mobile` headers by default.
Servers can opt-into receiving more detailed UA information using the other available Client Hints,
by delivering an `Accept-CH` header opt-in, that includes the information they are interested in.

Note the word "secure" in the paragraph above, and the `SecureContext` attribute in the IDL: these
client hints will not be delivered to plaintext endpoints. Non-secure HTTP will receive only the
locked `User-Agent` string, now and forever.

### For example...

A user agent's initial request to `https://example.com` will include the following request headers:

```http
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko)
            Chrome/71.1.2222.33 Safari/537.36
Sec-CH-UA: "Chrome"; v="74", ";Not)Your=Browser"; v="13"
Sec-CH-Mobile: ?0
```

If a server delivers the following response header:

```http
Accept-CH: Sec-CH-UA-Full-Version, Sec-CH-UA-Platform, Sec-CH-UA-Arch
```

Then subsequent requests to `https://example.com` will include the following request headers:

```http
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko)
            Chrome/71.1.2222.33 Safari/537.36
Sec-CH-UA: "Chrome"; v="74", ";Not)Your=Browser"; v="13"
Sec-CH-UA-Full-Version: "74.0.3424.124"
Sec-CH-UA-Platform: "macOS"
Sec-CH-UA-Arch: "arm"
```

The user agent can make reasonable decisions about when to honor requests for detailed user agent
hints and first-parties can use `Feature-Policy` to decide which third-parties they'd like to
privilege with detailed user agent information. In any event, moving to this opt-in model means
that the extent of a site's usage can be monitored and evaluated.

For developers that prefer using user agent information to make client-side decisions, they could use the JavaScript API:

```javascript
  const uaData = navigator.userAgentData;
  const brands = uaData.brands;     // [ {brand: "Google Chrome", version: "84"}, {brand: "Chromium", version: "84"} ]
  const mobileness = uaData.mobile; // false
  (async ()=>{
    // `getHighEntropyValues()` returns a Promise, so needs to be `await`ed on.
    const highEntropyValues = await uaData.getHighEntropyValues(
      ["platform", "platformVersion", "architecture", "model", "uaFullVersion"]);
    const platform = highEntropyValues.platform;               // "Mac OS X"
    const platformVersion = highEntropyValues.platformVersion; //"10_15_4"
    const architecture = highEntropyValues.architecture;       // "x86"
    const model = highEntropyValues.model;                     // ""
    const uaFullVersion = highEntropyValues.uaFullVersion;     // "84.0.4113.0"
  })();

```

# Use-cases

This section attempts to document the current uses for the `User-Agent` string,
and how similar functionality could be enabled using UA-CH.

## Differential serving

### Based on browser features
This use-case enables services like [polyfill.io](https://polyfill.io) to serve
custom-tailored polyfills to their users, without bloating up the experience of
modern browser users.
Similarly, when serving Javascript to users, one can avoid
transpilation (which can result in bloat and inefficient code) for browsers
that support the latest ES features that were used.
Finally, when serving images, some browsers don't update their `Accept` request
headers, while in other cases (*cough* WebP *cough*) the MIME type is not
descriptive enough to distinguish between different variants of the same
format. In those cases, knowing the browser and its version can be critical to
serving the right image variant.

For that use case to work, the server needs to be aware of the browser and its
meaningful version, and map that to a list of available features. That enables
it to know which polyfill or code variant to serve.

Services that wish to do that using UA-CH will need to inspect the `Sec-CH-UA`
header, that is sent by default on every request, and modify their response
based on that.

### Browser bug workaround
Some browser versions have well-known bugs which require content to workaround
them. Triggering those bugs can result in browser crashes, content breakage and
other issues, and those bugs are by definition not something that can be
feature detected. Therefore, content needs to avoid them altogether for
affected browser versions.

For that use case, servers need to be aware of the browser and its meaningful
version, be aware of browser bugs that impact them, and apply workarounds if
the current browser version is impacted.

Services that wish to do that using UA-CH will need to inspect the `Sec-CH-UA`
header, sent by default on every request, and use it to modify their response.

## Marketshare Analytics
A browser's market share can be extremely important. Having visibility into a
browser's usage can encourage developers to test in that particular browser,
ensuring fewer compatibility issues for its users. On top of that, a browser's
market share can have a direct impact on the browser vendors' business goals,
ensuring future development of the browser.

For marketshare analytics to work, the server needs to be aware of the server
and its meaningful version, in order to be able to register them and find their
relative market shares.

Sites that wish to provide market share analytics using UA-CH will need to
inspect the `Sec-CH-UA` header, that is sent by default on every request, and
keep a record of it.


## Content adaptation
Content adaptation is ensuring that users get content that's tailored to their
needs.  There are [many dimensions to content
adaptation](https://blog.yoav.ws/adapting_without_assumptions/) beyond the UA
string: [viewport
dimensions](https://developers.google.com/web/fundamentals/performance/optimizing-content-efficiency/client-hints),
[device memory](https://w3c.github.io/device-memory/), [user
preferences](https://wicg.github.io/savedata/) and more.  This sub-section
covers content adaptation needs that rely on information that is part of the
current `User-Agent` string.

### Browser based adaptation
Some sites choose to serve slightly different content to different browsers.
The reason for that vary. Some reasons are legitimate (e.g. wanting to serve
different experiences to different browsers due to their feature support).
Other reasons are slightly less legitimate (e.g. warning users that the site's
developers hasn't tested in their browser).  And then there are reasons which
are outright wrong (e.g. Willingness to block certain browsers' users from
accessing the site).

As browsers, we want to enable the former, while discouraging the latter.

### Mobile specific site
Many site owners serve different content between mobile and desktop sites.
While responsive web design has made it possible to serve multiple form factors
using a single code base, there are still cases where serving a mobile-specific
version can be better adapted.

For those cases, serving a mobile specific sites to users on mobile devices can
be helpful. For that to work, the server needs to be aware, at HTML serving
time, whether the user is on a mobile device or not.

Sites that wish to serve mobile-specific sites using UA-CH can do that using
the `Sec-CH-UA-Mobile` headers that are sent by default on every request.

### Low-powered devices
Some sites serve different content to low powered devices that cannot deal with
CPU intensive tasks, large video and images, etc.  Such content adaptation
typically uses the device model information that's integrated in the current
`User-Agent` string for that purpose, relying on server-side databased to
convert device models into memory, CPU power, and other categories on which
they want to split their content.

If the dimension on which the split is made is memory, the Device-Memory Client
Hint can be used to make that distinction.  Otherwise, with UA-CH, sites can
still retrieve the device model by opting in to the `Sec-CH-UA-Model` hint.

Both of these hints are not sent by default, so require some extra work.

Top-level origins will need to send `Accept-CH: Device-Memory, Model` headers
with their responses to opt-in to receiving those hints.  In case where they
absolutly need to perform that adaptation on every navigation request, a
redirect would be required here in case where the hints are not present in a
browser that supports them.  There's [ongoing
work](https://github.com/WICG/client-hints-infrastructure/issues/16) to
eliminate that extra step.

Third-party origins that need to perform such adaptation would need
[delegation](https://github.com/WICG/client-hints-infrastructure#cross-origin-hint-delegation)
from the top-level origin. The top-level origin would need to opt-in using
`Accept-CH`, as well as add `Feature-Policy` headers that delegate those hints
to the third-party origin.

### OS specific styles
Some sites may wish to tailor their interfaces to match the user's OS. While
progressive enhancement is likely to be a better path here (e.g. through the
application of different button styles using script), there may be cases where
folks would wish to deliver tailored inline styles based on the platform and
platform version.

Those cases are very similar to the case discussed above (in "Low-powered
devices"), only with the `Sec-CH-UA-Platform` and `Sec-CH-UA-Platform-Version` hints.

### OS integration
Similarly, some sites would want to change links to OS specific ones (e.g.
[Android intent
links](https://developer.chrome.com/multidevice/android/intents)).  While,
again, progressive enhancement can be used to modify those links using script,
rather than bake them into the HTML, some sites may prefer server-side
adaptation.

Again, like the "OS specific styles" case, they'd need to use the platform and
platform version hints to do so.

### Browser and OS specific experiments
Some servers may like to limit their multi variant experimentation to specific
browsers, specific platforms or specific versions of any of the above.  For
experiments that are limited to browser and version, those sites can use the
`Sec-CH-UA` values sent by default on requests.  If they require platform and
its version, they'd have to opt-in for those hints, or use client-side scripts
to control the experimentation.

## User login notification
Many sites, especially security sensitive ones, like to notify their users when
log-in from a new device happens. That enables users to be aware of those
logins, and take action in case it's not a login that's done by them or on
their behalf.

For those notifications to be meaningful, sites need to recognize and
communicate the commercial brand of the browser to the user. These messages
often also include the platform and its version in order to make sure the user
knows which device is in question.

Since such messaging doesn't require any server-side adaptation, it's better
for this case to use the `userAgentData.getHighEntropyData()` method in order
to retrieve the required information.

## Download of appropriate binary executables
Some sites are used to download binary executables of native applications, and
need to be able to propose the right binary to the user by default.  The right
binary executable for the current user depends on a few factors: their
operating system, its version, as well as their CPU architecture.

In order to tackle that use case, download sites can opt-in to receive the
`Sec-CH-UA-Platform`, `Sec-CH-UA-Platform-Version` and `Sec-CH-UA-Architecture` hints (or query them
through the API), in order to ensure the right binary is offered to the user by
default.

## Conversion modeling
Some machine learning models use various details from the `User-Agent` string
in order to estimate various things about users of those user agents.  Similar
modeling would still be possible, but will require explicit opt-in to collect
the required bits of information.

## Vulnerability filtering
In some environments, proxy servers may be used to verify that the different
users accessing information are not doing so from obsolete devices, that are
potentialy vulnerable to security issues.  While the browser and version
information available from `Sec-CH-UA` can provide some information, the
browser and OS full version are often useful for that kind of analysis.

Such proxies would have to add a redirect step that opts-in to getting the
browser full version and the platform version in order to continue to get
access to those hints.

## Logs and debugging
Many services log the `User-Agent` string today and can use it in various ways
when analyzing past traffic or when trying to debug errors related to their
service.  Those services will have to use the lower entropy values available
through `Sec-CH-UA` for logging purposes, or opt-in to receive higher-entropy
hints. The latter doesn't seem like something services should do just for
forensic purposes. On the other hand, when specific issues are encountered, it
may make sense for those services to opt-in to receive more details on the user
agent, or use the `userAgentData.getHighEntropyData()` API for that purpose.

## Fingerprinting

User fingerprinting is the practice of gathering multiple bits of user
information from multiple sources and intersecting them together to create a
unique signature of the user, that would enable to recognize them later on,
even if they clear state from their browsers (e.g. by deleting cookies).

For those cases, the origin needs to gather as much entropy as possible, so is
likely to collect all the hints.

### Spam filtering and bot detection
This is a case of fingerprinting that is not user-hostile, and therefore one we
would like to preserve.  With UA-CH this will be initially enabled by active
collection of the various hints.

We hope that alternative methods or APIs will exist to address the
spam filtering and bot detection use cases in the future, as browsers may decide
to intervene on behalf of their users by limiting the collection of
user-identifying entropy (e.g., the
[Privacy Budget](https://github.com/bslassey/privacy-budget) proposal).

### Persistent user tracking
This is a case of fingerprinting that this proposal *explicitly tries to make
harder*.  Like the case of "spam filtering", it would still be feasible to
actively collect all the hints about the user as bits of entropy. Unlike the
above case, this is something that proposals such as the Privacy Budget aim to
prevent, without providing any alternative mechanisms for persistent user
tracking.

## Blocking known bots and crawlers
Currently, the `User-Agent` string is often used as a brute-force way to block
known bots and crawlers.  There's a concern that moving "normal" traffic to
expose less entropy by default will also make it easier for bots to hide in the
crowd.  While there's some truth to that, that's not enough reason for making the crowd
be more personally identifiable.

Similar to the spam filtering case, there's hope that alternative methods would
be able to replace `User-Agent` string matching for this use-case.


# FAQ

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

Reseting expectations may help to prevent abuse of the UA string's brand in the
short term, but probably won't help in the long run.

Having `UA` be a set enables browsers to express their brand, as well as their
different equivalence sets. We hope that this enabled expressiveness will
enable sites to differentially serve based on capabilities while minimizing
wrongfully categorizing browsers in the wrong buckets.

Complementary to that, user agents might encourage standardized processing of
the `UA` string by randomly including additional, intentionally incorrect,
comma-separated entries with arbitrary ordering (similar conceptually to TLS's
[GREASE][4]).

Let's examine a few examples:
* In order to avoid sites from barring unknown browsers from their allow lists, Chrome 73 could send a UA set that
    includes an non-existent browser, and which varies once in a while.
  - `"Chrome"; v="73", ";Not=Browser"; v="12"`
* In order to enable equivalence classes based on Chromium versions, Chrome could add the rendering engine and its
    version to that.
  - `"Chrome"; v="73", ";Not=Browser"; v="12", "Chromium"; v="73"`
* In order to encourage sites to rely on equivalence classes based on Chromium versions rather than exact UA sniffing,
    Chrome might remove itself from the set entirely.
  - `"Chromium"; v="73", ";Not=Browser"; v="12"`
* Browsers based on Chromium may use a similar UA string, but use their own brand as part of the set, enabling sites to
    count them.
  - `"Awesome Browser"; v="60", "Chrome"; v="73", "Chromium"; v="73"`

We'd reflect this value in the `navigator.userAgentData.brands` attribute, which returns an array of dictionaries
containing brand and version.

[4]: https://tools.ietf.org/html/draft-ietf-tls-grease-01

## As a concrete example, what UA string would Chrome on iOS send?

Chrome on iOS (as well as Edge on Android and iOS, Firefox on iOS, and every other non-Safari
browser on iOS) are interesting cases in which the underlying browser engine on a given platform
doesn't match the engine that the relevant browser built themselves. What should we do in these
cases?

There are a few options for the string:

1.  `"Chrome"; v="73", ")Friendly-Browsing"; v="99"`, which has the least entropy, but also sets poor expectations.
2.  `"CriOS"; v="73", ")Friendly-Browsing"; v="99"`` (or `"Chrome on iOS", v="73"`, or similar) which is basically
    what's sent today, and categorizes the browser as distinct.
3.  `"CriOS"; v="73", ")Friendly-Browsing"; v="99"`, "Safari"; v="12"`, which is interesting.
4.  `")Friendly-Browsing"; v="99", "Chrome"; v="73", "Safari"; v="12"`, which is more interesting.


## Wait a minute, where is the Client Hints infrastructure specified?

The infrastructure is specified as a separate [document](https://wicg.github.io/client-hints-infrastructure), and integration with Fetch and HTML happens there.

## What's with the `Sec-CH-` prefix?

Based on some discussion in [w3ctag/design-reviews#320](https://github.com/w3ctag/design-reviews/issues/320#issuecomment-435874298),
it seems reasonable to forbid access to these headers from JavaScript, and demarcate them as
browser-controlled client hints so they can be documented and included in requests without triggering
CORS preflights. A `Sec-CH-` prefix seems like a viable approach.

## How does `Sec-CH-UA-Mobile` define "mobile"?

This is a tough question. The motivation for the header is that a majority of user-agent header
sniffing is used by the server to decide if a "desktop" or "mobile" UX should be served. This is
currently implicitly defined in most modern browsers because they have two distinct UIs, a
"desktop" version (i.e. Windows, Mac OS, etc.) and a "mobile" version (i.e. Android, iOS). In general,
most browsers will also explicitly send "Mobile" in user-agent strings on "mobile" platforms. As
such, servers will usually use the platform or presence of this "mobile" identifier. It's also worth
pointing out that most modern browsers also have an explicit "request desktop site" UI element in their
mobile versions which should be honored. In a more general sense, though, a "mobile" experience could be
seen as a UX designed with smaller screens and touch-based interface in mind.

# Considered alternatives
## Freezing the UA string and reducing its information density without providing an alternative mechanism
We've considered removing information from the UA string, similar to WebKit's [attempt](https://webkit.org/blog/8042/release-notes-for-safari-technology-preview-46/#post-8042:~:text=Froze%20the%20user%2Dagent%20string%20to%20reduce,to%20prevent%20its%20use%20for%20fingerprinting).

We ruled out this alternative as it seems to leave many use-cases unaddressed.

## Restructuring of the User-Agent string

Restructuring the User-Agent string could have addressed some of the compatibility concerns with today's use of the User-Agent string, and potentially discourage its abuse.
At the same time, it would have incurred same or higher migration costs as this proposal. In particular, attempts to restructure the string in-place would result in a lot of breakage of legacy apps.
On top of that, it wouldn’t have addressed any of the passive entropy concerns that are motivating this proposal.

