# Snakepit — Full Runbook / Design Specification

**Project:** Aspartame Linux
**Subsystem:** Snakepit
**Status:** Architecture / implementation specification
**Core principle:** **Put the Python in the box.**

---

## 1. What Snakepit Is

Snakepit is Aspartame's **Python runtime library, compatibility resolver, and execution layer**.

It exists because the normal Python model is backwards for what Aspartame wants to accomplish.

Normally:

```text
Application
    │
    ├── requires Python 3.11
    ├── requires package A
    ├── requires package B
    │
    ▼
Environment manager
    │
    ▼
Construct an environment specifically for this application
```

Snakepit reverses this:

```text
                     SNAKEPIT
                         │
       ┌─────────────────┼─────────────────┐
       ▼                 ▼                 ▼
   Python Pot        Python Pot        Python Pot
      3.10              3.12              3.14
       │                 │                 │
       │ capabilities    │ capabilities    │ capabilities
       ▼                 ▼                 ▼
      software each pot can execute
                         │
                         ▼
                    Application
```

Snakepit deliberately maintains **many Python runtimes ahead of demand**.

When software arrives, Snakepit does not begin with:

> What Python environment should I create for this?

It begins with:

> **Which of the Python environments I already possess can run this?**

That inversion is the defining idea.

---

# 2. Mission

Snakepit should make this statement true:

> **An Aspartame system can encounter Python software from many eras and execute it without turning Python installation and environment management into the user's problem.**

The user should not normally need to know about:

```text
/usr/bin/python
pip
pipx
venv
virtualenv
pyenv
Conda environments
site-packages
PYTHONPATH
ABI compatibility
Python minor versions
dependency conflicts
activation scripts
```

Those things may exist underneath Snakepit.

They are implementation details.

The low floor remains:

```text
Run
```

The ceiling remains:

```text
Inspect runtime
Change runtime
Open shell
Inspect packages
Test compatibility
Create pot
Repair pot
View logs
Use underlying tooling directly
```

Snakepit follows Aspartame's **low floor, no ceiling** principle.

---

# 3. The Pot

The fundamental Snakepit object is a **Pot**.

A Pot is an isolated Python execution environment with a known identity and known capabilities.

A Pot is **not synonymous with Docker container**.

A Pot is an Aspartame abstraction.

Possible underlying technologies include:

```text
Conda
filesystem prefixes
bubblewrap
Linux namespaces
overlayfs
systemd
cgroups
bind mounts
read-only base environments
copy-on-write layers
```

Snakepit may eventually use several of these together.

The UI, Activity, and developer should not depend upon which implementation is underneath.

---

# 4. Pot Identity

Every Pot receives a stable internal identifier.

Example:

```text
org.aspartame.snakepit.python312-general
```

Human-readable information:

```text
Name:
    Python 3.12 General

Runtime:
    CPython 3.12.11

Architecture:
    x86_64

Purpose:
    General Python software

Status:
    Ready
```

Internally:

```yaml
id: python312-general
runtime:
  implementation: cpython
  version: 3.12.11
  architecture: x86_64

profile:
  - cli
  - gui
  - sugar
  - network

status: ready
```

---

# 5. Pots Are More Than Python Versions

Do **not** reduce Snakepit to:

```text
python-3.10
python-3.11
python-3.12
python-3.13
```

Version is only one compatibility property.

A Pot describes its complete capabilities.

For example:

```yaml
id: python312-sugar

runtime:
  implementation: cpython
  version: 3.12.11

capabilities:
  - python
  - cli
  - gtk3
  - sugar3
  - dbus
  - gstreamer
  - network

packages:
  pillow: "11.3"
  requests: "2.32"
  numpy: "2.3"

system:
  ffmpeg: true
  cups: true
```

Another Pot could be:

```yaml
id: python27-sugar-legacy

runtime:
  implementation: cpython
  version: 2.7.18

capabilities:
  - python
  - cli
  - gtk2
  - sugar-legacy
  - dbus-legacy

profile:
  - legacy
```

Therefore:

```text
Python version
      +
Python packages
      +
native libraries
      +
desktop integration
      +
architecture
      +
capabilities
      +
compatibility policy
      =
POT
```

---

# 6. Runtime Library

Aspartame should intentionally maintain an unusually broad collection of Python runtimes.

Conceptually:

```text
Snakepit
│
├── CPython 2.7
├── CPython 3.6
├── CPython 3.7
├── CPython 3.8
├── CPython 3.9
├── CPython 3.10
├── CPython 3.11
├── CPython 3.12
├── CPython 3.13
└── CPython 3.14
```

This does **not** require shipping every Python micro-release ever made.

Snakepit should distinguish:

```text
CURRENT
COMPATIBILITY
LEGACY
ARCHIVED
OPTIONAL
```

Example:

```text
3.14    Current
3.13    Current
3.12    Current

3.11    Compatibility
3.10    Compatibility
3.9     Compatibility

3.8     Legacy
3.7     Legacy
3.6     Legacy

2.7     Historical compatibility
```

The exact versions will change over Aspartame's lifetime.

The architectural rule does not:

> **Snakepit maintains runtime diversity intentionally instead of considering old Python versions garbage to be removed immediately.**

---

# 7. Base Pots and Specialized Pots

Shipping one enormous environment for every Python version would eventually become ridiculous.

Snakepit therefore needs a distinction between **base Pots** and **profiles/specialized Pots**.

Example:

```text
Python 3.12 Base
       │
       ├── General
       │
       ├── Sugar
       │
       ├── Scientific
       │
       └── Media
```

Conceptually:

```text
python312-base
    Python
    stdlib
    pip tooling

python312-general
    common libraries

python312-sugar
    GTK
    sugar3
    GI
    D-Bus integration

python312-science
    NumPy
    SciPy
    pandas
    Pillow

python312-media
    Pillow
    GStreamer bindings
    media-related libraries
```

But avoid prematurely creating dozens of profiles.

The first implementation should use a very small number.

---

# 8. Reverse Package Management

This is the architectural center of Snakepit.

Traditional dependency resolution:

```text
SOFTWARE
   │
   ▼
requirements
   │
   ▼
construct environment
```

Snakepit:

```text
POTS
 │
 ├── Pot A capabilities
 ├── Pot B capabilities
 ├── Pot C capabilities
 └── Pot D capabilities
          │
          ▼
      resolver
          ▲
          │
    software needs
          │
          ▼
       SOFTWARE
```

Software says what it needs.

Snakepit's Pots say what they provide.

The resolver finds the intersection.

---

# 9. Software Requirements

Applications should **not** need to name a specific Pot.

Bad:

```yaml
snakepit_pot: python312-sugar-v4
```

That couples the application to the machine's internal runtime organization.

Prefer:

```yaml
runtime:
  python: ">=3.11"

requires:
  - sugar3
  - gtk3
  - pillow

capabilities:
  - gui
```

Snakepit can then answer:

```text
Count.activity

python310-general     ✗ Python too old
python311-sugar       ✓
python312-sugar       ✓
python313-sugar       ✓
python314-general     ✗ missing sugar3
```

The software describes **requirements**.

The Pot describes **capabilities**.

Snakepit performs the mapping.

---

# 10. Runtime Resolution

Resolution should occur in stages.

```text
Software
   │
   ▼
Read metadata
   │
   ▼
Determine requirements
   │
   ▼
Query Pot registry
   │
   ▼
Filter incompatible Pots
   │
   ▼
Rank compatible Pots
   │
   ▼
Apply policy
   │
   ▼
Select Pot
   │
   ▼
Launch
```

Hard requirements must be satisfied first.

Example:

```text
Python >=3.11,<3.14
sugar3
gtk3
Pillow
```

Then ranking determines the preferred candidate.

---

# 11. Resolver Ranking

When several Pots work, selection should be deterministic.

Suggested initial ranking:

```text
1. Explicit user override
2. Previously verified runtime for this software
3. Aspartame preferred runtime
4. Highest-supported current Python
5. Most specific capability match
6. Compatibility runtime
7. Legacy runtime
```

Example:

```text
Count

Compatible:
    Python 3.11 Sugar
    Python 3.12 Sugar
    Python 3.13 Sugar

Aspartame preferred:
    Python 3.13 Sugar

Result:
    Python 3.13 Sugar
```

---

# 12. Verified Compatibility

Snakepit should distinguish between:

```text
COMPATIBLE
```

and:

```text
VERIFIED
```

Those are not the same.

A resolver may determine:

```text
Count appears compatible with Python 3.14
```

But Snakepit may know:

```text
Count has successfully launched and passed its smoke test using Python 3.13.
```

Therefore compatibility records can include:

```yaml
software: org.aspartame.Count

verified:
  python313-sugar:
    status: passed
    timestamp: ...
    version: ...

compatible:
  - python312-sugar
  - python314-sugar
```

Ranking can prefer verified combinations.

---

# 13. Do Not Guess Recklessly

Snakepit must never silently claim compatibility simply because:

```text
Python >= 3.10
```

matches.

Native modules matter.

So do GTK, GI, ABIs and system libraries.

Compatibility should be derived from:

```text
declared requirements
runtime version
installed Python packages
package versions
native dependencies
architecture
capability metadata
known incompatibilities
verification history
```

Unknown should remain:

```text
UNKNOWN
```

not magically become:

```text
WORKS
```

---

# 14. Pot Registry

Snakepit needs one authoritative registry.

Conceptually:

```text
/var/lib/aspartame/snakepit/
├── registry/
├── pots/
├── compatibility/
├── cache/
├── locks/
└── logs/
```

Exact paths can change during implementation.

Do not bake them into Activity APIs.

The registry should answer questions such as:

```text
Which Pots exist?

Which are ready?

Which Python versions exist?

What does each Pot provide?

Which Pots can run Count?

What is currently using this Pot?

When was it updated?

Is it healthy?

Has this application run successfully here?
```

---

# 15. Pot Manifest

Every Pot should expose machine-readable metadata.

Example:

```yaml
schema: 1

id: python313-sugar
name: Python 3.13 Sugar

runtime:
  implementation: cpython
  version: 3.13.7
  architecture: x86_64

capabilities:
  - cli
  - gui
  - sugar3
  - gtk3
  - dbus
  - network

packages:
  pillow: "11.3.0"
  requests: "2.32.5"

system:
  cups: available
  pipewire: available

policy:
  tier: current
  preferred: true

health:
  state: ready
```

This manifest should preferably be **generated from the actual environment** rather than manually maintained duplicated truth.

---

# 16. Application Metadata

Eventually Aspartame Activities could contain something conceptually like:

```yaml
runtime:
  language: python
  version: ">=3.11"

requires:
  python:
    - pillow >= 11

  capabilities:
    - sugar3
    - gtk3
    - dbus
```

But do not force this metadata format into existence prematurely.

Existing Sugar Activities already have their own metadata conventions.

Snakepit needs to work with existing software first.

New metadata should extend rather than gratuitously replace Sugar's Activity model.

---

# 17. Legacy Activity Detection

Legacy Sugar software is one of Snakepit's best reasons to exist.

Imagine finding a fifteen-year-old Activity.

Instead of:

```text
This requires Python 2.

Fuck off.
```

Aspartame can attempt:

```text
Old.activity
      │
      ▼
inspect metadata/source
      │
      ▼
possible Python 2 software
      │
      ▼
Snakepit
      │
      ├── modern Pots ✗
      │
      └── legacy Sugar Pot ✓
                         │
                         ▼
                        Run
```

That could make Aspartame unusually good at preserving old Python/Sugar software.

---

# 18. Detection Versus Declaration

Snakepit should use two sources of compatibility information.

First:

```text
DECLARED
```

Software explicitly states what it requires.

Second:

```text
DETECTED
```

Snakepit can inspect obvious properties.

Examples:

```text
Python syntax
imports
Activity metadata
shebang
native extension metadata
known dependency files
```

But detection is supplementary.

Do **not** attempt to build a magical static analyzer that promises to determine whether arbitrary Python software works.

Python is too dynamic for that shit.

---

# 19. Missing Dependencies

Suppose:

```text
Thing.activity

requires:
    Python >=3.11
    Pillow
    foo
```

Snakepit finds:

```text
3.12-general
    Pillow ✓
    foo ✗

3.13-general
    Pillow ✓
    foo ✗

3.13-science
    Pillow ✓
    foo ✓
```

Result:

```text
Use 3.13-science
```

No environment construction was required.

---

# 20. What If No Pot Works?

This needs careful policy.

Do not immediately mutate a shared Pot.

Shared Pots must remain predictable.

Instead:

```text
No existing Pot satisfies requirements.
```

Snakepit can offer progressively more advanced options:

```text
Install a compatible Snakepit Pot
Create a derived Pot
Run with missing features
Inspect requirements
Cancel
```

For normal users, the first option should generally be automatic and understandable:

> **Additional Python support is needed for this Activity.**

Not:

> Resolve pip dependency graph into Conda prefix?

---

# 21. Derived Pots

Eventually there will be software with weird dependencies.

Snakepit can support derived Pots.

```text
python313-general
        │
        ▼
derived Pot
        │
        + weird-library
        + special dependency
        │
        ▼
Thing.activity
```

But the derived Pot belongs to **Snakepit's managed runtime library**, not conceptually to `.venv` inside the application directory.

That distinction matters.

If another application can use the exact same derived Pot:

```text
Thing A ───┐
           ├── python313-derived-81f3
Thing B ───┘
```

reuse it.

---

# 22. Immutable / Controlled Pots

Normal applications should not mutate shared Pots.

This is essential.

Never allow:

```text
Count
   │
   ▼
pip install random-shit
   │
   ▼
shared python313-sugar changes
   │
   ▼
Scale mysteriously breaks
```

Shared Pots should be treated as effectively immutable during application execution.

Changes go through Snakepit.

---

# 23. Python Packages

Conda remains a strong candidate for building/managing the Python runtime substrate.

The conceptual layering can remain:

```text
Aspartame
   │
   ▼
Snakepit
   │
   ├── runtime registry
   ├── compatibility resolver
   ├── execution
   ├── lifecycle
   └── policy
           │
           ▼
    environment backend
           │
       ┌───┴───┐
       ▼       ▼
     Conda    pip
```

Users interact with **Snakepit**.

Snakepit interacts with Conda/pip.

That keeps the original Aspartame principle intact:

> **Conda handles Python versions and native dependency complexity; pip is allowed inside controlled environments.**

---

# 24. Native System Dependencies

Snakepit must not pretend everything is Python.

For example:

```text
Clip
    Python UI
    Pillow
```

Fine.

But another Activity might need:

```text
Python
PyGObject
GStreamer
ffmpeg
CUPS
Poppler
Tesseract
```

Native infrastructure should remain native.

Snakepit records and evaluates availability.

It should **not duplicate CUPS inside every Pot** merely because Python code talks to CUPS.

```text
Activity
    │
    ▼
Python Pot
    │
    ▼
system service/API
    │
    ▼
CUPS
```

Remember:

> **We love printers here.**

---

# 25. Host Integration

A Pot must not become a prison that prevents Activities from being Sugar Activities.

The execution layer needs carefully controlled access to things such as:

```text
D-Bus
Sugar services
Journal/datastore
display server
PipeWire
CUPS
NetworkManager-facing APIs
user files when explicitly needed
GPU
input devices
```

That is one reason **Docker should not define the architecture**.

Snakepit is integrated into Aspartame itself.

---

# 26. Security Boundary

Runtime compatibility and security permissions are separate concepts.

A Pot may be technically capable of networking.

An Activity may not necessarily receive unrestricted network access.

Model these separately:

```text
Pot capability:
    network-capable

Activity permission:
    network allowed
```

Likewise:

```text
Pot:
    CUPS available

Activity:
    printer access allowed
```

Future Snakepit isolation can therefore contribute to Aspartame's Activity permission system without conflating permissions with dependencies.

---

# 27. Execution

Eventually the common operation becomes:

```text
snake run <thing>
```

Internally:

```text
resolve software
      │
      ▼
read requirements
      │
      ▼
query registry
      │
      ▼
rank Pots
      │
      ▼
select Pot
      │
      ▼
prepare isolation
      │
      ▼
bind required resources
      │
      ▼
launch process
      │
      ▼
monitor
      │
      ▼
record result
```

For Sugar Activities this should normally happen invisibly through Activity launch.

The ordinary user should not have to type `snake`.

---

# 28. CLI

The CLI should expose the system without becoming another sprawling Python package manager.

Possible initial commands:

```text
snake status
snake pots
snake inspect <pot>
snake resolve <software>
snake run <software>
snake shell <pot>
snake verify <software>
snake doctor
```

Examples:

```text
$ snake pots

Python 3.14 General       Ready       Preferred
Python 3.13 Sugar         Ready       Current
Python 3.12 Sugar         Ready       Current
Python 3.10 Compatibility Ready       Compatibility
Python 2.7 Sugar Legacy   Ready       Legacy
```

And:

```text
$ snake resolve Count.activity

Count

Requires
  Python >= 3.11
  Sugar 3
  GTK 3

Compatible runtimes
  Python 3.12 Sugar       Compatible
  Python 3.13 Sugar       Verified
  Python 3.14 Sugar       Unknown

Selected
  Python 3.13 Sugar
```

That's useful without requiring the user to understand Conda.

---

# 29. `snake run`

Developer convenience can still exist.

For example:

```bash
snake run script.py
```

Snakepit could inspect the script and use a suitable general-purpose Pot.

Or:

```bash
snake run -m pytest
```

But this is secondary to the OS architecture.

Snakepit must **not become primarily a replacement for `uv`/pyenv/venv**.

---

# 30. `snake shell`

The no-ceiling escape hatch:

```bash
snake shell python313-sugar
```

Result:

```text
Entering Python 3.13 Sugar Pot

Python: 3.13.7
Pot: python313-sugar
```

Then an advanced user can inspect it.

That is valuable for debugging.

---

# 31. `snake doctor`

This should become important.

```text
$ snake doctor

Snakepit
────────

Registry              OK
Runtime storage       OK
Resolver               OK

Python 3.14 General   OK
Python 3.13 Sugar     OK
Python 3.12 Sugar     OK
Python 3.10 Compat    OK
Python 2.7 Legacy     WARNING

1 warning

Python 2.7 Legacy uses unsupported upstream software.
It remains installed for compatibility.
```

No gigantic dump of Conda internals unless requested.

---

# 32. Snakepit UI

Snakepit does **not** necessarily need a giant standalone Activity initially.

Most users shouldn't have to manage it.

Integration points are more Sugar-like.

Activity palette:

```text
Count

Resume
View Source
Edit Source
Environment
Logs
```

Selecting **Environment**:

```text
Count

Python 3.13

This is the Python runtime currently
used to run Count.

✓ Tested with this Activity

Other compatible runtimes
    Python 3.12
    Python 3.14

[Technical Details]
```

That is enough for normal use.

---

# 33. Advanced Snakepit Activity

A dedicated Snakepit Activity could come later.

It should visualize **runtime capability**, not imitate Synaptic or Docker Desktop.

Something like:

```text
Snakepit

Python Runtimes

3.14    ● Current
        12 Activities

3.13    ● Current
        28 Activities

3.12    ● Compatibility
        19 Activities

3.10    ● Compatibility
         7 Activities

2.7     ● Legacy
         3 Activities
```

Selecting one:

```text
Python 3.13

Can run
    Count
    Scale
    Clip
    Pippy
    ...

Provides
    Sugar
    GTK
    Pillow
    NumPy
    ...

Status
    Healthy
```

Notice the inversion again:

**Pot → things it can run.**

That is the UI manifestation of reverse package management.

---

# 34. Universal Help Integration

Everything needs stable semantic help IDs.

Examples:

```text
org.aspartame.snakepit
org.aspartame.snakepit.pot
org.aspartame.snakepit.runtime
org.aspartame.snakepit.compatibility
org.aspartame.snakepit.verified
org.aspartame.snakepit.legacy
```

Plain-language explanation for a Pot:

> **This is one of the Python runtimes Aspartame keeps available. Activities that work with this runtime can share it instead of installing their own copy of Python.**

Compatibility:

> **This Activity appears able to run using this Python runtime.**

Verified:

> **Aspartame has successfully tested this Activity using this runtime.**

Legacy:

> **This older Python runtime is kept so older software can continue working.**

---

# 35. Journal

Do not dump Snakepit implementation garbage into the Journal.

The Journal contains the user's work.

Not:

```text
Python 3.12 environment
pip operation
runtime cache
dependency lockfile
```

Snakepit's internal state belongs elsewhere.

If an Activity creates something meaningful, **that object** goes into Journal.

---

# 36. Updates

Updates must preserve reproducibility.

Bad:

```text
pacman -Syu
    ↓
random pip update
    ↓
shared Pot mutates
    ↓
14 Activities break
```

Better:

```text
new Pot revision
      │
      ▼
build/validate
      │
      ▼
verify important Activities
      │
      ▼
activate
      │
      ▼
retain previous revision temporarily
```

Conceptually:

```text
python313-sugar:r17
         │
         ▼
python313-sugar:r18
```

If r18 causes failure:

```text
rollback → r17
```

This fits Aspartame's eventual staged/rollback update philosophy extremely well.

---

# 37. Pot Versioning

Runtime version and Pot revision are separate.

```text
Python:
    3.13.7

Pot:
    python313-sugar

Pot revision:
    18
```

Changing Pillow does not mean Python changed.

Therefore the actual identity is closer to:

```text
python313-sugar@18
```

The friendly UI should normally hide `@18`.

---

# 38. Compatibility Database

Snakepit needs to remember what it learns.

Conceptually:

```text
software
    │
    ├── requirements
    ├── compatible Pots
    ├── verified Pots
    ├── failed Pots
    ├── preferred Pot
    └── last successful launch
```

Example:

```yaml
software: org.aspartame.Count

preferred: python313-sugar

pots:
  python312-sugar:
    compatibility: compatible

  python313-sugar:
    compatibility: verified

  python314-sugar:
    compatibility: unknown

  python310-sugar:
    compatibility: incompatible
    reason: python-version
```

---

# 39. Failure Is Data

If an Activity crashes, don't immediately mark the Pot incompatible.

Applications crash for plenty of reasons.

But verification can explicitly test:

```text
imports
startup
Activity registration
basic smoke test
optional application-provided tests
```

If:

```text
ImportError: No module named foo
```

Snakepit can report a dependency compatibility problem.

If:

```text
ZeroDivisionError
```

that's probably the application's fucking problem.

Keep those categories separate.

---

# 40. Runtime Discovery

Snakepit should know its Pots without expensive rescanning on every Activity launch.

Use a registry/cache.

Pot installation/update:

```text
Create/update Pot
      │
      ▼
inspect runtime
      │
      ▼
generate capability manifest
      │
      ▼
validate
      │
      ▼
register
```

Application launch then becomes fast:

```text
read Activity metadata
        +
query registry
        +
resolve
        =
launch
```

---

# 41. Performance Goal

Snakepit must not make Python Activities feel like containerized cloud workloads.

Warm launch resolution should be effectively negligible relative to launching the Activity itself.

Avoid:

```text
Activity clicked
     ↓
Conda solving...
     ↓
pip checking...
     ↓
network request...
     ↓
12 seconds later
```

Normal path:

```text
Activity clicked
     ↓
registry lookup
     ↓
known Pot
     ↓
launch
```

Expensive resolution/build work belongs at installation/update time whenever possible.

---

# 42. Offline First

A healthy installed Aspartame system must be able to launch its installed Activities without internet access.

Therefore:

```text
launch ≠ dependency download
```

Never make ordinary Activity startup depend on PyPI or Conda servers.

Network is for acquiring/updating Pots.

Existing software continues to work offline.

---

# 43. Storage

Multiple runtimes will consume disk.

That's acceptable within reason, because **runtime diversity is the product**.

But avoid gratuitous duplication.

Possible future mechanisms:

```text
hardlinks
reflinks
filesystem deduplication
shared package caches
Conda package cache
read-only bases
copy-on-write derived Pots
Btrfs subvolumes
```

The optimization should remain below the abstraction.

Never distort the Pot model merely to save 200 MB.

---

# 44. Garbage Collection

Snakepit should know:

```text
Pot installed
Pot referenced
Pot preferred
Pot legacy
Pot derived
Pot unused
```

Then cleanup can safely say:

```text
Unused runtime

Python 3.11 Scientific
No installed Activities currently require this runtime.

[Remove]
```

Never blindly garbage-collect a legacy runtime needed by installed software.

---

# 45. Installation

When an Activity is installed:

```text
Activity bundle
      │
      ▼
inspect requirements
      │
      ▼
Snakepit resolve
      │
 ┌────┴────┐
 ▼         ▼
match     no match
 │          │
 ▼          ▼
ready    find available Pot
            │
       ┌────┴────┐
       ▼         ▼
    available   unavailable
       │           │
       ▼           ▼
    install      explain
       │
       ▼
     verify
```

This work belongs primarily at **install time**, not every launch.

---

# 46. Activity Launch

After installation:

```text
Click Count
    │
    ▼
Activity launcher
    │
    ▼
Snakepit
    │
    ▼
Count → python313-sugar
    │
    ▼
isolation prepared
    │
    ▼
exec
    │
    ▼
Activity appears
```

The user sees:

```text
Click → Count
```

That's the fucking point.

---

# 47. Non-Python Software

Snakepit should know its jurisdiction.

If an Activity is native C++:

```text
Activity
   │
   └── native executable
```

Snakepit doesn't need to shove it into Python because Aspartame has a snake joke.

Likewise:

```text
Firefox
Mesa
CUPS
PipeWire
NetworkManager
ffmpeg
kernel
```

remain native.

The principle remains:

> **Python is an application/orchestration layer where appropriate. Native components stay native.**

---

# 48. Activity Definition Is Language-Neutral

This also reinforces an important Aspartame rule:

> **Activities are Sugar Activities because of how they behave, not because of what language they're written in.**

Therefore:

```text
Count.activity
    Python → Snakepit

NativeThing.activity
    Rust → native runtime

WebThing.activity
    web runtime

Wrapped conventional app
    native
```

Sugar owns the interaction model.

Snakepit owns Python runtime complexity.

---

# 49. Initial Backend Recommendation

For the first real implementation, **do not start by building a custom container system**.

Start boring.

A practical first architecture is:

```text
Snakepit
   │
   ├── Python service/library
   ├── CLI
   ├── Pot registry
   ├── resolver
   └── launcher
          │
          ▼
       Conda
          │
          ▼
   isolated prefixes
```

Then add stronger execution isolation independently.

This gets the **resolver model working first**.

That's the novel part.

Linux namespace cleverness isn't.

---

# 50. Privilege Model

Snakepit's resolver should never run as root merely because some runtime-management operation needs privilege.

Separate:

```text
User-facing Snakepit
        │
        ▼
unprivileged resolver
        │
        ▼
narrow privileged service
        │
        ▼
system Pot installation/update
```

Use D-Bus/polkit for privileged operations.

Example privileged operations:

```text
Install system Pot
Remove system Pot
Update system Pot
Repair registry
```

Not:

```text
run arbitrary pip command as root
```

Never that.

---

# 51. Suggested Components

Eventually:

```text
snakepit/
├── resolver
├── registry
├── inspector
├── launcher
├── verifier
├── backend
│   └── conda
├── isolation
├── policy
└── cli
```

Conceptually:

```text
                  ┌──────────────┐
                  │    Sugar     │
                  └──────┬───────┘
                         │
                  Activity launch
                         │
                         ▼
                  ┌──────────────┐
                  │  Snakepit    │
                  │   Resolver   │
                  └──────┬───────┘
                         │
              ┌──────────┼──────────┐
              ▼          ▼          ▼
          Registry    Policy     Verifier
              │
              ▼
             Pot
              │
              ▼
           Launcher
              │
              ▼
          Isolation
              │
              ▼
          Python exec
```

---

# 52. Resolver API

Keep the core resolver independent of Sugar UI.

Conceptually:

```python
resolve(application) -> Resolution
```

Where:

```text
Resolution

software
requirements
candidates
selected
reason
confidence
verification
```

Example:

```json
{
  "software": "org.aspartame.Count",
  "selected": "python313-sugar",
  "status": "verified",
  "candidates": [
    "python312-sugar",
    "python313-sugar",
    "python314-sugar"
  ]
}
```

That allows CLI, Sugar, installer, Universal Help, and future Django interfaces to consume the same truth.

---

# 53. D-Bus API

Eventually expose narrow interfaces such as:

```text
org.aspartame.Snakepit
```

Potential operations:

```text
ListPots
GetPot
Resolve
GetCompatibility
Verify
Launch
```

Privileged management can be separate:

```text
org.aspartame.Snakepit.Manager
```

With:

```text
InstallPot
RemovePot
UpdatePot
RepairPot
```

Do not expose arbitrary shell execution through privileged D-Bus.

---

# 54. Django / Server Integration

If Aspartame Server eventually gains the Django control plane, Snakepit can expose:

```text
/api/v1/environments
```

or preferably language matching the actual abstraction:

```text
/api/v1/snakepit/pots
/api/v1/snakepit/software
/api/v1/snakepit/compatibility
```

Django should consume Snakepit's service/API.

It should not directly fuck around inside Conda directories.

---

# 55. MVP

The first Snakepit milestone should be deliberately small.

Build only enough to prove the inversion.

### MVP requirements

```text
1. Create three isolated Python Pots.

2. Register their Python versions.

3. Register several capabilities.

4. Describe one test application.

5. Ask Snakepit which Pots can run it.

6. Select preferred compatible Pot.

7. Execute application using that Pot.

8. Record successful execution.

9. Run it again without dependency solving.

10. Show resolution through CLI.
```

Example:

```text
python310
python312
python314
```

Test software:

```text
requires Python >=3.11,<3.14
```

Expected:

```text
python310 ✗
python312 ✓
python314 ✗

Selected: python312
```

Then actually run it.

That proves Snakepit.

---

# 56. MVP Phase 2 — Capabilities

Add:

```text
python312-general
python312-sugar
python314-general
```

Application:

```text
Python >=3.12
requires sugar3
```

Expected:

```text
python312-general ✗ missing sugar3
python312-sugar   ✓
python314-general ✗ missing sugar3
```

Now Snakepit is no longer merely pyenv.

---

# 57. MVP Phase 3 — Real Activity

Use **Count**.

```text
Count.activity
       │
       ▼
Snakepit inspect
       │
       ▼
requirements
       │
       ▼
python3xx-sugar
       │
       ▼
launch
```

Count is ideal because it is first-party, controlled, and already exercises Sugar/GTK/Journal-ish Activity behavior.

Do not start with some nightmare Python 2 Activity from 2009.

Prove the architecture with something understood first.

---

# 58. Phase 4 — Multiple Activities Sharing Pot

Add:

```text
Count
Clip
Scale
```

Then demonstrate:

```text
              python313-sugar
              /      |      \
             /       |       \
          Count     Clip     Scale
```

No:

```text
Count/.venv
Clip/.venv
Scale/.venv
```

This is the demonstration that makes the architecture immediately understandable.

---

# 59. Phase 5 — Legacy

Only after modern execution works:

```text
Legacy Sugar Activity
        │
        ▼
modern Pot ✗
        │
        ▼
legacy Pot ✓
        │
        ▼
launch
```

If Aspartame can resurrect old Sugar software cleanly, Snakepit has proven another major reason to exist.

---

# 60. Phase 6 — Derived Pots

Then handle the weird case:

```text
Application needs unusual package
           │
           ▼
no shared Pot works
           │
           ▼
derive from closest Pot
           │
           ▼
install dependency
           │
           ▼
register new Pot
           │
           ▼
verify
           │
           ▼
reuse later
```

Do this **after** shared Pot resolution works.

Otherwise the implementation will naturally collapse back into an ordinary environment manager.

---

# 61. Things Snakepit Must Not Become

This section should be treated as a design guardrail.

Snakepit is **not**:

```text
Docker Desktop
Docker wrapper
venv GUI
pyenv replacement
Conda GUI
pip frontend
Kubernetes for Python
per-project environment generator
Python IDE
package store
Activity sandbox UI
```

Some of those technologies may exist underneath it.

They do not define it.

---

# 62. Anti-Pattern: Project Owns Runtime

Avoid:

```text
project/
├── .venv/
├── Python
└── dependencies
```

as the conceptual model.

Prefer:

```text
Snakepit
├── Pot A
├── Pot B
├── Pot C
└── Pot D

Applications
├── Count ───────► Pot C
├── Clip ────────► Pot C
├── Scale ───────► Pot C
└── Legacy ──────► Pot A
```

---

# 63. Anti-Pattern: Runtime Created on Every Launch

Never:

```text
click Activity
      │
      ▼
resolve packages
      │
      ▼
download
      │
      ▼
build environment
      │
      ▼
run
```

Prefer:

```text
install/update
      │
      ▼
expensive work

then

click
  │
  ▼
lookup
  │
  ▼
run
```

---

# 64. Anti-Pattern: Shared Mutable `site-packages`

Do not simply point every runtime at one universal shared `site-packages`.

Native modules, ABI differences, Python versions and dependency conflicts make that unsafe.

Sharing happens at the **Pot level**:

```text
many applications
       │
       ▼
one known compatible Pot
```

not:

```text
every Python installation
       │
       ▼
one terrifying global site-packages directory
```

---

# 65. Anti-Pattern: Docker Requirement

Aspartame users should not have to install or manage Docker to use Snakepit.

If some future implementation uses container-like Linux primitives underneath:

fine.

But:

```text
docker build
docker run
docker ps
docker pull
```

must not become part of the normal Snakepit mental model.

---

# 66. Anti-Pattern: Everything Preinstalled Everywhere

Having many runtimes does **not** mean installing every Python package into every Pot.

That creates enormous, conflicting environments.

The ideal structure is closer to:

```text
many Python versions
+
small number of useful profiles
+
shared/reusable derived Pots when genuinely necessary
```

---

# 67. Anti-Pattern: Version Is Compatibility

Never assume:

```text
Python 3.12
```

means:

```text
can run every Python 3.12 program
```

A Pot is:

```text
runtime + capabilities + dependencies + ABI + policy + verification
```

---

# 68. Ideal User Experience

A normal Aspartame user:

```text
downloads Activity
      │
      ▼
opens Activity
      │
      ▼
Activity works
```

That's it.

They never encounter Snakepit unless something unusual happens.

---

# 69. Ideal Developer Experience

A developer can ask:

```text
$ snake resolve Count.activity
```

and receive:

```text
Count

Selected runtime
  Python 3.13 Sugar

Why?
  Python version compatible
  Sugar 3 available
  GTK 3 available
  Required packages available
  Previously verified

Alternatives
  Python 3.12 Sugar
  Python 3.14 Sugar
```

Then:

```text
$ snake shell python313-sugar
```

when they actually need the machinery.

---

# 70. Ideal Failure Experience

Not:

```text
ModuleNotFoundError: No module named 'gi'
```

dumped at a normal user.

Instead:

```text
Count couldn't start.

The Python runtime normally used by Count
is unavailable or damaged.

[Repair]    [Details]
```

Details can reveal:

```text
Missing capability:
    sugar3

Expected Pot:
    python313-sugar
```

And deeper still:

```text
Actual traceback
```

Again:

```text
low floor
   ↓
progressive disclosure
   ↓
no ceiling
```

---

# 71. Ideal Architecture

```text
                         SUGAR
                           │
                    Activity launch
                           │
                           ▼
                 ┌───────────────────┐
                 │     SNAKEPIT      │
                 │                   │
                 │ Resolver / Policy │
                 └─────────┬─────────┘
                           │
              ┌────────────┼────────────┐
              ▼            ▼            ▼
          Registry      Verifier    Compatibility
              │
              └────────────┬────────────┘
                           ▼
                    Selected Pot
                           │
                           ▼
                 ┌───────────────────┐
                 │ Execution Layer   │
                 │ isolation/binds   │
                 └─────────┬─────────┘
                           │
                           ▼
                    Python process
                           │
             ┌─────────────┼─────────────┐
             ▼             ▼             ▼
           Sugar         Journal       D-Bus
             │             │             │
             └─────────────┼─────────────┘
                           ▼
                       Aspartame
```

Below Snakepit:

```text
                    SNAKEPIT
                        │
                backend abstraction
                        │
          ┌─────────────┼─────────────┐
          ▼             ▼             ▼
       Conda           pip      Linux isolation
          │                           │
          ▼                           ▼
   Python prefixes            namespaces/bwrap/etc.
```

That boundary is important.

**Sugar should not know Conda.**

**Activities should not know Conda.**

**The installer should not have to understand Pot internals.**

They know Snakepit.

---

# 72. Ideal End State

Eventually an Aspartame machine could say:

```text
Snakepit

12 Python runtimes
8 active Pots
63 Python Activities/applications

57 verified
4 compatible
2 legacy

All runtimes healthy.
```

And behind that simple statement is:

```text
CPython versions
Conda
pip
native libraries
GI bindings
legacy compatibility
capability resolution
dependency metadata
isolation
permissions
runtime selection
verification
rollback
logging
```

The user doesn't have to deal with any of it.

That is exactly the sort of complexity Aspartame should absorb.

---

# 73. Design Test

Whenever adding a Snakepit feature, ask:

**Does this make the user manage Python, or does this make Snakepit manage Python for the user?**

If the proposed workflow becomes:

```text
choose Python
create environment
activate environment
install dependencies
configure interpreter
```

Snakepit has failed.

If it becomes:

```text
run Activity
```

while Snakepit figures out:

```text
what runtime
what capabilities
what dependencies
what isolation
what compatibility
```

then it is working.

---

# 74. Canonical Definition

Use this as the project-level definition:

> **Snakepit is Aspartame's Python runtime library and compatibility resolver. It maintains multiple isolated Python runtime Pots with known capabilities and determines which Activities, applications, and scripts each Pot can execute. Rather than constructing a private environment for every application, Snakepit routes software into an existing compatible runtime whenever possible, derives reusable Pots when necessary, and keeps Python versioning, dependencies, isolation, and compatibility beneath the operating-system abstraction.**

Short version:

> **Snakepit keeps the Python runtimes. Software gets routed to the right one.**

Developer version:

> **Reverse package management for Python runtimes.**

Aspartame version:

> **We don't ask which Python you need. We already have the fucking snakes.**

And the implementation mantra should remain:

> **Put the Python in the box — but don't make the user manage the box.**
