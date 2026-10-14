# Remote Coding Agents in Practice

## A Hands-On Guide to Docker Sandboxes, Shared GPU Inference, and an Ubuntu Workstation Managed from a MacBook

Your MacBook is a good place to review code.
Your desktop is a good place to compile it.
Once agents can work while you are away, the useful arrangement is to let each machine do its job.

The difficulty starts when you ask for ten tasks at once.
They need independent checkouts.
They need dependencies before they can do useful work.
They all want to run a server on port 3000.
Some need a model that already occupies most of your GPU.
One finishes while another waits for a credential that expired an hour ago.

Opening ten terminals does not resolve those conflicts.
It gives you ten places to discover them.

> Give each task a private machine, keep shared services narrow, and preserve the result before reclaiming the environment.

This book develops that arrangement on one Ubuntu workstation.
Docker Sandboxes supplies the local microVM runtime.
The GPU stays with a host inference service.
Tailscale carries remote administration and previews.
A small host controller records tasks, admits work, collects evidence, and cleans up.
The MacBook remains the place where you inspect results and decide what to publish.

The running example is a developer with a 128 GiB Ubuntu desktop, an NVIDIA GPU, and two repositories.
One repository contains a TypeScript service.
The other contains a Java service built with Gradle.
Ten independent coding tasks are the initial concurrency target.
The GPU's actual model capacity is measured separately.
The examples do not assume a particular card or promise that ten simultaneous long-context requests will fit.

The architecture is intentionally smaller than a hosted sandbox platform.
It serves one operator and repositories that operator is authorized to use.
It does not claim isolation suitable for hostile customer tenancy.
The tasks edit code, run tests, build containers, and publish temporary development previews.
Production deployment and remote Git writes stay outside their default authority.

**Relationship to the other books.**
[Self-Hosted Agent Sandboxes in Practice](SELF_HOSTED_AGENT_SANDBOXES_IN_PRACTICE.md) develops a Firecracker system for operators who need to own workspace images, restore behavior, leases, and runtime integration.
This book takes the adoption path.
It applies the same operating principles to a packaged runtime and adds shared inference and remote workstation management.
These are different scope decisions.
Choose the one whose maintenance responsibilities you want to own.

**How to read the examples.**
Documented `sbx`, Git, and Tailscale commands illustrate the checked interfaces.
Names such as `agentctl`, `inference-gateway`, and `export-worker` describe services to implement or adopt.
The YAML task and policy records are proposed contracts, not Docker configuration schemas.
Code marked as a sketch leaves integration work explicit.
Nothing here is a complete fleet installer.

**Version scope.**
The source research is “Running 10+ Sandboxed Coding Agents on an Ubuntu GPU Workstation, Managed from a MacBook,” prepared in October 2026.
Primary documentation was checked on October 14, 2026.
Some details differ from that research, including installation, SSH forwarding, network policy, and template capture.
The final chapter records the corrections and the claims that still need evidence from your host.

---

## Contents

1. [Define a remote task you can finish](#1-define-a-remote-task-you-can-finish)
2. [Choose the boundary and its owner](#2-choose-the-boundary-and-its-owner)
3. [Separate the laptop, host, and sandboxes](#3-separate-the-laptop-host-and-sandboxes)
4. [Prepare the workstation and its budget](#4-prepare-the-workstation-and-its-budget)
5. [Create the first sandbox deliberately](#5-create-the-first-sandbox-deliberately)
6. [Give each task an independent repository](#6-give-each-task-an-independent-repository)
7. [Make network access an explicit dependency](#7-make-network-access-an-explicit-dependency)
8. [Keep credentials and host integrations narrow](#8-keep-credentials-and-host-integrations-narrow)
9. [Share inference without sharing the GPU](#9-share-inference-without-sharing-the-gpu)
10. [Prove the agent and model work together](#10-prove-the-agent-and-model-work-together)
11. [Warm toolchains and dependencies](#11-warm-toolchains-and-dependencies)
12. [Publish templates you can explain](#12-publish-templates-you-can-explain)
13. [Manage sessions from the MacBook](#13-manage-sessions-from-the-macbook)
14. [Publish previews with a clear lifetime](#14-publish-previews-with-a-clear-lifetime)
15. [Build a small controller that can recover](#15-build-a-small-controller-that-can-recover)
16. [Collect results before cleanup](#16-collect-results-before-cleanup)
17. [Measure capacity and useful throughput](#17-measure-capacity-and-useful-throughput)
18. [Operate through failures and upgrades](#18-operate-through-failures-and-upgrades)
19. [Keep a fallback and roll out through evidence](#19-keep-a-fallback-and-roll-out-through-evidence)
20. [Appendix: contracts, drills, and sources](#20-appendix-contracts-drills-and-sources)

---

## 1. Define a remote task you can finish

Start with one change you know how to verify.

The TypeScript service accepts an invalid date range.
The task is to reject it, add a regression test, and preserve the existing valid cases.
The agent needs source files, dependency downloads, a model endpoint, and a test runner.
It does not need your home directory, cloud credentials, or the right to push to the default branch.

That task gives the system a useful shape.
The sandbox contains execution.
The host supplies a bounded set of services.
The reviewer decides whether the patch is correct.

### 1.1 Write the task contract

```yaml
schema: personal-agent-task/v1
task_id: task-20261014-001
repository_id: calendar-api
base_commit: "<full-reviewed-commit-id>"
objective: "Reject an end date before the start date."

acceptance:
  - invalid date ranges return the documented client error
  - valid date ranges retain their current behavior
  - a regression test fails before the change and passes afterward

environment:
  runtime: local-docker-sandboxes
  workspace_mode: clone
  resource_class: medium
  inference_profile: local-coding
  network_profile: calendar-api-build

authority:
  remote_git_write: false
  host_ssh_agent: false
  host_mcp_tools: []

limits:
  elapsed_minutes: 90
  model_requests: 200
  export_interval_minutes: 10
```

The numbers are example operating limits.
Change them after measuring your tasks.
The contract should remain small enough to review before dispatch.

Acceptance belongs to the product behavior.
Environment readiness belongs to the fleet.
A passing dependency install cannot establish that the validation change is correct.

### 1.2 Define what completion means

Treat four events separately:

| Event | Evidence |
|---|---|
| Environment ready | Exact checkout and baseline test work |
| Agent finished | Agent process or conversation stopped |
| Result preserved | Patch, commit, and evidence exist outside the sandbox |
| Change accepted | Reviewer approves the behavior and verification |

An agent can finish without preserving a result.
A result can be preserved even when its tests fail.
A reviewer can reject a technically passing change because it implements the wrong requirement.

Use those distinctions in status displays and notifications.
“Done” is too small a vocabulary for this workflow.

### 1.3 Define the threats you intend to contain

Assume generated commands can be wrong.
Assume repository scripts can execute arbitrary code.
Assume a downloaded package or instruction file can ask for broader authority than the task needs.

The first boundary should contain filesystem changes, processes, and Docker operations.
The second should constrain outbound access and credential use.
The third should separate generated results from host execution.

For example, an agent can modify a test script.
The VM contains the script while the agent runs it.
If you fetch that change and execute the script on your host, you have moved the execution boundary.
Review and verification must account for that move.

This book chooses a private test environment for returned code.
The trusted host export process moves data.
It does not run the agent's build scripts.

### 1.4 Choose the operator experience

The operator should be able to leave the laptop, return, and answer:

- Which tasks are still running?
- Which tasks need a decision?
- Which results are ready to review?
- Which failures require host repair?
- Which environments can be removed?

That is the useful remote experience.
The number of terminals is an implementation detail.

---

## 2. Choose the boundary and its owner

A sandbox runtime does two jobs.
It limits what generated code can reach.
It supplies an environment where ordinary development still works.

A runtime that blocks every build is easy to contain.
A runtime that grants every integration is easy to use.
The useful choice is the boundary whose guarantees and exceptions you can operate.

### 2.1 Start with the adopted microVM path

Docker documents local Sandboxes as microVMs with a separate kernel and private Docker Engine.
The host workspace, network proxy, skills store, SSH agent, and MCP gateway are explicit crossings of that boundary.[^docker-security]

That is a good starting point for this book's workloads.
The agent can build containers inside its VM.
The operator avoids assembling a VMM, guest images, and tap networking just to run the first task.

The remaining work is substantial.
You still own task admission, repository selection, effective policy review, inference capacity, exports, and retention.
A runtime cannot infer those decisions from a prompt.

### 2.2 Compare responsibilities

| Path | Execution boundary | What you retain responsibility for | Choose it when |
|---|---|---|---|
| Docker Sandboxes | Packaged microVM | Task lifecycle, shared-service policy, result review | Ordinary coding tasks fit its interfaces |
| Docker Engine with gVisor | Userspace kernel between workload and host | Images, networking, proxy, credentials, exports | A measured compatibility need justifies this work |
| Incus VMs | Managed virtual machines | VM images, provisioning, service policy | Persistent development machines are the requirement |
| Firecracker platform | MicroVM under your own integration | Guest artifacts, restore, networking, broker, operations | Owning those mechanisms is part of the objective |
| Agent-native process sandbox | Restricted processes on the current machine | Whole-agent integrations and account exposure | Additional restrictions inside an established outer boundary |

These are architectural choices.
The table is not a performance ranking.
The gVisor project documents `runsc` integration with Docker, while the existing Firecracker book covers the responsibilities of a custom microVM service.[^gvisor-docker]

Do not select an alternative because one synthetic boot number is smaller.
Measure the environment that reaches a passing repository test.

### 2.3 Keep the GPU requirement separate

The coding task needs model responses.
That does not imply it needs a GPU device inside its sandbox.

Sharing inference over a narrow API avoids giving every build process a route into the driver.
It also lets one inference service own batching, memory allocation, and model loading.

Docker's documented GPU passthrough is experimental VFIO on x86_64 Linux with an NVIDIA GPU that nothing else is using.[^docker-gpu]
It is a separate architecture for a dedicated device.
It does not solve sharing the same card with host inference.

If a future task needs to run CUDA code, define that workload separately.
Its device exposure, capacity reservation, and verification change.
Do not turn on passthrough for every coding task to satisfy one exception.

### 2.4 Decide what would make you switch

Write down the actual exit conditions:

```yaml
# Proposed runtime decision record.
default_runtime: docker-sandboxes

fallback_triggers:
  - a required build repeatedly fails on the supported runtime
  - required network policy cannot be enforced and tested
  - required result export cannot survive normal failure recovery
  - account or licensing requirements conflict with the deployment

insufficient_reasons:
  - a different runtime has a smaller advertised boot time
  - a blog describes a feature without a reproducible requirement
```

A failed setup step usually calls for diagnosis.
A verified missing capability calls for a runtime decision.
Keep those conversations separate.

---

## 3. Separate the laptop, host, and sandboxes

The MacBook should not become a hidden dependency for work running on the desktop.
Closing its lid should interrupt your view of a task.
It should not terminate the task, remove its credential, or erase its result.

### 3.1 Assign responsibilities

The laptop handles dispatch, conversation, previews, and review.
The Ubuntu host owns execution services and durable records.
Each sandbox owns only its task state.

```text
MacBook
  |
  | Tailscale: SSH and permitted preview connections
  v
Ubuntu workstation
  |
  +-- host task controller
  |     +-- admission and task ledger
  |     +-- result export and retention
  |     +-- preview ownership
  |
  +-- Docker Sandboxes runtime and policy proxy
  |     +-- task-001: private clone, agent, private Docker Engine
  |     +-- task-002: private clone, agent, private Docker Engine
  |     +-- ...
  |
  +-- inference gateway --> local model service --> NVIDIA GPU
  |
  +-- reviewed templates, package mirrors, Git sources
  |
  +-- monitoring and private notification delivery
```

This is the proposed reference topology.
The inference gateway is a separate service contract.
Docker's egress proxy does not by itself implement the gateway's task quotas or model admission.

### 3.2 Keep shared services smaller than shared machines

A package mirror serves dependency artifacts.
An inference service serves model responses.
An export store receives bounded result data.

Each service has a narrow purpose.
None needs to expose the host's entire filesystem.
None needs to accept an arbitrary shell command from the agent.

The controller has broader authority.
It can create environments, register routes, and revoke credentials.
Keep it on the host under the operator's control.
Repository content must not supply its executable configuration.

### 3.3 Treat the host account as an authority boundary

The account that administers sandboxes can reach runtime state and host services.
A shell in that account is an administrator interface even if it does not run as root.

Use a dedicated host account when the desktop also contains unrelated personal data.
Grant the operator remote access to that account.
Keep shared infrastructure configuration in a reviewed repository.

You may use the desktop interactively too.
That increases the consequences of host mistakes.
Record which files and services the fleet account can reach instead of calling the host disposable.

### 3.4 Map the crossing points

```yaml
# Proposed authority inventory.
crossings:
  workspace:
    mode: clone
    host_source: read_only
  network:
    effective_policy: recorded_per_sandbox
  credentials:
    provider: scoped_proxy_management
    repository_write: absent
  ssh_agent:
    forwarding: disabled
  shared_skills:
    mode: off
  mcp:
    host_tools: none
  previews:
    host_binding: loopback
    audience: permitted_tailnet_clients
```

The inventory is the desired configuration.
Inspect the installed runtime to establish the actual configuration.

The checked Docker security documentation contains inconsistent descriptions of the shared skills default in different sections.
Its CLI and settings references document explicit `off`, `readonly`, and `readwrite` choices.[^docker-create][^docker-settings]
Set the mode deliberately.
A default you did not inspect is not an operating decision.

---

## 4. Prepare the workstation and its budget

Ten agent sessions are cheap only while they wait.
Ten builds, ten private Docker stores, and a loaded inference server make the host a capacity problem.

Prepare it as a machine with reservations and recovery requirements.

### 4.1 Confirm the runtime prerequisites

The checked Docker installation guide requires Ubuntu 24.04 or later and KVM for local Linux sandboxes.
The user needs access through the `kvm` group.
Docker Desktop and host Docker Engine are not required for `sbx` itself.[^docker-install]

On the Ubuntu host:

```bash
uname -m
uname -r
ls -l /dev/kvm
id
```

Verify that the intended account can open `/dev/kvm`.
A device node alone does not prove permissions or usable virtualization.
After changing group membership, establish a new login session and repeat the check.

Install from the supported package path or a reviewed release artifact.
Record the package version and source.
The research's convenience installer example combines Docker Engine and SBX; it is not the SBX-only installation path.
Follow the distinction in Docker's installation guide.[^docker-install]

### 4.2 Reserve host memory first

For the example 128 GiB machine, begin with this planning budget:

| Consumer | Example reservation |
|---|---:|
| Desktop, kernel, runtime, and remote administration | 12 GiB |
| Inference host RAM and associated services | 20 GiB |
| Maintenance and failure headroom | 16 GiB |
| Ten task environments | 80 GiB |

The inference reservation is an assumption to measure.
CPU offload or a different model can exceed it.
GPU VRAM is a separate budget.

Ten times 8 GiB uses the entire task allocation before adding any unaccounted overhead.
Measure the runtime overhead and reduce admission if it does not fit the host reserve.
A table that sums correctly can still model the wrong working set.

Set sandbox memory and CPU explicitly.
The checked CLI documents automatic CPU allocation and a default memory selection based on host memory.[^docker-create]
Those defaults do not form a ten-task admission policy.

### 4.3 Distinguish CPU allocation from CPU capacity

Ten environments with four vCPUs each can request forty virtual CPUs.
That does not give a smaller host forty independent physical cores.

Begin with a small number of concurrent expensive builds.
Allow more tasks only when their measured mix leaves acceptable latency for the host and inference service.
Agents waiting for a response and agents running Gradle tests belong to different pressure patterns.

A cooperative build semaphore can improve throughput.
It cannot constrain a malicious shell that ignores it.
The outer runtime limits and host admission must remain sufficient on their own.

### 4.4 Budget disk growth

Reserve space for:

- Private workspaces, dependencies, and build outputs.
- Docker image layers and volumes inside each VM.
- Template candidates and retained versions.
- Model artifacts and package mirrors.
- Exported results, logs, and backups.

Choose a free-space floor before starting tasks.
Stop new admission while enough disk remains to export results and diagnose failure.
An example warning at 80 percent use is a notification threshold.
The admission floor should follow expected growth and recovery space.

Do not relocate opaque runtime state by moving directories while its daemon is running.
Use supported settings and a tested migration procedure.
A filesystem snapshot also needs an application consistency plan before it can count as recovery evidence.

### 4.5 Preserve access through a reboot

Confirm remote login, inference startup, runtime startup, and result storage after a real reboot.
Record which services are system services and which belong to the fleet account.
If a user service must survive logout, configure its lifecycle deliberately.

Keep a console or another tested recovery path before changing SSH, firewall, kernel, or driver configuration.
The useful outcome is a host you can recover from the MacBook after ordinary maintenance.

---

## 5. Create the first sandbox deliberately

The first sandbox should answer a small set of questions.
Can it start?
Does it have the expected checkout?
Can it run the baseline test?
Can it reach only the services you intended?
Can you recover its result?

Do this before asking it to fix a difficult bug.

### 5.1 Capture the installed interfaces

On the host:

```bash
sbx version
sbx create --help
sbx run --help
sbx exec --help
sbx ports --help
sbx template save --help
sbx settings ls --json
```

Save the useful output with the host configuration record.
Help text establishes supported syntax on that installation.
It does not establish that the implementation behaves as intended.

Complete Docker's host sign-in flow before provisioning:

```bash
sbx login
```

The checked installation guide uses browser OAuth.[^docker-install]
For a headless host, follow the installed CLI's supported login path and test renewal.
Runtime login is separate from the agent's model-provider authentication.

### 5.2 Configure the known host crossings

For a fresh host using this book's desired configuration:

```bash
sbx settings set ssh.agentForwardingEnabled false
sbx settings set skills.defaultMode off
sbx daemon restart
sbx policy init deny-all
```

The settings reference documents that disabling SSH forwarding needs a daemon restart and that existing skills mounts require recreation to change.[^docker-settings]
Apply this during setup, before active work exists.
On an established host, inspect effective settings and plan any restart as an interruption.

`deny-all` initializes a policy with no baseline allows.
Kits can still contribute per-sandbox rules.
Unmatched requests can require operator approval.
Inspect the effective policy after creation.[^docker-policy]

### 5.3 Create without launching the agent

Use a clean source repository at the reviewed base commit.
The first task name is `task-001`.

```bash
sbx create \
  --name task-001 \
  --clone \
  --skills off \
  --cpus 4 \
  --memory 8g \
  codex /srv/agent-sources/calendar-api

sbx policy ls task-001 --wide
sbx exec -it task-001 bash
```

The checked CLI distinguishes creation from attaching to an agent and supports a private clone.[^docker-create][^docker-exec]
Creating first gives you a place to inspect state before a model begins acting.

Inside the sandbox, locate the actual task clone.
Do not assume a workspace path from a different template.
Check `pwd`, Git status, the base commit, remotes, and tool versions.

The source directory is part of the trust boundary.
A dirty source, repository credential helper, or unexpected local configuration deserves investigation.
Use a dedicated clean source clone rather than your daily checkout.

### 5.4 Establish a baseline

For the TypeScript repository, the baseline might be:

```bash
# Inside the task clone, with the repository's pinned tools.
git status --short
git rev-parse HEAD
node --version
pnpm --version
pnpm install --frozen-lockfile
pnpm test -- --runInBand
```

These are repository-specific examples.
Use the real project's commands.
A different test framework may reject `--runInBand`.

Record the command, checkout, toolchain, and exit status.
If the baseline fails, classify the failure before delegating.
The agent should not quietly repair unrelated environment drift while implementing the date validation.

### 5.5 Run a small boundary test

Create a harmless host canary outside the shared source directory.
Try to read it from the sandbox.
Test an allowed destination and an operator-controlled denied destination.
Inspect whether an SSH agent socket is available.
Confirm that Docker inside the sandbox belongs to the private environment.

Keep these tests independent of the model.
They test the platform on which the model will act.

Do not use production credentials as probes.
A test credential with no valuable authority is enough to check a forwarding or masking mechanism.
Chapter 20 supplies the acceptance matrix.

---

## 6. Give each task an independent repository

The agent needs a place to make mistakes that belongs to its task.
Two agents writing the same working tree violate that requirement before either starts a build.

### 6.1 Use one private clone per task

This book uses one clone-mode sandbox for each task.
The agent creates its task branch inside that clone.
The host source stays a clean, read-only input.

Docker's Git guide explains that clone mode creates a separate repository and exposes its commits through a host `sandbox-<name>` remote.
It also copies applicable upstream remotes into the clone.[^docker-git]

Clone mode gives workspace separation.
Remote push restrictions require separate credential and network decisions.
An agent with a write token and an allowed Git endpoint can still push.

### 6.2 Keep the host source stable

Prepare a source clone under a dedicated directory.
Record its full commit ID and reject uncommitted content for normal fleet tasks.
While active sandboxes use that source, avoid branch switching, destructive cleanup, or removing Git objects they may need.

A source service can maintain separate directories for different baselines.
A task record selects one explicitly.
Start with that simple approach before attempting shared-object optimizations.

An example input identity:

```yaml
# Proposed repository source record.
repository_id: calendar-api
source_path: /srv/agent-sources/calendar-api
base_commit: "<full-commit-id>"
working_tree: clean
submodules: recorded-and-pinned
large_file_objects: locally-available-or-explicitly-fetched
remote_write_credentials: absent
```

Submodules and Git LFS need their own preparation.
A successful clone does not mean those external objects are ready.
Include them in the baseline.

### 6.3 Understand the worktree alternative

Host worktrees are useful when you want live edits and are willing to manage Git on the host.
The checked Docker guide says a sandbox mounting only a linked worktree cannot resolve its external Git directory and consequently lacks Git access.[^docker-git]

Do not solve that limitation by mounting your entire repository administration directory read-write without reviewing the consequences.
Worktrees share repository metadata and object storage.
They are not independent security boundaries.

For this fleet, private clones simplify ownership and export.
A developer collaborating interactively on one task may choose a direct mount instead.
Record the difference so that cleanup and review use the right assumptions.

### 6.4 Keep generated services task-local

Private files do not automatically create private external databases.
Give the task its own database, fixtures, and service configuration.

When using Compose inside the VM:

```yaml
# Illustrative project service, inside one sandbox's private Docker Engine.
services:
  db:
    image: postgres:16
    environment:
      POSTGRES_DB: taskdb
      POSTGRES_USER: taskuser
      POSTGRES_PASSWORD: development-only
    volumes:
      - task-db:/var/lib/postgresql/data

volumes:
  task-db:
```

Use a reviewed image digest in a repeatable environment.
The password is disposable test configuration.
It grants no host or production authority.

Avoid test URLs that point to shared development databases.
An agent that drops a table should lose only its own fixture.

### 6.5 Export more than committed changes

An interrupted task may leave valuable edits uncommitted.
An exported branch preserves committed history.
A binary-capable diff and an inventory of untracked files preserve the remaining work.

Record ignored outputs separately when they are relevant.
Do not copy an entire workspace just because a transcript says the task is finished.
Results need a deliberate scope, size limit, and provenance.

The export process returns in chapter 16.

---

## 7. Make network access an explicit dependency

A build that needs a registry should have access to that registry.
It should not acquire access to your NAS because both destinations happen to use HTTP.

Network policy belongs to the environment contract.
Discover dependencies during preparation, review them, and make the normal task run predictable.

### 7.1 Start with a small dependency inventory

For the example service:

| Dependency | Required operation | Proposed access |
|---|---|---|
| npm registry or private mirror | Retrieve packages | Selected registry hosts |
| Git source service | Retrieve reviewed source | Prepared input or read-only service |
| Container registry | Pull test images | Selected registry and authentication hosts |
| Inference gateway | Generate model responses | One host service port |
| Notification receiver | Submit bounded task events | One event endpoint |

Registry authentication and redirects can introduce additional hosts.
Capture the actual request chain during baseline preparation.
Do not guess the full dependency graph from the top-level URL.

A normal coding task should not be the first time the operator sees those destinations.
Otherwise every dependency install becomes an approval conversation.

### 7.2 Review kit rules as well as local rules

The local preset is only part of the effective policy.
Kits can add rules.
Organization governance can change which local rules have effect.

Inspect the sandbox context:

```bash
sbx policy ls task-001 --wide
sbx policy ls task-001 --source kit --type network --wide
sbx policy check network --sandbox task-001 registry.npmjs.org
```

Docker documents host-level policy checks separately from HTTP method and path evaluation.
A host check can pass while a specific request remains blocked.[^docker-policy]
Test the real package operation too.

Record rule origin, destination, and reason.
An allow rule without an owner tends to become permanent by accident.

### 7.3 Allow one host service

In this architecture, the task reaches a gateway on host port 8000.
The backend model server stays on a different port.

```bash
sbx policy allow network --sandbox task-001 localhost:8000
```

The sandbox uses:

```text
http://host.docker.internal:8000/v1
```

Docker's host-service guide documents the proxy translation from `host.docker.internal` to `localhost`.
Policy matches the forwarded destination.[^docker-host-model]

That rule permits a route.
It does not authenticate the task or validate the API.
The gateway must still enforce its contract.

Do not grant all of localhost to make one endpoint work.
The host also contains administrative services and unrelated application ports.

### 7.4 Test network attempts by their outcome

The research described non-HTTP TCP as universally dropped.
The checked policy guide documents permitted TCP proxying, including SSH when explicitly allowed.[^docker-policy]
For this architecture, deny unnecessary TCP destinations and test them.

Use an operator-controlled sink with a harmless canary.
Test the domain name and direct IP forms.
Test IPv4 and IPv6 when enabled.
Test a sibling sandbox's address and a host service outside the exception.

The expected evidence is that the sink received nothing and the platform recorded the denial.
An agent saying “connection failed” is incomplete evidence.
It may have failed for DNS, authentication, or an unrelated timeout.

### 7.5 Diagnose policy failures before changing policy

When an install fails, collect the denied request and package operation.
Check whether the destination belongs to the reviewed dependency chain.
If it does, add the narrow rule and retry the same operation.
If it does not, investigate the dependency or instruction that introduced it.

Avoid changing the whole preset to open access for one failing install.
That erases the boundary while hiding the original reason.

Allowed services remain possible data exit channels.
A repository can encode content in an allowed request.
Reduce reachable secrets and limit service authority even when destination policy works.

---

## 8. Keep credentials and host integrations narrow

A credential can stay outside the VM while still giving the VM useful authority.
That is how a credential proxy works.
It is also why masking a token is only part of the decision.

Ask two questions:

1. Can the task read the raw credential?
2. What can the task cause the credential to do?

The second question determines most of the consequences.

### 8.1 Scope service credentials

Docker documents host-side authentication header injection, sandbox-scoped secrets, and a global fallback.
OAuth passthrough can deliver real token responses inside the sandbox.[^docker-credentials]

For a hosted-provider task, an illustrative setup is:

```bash
sbx secret set openai --sandbox task-001
```

This interactive command avoids placing the key in the manuscript or shell arguments.
Use the relevant service identifier and required account flow for your chosen agent.

Do not assume removing a sandbox-specific secret removes access.
Docker documents that a global secret becomes effective again if present.[^docker-credentials]
Inspect the complete credential configuration during revocation.

For local inference, create a task-scoped gateway capability.
It should authorize only the assigned model profile and budget.
It should not be a GPU server administrator key.

### 8.2 Keep Git writes out of ordinary tasks

The host supplies source.
The task produces local changes.
The reviewer publishes accepted work using a separate host identity.

A package-read token may still be necessary.
Choose an identity with the minimum required read authority.
Confirm that another repository and remote write operations fail using disposable test resources.

Do not import the operator's broad GitHub login automatically.
A host credential resolver that returns that token preserves secrecy while delegating its permissions.
The task does not need to see the token to exercise them.

### 8.3 Disable SSH forwarding explicitly

A forwarded SSH agent lets sandbox processes request signatures.
The key file can stay on the host while authentication remains available to generated commands.

This book disables forwarding through the host settings and avoids forwarding the MacBook's agent into the fleet account.
Confirm the effective state after daemon restart.
Repeat the probe after runtime upgrades.

If one task needs SSH authentication, give it a separate capability and destination.
Document whether the credential permits source reads, remote shell access, signing, or writes.
Those are different permissions.

### 8.4 Treat shared skills as executable inputs

A shared instruction or script can influence every participating agent.
A read-only consumer can still consume a file changed by another producer.

For independent task sandboxes, use `--skills off`.
Bake reviewed task instructions into the template or distribute a versioned read-only artifact from outside the task fleet.

Do not let one active task publish global instructions for the next.
A useful improvement to an agent skill belongs in a reviewed change to the shared configuration.

### 8.5 Review MCP and host commands separately

Docker documents local stdio MCP servers as host processes outside the VM.[^docker-security]
A filesystem tool there may reach files the VM cannot.
A shell tool there may execute with host permissions.

Start with no host MCP tools for ordinary coding tasks.
When adding one, document its implementation, credential, operation set, and expected caller.
Test its own boundary.

The same principle applies to host lifecycle commands supplied by environment configuration.
Docker's settings guide explicitly identifies these as commands running with host permissions.[^docker-settings]
Store approved configuration outside writable task mounts.

### 8.6 Protect the host credential store

On a desktop, check that the intended keyring is available to the runtime account.
On a headless installation, determine which backing store is active.
Docker documents a file fallback protected by permissions when Linux Secret Service is unavailable.[^docker-credentials]

Backups of that directory require the same protection as the credentials themselves.
A transcript archive should not include it.
A template bake should never start from the operator's home directory.

---

## 9. Share inference without sharing the GPU

The GPU is one shared service resource.
Give it an owner.
The owner loads the model, manages memory, and exposes the inference API.
Coding tasks consume that API.

### 9.1 Separate gateway and backend

Use two loopback listeners:

```text
sandbox
  |
  | authorized request through sandbox egress proxy
  v
host 127.0.0.1:8000: inference gateway
  |
  | backend credential, selected model, bounded request
  v
host 127.0.0.1:8001: model server
  |
  v
GPU
```

The sandbox policy allows port 8000.
It does not allow port 8001.
The gateway strips client-supplied identity headers and derives task identity from authenticated authority.
The backend key never enters the task.

This is a proposed service design.
You must implement or adopt the gateway.
A reverse proxy that forwards every path is insufficient.

### 9.2 Define the gateway contract

```yaml
# Proposed inference-gateway contract.
listener: 127.0.0.1:8000
backend: http://127.0.0.1:8001

authentication:
  identity: issued-per-task
  expires_with: task-lease
  caller_identity_headers: discard

authorization:
  models: assigned-profile-only
  endpoints: agent-compatibility-allowlist
  server_administration: denied

limits:
  request_body_bytes: bounded
  input_tokens: profile-specific
  output_tokens: profile-specific
  concurrent_requests_per_task: 1
  total_active_requests: measured
  queued_requests: bounded
  deadline: bounded

failure:
  revoked_identity: reject
  unavailable_backend: explicit-service-error
  queue_full: explicit-capacity-error
  client_disconnect: cancel-backend-when-supported
```

The one-request task limit is an initial choice.
Some agents issue parallel requests.
Measure whether their compatibility requires more, then change the allocation deliberately.

Enforce input token limits with the selected model's tokenizer or another tested bound.
HTTP body size alone does not establish context size.
Include tool schemas and retained conversation history.

### 9.3 Serve one pinned model

vLLM documents an HTTP serving interface and API-key authentication.
Chat behavior depends on the model's chat template, and supported API shapes vary by version.[^vllm-server]

A command sketch on the host:

```bash
# Sketch: install and pin vLLM, select a tested model revision,
# supply authentication securely, and add model-specific tool settings.
/opt/inference/venv/bin/vllm serve /srv/models/coding-model \
  --host 127.0.0.1 \
  --port 8001 \
  --served-model-name local-coding \
  --max-model-len 32768
```

The sketch omits credential loading intentionally.
Use the installed server's supported secure input mechanism.
Do not paste a long-lived key into a command line merely because a quick-start example does.

The context limit is an example to measure.
Model weights fitting in VRAM does not prove that the requested concurrent contexts fit too.

vLLM's security guide identifies endpoints that its API-key control does not protect.[^vllm-security]
Keep the gateway allowlist in front of the server.
Authentication on a generation route is not authentication on every route the process serves.

### 9.4 Keep service supervision distinct from readiness

Run the selected backend under systemd or another host supervisor.
Use a dedicated service identity and reviewed executable.
Keep model files read-only to the running service where practical.
Set restart rate limits.

A process being alive does not make the model ready.
Run a small authenticated inference probe with a deadline.
Make readiness fail while loading, after GPU errors, or when the compatibility probe fails.

If using a systemd credential, the loader must actually read it and configure the server.
`LoadCredential=` does not magically map a file into an arbitrary application's API-key option.
Treat that loader as a small piece of trusted host code.

### 9.5 Keep inference free of host tools

The checked llama.cpp server documentation includes optional server-side filesystem tools, shell tools, and MCP child processes.[^llama-server]
Those features are outside this book's inference contract.

Do not enable them on the shared host model server.
The model service returns model output.
The agent's tool runner executes tools inside its task sandbox.

Also constrain administrative paths, model loading, local media paths, and unrelated UI endpoints at the gateway.
A process on loopback is still reachable if you deliberately grant its port to sandboxes.

### 9.6 Respect the shared confidentiality boundary

The host inference service sees prompts from every task.
Its logs, debugging captures, and caches belong to the shared infrastructure.
Limit access and retention accordingly.

Do not mix repositories with incompatible data policies merely because they fit on one GPU.
For this personal fleet, all tasks belong to one authorized operator.
That assumption should remain visible in the architecture.

---

## 10. Prove the agent and model work together

An endpoint that answers “hello” has passed a transport test.
A coding agent needs a longer contract.
It needs to ask for tools, consume their results, recover from errors, and continue without losing the objective.

### 10.1 Check the whole compatibility tuple

Record:

```yaml
# Proposed compatibility record.
agent:
  name: codex
  revision: "<installed-version>"
runtime:
  revision: "<installed-sbx-version>"
model:
  identity: "<artifact-and-revision>"
  quantization: "<chosen-format>"
server:
  implementation: "<vllm-or-llama.cpp>"
  revision: "<installed-version>"
  chat_template: "<identity>"
  tool_parser: "<identity-or-not-applicable>"
transport:
  protocol: "<tested-agent-wire-format>"
  streaming: tested
limits:
  context_tokens: "<tested-limit>"
  output_tokens: "<tested-limit>"
```

“OpenAI-compatible” or “Anthropic-compatible” is an interface claim.
It does not establish every request shape your installed agent uses.

### 10.2 Choose a connection path

There are two paths worth evaluating:

| Path | Advantage | Additional responsibility |
|---|---|---|
| Agent-native endpoint configuration | Direct mapping to your gateway | Agent-specific protocol and credential setup |
| SBX model-provider integration | Runtime supplies model selection and adaptation | Experimental feature behavior and bundled model service |

Docker documents experimental model-provider selection for supported built-in agents, custom endpoints, and a host `llmman` service.[^docker-models]
Test it as an integration choice.
Do not assume every kit or cloud workflow supports the same flags.

When using your own backend, ensure the bundled service does not also load a second local model and consume the same GPU.
Inspect actual processes and device memory after launch.

Docker also documents a broader credential-routing caveat for model-provider keys through `llmman`.[^docker-models]
Use a gateway key with narrow task authority.
Do not supply a broad hosted-provider key to a path whose effective routing you have not reviewed.

For the direct Codex path, configure the agent's sandbox-local `~/.codex/config.toml`:

```toml
# Merge into the sandbox's agent configuration before starting a task.
model = "local-coding"
model_provider = "fleet-local"

[model_providers.fleet-local]
name = "Fleet inference gateway"
base_url = "http://host.docker.internal:8000/v1"
env_key = "FLEET_INFERENCE_KEY"
wire_api = "responses"
request_max_retries = 2
stream_max_retries = 2
```

OpenAI documents custom provider selection, endpoint configuration, environment-supplied authentication, and the Responses wire format.[^codex-provider]
The gateway and backend must support that format's actual tool loop.
A Chat Completions endpoint alone does not satisfy this example.

The host controller issues a short-lived `FLEET_INFERENCE_KEY` for this task and makes it available to the launch process.
Then, inside the task's host tmux session:

```bash
# Host environment contains only this task's issued gateway capability.
sbx run --name task-001 \
  -e FLEET_INFERENCE_KEY \
  -- --sandbox workspace-write --ask-for-approval on-request
```

This task capability enters the VM deliberately.
It expires with the lease and grants no backend administration.
Do not substitute the backend key.
Exclude the capability from template bakes, exports, and logs.

The attended example retains inner approvals.
Choose unattended policy only after the compatibility and boundary tests.
Keep runtime model selection and agent-native provider configuration as distinct tested profiles instead of layering both accidentally.

### 10.3 Test a complete tool loop

Use a disposable repository with a deterministic task:

1. Read the implementation and test.
2. Run the failing test.
3. Edit the implementation.
4. Run the same test successfully.
5. Report the change and its verification.

Capture the protocol trace in a private test environment.
Confirm that tool arguments are parseable, tool results return under the expected identifiers, and streaming ends cleanly.

Test a tool failure too.
The agent should handle a missing file or nonzero test exit.
A model that emits ordinary prose where a tool call belongs cannot finish a coding loop reliably.

### 10.4 Evaluate useful tasks

After compatibility passes, run a small evaluation set drawn from your own repositories:

- A bounded bug fix with a regression test.
- A refactor with unchanged behavior.
- A change spanning two related files.
- A task with an intentionally missing requirement.
- A task that cannot be completed with the granted access.

Measure accepted results, unsupported claims, and recovery behavior.
Keep task verification outside the model's self-report.

A local model may fit routine changes while a hosted model handles a more demanding class.
Make that routing a reviewed task decision.
Automatic external fallback changes data movement and cost.

### 10.5 Configure agent autonomy inside the chosen boundary

OpenAI documents sandbox modes and approval policy as separate controls.
For an attended Codex session:

```bash
# Inside the sandbox, from the task repository.
codex --sandbox workspace-write --ask-for-approval on-request
```

For a deliberately unattended task, choose whether denials should fail or whether the established outer VM supplies execution isolation.
The documented `never` approval policy does not itself remove sandbox restrictions.[^codex-security]

If inner sandboxing fails, diagnose the nesting failure.
OpenAI documents using `danger-full-access` inside a container whose outer boundary supplies the required isolation.[^codex-security]
That choice also increases the importance of reviewing every shared integration.

Do not grant host authority to make an inner tool call work.
The VM's disposable execution environment is the place for generated commands.

### 10.6 Make context a budget

A task brief, repository instructions, tool definitions, and conversation history all spend context.
Provide the relevant files and tests first.
Avoid dumping the entire repository into the initial prompt.

Define what happens near the context limit.
The agent can summarize progress into a task-local checkpoint, export edits, and resume from a bounded new session.
Test the selected agent's actual recovery behavior.

If the server rejects an oversized request, preserve that error.
Silent truncation can remove the task's acceptance criteria while leaving the conversation apparently healthy.

---

## 11. Warm toolchains and dependencies

Fast environments begin with the work the agent needs.
A VM that starts quickly and spends ten minutes downloading dependencies has not made the task ready quickly.

### 11.1 Separate four kinds of state

| State | Sharing rule in this design |
|---|---|
| Toolchain binaries | Reviewed template inputs |
| Downloaded package artifacts | Reviewed template cache or read-only mirror |
| Build outputs and runtime databases | Private to one task |
| Credentials and session identity | Issued after allocation |

This separation keeps performance work from turning into cross-task authority.

For TypeScript, pin Node and the package manager.
For Java, pin the JDK and Gradle wrapper inputs.
A template should include the tools required to verify its repositories.

### 11.2 Keep writable caches task-local

An active task can modify its own package cache.
It should not be able to poison the next task's trusted input.

Start with private caches.
Warm them through a reviewed bake.
If download traffic becomes expensive, add a host artifact mirror with restricted publication rights.

A mirror should not accept arbitrary uploads from every sandbox.
A build cache that accepts task-produced outputs needs its own validation and trust model.
Content addressing identifies bytes.
It does not establish who was entitled to publish them.

### 11.3 Respect package-manager requirements

Some tools need writable metadata even when package contents are reusable.
A read-only mount may cause cache misses or failures.
Do not declare a shared cache correct just because the directory mounts successfully.

Test install, test, rebuild, and cleanup with the selected tool versions.
Confirm that private writable outputs remain private.
Avoid sharing `node_modules`, Gradle project caches, database files, or compiler outputs across active tasks by default.

Treat package installation scripts as code execution.
Run them inside the bake or task boundary with the intended access.
The host mirror serves artifacts.
It does not execute package hooks.

### 11.4 Use Git mirrors without creating hidden lifetimes

Host mirrors can reduce repeated source downloads.
Refresh them under a host-controlled identity.
Export reviewed baselines to stable source clones.

Git documents that reference and shared clones can depend on objects retained by another repository.
`--dissociate` breaks that borrowing relationship by making local copies.[^git-clone]

For an independent source clone:

```bash
# Host preparation example, outside the active task.
git clone \
  --reference-if-able /srv/git/calendar-api.git \
  --dissociate \
  https://git.example.com/team/calendar-api.git \
  /srv/agent-sources/calendar-api-next
```

This illustrates the Git optimization.
It is not a promise that SBX internally uses the same mechanism.
Measure the supported clone path before optimizing around undocumented storage behavior.

### 11.5 Start with stopped environments

A pre-created stopped environment avoids some provisioning work.
It still needs current identity, policy, task source, and readiness verification when claimed.

Begin with a small pool for one resource class and repository baseline.
Do not maintain ten pools before knowing which tasks recur.
Count stopped environments in disk admission.

A previously used environment is not a clean pool member.
Unless you can demonstrate a complete reset, retire it.
Reuse reviewed templates.
Do not reuse task residue.

### 11.6 Measure warm readiness

Use three measurements:

```text
runtime ready: start request -> environment running
repository ready: claim -> pinned baseline test passing
task complete: dispatch -> preserved reviewable result
```

Compare a cold template, a warm template, and a stopped pool member.
Measure at one, five, and ten tasks.
A warm optimization that saturates the disk can improve one-task latency while harming total throughput.

---

## 12. Publish templates you can explain

A template is a repeated starting condition.
That makes every file in it a repeated decision.

Bake it from reviewed inputs.
Keep the recipe and the resulting artifact identity.
Promote it only after a fresh environment passes the baseline and boundary checks.

### 12.1 Prefer a clean bake

Use a separate baker environment.
Install pinned toolchains and reviewed dependencies.
Run the baseline.
Stop background writers before capture.
Remove temporary authentication and task-specific files.

Do not save a template from an ordinary completed task.
That environment may contain prompts, modified instructions, tokens, database state, or an agent's experimental configuration.

The artifact can be reproducible in its declared inputs without being byte-identical.
State which property you have verified.
The companion workstation book develops that distinction in more detail.

### 12.2 Use the documented local capture interface

The checked CLI supports:

```bash
sbx template save \
  baker-typescript \
  fleet-typescript:2026-10-14 \
  --output /srv/agent-templates/fleet-typescript-2026-10-14.tar
```

Then use the saved template locally:

```bash
sbx create \
  --name task-002 \
  --clone \
  --skills off \
  --cpus 4 \
  --memory 8g \
  --pull never \
  --template fleet-typescript:2026-10-14 \
  codex /srv/agent-sources/calendar-api
```

Docker documents the local saved-image workflow and distinguishes it from cloud memory-and-disk capture.[^docker-template]
Do not import cloud checkpoint guarantees into a local workstation design.

### 12.3 Verify what capture includes

Prepare harmless sentinels in the locations that matter:

- Toolchain installation directory.
- User download cache.
- Task clone.
- Docker image store inside the VM.
- Kit-owned persistent storage.

Save the candidate.
Create a fresh environment.
Inspect which sentinels and baseline inputs survived.

The experiment establishes your installed capture behavior.
It does not make unknown locations safe.
Do not put real secrets in the baker to test whether they persist.

If a cache or workspace volume is excluded, arrange its preparation separately.
A saved image does not automatically imply a complete warm repository.

### 12.4 Publish an artifact record

```yaml
# Proposed template publication record.
template_id: fleet-typescript-20261014
local_tag: fleet-typescript:2026-10-14
artifact_sha256: "<digest-of-export>"
recipe_commit: "<reviewed-recipe-commit>"
sbx_version: "<checked-version>"
agent_version: "<checked-version>"

inputs:
  base_image: "<pinned-reference>"
  node: "<pinned-version>"
  pnpm: "<pinned-version>"
  baseline_commit: "<full-commit-id>"
  dependency_lock_digest: "<digest>"

evidence:
  baseline_test: passed
  fresh_instance: passed
  credential_canary_absent: passed
  boundary_probes: passed
  capture_inventory: recorded

promotion: candidate
```

A digest identifies the exported artifact.
The evidence explains what it was checked to do.
Neither replaces the recipe.

### 12.5 Promote and retain deliberately

Keep candidate, active, and previous-good versions.
Change the active selection only after acceptance checks.
Existing tasks retain their recorded template identity.

Retire a template when no active task or recovery requirement references it.
Record revocation if a template contains a compromised dependency.
Stopping future allocation is a separate action from interrupting existing tasks.

Rebuild because an input changed or an update policy requires it.
A nightly rebuild that silently selects new upstream versions makes yesterday's environment harder to explain.
Keep security updates timely through reviewed promotion, not through mutable task inputs.

---

## 13. Manage sessions from the MacBook

Remote administration should continue to work when the agent does not.
Keep the host shell, task conversation, and preview connection independent.

### 13.1 Choose how SSH authenticates

Tailscale SSH uses tailnet identity and its own SSH authorization rules.
The network policy and SSH policy must both permit the connection.
Ordinary OpenSSH over a Tailscale address remains another option.[^tailscale-ssh]

For the example, use Tailscale SSH to the dedicated `agentfleet` account.
On the already enrolled Ubuntu host:

```bash
sudo tailscale set --ssh
```

Make this change with a recovery connection available.
Tailscale documents that enabling its SSH server interrupts existing SSH connections to the Tailscale address.[^tailscale-ssh]

The tailnet policy should allow the operator to reach the host on port 22 and log in as `agentfleet`.
Prefer that named account over permission to log in as every non-root user.
Keep an additional recovery route if tailnet authentication becomes unavailable.

### 13.2 Keep the Mac configuration small

```sshconfig
# On the MacBook.
Host agentdesk
    HostName desk
    User agentfleet
    ForwardAgent no
    ServerAliveInterval 30
    ServerAliveCountMax 3
```

Replace `desk` with the tested tailnet host name.
This configuration avoids forwarding the Mac's agent into the fleet account.

Connect:

```bash
ssh agentdesk
```

Test the connection after a laptop network change and after a host reboot.
Remote operation needs both cases.

### 13.3 Keep interactive sessions on the host

tmux separates a running terminal session from the connection used to view it.[^tmux]
For the first attended task, create a host session:

```bash
# On the Ubuntu host.
tmux new-session -s task-001
```

Inside that session:

```bash
sbx run --name task-001
```

Detach with `Ctrl-b`, then `d`.
After reconnecting:

```bash
tmux attach-session -t task-001
```

This is deliberately a two-step sequence.
It keeps task names out of a constructed shell command.
The controller can automate the same behavior using argument arrays.

tmux preserves the interactive terminal across connection loss.
It does not supply task durability across host loss or enforce a lease.
The task ledger and exports serve those needs.

### 13.4 Inspect before launching again

Docker documents `sbx run --name` as the path to an existing sandbox.[^docker-run]
Agent session continuation still depends on the agent.

When you lose the terminal, inspect runtime state and the host tmux session before starting another agent.
If the original conversation is alive, reattach there.
If it exited, identify the saved session and checkout before resuming.

A second agent in the same clone can produce conflicting edits.
An idempotent dispatch record should reject duplicate launch attempts.
A session name alone cannot establish whether work already started.

### 13.5 Use IDE access for review

Remote SSH in an editor can make host source and exports convenient to inspect.
It also executes editor integrations in the remote environment.
Treat workspace tasks and extensions as execution when they consume returned agent code.

For returned changes, use a fresh verification sandbox.
Keep the host review checkout free of automatic build or dependency commands until the change is inspected.

The browser preview is another review channel.
It should not become an administration path into the fleet account.

### 13.6 Add notifications after the state exists

The host controller should notify on durable events:

- Input required.
- Result preserved and ready to inspect.
- Task failed with evidence.
- Host admission stopped.

The event contains a task ID and a bounded reason.
It should not include raw source, full prompts, or credentials.

A sandbox hook can report an observation to a narrow receiver.
It cannot certify that its result is correct.
Rate-limit events and authenticate the sender.
The controller decides which observation changes task status.

A notification failure should not erase the result.
Deliver from an outbox that can retry after the controller restarts.
The task ledger remains the place to check when the phone is quiet.

---

## 14. Publish previews with a clear lifetime

Every task can run its internal service on port 3000.
The host needs a different route for each preview.
The route also needs an owner and an expiry.

### 14.1 Test the development listener

Start the repository's development server inside the sandbox.
Inspect its bind address and confirm that the sandbox's publishing path can reach it.

A representative Vite command is:

```bash
# Inside the task clone; use the project's actual dev command.
pnpm dev --host 0.0.0.0 --port 3000
```

Binding to all interfaces here means the task's sandbox environment.
It does not justify binding the host listener to all interfaces.

If the connection fails, inspect each hop.
A server on guest loopback, a wrong port mapping, and a denied tailnet connection produce different fixes.
Test the installed template rather than treating `0.0.0.0` as a universal runtime requirement.

### 14.2 Publish explicitly to host loopback

For one attended preview:

```bash
sbx ports task-001 --publish 127.0.0.1:18001:3000/tcp4
sbx ports task-001 --json
```

The checked CLI documents host bindings, ephemeral allocation, address families, and JSON output.[^docker-ports]
Use explicit IPv4 loopback for the initial preview path.

For a fleet, reserve the host port through the controller.
Do not assign it by arithmetic on an untrusted task name.
Detect conflicts with unrelated host listeners.
Record the actual mapping returned by the runtime.

Ephemeral ports avoid choosing an occupied port up front.
They still require a durable routing record and restart reconciliation.
Chapter 20 identifies the lifecycle behavior to test.

### 14.3 Use an SSH tunnel for the simplest view

From the MacBook:

```bash
ssh -N \
  -o ExitOnForwardFailure=yes \
  -L 127.0.0.1:3000:127.0.0.1:18001 \
  agentdesk
```

Open `http://127.0.0.1:3000` on the Mac.
Choose a different local port if the Mac already uses that one.

This route lasts with the SSH connection.
It is a good first preview because it adds no persistent host route.
Verify forwarding with the selected SSH server and client.

### 14.4 Add Tailscale Serve for a persistent route

On the Ubuntu host:

```bash
sudo tailscale serve \
  --bg \
  --https=8443 \
  http://127.0.0.1:18001
```

Use the URL returned by the command.
Serve routes to a local service within the tailnet, and tailnet access rules still apply.
HTTPS certificate setup is required.[^tailscale-serve]

The background route persists across reboot until disabled.[^tailscale-serve-cli]
That makes it convenient and makes cleanup part of the task lifecycle.

Record:

```yaml
# Proposed preview record.
task_id: task-20261014-001
sandbox_name: task-001
sandbox_port: 3000
host_binding: 127.0.0.1:18001
serve_https_port: 8443
generation: 1
expires_at: "<task-lease-expiry>"
audience: operator-devices
```

Avoid Funnel for ordinary task previews.
It publishes to the internet and changes the audience of the task's output.[^tailscale-serve]

### 14.5 Treat development pages as generated output

The task controls the page, scripts, redirects, and responses you view.
Use disposable test accounts and synthetic data.
Keep production browser sessions out of the preview workflow.

Different ports give distinct origins, but cookies are not generally scoped by port.
Ten applications on one host name can overwrite each other's cookie names.
Use unique cookie names or configure distinct tested host names when session isolation matters.

A path such as `/task-001/` also requires application support for asset paths, redirects, and WebSockets.
A reverse proxy cannot infer every frontend's base-path assumptions.
Start with a port route unless the project already supports path deployment.

### 14.6 Remove the route before freeing the port

Disable the example Serve listener:

```bash
sudo tailscale serve --bg --https=8443 off
sbx ports task-001 --unpublish 127.0.0.1:18001:3000/tcp4
```

The Serve CLI documents removing an individual listener with the original flags and `off`.[^tailscale-serve-cli]
Avoid resetting all Serve configuration to remove one task.

Revoke the route, verify it is unavailable, then release its host port.
A stale route aimed at a reused port can show another task or an unrelated host service.
Runtime deletion alone does not remove a separately configured Serve route.

---

## 15. Build a small controller that can recover

The research's shell wrapper is a useful list of operator actions.
A reliable controller needs more than a switch statement.

Creating a sandbox, launching an agent, registering a preview, and exporting a result can each fail independently.
The controller must know which steps happened when its own process disappears.

### 15.1 Keep the interface small

```text
# Proposed interface; agentctl is not supplied by Docker.
agentctl submit task.yaml
agentctl list
agentctl inspect TASK_ID
agentctl attach TASK_ID
agentctl preview TASK_ID
agentctl cancel TASK_ID
agentctl export TASK_ID
agentctl retire TASK_ID
```

`cancel` ends execution authority.
`retire` removes the preserved task environment after export or an explicit decision to discard it.
Keeping them separate makes recovery easier to explain.

Implement only the operations the operator uses.
A web dashboard can come later if the command output stops being sufficient.
You do not need a scheduler cluster to operate one workstation.

### 15.2 Record lifecycle intent durably

A small SQLite ledger is enough for the first controller.
Use one writer or another explicit serialization strategy for lifecycle mutations.
Store each transition before performing its external side effects.

```sql
-- Schema sketch: application code validates transitions and transactions.
CREATE TABLE tasks (
    task_id TEXT PRIMARY KEY,
    request_id TEXT NOT NULL UNIQUE,
    spec_digest TEXT NOT NULL,
    repository_id TEXT NOT NULL,
    base_commit TEXT NOT NULL,
    template_id TEXT NOT NULL,
    resource_class TEXT NOT NULL,
    sandbox_name TEXT NOT NULL UNIQUE,
    phase TEXT NOT NULL CHECK (phase IN (
        'queued', 'allocating', 'ready', 'running',
        'input_required', 'exporting', 'reviewable',
        'failed', 'cancelled', 'retiring', 'retired'
    )),
    generation INTEGER NOT NULL CHECK (generation > 0),
    lease_expires_at TEXT NOT NULL,
    export_digest TEXT,
    updated_at TEXT NOT NULL
);

CREATE TABLE task_events (
    event_id INTEGER PRIMARY KEY,
    task_id TEXT NOT NULL REFERENCES tasks(task_id),
    generation INTEGER NOT NULL,
    kind TEXT NOT NULL,
    detail_json TEXT NOT NULL,
    observed_at TEXT NOT NULL
);

CREATE TABLE reservations (
    task_id TEXT PRIMARY KEY REFERENCES tasks(task_id),
    memory_mib INTEGER NOT NULL CHECK (memory_mib > 0),
    disk_growth_mib INTEGER NOT NULL CHECK (disk_growth_mib > 0),
    build_slots INTEGER NOT NULL CHECK (build_slots > 0)
);
```

Enable foreign-key enforcement on each connection.
The schema rejects some invalid records.
It does not enforce transition order, clock behavior, or atomic resource admission by itself.

Treat a reused `request_id` with a different specification digest as an error.
Retries should return the existing task.
They should not overwrite its objective or allocate another environment.

### 15.3 Make allocation recoverable

Use a deterministic runtime name from the task record.
Reserve resources in the same transaction that moves the task to `allocating`.
Then create the environment.

If the controller dies after creation but before recording success, reconciliation finds the known runtime name.
It inspects the environment and finishes preparation or quarantines the mismatch.
It does not blindly create a second sandbox.

Runtime CLI output is an observation.
Persist the environment identity and checked configuration.
Reject an existing name whose repository, resource class, or template does not match the intended task.

### 15.4 Enforce a host-owned lease

The ledger sets a deadline.
The controller renews it only through an authorized host request.
The task cannot extend its own lease by rewriting a workspace file.

Lease expiry should:

1. Reject new inference requests for that task.
2. Revoke notification and preview authority.
3. Stop task execution.
4. Attempt bounded result preservation.
5. Retain the stopped environment or complete retirement according to policy.

The local `sbx` examples do not rely on cloud-only TTL flags.[^docker-run]
Implement local deadlines through the controller and gateway.

For an expired lease, avoid restarting the whole environment with all capabilities restored just to fetch a Git branch.
Chapter 16 distinguishes ordinary quiesced export from recovery export.

### 15.5 Reconcile expected and observed state

Run reconciliation at startup and periodically:

```text
ledger says running, runtime absent
  -> preserve known evidence; mark failure; release verified resources

runtime exists, known allocating task
  -> inspect; complete preparation or quarantine

runtime exists, no ownership record
  -> report orphan; do not delete an unrelated user's environment

preview exists, task expired
  -> remove owned route; verify removal

reservation exists, task retired
  -> confirm runtime absence; release reservation
```

Quarantine means keeping execution and authority disabled while preserving diagnostic state.
It should have a retention limit and an operator-visible reason.

Compare durable task generations on callbacks.
A late notification from generation 1 must not finish a replacement task at generation 2.

### 15.6 Keep subprocess calls structured

Use argument arrays:

```python
# Sketch: the task record was already validated and authorized.
import subprocess

args = [
    "sbx", "create",
    "--name", task.sandbox_name,
    "--clone",
    "--skills", "off",
    "--cpus", str(task.cpus),
    "--memory", task.memory,
    "--pull", "never",
    "--template", task.template_tag,
    task.agent,
    task.source_path,
]

result = subprocess.run(
    args,
    check=False,
    capture_output=True,
    text=True,
    timeout=300,
)
```

Validate the agent, template, source path, and resource class against host-owned records.
Do not accept arbitrary extra flags from a task prompt.
Apply output bounds and redaction before storing subprocess logs.

A timeout means the command did not return in time.
It does not prove the runtime side effect failed.
Reconcile by the recorded environment name.

---

## 16. Collect results before cleanup

The most expensive task failure is losing a useful result after the work already happened.
Make preservation a normal phase.
Use the same path for passing, failing, and cancelled work.

### 16.1 Freeze a consistent view

For normal completion, end the agent conversation and stop repository writers.
Keep the environment available for the host export path.
Revoke model authority so the agent cannot resume through a stray client.

Docker documents that clone mode's Git daemon is available only while the sandbox runs.
Stopping the sandbox makes its host remote unavailable until it starts again.[^docker-git]
Collect the branch while the environment is quiescent and available.

Quiescence is a condition to verify.
A stopped agent can leave a watcher or build process writing files.
Compare repository status before and after export, or use a supported consistent capture path.
Mark an export incomplete if you cannot establish consistency.

### 16.2 Fetch into a separate export repository

The host source repository is not the reviewer checkout.
Keep an export object store whose configuration comes from trusted host code.

An illustrative command in the source repository:

```bash
git fetch \
  --no-tags \
  sandbox-task-001 \
  refs/heads/task/task-001:refs/agent-results/task-001

git log --oneline refs/agent-results/task-001
git diff --stat "<recorded-base-commit>" refs/agent-results/task-001
```

The example assumes the agent used the contracted branch name.
If it did not, inspect refs and record the actual result.
Do not substitute an unvalidated ref name into a host shell command.

Move the accepted result objects to the durable export store before deleting the sandbox.
A fetched ref that still exists only in a disposable host source directory is not enough.

### 16.3 Preserve uncommitted and untracked work

Inside the task repository:

```bash
git diff --binary HEAD
git ls-files --others --exclude-standard -z
git status --porcelain=v1 -z
```

`git diff --binary HEAD` captures tracked changes relative to the current commit.
The untracked-file inventory identifies other candidate results.
Select those files under the export contract.

Copy only bounded data through a supported runtime interface.
Docker documents `sbx cp` for local file transfer.[^docker-cp]
Verify the installed syntax and preserve ownership safely.

Treat filenames and archive metadata as untrusted.
Reject path traversal, absolute paths, symlinks escaping the export root, and special files.
Enforce total bytes and file counts before extraction.
An export worker needs no repository credentials and no ability to execute exported content.

### 16.4 Publish a result record

```json
{
  "schema": "personal-agent-result/v1",
  "task_id": "task-20261014-001",
  "generation": 1,
  "base_commit": "<recorded-base>",
  "result_commit": "<exported-result>",
  "tracked_patch_digest": "<sha256>",
  "untracked_inventory_digest": "<sha256>",
  "template_id": "fleet-typescript-20261014",
  "agent_revision": "<installed-version>",
  "model_profile": "local-coding",
  "agent_outcome": "reported-complete",
  "verification_outcome": "targeted-test-passed",
  "export_complete": true,
  "review_outcome": "pending"
}
```

The export worker validates its observations.
The agent's own summary remains attributed as a report.
Verification evidence records commands, exit status, checkout, and artifacts.

A test log saying “passed” is not enough if the agent changed the test to remove the assertion.
Review the behavior and the verification together.

### 16.5 Recover interrupted tasks with reduced authority

For an expired or suspicious task, revoke external access before recovery.
If file extraction requires the environment to run, use an isolated recovery mode with its network, credentials, model access, and previews disabled.
Do not launch the agent again.

Test whether your installed runtime can provide that recovery path.
If it cannot, preserve the stopped runtime state and the last checkpoint.
Do not claim a live export is safe while the task still has its original integrations.

Periodic exports reduce this dependency.
A ten-minute example interval limits ordinary work loss only if the exports actually succeed.
Alert on checkpoint age and record incompleteness.

### 16.6 Retire in a known order

After the result is durable:

1. Revoke task capabilities and confirm no global credential fallback grants access.
2. Remove owned preview routes and verify their absence.
3. Stop and remove the sandbox through the runtime.
4. Confirm runtime, mappings, and task-owned session are absent.
5. Release reservations and mark the task retired.

Store the retirement event and retained export location.
Cleanup should be retryable.
A missing sandbox is a successful absence observation, not a reason to skip route cleanup.

Avoid `sbx run --rm` for normal fleet work.
The checked CLI documents automatic removal after the agent session exits.[^docker-run]
That lifetime is shorter than the export and review contract in this book.

---

## 17. Measure capacity and useful throughput

The target is useful parallel work.
Ten agents waiting for model responses can be less productive than four agents completing reviewed changes.

Measure the pipeline from dispatch to accepted result.
Keep environment speed, inference speed, and code quality visible separately.

### 17.1 Build a benchmark from real tasks

Choose repeatable repository operations:

| Workload | What it reveals |
|---|---|
| Cold dependency install | Network and mirror behavior |
| Warm baseline test | Preparation benefit and disk pressure |
| TypeScript build with test database | CPU, memory, and service startup |
| Gradle build and targeted test | JVM peaks and project cache behavior |
| Small agent bug fix | Tool-loop latency and accepted output |

Record the exact inputs.
Use a dedicated benchmark baseline and disposable data.
Avoid comparing a cold run at one revision with a warm run after someone updated dependencies.

Run each workload enough times to expose variation.
Retain failures and timeouts.
Excluding failed runs makes an unstable system look fast.

### 17.2 Measure at the promised concurrency

Run one, five, and ten tasks.
Include a burst where several builds begin together.
Include a mixed workload where inference and builds overlap.

Record:

```yaml
# Proposed benchmark record.
inputs:
  host_configuration: "<identity>"
  runtime_version: "<identity>"
  template_ids: ["<identity>"]
  repository_commits: ["<identity>"]
  model_revision: "<identity>"
  concurrency: 10

measurements:
  admission_wait_ms: "<distribution>"
  baseline_ready_ms: "<distribution>"
  model_queue_ms: "<distribution>"
  first_token_ms: "<distribution>"
  result_preserved_ms: "<distribution>"
  accepted_results: "<count>"
  failed_results: "<count>"
  host_memory_peak_mib: "<observed>"
  gpu_memory_peak_mib: "<observed>"
  disk_growth_mib: "<observed>"
```

Report a distribution and the number of runs.
A p95 estimate from a handful of samples is not a reliable capacity statement.
For a small initial experiment, publish individual runs and their range.

### 17.3 Give inference its own admission

Sandbox admission reserves host memory and disk.
Inference admission reserves the backend's active work.

Ten task VMs can exist while only two requests generate tokens.
A bounded queue absorbs short bursts.
Its deadline prevents tasks waiting indefinitely.

Long prompts cost both prefill work and memory.
Output caps protect generation time but do not bound prompt processing.
Measure the selected model and context lengths together.

Treat task-level concurrency as a separate choice from server-level concurrency.
One task issuing many parallel requests should not consume the entire model queue.

### 17.4 Evaluate quality beside latency

A faster model that produces twice as many rejected patches may reduce useful throughput.
A larger context that slows every request may still help a task class finish with fewer loops.

Count accepted changes per operator hour.
Also record review effort, rejected assumptions, and verification failures.
Those measurements connect local inference to the actual work.

Keep a fixed task set for regression.
Add representative failures when the system discovers them.
The eval books in this repository supply the broader evaluation method.

### 17.5 Monitor the host's ability to recover

Useful signals include free disk, reservation totals, host memory pressure, GPU faults, inference readiness, export age, and expired task capabilities.
Collect them independently of the agents.

Begin with local inspection and a few durable counters.
Add a metrics stack when trend visibility helps an actual operating decision.
Keep its collectors and dashboard reachable only through intended administration paths.

Never report GPU utilization alone as progress.
A fully occupied GPU can be serving rejected retries.
Correlate activity with completed tool loops and preserved results.

### 17.6 Write the capacity statement honestly

An initial statement might be:

> This host completed ten mixed task environments at the recorded template and model revisions.
> It admitted two simultaneous inference requests and three expensive builds.
> The benchmark record contains readiness times, export outcomes, and rejected patches.

That is an example format, not a result measured for this manuscript.
Your statement should contain your evidence.
Increase capacity only while it remains true.

---

## 18. Operate through failures and upgrades

The fleet becomes useful when normal failures stop requiring improvisation.
Write the recovery steps before a long task runs overnight.

### 18.1 Classify failures by the boundary that failed

| Symptom | First evidence | First action |
|---|---|---|
| MacBook connection lost | Host session and runtime state | Reattach; avoid duplicate launch |
| Build cannot download a package | Proxy decision and registry request | Diagnose the narrow dependency |
| Model request stalls | Gateway queue, deadline, backend health | Bound the wait and preserve task state |
| GPU OOM | Backend error and GPU telemetry | Reduce request capacity; check context budget |
| Host memory pressure | Reservations and measured task peaks | Stop new admission; reduce active pressure |
| Disk approaching floor | Filesystem use and task growth | Stop admission; preserve results; collect owned data |
| Preview shows the wrong task | Route record and port mapping | Remove the stale route; inspect ownership |
| Export failed | Export event and surviving environment | Retain state; retry through the tested recovery path |

Do not restart every layer for one symptom.
Restarting the runtime daemon can interrupt unrelated tasks.
Restarting the agent can repeat side effects.
Recover the failed boundary first.

### 18.2 Handle backend loss without losing edits

If the inference backend fails, the gateway returns an explicit error with a bounded retry policy.
The task retains its checkout.
The controller records that model readiness is false.

After recovery, check the model identity before resuming work.
A server restarting with a different model should not continue under the old compatibility record.

Cancel repeated failed requests when their task budget is exhausted.
A model outage should not become an unbounded retry storm.
Hosted fallback requires an authorized task policy.

### 18.3 Handle disk pressure before it becomes disk full

Stop new tasks at the admission floor.
Check whether growth belongs to active work, retained templates, exports, models, or abandoned environments.
Delete only objects whose retention and ownership permit it.

Keep a reserve for export and diagnosis.
If the runtime cannot write, stop low-priority execution before collecting evidence.
Do not run a global prune as the first response.

The agent may store useful work in a stopped sandbox.
Docker's prune command removes stopped sandboxes and their contents.[^docker-prune]
Use the ledger's retirement decisions rather than treating “stopped” as “disposable.”

### 18.4 Recover after a host reboot

On startup:

1. Restore administration and inspect persistent state.
2. Start and verify shared services.
3. Reconcile tasks, reservations, and preview routes.
4. Mark interrupted tasks explicitly.
5. Resume selected tasks through an operator decision or a tested task policy.

Do not blindly relaunch every prompt.
A task may already have written a migration, created an artifact, or completed its patch before the host disappeared.
Use the saved conversation and checkpoint.

Tailscale Serve can restore routes before the controller finishes reconciliation.
A design requiring strict route ordering should use an explicit startup gate.
At minimum, remove expired routes before reusing their host ports.

### 18.5 Upgrade a compatibility tuple

An agent upgrade, runtime upgrade, model revision, server upgrade, and driver change affect different parts of the workflow.
Changing all of them at once makes failures harder to explain.

Upgrade one candidate tuple.
Run baseline, tool-loop, boundary, preview, export, and cleanup checks.
Run the small task regression set.
Then promote it.

Keep the previous supported tuple available where the installed runtime permits it.
Record whether persistent VM state survives a runtime downgrade.
If that behavior is unknown, export active work before upgrading.

### 18.6 Keep security maintenance timely

The research proposes holding kernel and NVIDIA packages.
An indefinite hold can also delay security fixes.

For this book, use deliberate maintenance windows and tested candidate configurations.
If a temporary hold is necessary, record the reason, owner, and expiry.
Review available security updates during that period.

Test inference and runtime readiness after reboot.
Keep the GPU driver compatible with the selected server and model stack.
The compatibility record belongs with the host configuration.

### 18.7 Back up what you cannot reconstruct

Back up the task ledger, reviewed configuration, template recipes, and preserved results.
Protect credential backups separately.
Model weights and package artifacts can be reconstructed only if their identities and sources remain available.

A backup of SQLite needs a consistent capture method.
A copy of a live database file without its transaction state can be incomplete.
Use the database's backup interface or stop the writer during capture.

Test restoring into a fresh directory or machine.
Confirm that results remain inspectable without the original sandbox.
The useful recovery unit is the record and artifact together.

---

## 19. Keep a fallback and roll out through evidence

The fallback should preserve the task contract.
Changing runtimes should not change whether a task can read your keys, push code, or lose its result.

### 19.1 Keep gVisor as a measured alternative

Install and register `runsc` through the project's Docker integration guide.[^gvisor-docker]
Use an image with pinned tools and a non-root user.
Keep host Docker administration outside the workload.

An illustrative creation sketch:

```bash
# On a disposable acceptance host with runsc registered.
# Replace the placeholder digest before use.
docker network create --internal task-001-net

docker create \
  --name task-001 \
  --runtime runsc \
  --network task-001-net \
  --cap-drop ALL \
  --security-opt no-new-privileges \
  --read-only \
  --tmpfs /tmp:rw,size=2g \
  --pids-limit 512 \
  --memory 8g \
  --cpus 4 \
  --user 1000:1000 \
  --mount type=bind,src=/srv/agent-work/task-001,dst=/workspace \
  "agent-base@sha256:<reviewed-digest>" \
  sleep infinity
```

The workspace must be private to the task and writable by its user.
The home directory and caches also need deliberate writable locations.
Adapt those paths to the image before expecting the agent to run.

The sketch creates no egress path and no preview route.
It is an isolation starting point, not a complete substitute for SBX.

### 19.2 Put the fallback proxy outside the task

Add an external proxy with an interface on the internal task network and a separate interface for its own approved egress.
Give it an allowlist, request limits, and task identity.
Keep its administration unavailable from the task network.

Test direct connections that ignore proxy environment variables.
An `HTTPS_PROXY` setting asks clients to cooperate.
It does not create a network boundary.

Constrain access to the host gateway, sibling networks, metadata endpoints, and other host interfaces.
The precise enforcement depends on your Docker firewall backend and topology.
Follow the corresponding Docker firewall documentation.[^docker-firewall]

Do not paste `DOCKER-USER` rules into a host that uses a different firewall backend.
Test IPv4 and IPv6.
Preserve remote administration while changing routing.

### 19.3 Preserve inference and result contracts

The fallback task still receives one task-scoped gateway capability.
It still has private source, bounded resources, and a lease.
It still exports to a host-owned store before retirement.

Give previews a separate controlled route.
Do not join every task to one unrestricted shared bridge just to let a reverse proxy see them.

Do not mount the host Docker socket.
A task needing Compose is a compatibility requirement to test separately.
gVisor documents nested Docker configuration with specific filesystem and networking requirements.[^gvisor-nested]
That workload should earn its own accepted image and boundary tests.

### 19.4 Avoid adding GPU access to the fallback by default

Retain the shared inference endpoint.
Changing the sandbox runtime does not require changing GPU ownership.

If direct CUDA execution becomes necessary, gVisor documents nvproxy support with driver compatibility requirements and limits to its driver protection.[^gvisor-gpu]
Treat it as a dedicated task class.
Measure its exposure and capacity independently.

Choose Incus VMs for persistent machine requirements.
Choose the Firecracker path when owning the runtime lifecycle is itself required.
The source research's technology list is useful context.
It does not substitute for a workload requirement.

### 19.5 Roll out in four stages

**Stage 1: one attended task.**
Create one sandbox with explicit resource limits.
Disable unused host crossings.
Complete a real task, preview it, export it, and remove the environment.
Exit when the result survives without the sandbox.

**Stage 2: one unattended task.**
Add the durable task record, deadline, gateway identity, and notifications.
Lose the Mac connection deliberately.
Restart the controller during allocation and export.
Exit when retries do not duplicate work and the result remains recoverable.

**Stage 3: five mixed tasks.**
Use both repositories.
Measure memory, disk, model queueing, and accepted results.
Exercise one dependency denial and one backend outage.
Exit when the controller preserves ownership through those failures.

**Stage 4: ten tasks and maintenance.**
Run the mixed benchmark at the target concurrency.
Expire one task and restart the host under the tested recovery plan.
Promote a candidate template and verify cleanup of old versions.
Exit when the capacity statement and recovery evidence describe what actually happened.

Do not attach a calendar promise to a stage whose acceptance is unknown.
The next stage earns its capacity through the previous one's evidence.

### 19.6 Keep the system small

The final arrangement should make work easier to supervise.
If the controller needs more attention than the code changes it enables, inspect its scope.

Keep task contracts small.
Keep services narrow.
Keep accepted results measurable.
Add an integration when it removes an observed constraint and you can test the authority it introduces.

That is how ten remote coding agents become useful workstation capacity.

---

## 20. Appendix: contracts, drills, and sources

This appendix collects the checks that turn the architecture into an accepted installation.
The manuscript supplies a design and documentation-backed interfaces.
It does not supply measurements from the reader's Ubuntu host.

### 20.1 Acceptance matrix

| Boundary | Probe | Required evidence |
|---|---|---|
| Workspace | Modify a task file and inspect source and sibling clone | Only the task clone changes |
| Host filesystem | Read a harmless canary outside approved shares | No content returned |
| SSH authority | Inspect agent socket and attempt disposable authentication | Forwarding absent unless expressly assigned |
| Shared skills | Inspect mounts and attempt a harmless shared write | Store absent for the fleet task |
| MCP | Enumerate available integrations | Only the contracted tools are available |
| Network | Allowed, denied, direct-IP, private-range, and IPv6 probes | Outcome matches recorded effective policy |
| Inference | Reach gateway and try backend port directly | Gateway works; backend is unreachable from task |
| Gateway identity | Expire one test task and reuse its capability | New request rejected |
| Tool compatibility | Read, edit, failed test, passing test, final report | Complete coherent tool loop |
| Template | Restore candidate and inspect canaries and baseline | Reviewed inputs present; sensitive state absent |
| Preview | Test listener, route, expiry, and port reuse | Intended audience only; route removed before reuse |
| Export | Return committed, dirty, binary, and untracked edits | Bounded complete record outside sandbox |
| Recovery | Kill controller after an external side effect | Reconciliation avoids duplicate allocation |
| Retirement | Interrupt cleanup and repeat it | Resources eventually absent; result retained |

Run probes against disposable resources.
Record installed versions and outcomes.
Repeat the affected probes when an input or runtime interface changes.

### 20.2 Failure drills

**Disconnect the laptop.**
Start a task in its host session.
Change the MacBook network or close the connection.
Reconnect and inspect before attaching.
The agent should have one launch record and one working clone.

**Kill the controller after creation.**
Allow the runtime creation to complete.
Terminate the controller before it records readiness.
Restart it.
The known environment should be reconciled rather than duplicated.

**Expire inference authority.**
Use a task-scoped test key.
Expire the task while a request is queued.
Verify that no new request starts and that queued or active work follows the documented cancellation policy.

**Break a dependency route.**
Deny a known test registry.
Run the baseline install.
Confirm a policy event and a classified task failure.
Restore only the required rule and rerun the same operation.

**Interrupt export.**
Create committed and uncommitted edits.
Fail the export destination.
The controller must keep the environment and an incomplete export record.
Restore storage and verify the result before retirement.

**Reuse a preview port.**
Expire a preview.
Verify route removal.
Bind a harmless test service to the old port.
The expired tailnet URL should not expose it.

**Reboot the host.**
Preserve checkpoints first.
Reboot.
Inspect routes, reservations, and task state before relaunch.
The controller should explain the interruption and retained evidence.

### 20.3 Minimal runbook

```text
before dispatch
  record objective, acceptance, base commit, and resource class
  select accepted template and model compatibility tuple
  inspect effective policy and unused host integrations
  reserve capacity and issue bounded task authority

while running
  track baseline, model queue, lease, and checkpoint age
  deliver durable status events
  retain errors and deny reasons

before retirement
  stop writers and revoke task authority
  preserve committed and uncommitted results
  verify export completeness and provenance
  remove owned routes, runtime state, and sessions
  release reservations after absence checks
```

The order can differ for suspicious or expired tasks.
Those tasks use the reduced-authority recovery path.
Keep that exception in the runbook instead of improvising it during an incident.

### 20.4 Corrections to the supplied research

The research establishes the topic and the broad adoption recommendation.
Several details needed correction or a narrower claim.

| Research detail | Treatment in this book |
|---|---|
| Convenience script installs SBX without host Engine | The install guide distinguishes Engine-plus-SBX from SBX-only installation |
| Clone mode means the agent cannot push | Clone mode isolates files; remotes and credential authority still determine push capability |
| SSH agent forwarding should be avoided | Explicitly disable the documented default and verify the result |
| Shared skills omitted from the main boundary inventory | Treat them as shared instructions with an explicit `off` choice |
| All non-HTTP TCP is dropped | Current policy documentation permits selected TCP destinations; test denied routes |
| `deny-all` implies no other allows | Review effective kit and organization rules as well as the preset |
| No documented SBX snapshot command | Local template save exists; cloud checkpoint capture is a separate feature |
| Model API key makes the shared service safe | Use a narrow gateway; review exposed endpoints and server-side tools |
| A nightly prune removes old work safely | Retire by ownership and export state; stopped environments may contain results |
| Hold driver and kernel packages for deliberate upgrades | Give any hold an owner and expiry, and keep security maintenance timely |

These are documentation and design corrections.
They are not evidence that the running fleet has passed its acceptance tests.

### 20.5 Claims left for local verification

The following details depend on the installed runtime, selected agents, or host:

- Actual memory overhead and ten-task build capacity.
- GPU model capacity, context limits, batching, and cancellation.
- Contents retained by a local saved template.
- Preview reachability for the project's bind address and address family.
- Fixed and ephemeral port behavior across restart.
- Safe recovery export from a stopped or suspicious environment.
- Agent session continuation and inner sandbox compatibility.
- Credential availability during kit install and runtime phases.
- Per-task credential identity on the chosen model-provider path.

Do not fill these gaps with a secondary-source performance number.
Publish the acceptance record from the installation.

### 20.6 Primary sources

The footnotes link to the primary documents checked while preparing the book.
They support product interfaces and documented behavior.
Task contracts, gateway design, lifecycle rules, budgets, and rollout stages are recommendations developed for the running example.

Save the installed versions alongside the verification record.
The linked documentation can change after this manuscript.
Where a page's descriptions disagree, this book uses an explicit setting and a local test instead of choosing the convenient default.

[^docker-security]: Docker, [Security model](https://docs.docker.com/ai/sandboxes/security/).
    MicroVM isolation, shared-resource crossings, local MCP execution, and the shared-skills default inconsistency.

[^docker-install]: Docker, [Install Docker Sandboxes](https://docs.docker.com/ai/sandboxes/install/).
    Ubuntu and KVM prerequisites, SBX-only installation, and sign-in requirements.

[^docker-create]: Docker, [sbx create](https://docs.docker.com/reference/cli/sbx/create/).
    Creation syntax, clone mode, resource allocation, templates, and explicit skills modes.

[^docker-settings]: Docker, [Docker Sandboxes settings](https://docs.docker.com/ai/sandboxes/configuration/settings/).
    SSH forwarding, shared skills, host lifecycle commands, and when changes take effect.

[^docker-policy]: Docker, [Local policy](https://docs.docker.com/ai/sandboxes/governance/access-controls/local/).
    Presets, effective kit rules, TCP proxying, rule inspection, and policy checks.

[^docker-credentials]: Docker, [Manage credentials](https://docs.docker.com/ai/sandboxes/configuration/credentials/).
    Scoped secrets, global fallback, OAuth passthrough, Linux storage, and SSH forwarding.

[^docker-git]: Docker, [Use Git with sandboxes](https://docs.docker.com/ai/sandboxes/workflows/git/).
    Private clones, upstream remotes, live clone-mode Git export, and host worktree limitations.

[^docker-exec]: Docker, [sbx exec](https://docs.docker.com/reference/cli/sbx/exec/).
    Commands inside an existing sandbox.

[^docker-host-model]: Docker, [Run Claude Code in a Docker Sandbox with Docker Model Runner](https://docs.docker.com/guides/claude-code-sandbox-model-runner/).
    Host-service translation, local endpoint configuration, and connectivity verification.

[^docker-models]: Docker, [Use local and hosted models](https://docs.docker.com/ai/sandboxes/configuration/models/).
    Experimental provider integration, `llmman`, supported agents, and provider credential routing.

[^docker-template]: Docker, [sbx template save](https://docs.docker.com/reference/cli/sbx/template/save/).
    Saved local images, tar export, `--pull never`, and cloud-only checkpoint capture.

[^docker-ports]: Docker, [sbx ports](https://docs.docker.com/reference/cli/sbx/ports/).
    Port specifications, loopback binding, address families, and structured output.

[^docker-run]: Docker, [sbx run](https://docs.docker.com/reference/cli/sbx/run/).
    Existing-environment launch, agent argument forwarding, automatic removal, and cloud-only TTL.

[^docker-cp]: Docker, [sbx cp](https://docs.docker.com/reference/cli/sbx/cp/).
    Local file-transfer interface.

[^docker-prune]: Docker, [sbx prune](https://docs.docker.com/reference/cli/sbx/prune/).
    Removal of stopped environments.

[^docker-gpu]: Docker, [Enable NVIDIA GPU passthrough](https://docs.docker.com/ai/sandboxes/configuration/gpu-passthrough/).
    Experimental dedicated-device VFIO requirements.

[^docker-firewall]: Docker, [Packet filtering and firewalls](https://docs.docker.com/engine/network/packet-filtering-firewalls/).
    Engine firewall behavior and backend-dependent configuration.

[^codex-security]: OpenAI, [Agent approvals and security](https://learn.chatgpt.com/docs/agent-approvals-security) and [Sandbox](https://learn.chatgpt.com/docs/sandboxing).
    Sandbox and approval controls, non-interactive operation, and container nesting guidance.

[^codex-provider]: OpenAI, [Advanced configuration](https://learn.chatgpt.com/docs/config-file/config-advanced) and [Configuration reference](https://learn.chatgpt.com/docs/config-file/config-reference).
    Custom providers, provider selection, authentication, wire format, and request retries.

[^vllm-server]: vLLM, [Online serving](https://docs.vllm.ai/en/latest/serving/online_serving/) and [serve command](https://docs.vllm.ai/en/latest/cli/serve/).
    Serving interface, model chat templates, and server options.

[^vllm-security]: vLLM, [Security](https://docs.vllm.ai/en/latest/usage/security/).
    API-key scope and serving-process boundaries.

[^llama-server]: llama.cpp, [Server documentation](https://github.com/ggml-org/llama.cpp/blob/master/tools/server/README.md).
    Inference APIs and optional server-side tool and MCP execution.

[^tailscale-ssh]: Tailscale, [Tailscale SSH](https://tailscale.com/docs/features/tailscale-ssh).
    SSH identity, access policy, server activation, and recovery considerations.

[^tailscale-serve]: Tailscale, [Tailscale Serve](https://tailscale.com/docs/features/tailscale-serve).
    Tailnet access, HTTPS requirements, and the distinction from Funnel.

[^tailscale-serve-cli]: Tailscale, [tailscale serve command](https://tailscale.com/docs/reference/tailscale-cli/serve).
    Background persistence, listener removal, and status inspection.

[^tmux]: tmux project, [Getting started](https://github.com/tmux/tmux/wiki/Getting-Started).
    Sessions, detach, and reattach behavior.

[^git-clone]: Git, [git-clone](https://git-scm.com/docs/git-clone).
    Reference repositories, object borrowing, and dissociation.

[^gvisor-docker]: gVisor, [Docker quick start](https://gvisor.dev/docs/user_guide/quick_start/docker/).
    Installing and selecting the `runsc` runtime.

[^gvisor-nested]: gVisor, [Docker in gVisor](https://gvisor.dev/docs/tutorials/docker-in-gvisor/).
    Nested Docker compatibility requirements.

[^gvisor-gpu]: gVisor, [GPU support](https://gvisor.dev/docs/user_guide/gpu/).
    nvproxy compatibility and GPU driver exposure.
