# TAG Security & Privacy Questionnaire Answers #

* **Author:** miketaylr@chromium.org
* **Questionnaire Source:** https://www.w3.org/TR/security-privacy-questionnaire/#questions

## Questions ##

1. What information might this feature expose to Web sites or other parties, and for what purposes is that exposure necessary?

This feature exposes all the same information exposed today via the User-Agent string. See the response to question 8 for more details. Developers typically use this information for differential content serving, or to work around bugs. See https://github.com/WICG/ua-client-hints#use-cases for more use cases in more detail.

The key difference between the UA string and UA-CH is that it moves the model from passive to active. Rather than a site passively receiving all possible information for all requests, UA-CH requires the site to make active requests for hints it needs in such a way that a User Agent may observe such calls and intervene, depending on UA privacy policies or user preferences.

2. Is this specification exposing the minimum amount of information necessary to power the feature?

We believe so, yes.

3. How does this specification deal with personal information or personally-identifiable information or information derived thereof?

It does not deal directly in PII, however information provided by the User-Agent string may be used for passive fingerprinting. The EFF estimates 10 bits of entropy can be obtained from the UA string. UA-CH provides a path forward to reduce the default passive entropy sent for all requests. By moving this information to an active surface, UAs could monitor and apply privacy interventions based on principled UA policies such as Safari’s [Intelligent Tracking Prevention](https://webkit.org/tracking-prevention/), Firefox’s [Enhanced Tracking Protection](https://blog.mozilla.org/en/products/firefox/latest-firefox-rolls-out-enhanced-tracking-protection-2-0-blocking-redirect-trackers-by-default/), Chrome’s [Privacy Budget](https://github.com/bslassey/privacy-budget) proposal, or similar.

4. How does this specification deal with sensitive information?

If a user agent considers certain UA hints to be sensitive (for example, bitness, device model, or CPU architecture), this specification encourages them to return the empty string or a fictitious value (see the end of https://wicg.github.io/ua-client-hints/#http-ua-hints). UAs are free to identify which hints might be sensitive, and in which contexts, according to their privacy policies (ITP, ETP, Privacy Budget, etc.).

5. Does this specification introduce new state for an origin that persists across browsing sessions?

This specification does in the sense that it provides additional hints that may be in the Accept-CH cache (provided by Client Hints Infrastructure): https://wicg.github.io/client-hints-infrastructure/#accept-ch-cache-definition

6. What information from the underlying platform, e.g. configuration data, is exposed by this specification to an origin?

In addition to information about a user agent, this specification may expose the brand name, version, architecture, bitness, and “mobileness” of the underlying operating system, as well as device model (for non-desktop platforms). The same info can be obtained directly (or inferred indirectly) from the UA string today, e.g., the string “(Windows NT 10.0; Win64; x64)” denotes a 64-bit browser running on Windows 10 (64-bit). With UA-CH, only the platform name and “mobileness” is exposed by default. To get the other information, an origin needs to explicitly request it.

7. Does this specification allow an origin access to sensors on a user’s device

No.

8. What data does this specification expose to an origin? Please also document what data is identical to data exposed by other features, in the same or different contexts.

In addition to the data described in question 6, this specification may expose UA brand name, UA major version, and UA full version. All of this information is already exposed in the User-Agent header.

The difference between UA-CH and the UA string, is that an origin needs to request this data -- with the exception of platform name, UA brand name, UA major version, and platform mobileness. It’s not sent by default for all requests (and subresource requests). The UA is free to grant, deny, or even to return the empty string or fake data for such requests, according to their privacy policies (ITP, ETP, Privacy Budget, etc.).

9. Does this specification enable new script execution/loading mechanisms?

No.

10. Does this specification allow an origin to access other devices?

No.

11. Does this specification allow an origin some measure of control over a user agent’s native UI?

No.

12. What temporary identifiers might this specification create or expose to the web?

Each new client hint in this specification can be abused to store 1 extra bit in the Accept-CH cache, as described at https://wicg.github.io/client-hints-infrastructure/#accept-ch-cache-definition. No other new identifying information is exposed.

13. How does this specification distinguish between behavior in first-party and third-party contexts?

This specification inherits the following characteristics from Client Hints Infrastructure, “Origin opt-in applies to same-origin assets only and delivery to third-party origins is subject to explicit first party delegation via Permissions Policy, enabling tight control over which third party origins can access requested hint data.” Hints need to be explicitly delegated by developers from a first- to third-party context.

14. How does this specification work in the context of a user agent’s Private Browsing or "incognito" mode?

It will work the same. Accept-CH preferences will not be maintained between “private” and “non-private” browsing sessions. See https://wicg.github.io/client-hints-infrastructure/#accept-ch-cache-definition

15. Does this specification have a "Security Considerations" and "Privacy Considerations" section?

Yes. https://wicg.github.io/ua-client-hints/#security-privacy

16. Does this specification allow downgrading default security characteristics?

No.
