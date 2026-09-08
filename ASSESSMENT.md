# Assessment notes

Where this package's boundary falls, what agility it has, and what constrains
its lifecycle.

The README already carries an unusually direct **What this package does not do**
section. This document does not repeat it. Read that first; this adds the parts
an assessment needs and the README does not cover.

Algorithm conformance belongs to
[`kxco-post-quantum`](https://www.npmjs.com/package/kxco-post-quantum) and is
published in that package's evidence bundle. It is referenced here, never
restated.

## Boundary

**This is the one package in the family whose boundary is mostly other
people's.** It is a thin custody layer over a token it does not implement, and
almost every assurance a buyer wants from it is really an assurance about the
HSM. Saying so is the whole point of assessing it.

**Three backends, and only one of them is custody.** `src/backends/` holds
`memory.js`, `file.js` and `pkcs11.js`. Two of the three keep key material in
the host process, which is the situation custody exists to prevent. They are
legitimate for development and for callers who have decided the risk is
acceptable, and they are not custody. An assessment that names this package
without naming the backend has not named the configuration.

**Operate: the vendor's channel is outside the assessment.** With
`Pkcs11Backend`, this package calls into a PKCS#11 module supplied by the token
vendor. What happens between that library and the hardware, including the
transport to a network HSM, is the vendor's design and their certificate's
scope. This package can neither see it nor speak for it.

**Operate: no network of its own.** Nothing in `src/` opens a socket. The only
external interface is the PKCS#11 module on the local machine.

**The dependency an assessor will trip over.** `pkcs11js` is an
`optionalDependency`. It is a native binding, so it needs a toolchain, and on a
machine where it does not build the package still installs. Two consequences
worth stating:

- On-token custody is unavailable and the failure is at use, not at install.
- `npm sbom` fails outright with `ESBOMPROBLEMS`, because a package present in
  the lock file is absent from the tree. The evidence bundle in `dist/evidence`
  records that failed step rather than omitting it; the bundle built on the
  reference machine has exactly this failure in `00-MANIFEST.json`. A buyer
  asking for an SBOM on a machine without the native binding gets an error, and
  should know that before asking.

**Protect records, enforce policy, retain history.** No records and no policy
engine here. What this package does hold is the key that other packages' records
are signed with, so the retention question it owns is key survival rather than
record survival: an on-token key survives a restart, a wrapped key does not, and
the README says so.

**Start and update.** No release signing of its own. Published through CI with
npm provenance.

## Agility

**Inherited, and then constrained by hardware.** The parameter sets, the two
backends and the wire formats belong to `kxco-post-quantum`. See that package's
`AGILITY.md`.

What this package adds is the constraint that makes agility real rather than
theoretical: **a token performs the mechanisms its firmware implements, and
nothing else.** Where every other package in the family can move parameter set
with a release, this one cannot move past what the hardware supports. That is
the single genuine hardware ceiling in the stack, and it is the reason the
lifecycle question below matters more here than anywhere else.

`signingMode` reports whether a probe signature succeeded on the backend. It is
per token and not per key, and the README is explicit that a wrapped key and an
on-token key can coexist. So it is a useful signal and not an inventory.

**ML-KEM is wrapped only.** `decapsulate` always unwraps into host memory.
On-token generation covers ML-DSA. A migration plan that assumed symmetric
treatment of the two would be wrong.

## Lifecycle

**Supported versions.** One line moving forward, matching the family. Fixes
land in the next release rather than being backported.

**Pin inconsistency.** `@noble/ciphers` and `@noble/hashes` are declared at
exactly `2.4.0`. `kxco-post-quantum` is declared `^1.3.0`. The tree the
evidence bundle was last built from resolved that range to **1.4.0**, against a
current primitives release of 1.7.2. `02-primitives.json` records the resolved
version, and that is the version any claim about the bundle applies to.

The primitives package pins its own dependencies exactly and states why: a
range lets the code that runs the cryptography change without a release. We do
not apply that rule here. Changing it costs a release of this package per
primitives release, and the decision has not been made.

**Ceiling: this is where hardware replacement is a real answer.** Everywhere
else in the family the ceiling is a runtime and the fix is a Node upgrade. Here
the ceiling is the token's firmware. If your HSM does not implement ML-DSA, no
version of this package makes it do so, and the remedy is a firmware update
from the vendor or a different token. Ask the vendor for their post-quantum
mechanism roadmap before committing to a migration date. That question is not
answerable from this repository.

**Roadmap.** No external audit of this package, no bug bounty. The primitives
package's roadmap in its `AUDIT.md` names a FIPS 140-3 CMVP application for a
module deployment using that library with an HSM; this package is the seam that
application would run through, and nothing has been submitted.

## Correcting this document

Every claim here is checkable against `src/` and the README. If one does not
match, that is a defect worth reporting through the repository's issues.
