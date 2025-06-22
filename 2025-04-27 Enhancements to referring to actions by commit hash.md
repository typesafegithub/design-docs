# Enhancements to referring to actions by commit hash

<!-- TOC -->
* [Enhancements to referring to actions by commit hash](#enhancements-to-referring-to-actions-by-commit-hash)
  * [Related items](#related-items)
  * [Problem](#problem)
  * [Out of scope](#out-of-scope)
    * [Pinning to commits in version catalog, to have stable typings](#pinning-to-commits-in-version-catalog-to-have-stable-typings)
    * [Pinning to a specific binding (e.g. by JAR's checksum)](#pinning-to-a-specific-binding-eg-by-jars-checksum)
  * [Proposed solution](#proposed-solution)
<!-- TOC -->

## Related items

* a feature request: [[typesafegithub/github-workflows-kt] [Core feature request] Pinning action versions to commit hashes updateable by bots](https://github.com/typesafegithub/github-workflows-kt/issues/1691)

## Problem

When consuming an action through the bindings server, using the most popular pattern, like so:

```kotlin
@file:Repository("https://bindings.krzeminski.it")
@file:DependsOn("actions:checkout:v4")
```

the script may get a different artifact each time the script is compiled, and a different action may be run when the
workflow is run.

It's because unlike with Maven Central, the Maven-compatible custom bindings server doesn't have an inherent property of
artifact immutability (on why it's not possible, see
[Pinning to a specific binding (e.g. by JAR's checksum)](#pinning-to-a-specific-binding-eg-by-jars-checksum)). The `v4`
version is usually modeled as a moving tag/branch, pointing to the newest released version from major version 4.

Even tags with full version:

```kotlin
@file:DependsOn("actions:checkout:v4.1.2")
```

aren't guaranteed to be immutable. They should be by convention, but there's no mechanism to enforce it.

It's sometimes desired to pin to a specific action logic to ensure that no changes are made without the user knowing
about it, and to cope with supply chain attacks like
[this one](https://unit42.paloaltonetworks.com/github-actions-supply-chain-attack/).

That's why users wanting to be 100% sure the same action logic is run every time, use commit hashes like so:

```yaml
- uses: actions/checkout@85e6279cec87321a52edac9c87bce653a07cf6c2
```

which is also possible through github-workflows-kt:

```kotlin
@file:DependsOn("actions:checkout:85e6279cec87321a52edac9c87bce653a07cf6c2")
```

However, there are the following problems:
1. If an action's typing is stored in the typing catalog, it's not used during binding generation, which in practice
   means that all inputs are of type `String`.
2. It's not possible to make it dependency updating bots-friendly. With the YAML approach, one can write:
   ```yaml
   - uses: actions/checkout@85e6279cec87321a52edac9c87bce653a07cf6c2 # v4.1.2
   ```
   and the bots will generally suggest updates to the hash and the version in the comment. With github-workflows-kt,
   not only the bots won't learn about available versions accessible through the commit hashes, but also the comment
   with the version won't end up in the YAML.

## Out of scope

### Pinning to commits in version catalog, to have stable typings

For better reproducibility, users might want to specify a particular commit in
[github-actions-typing-catalog](https://github.com/typesafegithub/github-actions-typing-catalog). Right now, typings
stored in the catalog are versioned by the major versions, so e.g. if a typing change is required because of a change in
the action between v1.0 and v1.1, it may be a breaking change for the Kotlin binding consumer.

To solve it, we could provide a way to freeze not only the action's commit hash, but also the catalog typing's commit
hash. However, this problem is neglected for the purpose of this document because demand for this feature is unknown
(there is no signal about it from the users), and it would lead to further complicating the system.

### Pinning to a specific binding (e.g. by JAR's checksum)

One could imagine an attack on the bindings server where a malicious piece of code is injected to the Kotlin-based
logic. To address it, we could have a way of specifying e.g. a checksum of the binding's JAR, to be sure that the
delivered binding is always the same.

However, there's no native support for dependency checksums in Kotlin Scripting, and implementing it in an acceptable
way from user's point of view is not possible. It would be also challenging to ensure that the JAR provided by the
server doesn't change - in practice, it's not possible as even a subtle change e.g. in tooling used to build the JAR may
lead to changing the checksum.

Recommended workarounds are
* private hosting of the bindings server, either by building it from source (then one can pin the
  [repo](https://github.com/typesafegithub/github-workflows-kt)'s commit) or pinning to an image's digest in
  [DockerHub](https://hub.docker.com/r/krzema12/github-workflows-kt-jit-binding-server/tags)
* fetching the binding to a static location, and consuming it from the script. However, this
  way makes using dependency updating bots extremely hard

## Proposed solution

Add two new ways of consuming an action binding:

### 1. With validation of (version, commit hash) pair

Example:

```kotlin
@file:Repository("https://bindings.krzeminski.it")
@file:DependsOn("actions:checkout___commit:v4.1.2__85e6279cec87321a52edac9c87bce653a07cf6c2")
```

which would end up in the YAML as:

```yaml
- uses: actions/checkout@85e6279cec87321a52edac9c87bce653a07cf6c2 # v4.1.2
```

Additionally, the bindings server would validate if the version ref (here: `v4.1.2`) points to the mentioned commit
hash. Thanks to the extra validation, the user has an extra assurance than with the YAML approach - the version is for sure
correct.

Since the version is known and validated, the server can also look up the typing in the catalog.

### 2. Without validation of (version, commit hash) pair

Example:

```kotlin
@file:Repository("https://bindings.krzeminski.it")
@file:DependsOn("actions:checkout___commit_lenient:v4.1.2__85e6279cec87321a52edac9c87bce653a07cf6c2")
```

which also would end up in the YAML as:

```yaml
- uses: actions/checkout@85e6279cec87321a52edac9c87bce653a07cf6c2 # v4.1.2
```

It's similar to the above approach, with these differences:
* the version is just used to add a comment in the YAML - no validation is performed
* the version is used to look up the typings in the catalog

This mode is created to closer resemble how the YAML approach works. Because of no extra validation, it allows handling
certain edge cases, when the commit hash is intentionally out of sync with the version. For example: the version tag was
deleted because of a security vulnerability, and the user prefers to keep the action usage pinned to the commit hash to
keep the workflow working, as opposed to failing in the consistency check job. Or the user wants to use a newer hash with some fix that did not make it into a release yet.

This mode is made more verbose in the code on purpose, to promote the first, safer approach.
