# Enhancements to referring to actions by commit hash

## Related items

* a feature request: [[typesafegithub/github-workflows-kt] [Core feature request] Pinning action versions to commit hashes updateable by bots](https://github.com/typesafegithub/github-workflows-kt/issues/1691)

## Problem

## Out of scope

### Pinning to commits in version catalog, to have stable typings

For better reproducibility, users might want to specify a particular commit in
[github-actions-typing-catalog](https://github.com/typesafegithub/github-actions-typing-catalog). Right now, typings
stored in the catalog are versioned by the major versions, so e.g. if a typing change is required because of a change in
the action between v1.0 and v1.1, it may be a breaking change for the Kotlin binding consumer.

To solve it, we could provide a way to freeze not only the action's commit has, but also the catalog typing's commit
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

A recommended workaround is fetching the binding to a static location, and consuming it from the script. However, this
way makes using dependency updating bots extremely hard.

## Proposed solution
