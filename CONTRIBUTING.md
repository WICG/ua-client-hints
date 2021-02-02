# Contributing Guide

You are welcome to contribute to this specification. In this guide, we lay out
the processes we try to stick to as we work on it.

## Filing an Issue

If you're reading the specification and something is confusing, broken, has a typo,
or you feel like important use cases or information are missing, please let us
know by filing an issue.

A few useful tips:

* It's good idea to check out the
[list of other open issues](https://github.com/WICG/ua-client-hints/issues) in
case one already exists. Comments to existing issues are a great way to
contribute.
* Choose a descriptive title when creating a new issue

## Pull Requests

Pull requests to this specification are encouraged and welcome.

Before doing the work to update the spec, please start with a new issue so we
can ensure that your changes align with the goals of this specification (or so
you can explain why the goals of this specification should change, if that's the
case).

Please note the [section on CLA requirements](#web-platform-incubator-community-group-and-w3c-cla)
before any PR can be merged.

## Building the Spec

This spec lives inside [`index.bs`](https://github.com/WICG/ua-client-hints/blob/master/index.bs)
and uses [Bikeshed](https://tabatkins.github.io/bikeshed/) as a tool for spec
generation.

Once installed, you can build the spec locally by
invoking the [`bikeshed spec`](https://tabatkins.github.io/bikeshed/#cli-spec)
command. After that completes, you can open `index.html` in a browser to see
your changes.

[`bikeshed watch`](https://tabatkins.github.io/bikeshed/#cli-watch) is also
handy when making more than a single edit.

## Web Platform Incubator Community Group and W3C CLA

This repository is being used for work in the W3C Web Platform Incubator
Community Group, governed by the
[W3C Community License Agreement (CLA)](http://www.w3.org/community/about/agreements/cla/).
To make substantive contributions, you must join the CG.

If you are not the sole contributor to a contribution (pull request), please
identify all contributors in the pull request comment.

To add a contributor (other than yourself, that's automatic), mark them one per
line as follows:

```
+@github_username
```

If you added a contributor by mistake, you can remove them in a comment with:

```
-@github_username
```

If you are making a pull request on behalf of someone else but you had no part
in designing the feature, you can remove yourself with the above syntax.
