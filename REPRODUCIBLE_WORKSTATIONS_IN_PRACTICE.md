# Reproducible Workstations in Practice

## A Hands-On Guide from Ubuntu to Docker with One Repository

A workstation usually becomes useful through a hundred small decisions.
You install a package, fix a shell setting, add a language runtime, configure the GPU, and find a command that makes an annoying problem disappear.
Six months later, the machine works.
The instructions for rebuilding it do not exist.

Putting your dotfiles in Git solves part of that problem.
The rest lives in package managers, toolchain installers, desktop settings, container images, credentials, and the things you changed while trying to get back to work.

> A reproducible workstation is a maintained explanation of how your working environment is built.

This book develops that explanation into a system.
One repository declares an Ubuntu workstation and builds a Docker development image from the same package lists and installation modules.
chezmoi manages personal files.
mise manages developer tools.
Small shell modules manage the system layer.
Docker packages the shared environment.
Tests, drift reports, and reviewed updates keep those declarations useful.

The running example is an engineer who works across Python, Node, Go, and Rust, uses GDAL and PROJ for geospatial work, and runs local inference on an NVIDIA desktop.
A second machine has no GPU.
Both should share the tools that matter without pretending to be the same machine.

The examples use Ubuntu, Bash, TOML, Dockerfiles, and plain text.
The operating model transfers to other platforms.
The system commands do not.

**How to read the examples.**
The named scripts are reference implementations for the boundary being discussed.
The repository grows through the labs; a directory tree is not a claim that every file has been supplied.
Image digests, signing identities, bootstrap versions, and hardware choices must come from your own verified inputs.
The complete host, desktop, and GPU setup still needs testing on your supported Ubuntu release and hardware.

**Version scope.**
The source research was prepared in October 2026.
This book uses the supported Ubuntu LTS release you select as an explicit input.
Choosing a newer LTS means testing that release, its package names, third-party repositories, and GPU compatibility.
Release trivia does not belong in the recovery contract.

---

## Contents

1. [Define what you need to reproduce](#1-define-what-you-need-to-reproduce)
2. [Choose tools by the state they own](#2-choose-tools-by-the-state-they-own)
3. [Build the repository boundary](#3-build-the-repository-boundary)
4. [Write system modules that can converge](#4-write-system-modules-that-can-converge)
5. [Manage personal files with chezmoi](#5-manage-personal-files-with-chezmoi)
6. [Lock developer tools with mise](#6-lock-developer-tools-with-mise)
7. [Bootstrap without circular dependencies](#7-bootstrap-without-circular-dependencies)
8. [Handle credentials and machine identity](#8-handle-credentials-and-machine-identity)
9. [Build the Docker image from shared inputs](#9-build-the-docker-image-from-shared-inputs)
10. [Use the image in development containers](#10-use-the-image-in-development-containers)
11. [Keep desktop and GPU state on the host](#11-keep-desktop-and-gpu-state-on-the-host)
12. [Test the claims the repository makes](#12-test-the-claims-the-repository-makes)
13. [Publish artifacts you can verify](#13-publish-artifacts-you-can-verify)
14. [Detect and reconcile drift](#14-detect-and-reconcile-drift)
15. [Update deliberately and recover from failure](#15-update-deliberately-and-recover-from-failure)
16. [Add AI assistance at a narrow boundary](#16-add-ai-assistance-at-a-narrow-boundary)
17. [A four-week adoption plan](#17-a-four-week-adoption-plan)
18. [Appendix: working contracts and checklists](#18-appendix-working-contracts-and-checklists)
19. [Sources and verification notes](#19-sources-and-verification-notes)

---

## 1. Define what you need to reproduce

The first decision is the recovery target.
“Everything on my machine” includes temporary experiments, obsolete dependencies, browser sessions, downloaded models, and mistakes.
Rebuilding all of it preserves the mess.

Start with the jobs the machine must support.
For the running example, those jobs are editing code, running project tests, processing a small geospatial dataset, and running inference when a GPU is available.
Each job needs an observable acceptance check.

### 1.1 Write an environment contract

```yaml
environment: engineering-workstation
owner: workstation-maintainer

targets:
  - ubuntu-desktop-amd64
  - ubuntu-laptop-without-gpu
  - linux-development-container-amd64

required_jobs:
  - clone a project and run its tests
  - execute the selected Python and Node toolchains
  - inspect and transform a geospatial fixture
  - use personal shell and editor defaults

optional_jobs:
  - run local inference on approved NVIDIA hardware
  - build an arm64 development image

recovery:
  inputs: reviewed Git revision plus separately recovered credentials
  result: passing acceptance checks on a clean supported OS
  budget: measured locally, with download and reboot time reported

excluded_state:
  - project source and uncommitted work
  - browser sessions and personal documents
  - model weights and application databases
  - temporary debugging packages
```

The exclusions have owners too.
Project source needs its own backup.
Model weights need a storage policy.
Credentials need a recovery procedure.
An environment repository does not replace those systems.

### 1.2 Separate four meanings of reproducible

| Property | Claim | Evidence |
| --- | --- | --- |
| Recoverable | A clean machine can become productive | A recorded clean rebuild |
| Convergent | Reapplying declarations repairs managed state | A deliberate drift-and-repair test |
| Repeatable | The selected inputs produce the same relevant versions and behavior | Package inventories, locks, and acceptance tests |
| Bit-for-bit reproducible | Independent builds produce identical artifact bytes | Digest comparison under controlled build conditions |

These claims have different costs.
A successful bootstrap establishes only that one execution succeeded.
An idempotent package installer does not prove two independently built images contain identical bytes.
A Docker digest identifies an existing artifact; it does not prove you can recreate it.

For a personal workstation, begin with recovery, convergence, and repeatable shared tooling.
Pursue identical artifact bytes when a requirement makes them useful.
Keep the claim smaller than the evidence.

### 1.3 Give the host and image different update policies

A daily workstation needs a timely security-update policy.
A development image needs reviewed inputs so yesterday's build can be explained.
Those needs overlap, but they are not identical.

Ubuntu's snapshot service can select archive contents by timestamp.
It also freezes the selected security pocket, so Canonical explicitly asks users to check for newer security fixes.[^ubuntu-snapshots]

The recommendation in this book is therefore:

- The host follows the configured Ubuntu security-update policy.
- Image builds use a reviewed Ubuntu base digest and archive snapshot.
- Developer tools use committed lockfiles where the backend supports the required guarantees.
- Projects keep their own dependency locks.

The host and image share declared capabilities.
They may have different system-package versions.
Record that distinction rather than quietly promising exact parity.

### Lab: define five acceptance checks

Choose five checks that represent real work.
Include one that opens or processes a fixture, one that runs as an ordinary user, and one that uses a freshly cloned project.

**Done when:** another engineer can tell what a successful recovery means without looking at your current machine.

---

## 2. Choose tools by the state they own

An environment contains several kinds of state.
Files, operating-system packages, developer tools, and project dependencies have different installers and different failure modes.
Assign one owner to each.

### 2.1 The reference division of labor

| State | Owner in this book | Reviewable declaration |
| --- | --- | --- |
| Personal files and templates | chezmoi | `home/` |
| Language runtimes and selected CLIs | mise | `toolchains/mise.toml` and its lockfile |
| Ubuntu package requirements | Shell modules | `packages/apt/*.txt` |
| Third-party system repositories | Dedicated root module | Repository URL, signing policy, supported release |
| Desktop preferences | Host module | Selected `dconf` settings |
| Container packaging | Dockerfile | Shared modules plus image-specific assembly |
| Project dependencies | The project package manager | `uv.lock`, `package-lock.json`, or equivalent |

This is an architectural recommendation.
It keeps the seams inspectable and gives each failure a likely owner.
It is not a claim that these are the only tools that can solve the problem.

Avoid installing the same CLI through apt, mise, and a language package manager.
Otherwise, `command -v` may succeed while the shell runs an unintended version.
The package owner and the effective executable must agree.

### 2.2 When a different approach fits

| Approach | Choose it when | Boundary to understand |
| --- | --- | --- |
| GNU Stow or a plain file repository | Your problem is mostly file placement | Add package and recovery policies separately |
| chezmoi plus shell modules | You want personal templating and a small Ubuntu system layer | You own the convergence behavior of every script |
| Ansible | Remote operations, inventories, and existing roles matter | Define how playbooks and personal-file management divide ownership |
| Nix plus Home Manager | Declared user-space build inputs are the central requirement | Learn its package and configuration model before committing to migration |
| Dev Containers alone | The project container is the entire supported environment | Host hardware and personal recovery remain separate concerns |

Machine count is a signal, not a threshold.
Two remote machines with different policies may justify Ansible.
Several identical personal machines may still be easy to manage with the reference stack.
Choose from operating requirements.

Nix offers a different model built around derivations with explicit inputs.[^nix]
That can justify a user-space migration.
It does not make Ubuntu drivers, firmware, desktop sessions, or project data disappear from the ownership map.

### 2.3 Keep convenience separate from pinning

A package list can make recovery convenient without preserving exact versions.
A tool manager can preserve top-level versions without preserving every build dependency.

Homebrew is explicit that Bundle does not provide a Brewfile lockfile for arbitrary historical versions.[^homebrew]
That makes it a different kind of declaration from a locked project dependency graph.
Evaluate tools against the property you need.

The useful question is:

> Which bytes, versions, and behavior does this declaration actually constrain?

If the answer is unclear, inspect a generated lock entry and a clean installation.

### Lab: remove duplicate ownership

Inventory your important executables.
For each, record the effective path, installation mechanism, and intended owner.
Resolve duplicate ownership before adding automation.

**Done when:** the shell cannot accidentally choose between three independently updated copies of a critical tool.

---

## 3. Build the repository boundary

The repository should make intent obvious.
A reader needs to distinguish declarations from installation code, private machine observations, and derived artifacts.

### 3.1 The repository we will grow

```text
workstation/
├── README.md
├── bootstrap
├── install.sh
├── .chezmoiroot
├── .dockerignore
├── home/
│   ├── .chezmoi.toml.tmpl
│   ├── .chezmoiignore
│   ├── dot_bashrc.tmpl
│   ├── dot_gitconfig.tmpl
│   ├── dot_config/mise/
│   │   ├── config.toml.tmpl
│   │   └── mise.lock.tmpl
│   └── dot_config/systemd/user/
│       ├── workstation-drift.service
│       └── workstation-drift.timer
├── toolchains/
│   ├── mise.toml
│   └── mise.lock
├── packages/apt/
│   ├── base.txt
│   ├── dev.txt
│   ├── geo.txt
│   ├── desktop.txt
│   └── SNAPSHOT
├── modules/
│   ├── apt.sh
│   ├── repos.sh
│   ├── desktop.sh
│   └── nvidia-host.sh
├── desktop/
│   └── keybindings.ini
├── docker/
│   └── Dockerfile
├── .devcontainer/
│   ├── cpu/devcontainer.json
│   └── gpu/devcontainer.json
├── tests/
│   ├── acceptance.sh
│   ├── convergence.sh
│   └── fixtures/
├── scripts/
│   ├── drift.sh
│   └── verify-inputs.sh
└── .github/workflows/
    ├── ci.yml
    └── publish.yml
```

The `.chezmoiroot` file contains `home`.
chezmoi reads source state from that subdirectory, including the configuration template and ignore file.[^chezmoi-root]
The package lists and Dockerfile remain ordinary repository content.

The tree is the target layout.
Add files as their behavior becomes defined and tested.
Do not create empty machinery to make the repository look complete.

### 3.2 Use one canonical toolchain declaration

Keep `toolchains/mise.toml` and `toolchains/mise.lock` together.
The host receives rendered copies through chezmoi.
The image receives copies through Docker.
Neither copy becomes an independently maintained declaration.

For `home/dot_config/mise/config.toml.tmpl`:

```gotemplate
{{ include (joinPath .chezmoi.workingTree "toolchains/mise.toml") }}
```

For `home/dot_config/mise/mise.lock.tmpl`:

```gotemplate
{{ include (joinPath .chezmoi.workingTree "toolchains/mise.lock") }}
```

chezmoi exposes both `.chezmoi.sourceDir` and `.chezmoi.workingTree`.[^chezmoi-variables]
With a `home/` source root, use the working tree for siblings such as `toolchains/`.
Do not infer the repository root by blindly removing a directory from `chezmoi source-path`.

If your mise version creates dependency-lock sidecars, include those in this projection too.
The two templates above are sufficient only for a configuration whose required lock inputs fit in those two files.
The Docker copy must preserve the same relative sidecar layout.

### 3.3 Observations are not declarations

Store local drift reports outside the repository, under a private state directory.
An apt inventory tells you what exists.
It does not tell you why it belongs in the shared environment.

Keep curated exceptions in Git when useful.
Do not automatically publish full host inventories, GNOME dumps, SSH configuration, or paths containing customer names.
A public repository needs a deliberate publication boundary.

### 3.4 Profiles select capabilities

Begin with three profiles:

| Profile | System packages | Personal files | Desktop | GPU host setup |
| --- | --- | --- | --- | --- |
| `desktop` | `base`, `dev`, selected domain groups, `desktop` | Full approved set | Yes | Explicit opt-in |
| `server` | `base`, `dev`, selected domain groups | Headless set | No | Explicit opt-in |
| `container` | Already provided by the image | Container-safe subset | No | Never |

A profile describes intent.
Hardware detection provides evidence.
Do not make `nvidia-smi` the sole test for whether a clean machine needs an NVIDIA driver: the command may be absent precisely because setup has not happened.

### Lab: create the smallest repository

Start with `base.txt`, `dev.txt`, the toolchain pair, one shell file, and a README describing ownership.
Add the other directories when the corresponding lab needs them.

**Done when:** every committed file is either an intentional input, code that applies it, or a check that verifies it.

---

## 4. Write system modules that can converge

The shared system layer should work in two contexts: root during an image build and an explicitly elevated command on the host.
It should not require a desktop session, a terminal, or a running systemd instance.

Keep that contract small.
A package module installs named packages.
A repository module owns declared apt source files.
A desktop module requires a user session.
Putting them in one script makes their preconditions difficult to test.

### 4.1 Declare package names as data

`packages/apt/base.txt`:

```text
# Transport and repository operations
ca-certificates
curl
git
```

`packages/apt/dev.txt`:

```text
# Build and inspection tools owned by Ubuntu
build-essential
jq
```

Use one package name per line.
Decide whether inline comments, version expressions, and architecture qualifiers are supported.
The following implementation intentionally accepts only simple package names.
That restriction keeps validation and drift comparison understandable.

### 4.2 A presence module with explicit limitations

This script ensures packages are installed.
It does not remove undeclared packages, downgrade installed versions, or perform a general system upgrade.
The host security policy owns upgrades.
Clean image builds own selection from the chosen snapshot.

```bash
#!/usr/bin/env bash
# modules/apt.sh
set -Eeuo pipefail

fail() {
  printf 'apt module: %s\n' "$*" >&2
  exit 1
}

((EUID == 0)) || fail "run this module as root"
(($# > 0)) || fail "provide at least one package group"

module_dir=$(cd -- "$(dirname -- "${BASH_SOURCE[0]}")" && pwd)
repo_root=$(cd -- "$module_dir/.." && pwd)
declare -a packages=()

for group in "$@"; do
  [[ "$group" =~ ^[a-z][a-z0-9-]*$ ]] || fail "invalid group: $group"
  manifest="$repo_root/packages/apt/$group.txt"
  [[ -r "$manifest" ]] || fail "missing manifest: $group"

  while IFS= read -r line || [[ -n "$line" ]]; do
    [[ "$line" =~ ^[[:space:]]*(#|$) ]] && continue
    [[ "$line" =~ ^[a-z0-9][a-z0-9+.-]+$ ]] ||
      fail "invalid package entry in $group: $line"
    packages+=("$line")
  done < "$manifest"
done

((${#packages[@]} > 0)) || fail "selected groups contain no packages"
declare -a unique=()
mapfile -t unique < <(printf '%s\n' "${packages[@]}" | LC_ALL=C sort -u)
declare -a missing=()

for package in "${unique[@]}"; do
  if status=$(dpkg-query -W -f='${Status}' "$package" 2>/dev/null); then
    [[ "$status" == "install ok installed" ]] || missing+=("$package")
  else
    missing+=("$package")
  fi
done

if ((${#missing[@]} == 0)); then
  printf 'apt: declared packages are present\n'
  exit 0
fi

declare -a apt_options=(-o APT::Update::Error-Mode=any)
if [[ -n "${WORKSTATION_APT_SNAPSHOT:-}" ]]; then
  [[ "$WORKSTATION_APT_SNAPSHOT" =~ ^[0-9]{8}T[0-9]{6}Z$ ]] ||
    fail "invalid snapshot timestamp"
  apt_options+=(-o "APT::Snapshot=$WORKSTATION_APT_SNAPSHOT")
fi

export DEBIAN_FRONTEND=noninteractive
apt-get "${apt_options[@]}" update
apt-get "${apt_options[@]}" install -y --no-install-recommends "${missing[@]}"
```

The snapshot setting selects apt behavior rather than assuming the interactive `apt` CLI and `apt-get` have identical option surfaces.[^ubuntu-snapshots]
Test the module against the apt version in your selected base.
Third-party archives need their own policy; the Ubuntu snapshot service does not freeze every repository you add.

The missing-package check gives the host a small, predictable operation.
It also creates a limitation: a package already inherited from the base image remains at the base's version.
Your reproducibility inputs therefore include both the base digest and the snapshot.
If you require every managed package to match a snapshot candidate, implement that version-convergence policy separately and test upgrades and downgrades.

### 4.3 Do not lose errors while gathering input

Shell scripts can fail in ways that look like success.
A pipeline producing an empty array is not evidence that an empty manifest was intended.
An optional command that failed is not evidence that the machine has no such packages.

The script above reads manifests directly so missing files fail before apt runs.
Its process substitution only sorts an already validated in-memory list.
For more complicated collectors, write intermediate results to a private temporary directory and check the producer's exit status.

Avoid logging complete command arguments in a generic error trap.
A later module may carry a token or secret path.
Log the module, operation, and failure location.
Redact sensitive values by design.

### 4.4 Root writes need an ownership policy

For an `/etc` file, define whether the repository owns the whole file or selected keys.
Whole-file ownership is easy for a file you create.
It is dangerous for a shared configuration such as Docker's daemon settings.

A good root-file operation:

1. Reads the current content.
2. Computes the intended result.
3. Validates the result before installation.
4. Writes only when the content differs.
5. Reloads the affected service only after an actual change.

For a third-party apt repository, record the supported Ubuntu releases and architectures, expected signing identity, key rotation procedure, and `Signed-By` scope.
A successful HTTPS download alone does not establish the publisher you intended to trust.

### Lab: run, repeat, break, repair

In a disposable Ubuntu environment, apply a package group twice.
Record the installed package inventory after each run.
Then remove one declared package and apply the module again.

**Done when:** the repeated run preserves managed state, the missing package is repaired, and an invalid manifest fails before installation.

---

## 5. Manage personal files with chezmoi

Personal files contain both shared preferences and machine-specific data.
chezmoi gives that boundary a file-oriented model.
Use it to make the target files understandable.
Keep system orchestration explicit.

### 5.1 Start with a small source state

Add a shell configuration and Git defaults first.
Avoid importing every file from a machine that has accumulated years of experiments.

The three relevant states are:

```text
Source state in Git
        ↓ render with machine data
Target state chezmoi intends to write
        ↓ compare and apply
Destination state currently on the machine
```

Review the intended result before applying it:

```bash
chezmoi diff
chezmoi apply
chezmoi verify
```

These operations belong in the file-management workflow.
They do not prove the GPU works or the system package policy is satisfied.
Those layers need their own checks.

### 5.2 Keep profile input explicit

For `home/.chezmoi.toml.tmpl`:

```gotemplate
{{- $profile := promptString "profile" -}}
{{- if not (or (eq $profile "desktop") (eq $profile "server") (eq $profile "container")) -}}
{{- fail "profile must be desktop, server, or container" -}}
{{- end }}
sourceDir = {{ .chezmoi.workingTree | quote }}

[data]
profile = {{ $profile | quote }}
```

The bootstrap supplies the answer with `--promptString profile=container` or the selected host profile.
chezmoi's prompt overrides match the prompt text.[^chezmoi-init]
Using the same literal for the key and prompt prevents a wrapper from silently missing it.
The configuration records the checkout path so later `chezmoi diff` commands use the enrolled repository.
The explicit prompt also lets a later enrollment select a different profile instead of silently retaining a previously saved answer.

Keep Git email and host-specific identity in separately supplied machine data.
Do not bake a personal identity into an image shared with another developer.

For `home/.chezmoiignore`:

```gotemplate
{{ if eq .profile "container" }}
.config/mise/config.toml
.config/mise/mise.lock
.config/systemd/user/**
.ssh/**
{{ end }}
```

The image already owns its baked toolchain.
This container profile adds safe preferences without rewriting that installation.
The host profile projects the canonical toolchain into the user's global mise configuration.

Ignore patterns must use destination names.
Test the actual rendered target set for each profile, including encrypted entries and any future sidecars.

### 5.3 Treat scripts as scripts

chezmoi supports scripts that run every apply, scripts that run when rendered content changes, and scripts that run once per unique rendered content.
Its documentation explicitly requires idempotency even for `run_once_` and `run_onchange_` scripts.[^chezmoi-scripts]

That distinction matters after drift.
A `run_onchange_` apt wrapper may not run when you manually uninstall a package, because the wrapper's content did not change.
It optimizes scheduling.
It does not replace a convergence check.

The reference design therefore keeps system reconciliation in `bootstrap`.
A normal `chezmoi apply` remains a personal-file operation.
If you later add system wrappers, describe the expanded behavior in the README and test it.

### 5.4 A changed-input hash is only as complete as its inputs

An optional desktop wrapper might include:

```gotemplate
#!/usr/bin/env bash
# selected settings: {{ include (joinPath .chezmoi.workingTree "desktop/keybindings.ini") | sha256sum }}
# implementation: {{ include (joinPath .chezmoi.workingTree "modules/desktop.sh") | sha256sum }}
set -Eeuo pipefail
{{ if eq .profile "desktop" }}
bash {{ joinPath .chezmoi.workingTree "modules/desktop.sh" | shellQuote }}
{{ end }}
```

Save that as an appropriately named `run_onchange_after_` template only if you choose chezmoi-based desktop scheduling.
Hash the implementation as well as its data.
A fixed data hash will not trigger a changed script.

Even then, verify the destination settings independently.
A successful execution record says a script ran.
It does not say its effect survived.

### Lab: test every profile without credentials

Render source state for desktop, server, and container in isolated homes.
Review the target file list and permissions.
Use fake identity values and no personal vault.

**Done when:** container rendering cannot request host secrets, install user services, or change the baked toolchain.

---

## 6. Lock developer tools with mise

The toolchain declaration should explain which tools you want.
The lockfile should record what that request resolved to.
Keep both under review.

### 6.1 A small canonical configuration

For `toolchains/mise.toml`:

```toml
[settings]
lockfile = true
lockfile_platforms = ["linux-x64", "linux-arm64"]

[tools]
python = "3.13"
node = "24"
go = "1.26"
rust = "stable"
uv = "latest"
ripgrep = "latest"
chezmoi = "latest"
```

The version selectors illustrate the policy.
Choose supported versions for your projects and generate the real lockfile.
Do not handwrite URLs, checksums, or a pretend lock format.

From the `toolchains/` directory:

```bash
mise trust ./mise.toml
mise lock --platform linux-x64,linux-arm64
mise install --locked
```

mise documents platform entries and a strict locked installation that fails when required matching lock information is missing.
The details recorded vary by backend.[^mise-lock]

Review the exact config before trusting it.
Tool configuration can become an execution surface as you add hooks and tasks.[^mise-trust]

### 6.2 Inspect what the lock actually contains

Do not assume every tool has a download URL, checksum, complete dependency graph, and publisher verification.
Some backends record more than others.
Some source builds resolve additional inputs through their own package manager.

For each important tool, inspect:

- The selected version and backend.
- Entries for each supported platform.
- Artifact URL and checksum coverage.
- Any dependency graph or sidecar files.
- The verification behavior of the pinned mise release.

For Rust or a CLI compiled through Cargo, a top-level version may be a weaker constraint than a verified prebuilt release.
That can be acceptable.
Record the weaker guarantee and test the result.

The same distinction applies to npm and Python tools.
A pinned package name and version is not automatically a complete transitive lock.
Follow the lock behavior of your selected backend and mise release.

### 6.3 A toolchain lock is not a project lock

A selected Python interpreter does not constrain everything `pip` might install.
A selected Node release does not constrain a project's npm dependency graph.

Keep `uv.lock`, `package-lock.json`, and equivalent files in their project repositories.
Use locked project installation in the project acceptance checks.

uv's tool commands install applications in isolated environments.[^uv-tools]
That is useful for CLI ownership.
It is not a reason to put project libraries in a global tool environment.

### 6.4 Use explicit execution in automation

Interactive shell activation is for interactive shells.
Automated checks should use an explicit configuration context:

```bash
cd toolchains
MISE_LOCKED=1 mise exec -- python --version
MISE_LOCKED=1 mise exec -- node --version
```

An image can expose baked tools through shims and also preserve `/opt/toolchains` for explicit execution.
After a project checkout, that project's mise config may deliberately select different versions.
Test the common baseline and the project workflow separately.
`mise exec` provides the command environment; `MISE_LOCKED=1` supplies the strict lock policy.[^mise-exec]

### 6.5 Update the canonical files

Generate updates in `toolchains/`.
Review the configuration, lockfile, and any sidecars as one change.
Apply the host projection afterward.

If you run `mise use -g` on the host, you change the deployed global configuration.
That creates local drift from the canonical files.
Do not reconcile it by blindly re-adding rendered templates into chezmoi.
Edit the canonical inputs and regenerate their projections.

### Lab: test lock coverage

Install from the committed lock in a clean supported environment.
Try a deliberately incomplete platform entry in a disposable copy.
Check that strict installation fails rather than resolving a new version.

**Done when:** you know which tool inputs are locked and can distinguish incomplete lock data from an upstream download failure.

---

## 7. Bootstrap without circular dependencies

A bootstrap has to work before the managed environment exists.
That is where many otherwise clean designs become circular.

If mise installs chezmoi and chezmoi deploys the only mise configuration, something outside that loop has to establish the first executable.
Name that trust root.

### 7.1 Make the prerequisites boring and explicit

The reference bootstrap assumes:

- A supported Ubuntu installation with apt configured.
- Git and a reviewed checkout of the workstation repository.
- Approved versions of mise and chezmoi installed through a documented verified procedure.
- Root authorization for the selected host modules.

A signed distribution package or a release artifact verified against an independently trusted publisher key can establish the bootstrap executables.
Pin and document the selected procedure.
Downloading an artifact and its checksum from the same compromised location only detects accidental corruption.

Keep bootstrap versions separate from the development toolchain they install.
If mise manages a later chezmoi version, define which one the bootstrap invokes.
Two untracked executable paths are another form of drift.

### 7.2 A host bootstrap with one orchestrator

The following script handles the shared package and personal-file layers.
Desktop and NVIDIA modules are separate steps until you implement and test their host-specific contracts.

```bash
#!/usr/bin/env bash
# bootstrap
set -Eeuo pipefail

usage() {
  printf 'usage: bootstrap --profile desktop|server|container\n' >&2
  exit 2
}

[[ $# == 2 && $1 == "--profile" ]] || usage
profile=$2
case "$profile" in
  desktop|server|container) ;;
  *) usage ;;
esac

((EUID != 0)) || {
  printf 'run bootstrap as the intended ordinary user\n' >&2
  exit 1
}

repo_root=$(cd -- "$(dirname -- "${BASH_SOURCE[0]}")" && pwd)
command -v chezmoi >/dev/null
command -v mise >/dev/null

if [[ "$profile" != "container" ]]; then
  sudo -v
  groups=(base dev geo)
  [[ "$profile" != "desktop" ]] || groups+=(desktop)
  sudo env WORKSTATION_APT_SNAPSHOT= \
    bash "$repo_root/modules/apt.sh" "${groups[@]}"
fi

chezmoi init --source "$repo_root" --promptString "profile=$profile"
chezmoi apply --source "$repo_root"

if [[ "$profile" != "container" ]]; then
  (
    cd -- "$HOME"
    mise install --locked
  )
fi

printf 'bootstrap: shared layers applied for %s\n' "$profile"
```

The host explicitly clears the image snapshot selection.
The package module owns privilege-sensitive work.
chezmoi and mise run as the intended user.

Populate every selected manifest before using this script.
The `geo` group belongs to this book's running example; remove it if your contract does not require it.
The `desktop` group installs declared packages but does not itself establish desktop-session settings.

The template input is supplied, but that alone is not a noninteractive guarantee.
Any future secret template or prompt must have a declared unattended behavior.
CI should execute without a TTY and fail if credentials or input are unexpectedly required.

### 7.3 Give containers a narrower entry point

For the dotfiles hook, `install.sh` can delegate to the container profile:

```bash
#!/usr/bin/env bash
# install.sh
set -Eeuo pipefail
repo_root=$(cd -- "$(dirname -- "${BASH_SOURCE[0]}")" && pwd)
exec bash "$repo_root/bootstrap" --profile container
```

This assumes the tested image already includes the prerequisites.
The hook applies personal preferences.
It does not repair an arbitrary third-party image into the full workstation.

If you need that broader behavior, define it as a different entry point with its own tests.

### 7.4 Plan for partial completion

A network failure may happen after apt succeeds and before tool installation completes.
On rerun, completed steps should preserve their state and failed steps should retry.
Do not put a single “bootstrap complete” marker ahead of the actual checks.

Record failures by phase.
A tool download failure should not send you back to debugging GNOME.
A desktop-session failure should not invalidate the already verified CLI baseline.

### Lab: interrupt and resume

Fail a tool download in a disposable environment after the package step.
Restore access and rerun bootstrap.
Check ownership of the managed files and tool directories.

**Done when:** rerunning repairs the incomplete phase without leaving root-owned files in the user's home.

---

## 8. Handle credentials and machine identity

Recovery needs credentials, but reproducible builds should not contain a person's credentials.
Make that boundary visible in both file management and image construction.

### 8.1 Classify before choosing encryption

| Material | Storage policy | Recovery behavior |
| --- | --- | --- |
| Public Git defaults | Shared declaration | Render without a vault |
| Personal Git email | Machine data | Supply during host enrollment |
| SSH private key | Separate credential system | Recover or create according to its policy |
| API token | Runtime secret provider | Inject only into the intended process |
| Encrypted managed file | Git ciphertext, key elsewhere | Decrypt only in an approved profile |
| Public signing key or expected identity | Reviewed trust declaration | Available before artifact verification |

Encrypted content can still reveal filenames and operational structure.
Keep the public repository small enough to review.

### 8.2 Key recovery is part of the system

age or a password manager can fit the host file workflow.
Choose from how you will recover access on a new machine.

A key stored only on the lost workstation is not a recovery design.
A private key stored beside its ciphertext in plaintext is not a secret boundary.
Test the recovery path independently of the machine you are replacing.

For the first clean test, use dummy encrypted fixtures.
Prove that an authorized host can decrypt them and the container profile excludes them.
Real credentials belong in a later enrollment step.

### 8.3 Keep credentials out of the build context

Do not copy the entire personal repository into the image and delete secrets afterward.
Use a build-context allowlist and targeted `COPY` instructions.
Earlier layers and build intermediates may retain copied content.

BuildKit secret mounts provide a build-time channel for required tokens.
Maximum provenance records build arguments, so secrets must not travel through `ARG`.[^docker-provenance]

A secret mount only controls delivery.
An install script can still copy the secret into output, write it to a config file, or print it.
Review the consuming command and inspect the resulting artifact.

### 8.4 Runtime credentials remain runtime state

The image should build and pass its baseline checks with no personal vault.
A runtime needing an inference-service token receives that token when the service starts.
The token never becomes a prerequisite for unrelated shell configuration.

Minimize inherited credentials in development containers.
An agent running there can use whatever the environment makes available.
Directory mounts and Docker socket access are also authority, even when they do not look like secrets.

### Lab: recover without personal secrets

Build the baseline image and render the container profile with a clean home and no vault.
Then enroll a disposable host with a dummy encrypted fixture.

**Done when:** public build inputs are sufficient for the shared environment, and private enrollment has a separate, tested procedure.

---

## 9. Build the Docker image from shared inputs

The image should package the declared shared environment.
Keep the installation rules in the modules and the assembly rules in Docker.

You still need Docker-specific decisions: layer order, user creation, cache locations, tool-directory ownership, and build inputs.
Pretending those decisions do not exist produces an unreadable abstraction.

### 9.1 Begin with two targets

```text
Reviewed Ubuntu base
        ↓ shared apt module: base
base target
        ↓ shared apt module: dev + geo
        ↓ canonical locked toolchain
tooling target
```

Add an AI target only when you have a separate user-space dependency set to package.
An upstream inference service can run beside the tooling container.
It does not need to be embedded inside it.

### 9.2 Restrict the build context

For `.dockerignore`:

```text
**
!modules/
!modules/apt.sh
!packages/
!packages/apt/
!packages/apt/base.txt
!packages/apt/dev.txt
!packages/apt/geo.txt
!packages/apt/SNAPSHOT
!toolchains/
!toolchains/mise.toml
!toolchains/mise.lock
!docker/
!docker/Dockerfile
```

This context contains only the inputs used below.
When you add toolchain sidecars or another module, extend the allowlist deliberately.
Reviewing a build context is easier when everything admitted has a reason to be there.

The context boundary also keeps a shell-preference edit from invalidating an expensive tool installation.
Docker recommends copying dependency inputs before frequently changing source content and using cache mounts for package downloads.[^docker-cache]

### 9.3 An image assembly reference

Set `UBUNTU_IMAGE` and `MISE_IMAGE` to approved full references containing `@sha256:` digests.
They have no floating defaults.
The snapshot comes from the committed file.

```dockerfile
# syntax=docker/dockerfile:1
# docker/Dockerfile
ARG UBUNTU_IMAGE
ARG MISE_IMAGE

FROM ${MISE_IMAGE} AS mise_binary

FROM ${UBUNTU_IMAGE} AS base
SHELL ["/bin/bash", "-Eeuo", "pipefail", "-c"]
ENV LANG=C.UTF-8 TZ=Etc/UTC

COPY modules/apt.sh /opt/workstation/modules/apt.sh
COPY packages/apt/base.txt /opt/workstation/packages/apt/base.txt
COPY packages/apt/SNAPSHOT /opt/workstation/packages/apt/SNAPSHOT
RUN --mount=type=cache,target=/var/cache/apt,sharing=locked \
    --mount=type=cache,target=/var/lib/apt/lists,sharing=locked \
    WORKSTATION_APT_SNAPSHOT="$(</opt/workstation/packages/apt/SNAPSHOT)" \
    bash /opt/workstation/modules/apt.sh base

FROM base AS tooling
COPY packages/apt/dev.txt packages/apt/geo.txt /opt/workstation/packages/apt/
RUN --mount=type=cache,target=/var/cache/apt,sharing=locked \
    --mount=type=cache,target=/var/lib/apt/lists,sharing=locked \
    WORKSTATION_APT_SNAPSHOT="$(</opt/workstation/packages/apt/SNAPSHOT)" \
    bash /opt/workstation/modules/apt.sh dev geo

COPY --from=mise_binary /usr/local/bin/mise /usr/local/bin/mise
ENV MISE_DATA_DIR=/opt/mise \
    MISE_CONFIG_DIR=/opt/toolchains \
    MISE_CACHE_DIR=/var/cache/mise
ENV PATH="/opt/mise/shims:${PATH}"

COPY toolchains/mise.toml /opt/toolchains/config.toml
COPY toolchains/mise.lock /opt/toolchains/mise.lock
WORKDIR /opt/toolchains
RUN --mount=type=cache,target=/var/cache/mise \
    --mount=type=secret,id=github_token,env=MISE_GITHUB_TOKEN \
    mise install --locked

# Reuse an existing UID 1000 account; fail on an unexpected group collision.
RUN if getent passwd 1000 >/dev/null; then \
      old_user="$(getent passwd 1000 | cut -d: -f1)"; \
      if [[ "$old_user" != dev ]]; then usermod -l dev "$old_user"; fi; \
      usermod -d /home/dev -m -s /bin/bash dev; \
    else \
      useradd --create-home --uid 1000 --shell /bin/bash dev; \
    fi; \
    mkdir -p /workspaces /var/cache/mise; \
    chown dev: /workspaces /var/cache/mise

USER dev
ENV HOME=/home/dev
WORKDIR /workspaces
CMD ["bash"]
```

The mise cookbook documents copying its static binary from an official image and explicitly configuring the destination directories.[^mise-docker]
The source image's environment does not come along with `COPY --from`.

This reference leaves installed tools owned by root and lets the ordinary user write the runtime cache.
It supports the baked versions.
If users need to install additional versions, define a writable installation strategy and test it; this example does not grant write access to `/opt/mise`.
Do not fix a runtime permission failure by making the entire tool directory world-writable.

The user block accommodates a base that already has a UID 1000 account.
It does not recursively delete that account's home to create a new one.
An unexpected account or group conflict should be a visible build failure.
If your selected base has additional identity requirements, use a tested user-creation module.

The Dockerfile frontend is another input.
Before calling this a fully pinned build, replace the syntax tag with an approved frontend digest and pin the BuildKit version used by CI.
Record them with the base and mise references.

### 9.4 Build one supported architecture first

The command assumes your environment contains the approved digest references:

```bash
docker buildx build \
  --file docker/Dockerfile \
  --target tooling \
  --build-arg UBUNTU_IMAGE="$UBUNTU_IMAGE" \
  --build-arg MISE_IMAGE="$MISE_IMAGE" \
  --platform linux/amd64 \
  --load \
  --tag workstation:test \
  .
```

The token mount is optional.
When downloads need authentication, supply a narrowly scoped token through `--secret id=github_token,env=GITHUB_TOKEN`.
BuildKit documents both file and environment-variable secret mounts.[^docker-secrets]

This build does not claim to supply a tested GDAL Python environment.
The `geo` package group provides the declared system capabilities.
Project Python bindings receive their own compatibility test.

### 9.5 Cache affects speed, not intent

A cache hit means BuildKit reused an earlier result.
It does not prove upstream packages are current.
With a fixed snapshot and fixed base, an unchanged rebuild should not magically pick up newer Ubuntu security packages.

A scheduled update must change the reviewed inputs.
A scheduled clean rebuild can test whether those existing inputs are still retrievable.
Those are different jobs.

Cache mounts also need a supported concurrent-access policy.
Docker's apt examples use `sharing=locked`.[^docker-cache]
Keep caches disposable and run periodic clean builds.

### Lab: compare host and image inputs

Build `base` and `tooling`.
Inspect the package manifests used by each and the effective mise configuration.
Run the same shared-capability checks on the host and image.

**Done when:** a package declaration has one installation implementation, and every intentional host/image difference is documented.

---

## 10. Use the image in development containers

The Dockerfile builds the environment.
`devcontainer.json` describes how a supported development tool uses it.
Keep those responsibilities distinct.

### 10.1 Consume a tested image by digest

For `.devcontainer/cpu/devcontainer.json`:

```json
{
  "name": "Engineering workstation",
  "image": "ghcr.io/your-org/workstation@sha256:REPLACE_WITH_TESTED_DIGEST",
  "remoteUser": "dev",
  "updateRemoteUserUID": true,
  "init": true,
  "customizations": {
    "vscode": {
      "extensions": [
        "ms-python.python"
      ]
    }
  }
}
```

The digest placeholder is deliberately incomplete.
Replace it with the registry digest from your tested publication.
A CI check should reject unresolved placeholders in any configuration you intend to deploy.

The Dev Container specification defines `remoteUser`, UID/GID adjustment on Linux, image references, and lifecycle commands.[^devcontainers]
A published image digest is a stable environment input.
Editor preferences remain an experience layer above it.

### 10.2 Test bind-mount ownership

An ordinary user should be able to create a file in the project workspace.
The host should see the expected ownership afterward.
Exercise the actual Dev Container client, because plain `docker run` does not test that client's UID adjustment.

Also check baked-tool access after the adjustment.
A tool under `/root` may exist in the image and still be unavailable to `dev`.
A writable cache with the wrong UID may fail only after startup.

### 10.3 Keep lifecycle hooks small

Use creation hooks for project initialization that cannot be baked into the shared image.
Use project package managers for project dependencies.
Avoid reinstalling the shared toolchain every time an editor attaches.

A useful creation hook might run a project's locked dependency install.
A poor attachment hook upgrades global tools and performs apt operations during every terminal session.

Hook failures need visible logs.
The environment should not appear ready while its setup failed in the background.

### 10.4 Personal dotfiles belong to the client setup

Configure your client or Codespaces dotfiles integration to clone the workstation repository and run `install.sh`.
That clone must preserve the `home/`, `toolchains/`, and module paths expected by the entry point.

Treat this as a client-specific setup task.
It is not a portable property of every Dev Container consumer.
Test the client you actually use.

For a team image, omit personal Git identity and vault-backed files.
Each person supplies their own container-safe preferences.

### 10.5 Make GPU selection explicit

For `.devcontainer/gpu/devcontainer.json`, use the same tested image or an independently tested GPU-capable user-space image:

```json
{
  "name": "Engineering workstation with GPU",
  "image": "ghcr.io/your-org/workstation@sha256:REPLACE_WITH_TESTED_DIGEST",
  "remoteUser": "dev",
  "updateRemoteUserUID": true,
  "init": true,
  "hostRequirements": {
    "gpu": true
  },
  "runArgs": [
    "--device",
    "nvidia.com/gpu=all"
  ]
}
```

This configuration assumes a Docker runtime with native CDI support and valid host device declarations.
The CPU variant omits the device request.
Declaring a GPU optional does not rewrite an explicit runtime device argument when the device is absent.

If your selected runtime uses `--gpus all` and the NVIDIA runtime integration instead, document and test that path.
Do not combine several injection mechanisms without understanding which one owns device setup.

### Lab: open the actual project

Launch the CPU configuration on a machine without a GPU.
Create a workspace file, run the project's locked install, and execute its tests.
On the GPU machine, launch the GPU configuration separately.

**Done when:** the development workflow works through your real client, with correct ownership and no manual setup hidden in your terminal history.

---

## 11. Keep desktop and GPU state on the host

A workstation is more than a container.
Its display session, kernel, drivers, device permissions, and user services need host-level verification.

The shared modules stop at the boundary they can actually support.
Host modules build the rest.

### 11.1 Capture selected desktop preferences

Start with the preferences that affect daily work: keyboard shortcuts, input sources, and a few terminal settings.
Export the relevant subtree, review the keys, and commit only the intended values.

For example:

```bash
dconf dump /org/gnome/desktop/wm/keybindings/
```

Run desktop operations as the logged-in user in a valid session.
Root is not a substitute for that session.
A container with the `dconf` executable installed does not establish a meaningful desktop test.

Choose an ownership model.
A selected-key model preserves settings outside your declaration.
A whole-subtree model may reset additional values.
Document which behavior your module implements.

A `dconf load` alone does not explain how removed declarations should be handled.
Test what happens when you delete a previously managed key.
Reset only state your module explicitly owns.

### 11.2 Write user services with stable paths

Install the drift command as a user executable, for example `~/.local/bin/workstation-drift`.
The systemd unit should reference that path.
Do not make it depend on the incidental directory where the repository was first cloned.

For `home/dot_config/systemd/user/workstation-drift.service`:

```ini
[Unit]
Description=Report workstation configuration drift

[Service]
Type=oneshot
ExecStart=%h/.local/bin/workstation-drift
```

For `home/dot_config/systemd/user/workstation-drift.timer`:

```ini
[Unit]
Description=Weekly workstation drift review

[Timer]
OnCalendar=Mon *-*-* 10:00:00
Persistent=true
RandomizedDelaySec=15m

[Install]
WantedBy=timers.target
```

`Persistent=true` enables catch-up behavior for missed calendar activations when the timer becomes active again.[^systemd-timers]
It does not make a powered-off laptop run a job.
Nor does a user timer guarantee execution while the user's service manager is stopped.

After installing or changing the units:

```bash
systemctl --user daemon-reload
systemctl --user enable --now workstation-drift.timer
systemctl --user list-timers workstation-drift.timer
```

The service invokes the installed drift script.
Chapter 14 defines that script's behavior; add its deployment to `home/` when implemented.

### 11.3 Treat NVIDIA setup as a host capability

The host owns the driver and NVIDIA Container Toolkit.
The workload image owns the CUDA user-space libraries its framework requires.
A GPU-enabled development container consumes device access from the runtime.

```text
Hardware and firmware
        ↓
Host kernel + NVIDIA driver
        ↓
Container runtime + device injection
        ↓
Workload CUDA libraries + framework
        ↓
Inference acceptance test
```

Installing a driver inside the tooling image does not satisfy this chain.
Test from the bottom up.

Start with `nvidia-smi` on the host.
Then inspect the runtime's device configuration.
Finally run a pinned workload image and a small inference request.

### 11.4 CDI configuration needs a lifecycle

NVIDIA documents CDI support and, from Container Toolkit 1.18, an automatic refresh service.
It also identifies cases such as MIG reconfiguration and driver removal that require additional handling.[^nvidia-cdi]

Inspect available device declarations:

```bash
nvidia-ctk cdi list
```

After validating native CDI support in the selected Docker version:

```bash
docker run --rm \
  --device nvidia.com/gpu=all \
  "$GPU_TEST_IMAGE" \
  nvidia-smi -L
```

`GPU_TEST_IMAGE` must be a reviewed compatible digest reference.
Device enumeration establishes access.
It does not prove your framework, model, or kernels work.

A separate acceptance case should load a small model or execute a representative GPU operation.
Record the driver, runtime, CUDA, and framework versions with the result.

### 11.5 Avoid solving GDAL compatibility with PATH tricks

Ubuntu's Python bindings target the distribution's Python environment.
A mise-managed Python environment is a separate interpreter.
Installing `python3-gdal` does not establish that a mise-selected interpreter can import it.

GDAL's binding documentation describes the native library and header dependency and shows binding versions matched to `gdal-config`.[^gdal]

Choose one supported route:

- System GDAL CLI tools with a tested distribution-Python workflow.
- A locked project environment that builds compatible bindings against the selected native library.
- A separate tested geospatial environment with its own package ownership.

Do not export an arbitrary system `dist-packages` directory into every Python process.
That hides the compatibility boundary and makes unrelated projects harder to explain.

### Lab: test the host after reboot

Apply the host modules on a disposable VM where possible and on approved GPU hardware where necessary.
Reboot.
Verify the selected desktop settings, timer activation, device visibility, and a representative workload.

**Done when:** host capabilities survive restart and their checks identify which layer failed.

---

## 12. Test the claims the repository makes

The test plan follows the recovery contract.
Each layer needs evidence at the boundary where it runs.

### 12.1 A useful validation matrix

| Test | Environment | Claim it can establish |
| --- | --- | --- |
| Syntax, lint, and manifest validation | Any compatible development runner | Inputs satisfy local structural rules |
| Package module | Disposable Ubuntu container | Named packages can be installed and reapplied |
| Non-root file and tool setup | Disposable home in a container | Personal setup respects identity and lock policy |
| Built image acceptance | Each supported architecture | The packaged environment performs required CLI jobs |
| Dev Container startup | Actual client and bind mounts | Client setup, lifecycle, and ownership work |
| Desktop and service test | Ubuntu VM with a user session | Host configuration and reboot behavior work |
| GPU workload test | Supported physical hardware | The complete device and inference path works |

A passing Docker build does not cover the last three rows.
Name untested rows in the release record.

### 12.2 Check structure before downloading tools

Run ShellCheck and a formatter on shell modules.
Validate TOML, JSON, workflow syntax, and package manifests.
Run a secret scanner over the publishable inputs.

Reject unknown profiles, invalid package groups, unresolved deployment placeholders, and unsupported platform declarations.
These failures should happen before an installation or publication job acquires write credentials.

Keep formatting checks predictable.
A formatter that rewrites source in a validation job can hide the difference between a check and a mutation.
Use check mode in CI.

### 12.3 Idempotency needs an observation boundary

Running bootstrap twice and receiving two zero exit codes proves very little.
Both runs may append the same line, overwrite user preferences, change permissions, or restart a service.

Define a managed-state observation:

```yaml
managed_observation:
  apt:
    - installed package names, versions, and architectures
  files:
    - content hash
    - owner and group
    - permission mode
  tools:
    - resolved executable and version
    - effective configuration and lock hashes
  host:
    - selected service enablement
    - declared desktop keys

excluded_from_equality:
  - package-manager logs
  - download caches
  - observation timestamps
```

Use that observation in a sequence:

```text
Clean supported environment
        ↓ apply
Acceptance checks + observation A
        ↓ apply unchanged declarations
Acceptance checks + observation B
        ↓ compare A and B
        ↓ introduce one managed drift
Apply + acceptance checks + repair observation
```

State equality is one part of the check.
For operations where side effects matter, observe apt invocations, file replacement, and service restarts too.
An unchanged file hash does not prove the service was left running.

### 12.4 Test ordinary-user behavior

The image build runs package installation as root.
The developer runs tools as an ordinary user.
Both contexts need checks.

A small image acceptance script:

```bash
#!/usr/bin/env bash
# tests/acceptance.sh
set -Eeuo pipefail

((EUID != 0)) || {
  printf 'acceptance: expected an ordinary user\n' >&2
  exit 1
}

command -v git >/dev/null
command -v jq >/dev/null
command -v mise >/dev/null

MISE_LOCKED=1 mise exec -- python --version
MISE_LOCKED=1 mise exec -- node --version
MISE_LOCKED=1 mise exec -- go version
MISE_LOCKED=1 mise exec -- rustc --version
MISE_LOCKED=1 mise exec -- rg --version
MISE_LOCKED=1 mise exec -- chezmoi --version

probe=$(mktemp /workspaces/workstation-acceptance.XXXXXX)
trap 'rm -f -- "$probe"' EXIT
printf 'workspace-write-ok\n' > "$probe"
test "$(head -n 1 "$probe")" == "workspace-write-ok"
```

Run it against the baked image with no personal dotfiles.
Then add domain fixtures and a project test.
Version output is a baseline check, not the finished acceptance suite.

For geospatial work, transform a known coordinate or inspect a known raster and assert a meaningful result.
For inference, perform an actual request against the pinned service.

### 12.5 Exercise failure paths

At least five failures deserve deliberate tests:

1. A missing or invalid package manifest.
2. An incomplete lock for a supported platform.
3. A failed download or checksum mismatch.
4. A container profile attempting to use a secret.
5. A collector that cannot observe its assigned state.

Make the expected behavior explicit.
A checksum mismatch stops installation.
A missing optional collector is reported as unavailable.
A required collector failure makes the report incomplete.
None should become an empty successful result.

### 12.6 Containers and VMs need separate fixtures

A bootstrap-test container may need sudo installed and a test user configured.
That is a test fixture.
It is not part of the final tooling image unless the image contract requires it.

VM tests should start from the declared Ubuntu installer or cloud-image revision.
Record the baseline before workstation changes.
Use a real user session for desktop assertions.

Run periodic tests from an empty tool cache.
Otherwise the environment may look recoverable only because the old machine still has the downloads.

### Lab: write a failed recovery record

Break one input deliberately and run the recovery process.
Record the phase, evidence, diagnosis, and repair.
Improve any error message that sends you to the wrong layer.

**Done when:** failure is visible, correctly attributed, and recoverable through the documented process.

---

## 13. Publish artifacts you can verify

An image is useful when a consumer can identify it, test it, and understand why it should be trusted.
Tags alone do not establish that chain.

### 13.1 Separate validation from publication

Pull requests need read access and test infrastructure.
A trusted default-branch publication job needs registry write access.
Only the signing step needs its signing authority.

Do not give every workflow those permissions at the top level.
An untrusted branch changes the code that a build executes.
Keep publication credentials out of that execution context.

The publication sequence is:

```text
Reviewed revision
        ↓ validate inputs and build
Per-platform acceptance checks
        ↓ push candidate and record digest
Scan + inspect attestation metadata
        ↓ sign approved digest
Verify expected identity and claims
        ↓ update consumer digest
```

Treat a pushed candidate as staging until its checks pass.
If the published multi-platform build differs from the locally tested build, test the pushed per-platform digests too.
Do not equate “same Dockerfile” with “same artifact.”

### 13.2 Keep distinct evidence distinct

| Evidence | Question it helps answer | What it does not prove |
| --- | --- | --- |
| Digest | Which artifact is this? | That the artifact is useful or safe |
| SBOM | What software did the scanner identify? | A complete inventory of every relevant runtime component |
| Provenance | How and from what inputs was it built? | That the source or build logic was correct |
| Signature | Did an accepted identity sign this artifact? | That every component is trustworthy |
| Acceptance result | Did these jobs succeed on this artifact? | Correctness for every workload |

BuildKit can attach SBOM and provenance attestations.
Support depends on the driver and image store; the classic Docker image store cannot preserve image indices with attestations.[^docker-attestations]

Use a registry push path that retains the metadata.
A local `--load` test is useful but does not replace a registry-level attestation check.

### 13.3 Verify the accepted identity

For keyless signing, verification needs the expected certificate identity and OIDC issuer.[^cosign]
Supply those values from a reviewed trust policy.

```bash
cosign verify \
  --certificate-identity "$EXPECTED_SIGNING_IDENTITY" \
  --certificate-oidc-issuer "$EXPECTED_OIDC_ISSUER" \
  "$PUBLISHED_IMAGE_DIGEST"
```

Here, `PUBLISHED_IMAGE_DIGEST` is a full image reference containing the registry path and `@sha256:`.
The identity variables name the workflow identity and issuer you approved.
They must not be copied blindly from whichever certificate an untrusted artifact presents.

Inspect provenance against the expected repository, revision, build process, and materials.
A valid signature from the wrong workflow should fail the policy.

### 13.4 Scan with an actionable policy

Pick a scanner and preserve its version, database revision, findings, and suppression policy.
Begin with a rule that blocks severe findings with available fixes in relevant packages.
Document any accepted exception with an owner and expiry.

A scanner result is time-sensitive.
The same immutable image can become newly vulnerable as the advisory database changes.
Re-scan retained images as well as new ones.

The remedy is often a reviewed base or snapshot update.
Repeatedly rebuilding fixed inputs does not change the vulnerable package.

### 13.5 Add architectures as supported products

Build an arm64 target only after checking package availability, tool releases, lock coverage, and domain tests.
QEMU can help produce a build.
Native testing gives stronger evidence for native dependencies and workloads.

A multi-platform index points to separate platform images.
Record both the index digest and the platform digests used for testing.
GPU workload support may remain amd64-only even when the general tooling image supports arm64.

### Lab: verify a consumer

Publish a candidate through the reviewed pipeline.
Pull it by digest, verify the accepted identity, inspect attestations, and run acceptance checks.
Point a disposable Dev Container at that digest.

**Done when:** publication and consumption form a tested chain, including the artifact the developer actually runs.

---

## 14. Detect and reconcile drift

The current machine is evidence.
The repository is intent.
Drift detection compares them without confusing one for the other.

### 14.1 Make the declared path easy

If changing the repository is awkward, people install tools directly and promise to document them later.
That promise becomes the next drift report.

Provide small helpers:

- Add an apt package to an explicit profile group.
- Edit the canonical mise declaration and regenerate its lock.
- Add or update an approved personal file.
- Produce a private drift report.
- Run acceptance checks.

The helpers should display the change and its effect.
They should not automatically publish observations or commit a dirty checkout containing unrelated work.

### 14.2 Define the apt comparison

Let:

```text
actual   = currently installed packages marked manual
baseline = manual packages observed before workstation enrollment
declared = package names selected by this machine's profile

undeclared = actual minus baseline minus declared
missing    = declared minus all currently installed packages
```

The two comparisons answer different questions.
A declared package marked automatic is still installed.
Checking only manual packages would report a false missing package.

Select manifests through the same profile policy used by bootstrap.
Concatenating every `*.txt` file would make a container appear to declare desktop packages.

Capture the baseline before installing your bootstrap prerequisites.
Otherwise your observation silently accepts those additions as installer state.
A mature system also records the Ubuntu installation source and baseline policy, because not every machine starts with the same seed.

### 14.3 Preserve collector status

A drift command can inspect:

| Collector | Observation | Interpretation |
| --- | --- | --- |
| apt | Installed/manual names and installed versions | Declared presence and unexpected additions |
| mise | Effective config, selected versions, installed versions | Overrides and tool-resolution differences |
| chezmoi | Status and verification of managed files | Destination drift |
| Desktop | Selected declared keys | Preference changes within owned scope |
| Other package managers | Installed applications or global tools | Potential duplicate ownership or undeclared experiments |

Every collector needs a status such as `ok`, `unavailable`, or `failed`.
A shell pipeline ending in `|| true` discards that distinction.
Keep the failure and the partial report.

A useful report entry:

```json
{
  "collector": "apt",
  "status": "ok",
  "scope": "desktop-profile",
  "undeclared": ["example-debugger"],
  "missing": [],
  "review_needed": true
}
```

The package name is illustrative.
The schema communicates the observation; it does not authorize installation or deletion.

### 14.4 Store reports privately

For the report directory:

```bash
umask 077
state_root="${XDG_STATE_HOME:-$HOME/.local/state}/workstation"
mkdir -p -- "$state_root"
report_dir=$(mktemp -d "$state_root/report.XXXXXX")
printf 'drift report directory: %s\n' "$report_dir"
```

Use a unique directory for each run.
Keep an explicit retention policy.
Use private temporary files for sorted inputs instead of a shared `/tmp/declared` filename.

The drift command reads live state and writes its own private observations.
It does not apply configuration, update lockfiles, or reconcile files.
That separation makes the weekly timer safe to understand.

### 14.5 Reconcile by intent

Every difference needs one of four decisions:

| Decision | Example | Action |
| --- | --- | --- |
| Promote | A CLI became part of daily work | Add it through its intended owner |
| Keep local | A temporary debugging tool | Record a private exception if useful |
| Revert | An accidental managed-file change | Restore the declared state after review |
| Retire | A tool no longer belongs in the environment | Update declarations and plan removal |

Removing a package from a list does not remove it from existing hosts.
The presence module does not implement exact-set enforcement.
Package retirement therefore needs a separate, reviewed uninstall policy.

Avoid deriving a removal list from `actual minus declared`.
Baseline packages, local exceptions, and active project needs exist.
A drift report is not a deletion plan.

### 14.6 File reconciliation needs a direction

For a direct source file, a reviewed `chezmoi re-add` can incorporate a useful destination edit.
For a rendered template, edit the template or its canonical data.
Do not collapse a machine-specific rendered value into the shared source by accident.

For conflicts, inspect both the source change and the local change.
That is a merge problem with intent, not a reason to overwrite one side automatically.

### Lab: classify a week of changes

Collect one private report after ordinary work.
Classify each difference as promote, local, revert, or retire.
Apply only the reviewed decisions and collect again.

**Done when:** reports lead to small intentional changes, and an observation failure cannot masquerade as a clean machine.

---

## 15. Update deliberately and recover from failure

Reproducibility becomes a liability when “pinned” means “never maintained.”
The system needs a controlled way to change its inputs.

### 15.1 Give every input an update route

| Input | Update route | Evidence before acceptance |
| --- | --- | --- |
| Ubuntu base digest | Reviewed registry update | Image build, inventory, scan, acceptance |
| Ubuntu snapshot | Selected available timestamp | Package diff and clean build |
| mise executable | Verified release update | Lock compatibility and install checks |
| Developer tools | Canonical config and lock update | Backend coverage and workload checks |
| Workflow actions | Reviewed commit update | Workflow validation and restricted test run |
| Personal files | Reviewed source or template change | Profile rendering and destination diff |
| GPU stack | Host maintenance change | Compatibility review, reboot, workload test |

Renovate can propose dependency changes through its mise manager and custom regex managers.[^renovate]
That is a proposal mechanism.
It does not establish that the resulting toolchain works.

If the updater changes version requests but not the complete lock inputs, have a trusted regeneration step produce the remaining diff.
Test your exact Renovate and mise versions together.
Do not assume lock maintenance exists because a manager recognizes the config.

### 15.2 A schedule must change the intended input

A weekly clean build tests availability and compatibility of the current declaration.
A snapshot-update job selects and proposes a new archive timestamp.
A scanner may report new vulnerabilities without rebuilding anything.

Keep those jobs distinct enough that their failures are understandable.
The snapshot updater should verify availability, modify the declaration, and open a reviewable change.
Writing today's timestamp is not proof that the service has usable data for it.

GitHub documents that scheduled workflows run from the default branch and may be delayed.
In public repositories, inactivity can disable them after 60 days.[^github-schedules]
Treat the schedule as an execution mechanism, not a freshness guarantee.
Observe successful maintenance runs.

### 15.3 Keep update groups reviewable

Do not bundle a new Ubuntu release, a Python major version, CUDA changes, and shell cleanup into one update.
Each has a different failure boundary and recovery path.

Begin with reviewed merges.
Later, consider narrowly scoped automerge for changes whose required checks are reliable and whose rollback is simple.
A signing-policy update deserves different treatment from a low-risk CLI patch.

A minimum-release-age policy can reduce exposure to newly published releases.
It does not establish publisher trust or detect every compromised dependency.
Keep the policy as one signal in the update process.

### 15.4 Rollback has several meanings

| Layer | Recovery action | Limit |
| --- | --- | --- |
| Container image | Return the consumer to a retained tested digest | Persisted data may have changed |
| Toolchain | Restore reviewed config/lock and reinstall missing versions | Old artifacts must remain available |
| Personal files | Restore source revision and inspect/apply its target diff | Existing unmanaged edits need reconciliation |
| apt packages | Restore a tested OS snapshot or rebuild the host | An older package list does not undo upgrades |
| GPU driver | Use the tested host rollback procedure | Kernel, driver, and reboot compatibility matter |

Git rollback is not machine rollback.
It restores declarations.
Every stateful layer needs a separate recovery action.

For images, registry retention is part of recovery.
An old digest that garbage collection removed is not a rollback target.
For host changes, preserve a tested backup or snapshot before a maintenance event that could interrupt work.

### 15.5 Measure recovery from a clean start

Track successful rebuild time, failed phases, manual interventions, and external dependencies.
Separate download time, setup time, credential enrollment, and required reboots.

The recovery target should come from measurements on your network and hardware.
An arbitrary “productive in thirty minutes” promise is less useful than a rebuild record showing the two dependencies that still require manual work.

### Lab: perform one rollback

Upgrade a tool in a disposable environment, run acceptance checks, and restore the previous lock.
Start a project with the previous retained image digest.
Record which state needed separate handling.

**Done when:** the rollback instructions have been executed and their limits are visible.

---

## 16. Add AI assistance at a narrow boundary

An AI agent can help classify drift and draft changes.
Its useful input is a structured observation and an explicit ownership model.

Build those first.
An agent reading shell history and guessing what belongs in your workstation is missing the contract.

### 16.1 Give the agent a bounded job

```yaml
job: propose workstation declaration changes

inputs:
  - redacted drift report
  - selected profile declarations
  - tool ownership policy
  - approved local exceptions

allowed_changes:
  - approved package manifests
  - toolchain version requests
  - explanatory documentation

forbidden_changes:
  - credentials and encrypted content
  - workflows and signing policy
  - root modules and trust roots
  - automatic installation or package removal

output:
  - proposed patch
  - rationale per change
  - uncertain items left for review
```

The policy above is a suggested starting point.
Enforce the meaningful boundaries with filesystem permissions, credentials, and tool access.
A prompt alone does not enforce them.

Lock generation can run in a separate deterministic step after the agent proposes a version request.
That keeps invented checksums and fabricated lock entries out of the workflow.

### 16.2 Treat observations as untrusted input

Package descriptions, repository files, logs, and configuration comments can contain instructions.
They are data for the task.
They must not expand the agent's authority.

Do not give the drift assistant host sudo, a writable Docker socket, or publication credentials.
A patch-writing job does not require them.

If it can open a pull request, constrain that credential to the intended repository and proposal workflow.
The proposal still passes normal review and deterministic checks.
No automatic installation follows from an agent's classification.

### 16.3 Evaluate decisions with small fixtures

Build cases for:

- A useful undeclared CLI that should be promoted.
- A temporary debugger that should remain local.
- A duplicate executable that needs ownership review.
- A failed collector that should remain unknown.
- An injected instruction that should not change authority.

Include an ambiguous case.
The correct result may be “needs review.”
Forcing every difference into a permanent declaration produces a confident archive of accidents.

Measure unauthorized changes, mistaken promotions, and missed uncertainty.
A polished explanation is not the success criterion.

### 16.4 Keep the system usable without the agent

The report, helper commands, acceptance checks, and review process should work when the model service is unavailable.
AI reduces the effort of interpretation.
The repository and its evidence remain understandable on their own.

### Lab: review a proposed patch

Give an agent a redacted fixture report in an isolated checkout.
Inspect the patch, run the checks, and compare its decisions with your expected labels.
Make no live host changes.

**Done when:** the assistance saves review effort while uncertain cases remain explicit and authority stays inside the declared job.

---

## 17. A four-week adoption plan

Adoption should produce working recovery evidence every week.
Do not spend the first month cataloging settings that nobody knows how to restore.

### Week 1: establish the CLI baseline

Write the environment contract and capture a clean-machine baseline.
Choose one Ubuntu release and one architecture.
Create the package manifests, canonical toolchain, and a small chezmoi source state.

Implement the package module and shared-layer bootstrap.
Run them in a disposable Ubuntu environment as the intended user.
Exercise a failed download and a rerun.

**Exit condition:** the shared CLI environment can be recovered without relying on the existing workstation.

### Week 2: build and consume the image

Pin the base and mise image references.
Select an available archive snapshot.
Build the tooling image from the shared package module and toolchain.

Run ordinary-user acceptance checks.
Open a real project through the CPU Dev Container configuration.
Check workspace ownership and the project's locked dependency installation.

**Exit condition:** host and image perform the shared jobs, and their declared differences are understood.

### Week 3: establish maintenance and artifact trust

Add private drift observations, profile-specific comparison, and a weekly timer.
Make the declared installation path convenient.
Perform one drift review without automatically importing the inventory.

Add restricted CI, publication, scanning, signing, and consumer verification.
Test a retained image rollback and a toolchain rollback.

**Exit condition:** you can update the environment and explain which artifact a consumer trusts.

### Week 4: earn the host-specific features

Apply selected desktop settings in a VM with a real user session.
Check behavior after reboot.
Test the GPU chain on supported hardware and the geospatial fixture in its selected Python environment.

Add arm64 only if it is part of the actual recovery contract.
Add AI drift assistance only if report review is now a demonstrated bottleneck.

**Exit condition:** every capability called supported has a recorded test in the environment where it runs.

### Ongoing cadence

Review drift weekly.
Review proposed version changes according to their impact.
Run periodic clean-cache builds and complete host recovery exercises.
Revisit the tool ownership model when new requirements make a boundary awkward.

The durable habit is to capture an intentional change while you make it.
The weekly report catches what that habit misses.

---

## 18. Appendix: working contracts and checklists

### A. Recovery record

```yaml
recovery_id:
date:
operator:
repository_revision:

target:
  profile:
  os_release:
  installer_or_base_revision:
  architecture:
  hardware_class:

inputs:
  bootstrap_tool_versions:
  toolchain_lock_hash:
  sidecar_hashes:
  image_digest:
  apt_snapshot:

results:
  shared_cli:
  project_acceptance:
  workspace_permissions:
  desktop_after_reboot:
  gpu_workload:

duration:
  downloads_minutes:
  installation_minutes:
  enrollment_minutes:
  reboot_minutes:

manual_steps: []
unverified_capabilities: []
failures: []
next_actions: []
```

### B. Module contract

```yaml
module:
owner:
scope:

preconditions:
  privilege:
  session:
  supported_os:
  supported_architectures:

inputs: []
managed_state: []
unmanaged_state: []

behavior:
  first_apply:
  repeated_apply:
  removed_declaration:
  partial_failure:

verification:
  acceptance:
  unchanged_state:
  drift_repair:
  rollback:
```

`removed_declaration` prevents an installer from being mistaken for an exact-state manager.
`partial_failure` prevents a marker file from being mistaken for convergence.

### C. Drift review item

```yaml
item_id:
collector:
collector_status:
profile:
observed_difference:
declared_owner:

decision: promote | keep-local | revert | retire | needs-review
rationale:
privacy_review:
proposed_patch:
required_checks: []
removal_plan:
reviewer:
```

### D. Image release record

```yaml
release:
repository_revision:

build_inputs:
  ubuntu_base_digest:
  ubuntu_snapshot:
  mise_image_digest:
  dockerfile_frontend_digest:
  buildkit_version:
  toolchain_lock_hash:

artifacts:
  index_digest:
  platform_digests: {}
  sbom:
  provenance:
  accepted_signing_identity:
  accepted_oidc_issuer:

checks:
  input_validation:
  clean_build:
  ordinary_user_acceptance:
  platform_workloads:
  vulnerability_policy:
  signature_verification:
  consumer_startup:

previous_release:
retention_until:
exceptions: []
```

### E. Readiness checklist

#### Declarations

- Every important executable has one intended owner.
- Profiles select package groups explicitly.
- Canonical toolchain files and required sidecars are committed.
- Machine observations remain separate from shared intent.
- Every claimed platform has acceptance evidence.

#### Convergence

- Invalid inputs fail before mutation.
- Repeated application preserves observed managed state.
- Introduced drift has a tested repair path.
- Retired declarations have an explicit removal policy.
- Partial failures can be resumed.

#### Credentials

- The baseline image builds without personal secrets.
- The build context admits only intended inputs.
- Credential recovery has been exercised independently.
- Runtime secrets stay outside published layers and metadata.
- Signing verification uses an approved identity and issuer.

#### Operations

- Drift reports preserve collector failures.
- Scheduled jobs have observable successful executions.
- Updates change reviewed inputs and run relevant checks.
- Retained images and host backups support the stated rollback.
- A clean rebuild has been timed and recorded.

---

## 19. Sources and verification notes

The manuscript began with supplied research on an Ubuntu workstation and Docker image built from one repository.
The architecture and labs here are engineering recommendations developed from that material.
Tool-specific behaviors were checked against primary documentation while preparing this draft on October 12, 2026.

The source research's exact Ubuntu release features, popularity counts, build-speed claims, and proposed agent-action flags are not necessary to the design and are not carried forward.
The examples also avoid assuming that every mise backend locks the same inputs, that a second zero exit code proves idempotency, or that an image rebuild changes pinned packages.

The shell and configuration examples are intended for local validation and the layered tests described in chapter 12.
Their presence in the book is not a claim that the full workstation has been deployed on your hardware.
Use the recovery record to capture that evidence.

[^ubuntu-snapshots]: Canonical, “How to use the Ubuntu snapshot service.”
    Documents archive selection and the warning about newer security-pocket packages.
    `https://ubuntu.com/server/docs/how-to/software/snapshot-service/`
    See also apt's snapshot configuration reference:
    `https://manpages.ubuntu.com/manpages/noble/man5/apt.conf.5.html`

[^nix]: Nix reference manual, “Derivations.”
    Describes derivation inputs and build dependencies.
    `https://nix.dev/manual/nix/2.34/language/derivations.html`

[^homebrew]: Homebrew documentation, “Homebrew Bundle, brew bundle and Brewfile.”
    Explains the absence of a Brewfile lockfile and historical version support.
    `https://docs.brew.sh/Brew-Bundle-and-Brewfile`

[^chezmoi-root]: chezmoi reference, “.chezmoiroot.”
    Defines relocation of the source-state root within the working tree.
    `https://www.chezmoi.io/reference/special-files/chezmoiroot/`

[^chezmoi-variables]: chezmoi reference, “Variables.”
    Defines `.chezmoi.sourceDir` and `.chezmoi.workingTree`.
    `https://www.chezmoi.io/reference/templates/variables/`

[^chezmoi-init]: chezmoi reference, “init.”
    Describes configuration initialization and prompt override pairs.
    `https://www.chezmoi.io/reference/commands/init/`

[^chezmoi-scripts]: chezmoi user guide, “Use scripts to perform actions.”
    Explains rendered-script scheduling and the idempotency requirement.
    `https://www.chezmoi.io/user-guide/use-scripts-to-perform-actions/`

[^mise-lock]: mise documentation, “Lockfile (mise.lock).”
    Describes strict mode, platform coverage, backend differences, sidecars, and the boundary between tools and project dependencies.
    `https://mise.jdx.dev/dev-tools/mise-lock.html`

[^mise-trust]: mise reference, “mise trust.”
    Explains config-file trust.
    `https://mise.jdx.dev/cli/trust.html`

[^mise-exec]: mise reference, “mise exec,” and the `locked` setting.
    Describe command execution and the `MISE_LOCKED` policy.
    `https://mise.jdx.dev/cli/exec.html`
    `https://mise.jdx.dev/configuration/settings.html`

[^uv-tools]: Astral, “Using tools.”
    Describes isolated Python tool installation and upgrades.
    `https://docs.astral.sh/uv/guides/tools/`

[^mise-docker]: mise documentation, “Docker.”
    Documents copying the binary and configuring its destination environment.
    `https://mise.jdx.dev/mise-cookbook/docker.html`

[^docker-cache]: Docker documentation, “Optimize cache usage in builds.”
    Covers layer order, context boundaries, and package-manager cache mounts.
    `https://docs.docker.com/build/cache/optimize/`

[^docker-secrets]: Docker documentation, “Build secrets.”
    Defines build-time secret mounts.
    `https://docs.docker.com/build/building/secrets/`

[^docker-provenance]: Docker documentation, “Provenance attestations.”
    Explains provenance modes and disclosure of build arguments in maximum mode.
    `https://docs.docker.com/build/metadata/attestations/slsa-provenance/`

[^docker-attestations]: Docker documentation, “Build attestations.”
    Describes attestation generation and driver/image-store support.
    `https://docs.docker.com/build/metadata/attestations/`

[^devcontainers]: Dev Container specification, “devcontainer.json reference.”
    Defines image, identity, lifecycle, and host-requirement properties.
    `https://github.com/devcontainers/spec/blob/main/docs/specs/devcontainerjson-reference.md`

[^systemd-timers]: systemd source documentation, “systemd.timer.”
    Defines calendar timers and persistence behavior.
    `https://github.com/systemd/systemd/blob/main/man/systemd.timer.xml`

[^nvidia-cdi]: NVIDIA Container Toolkit documentation, “Support for Container Device Interface.”
    The 1.18.0 reference documents the automatic refresh service and its limitations; the current reference covers current device-injection behavior.
    `https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/1.18.0/cdi-support.html`
    `https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/cdi-support.html`

[^gdal]: GDAL documentation, “General information” for Python bindings.
    Describes native-library requirements and matching bindings.
    `https://gdal.org/en/stable/api/python/python_bindings.html`

[^cosign]: Sigstore documentation, “Verifying Signatures.”
    Describes expected certificate identity and OIDC issuer verification.
    `https://docs.sigstore.dev/cosign/verifying/verify/`

[^renovate]: Renovate documentation, “mise Manager” and “Custom Manager Support using Regex.”
    Define the built-in manager and custom extraction mechanism.
    `https://docs.renovatebot.com/modules/manager/mise/`
    `https://docs.renovatebot.com/modules/manager/regex/`

[^github-schedules]: GitHub Actions documentation, “Events that trigger workflows,” schedule section.
    Documents default-branch execution, possible delays, and inactivity-related disabling.
    `https://docs.github.com/en/actions/reference/workflows-and-actions/events-that-trigger-workflows#schedule`
