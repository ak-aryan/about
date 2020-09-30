---
title: Code search turned static code checker
author: Rijnard van Tonder
authorUrl: https://twitter.com/rvtond
publishDate: 2020-09-30T14:00-07:00
tags: [blog]
slug: code-search-as-lightweight-on-demand-analysis
heroImage: /blog/XXX.png
published: true
---

<style>
  .gatsby-highlight {
    max-width: 100%;
    width: 40rem;
    margin-left: auto;
    margin-right: auto;
  }
</style>

I find static code checkers most valuable when they help teach me better ways to
code in a language or framework. For example, the Go
[staticcheck](https://staticcheck.io/docs/checks#SA6005) tool detects expensive
string comparisons like:

```go
if strings.ToLower(string1) == strings.ToLower(string2) {
  ...
}
```

and suggests the optimization:

```go
if strings.EqualFold(string1, string2) {
  ...
}
```

I find these short-and-sweet replacements are a great way to learn framework
idioms or library functions, like `strings.EqualFold` in Go.<sup>1</sup> As a
codebase grows these small inefficiences, inconsistencies, and missed
opportunities compound. Code patterns creep in that affect readability and
performance—[and it matters](https://www.digitalocean.com/blog/how-to-efficiently-compare-strings-in-go/?).

## Productivity workflows: Code checks vs. code search

Code checkers like staticcheck typically integrate with continuous integration
(CI) pipelines, or maybe a pre-commit hook in your development environment.
Others, like lint checks, typically integrate with editors. These workflows need
some upfront configuration, the code checks are fixed in place, and productivity
ensues (🤞).

Code search can also detect patterns for updating expressions or functions like
`EqualFold`. In practice though, search engines generally treat code as
plaintext, and can't offer the fidelity of dedicated code checkers. Code
checkers do more work to gather static information of programs, e.g., parsing
syntax into trees and using build outputs like types and dependency graphs. It
takes time to gather this info, it takes knowledge to write checks that use this
info, and it takes time to hook it all up. These constraints take away the
flexibility and speed of search workflows when you're simply looking for changes
to your favorite function called
`BananaNutChocolateCake`[↗](https://sourcegraph.com/search?q=BananaNutChocolateCake&patternType=literal).

And yet, I can't shake that there's a middle ground between crude plaintext
search and dedicated code checkers. What if the `EqualFold` check I read about
could be reduced to a simpler search query? Could we find, and maybe eradicate,
all the code that should be calling `EqualFold` instead? What would that look
like? And so sprung the idea to explore code checking with the ease of a
flexible, push-button search workflow.


## Pushing code search toward static code checking

Earlier this year Sourcegraph introduced [structural search](blog/going-beyond-regular-expressions-with-structural-code-search/)
to search over syntactic structures in code. Structural search implements
a basic building block in traditional code checkers: it treats programs as
concrete syntax trees. Sourcegraph search queries now also supports patterns with
`or` clauses to search for multiple patterns. Using these and search filtering,
it's possible to write configurable code checks as self-contained search queries.
Let's explore this idea!

Here's a search query inspired by
[staticcheck](https://staticcheck.io/docs/checks#S1003) where `strings.Index`
calls can be replaced with `strings.Contains`:

```python
language:go
-file:test
-file:vendor

strings.Index(..., ...) > -1

or

strings.Index(..., ...) >= 0

or

strings.Index(..., ...) != -1
```

This query will match against all `.go` files, excluding file paths that contain
`test` or `vendor`. It is sensible to exclude `test` and `vendor` paths if we
want to actually propose changes to a project (more on that later). The code
patterns `strings.Index(..., ....)` match the syntax of `strings.Index` calls,
and the `...` ellipses are special placeholders that match at least two
arguments.<sup>2</sup> The `or` keywords separate the patterns into separate
expressions.


---

<sup>1</sup> Another one of my favorites tools using the same principle is [Kibit](https://github.com/jonase/kibit) for Clojure.
<sup>2</sup> See the [structural search reference](https://docs.sourcegraph.com/@main/user/search/structural#syntax-reference) for more details about special syntax.

---

## Feedback

- Have a usage question or suggestion about structural search? [Send us a tweet](https://twitter.com/srcgraph) or e-mail us at <feedback@sourcegraph.com>
- Run into a bug? [Create an issue on GitHub](https://github.com/sourcegraph/sourcegraph/issues/new?assignees=&labels=&template=bug_report.md&title=)
