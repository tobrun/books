# Self-Hosted Agent Sandboxes in Practice

## A Hands-On Guide to Firecracker, Warm Workspaces, and Parallel Coding Agents on One Ubuntu Host

The first coding agent is easy to run.
Give it a checkout, install the dependencies, and let it work.

The tenth agent changes the problem.
Two builds compete for memory.
One task rewrites a shared cache.
Another leaves a process running after its caller disappears.
A repository instruction asks the agent to send a token somewhere it does not belong.

You need more than another terminal.
You need a system that gives each task a private environment, limits what that environment can reach, and knows when the task has finished.

You also need the environment to be useful quickly.
An empty VM that boots in a fraction of a second can still spend minutes downloading dependencies.
A warm checkout that produces the wrong build is fast in a less useful way.

> A useful sandbox starts from known inputs, receives bounded authority, preserves its results, and disappears without leaving that authority behind.

This book builds that system on one Ubuntu host.
Firecracker provides a separate guest kernel for each sandbox.
Versioned workspace images preserve repository dependencies and build caches.
Golden memory snapshots reduce repeated initialization.
A small broker owns claims, leases, admission, and cleanup.
Host services own network policy, credentials, Git mirrors, and evidence.

The running example is an engineer who wants to run ten coding agents initially and grow toward fifty.
The first repositories are a TypeScript service and a Java application built with Gradle.
Both agents edit code and run tests.
Some tasks need Docker inside the guest.
The host has no requirement to provide an Android emulator, GPU passthrough, or mutually untrusted customer hosting.

The examples use Linux, Bash, Python, SQL, JSON, and YAML.
They develop implementation boundaries rather than a complete sandbox platform.
Names such as `sandboxctl`, `guestctl`, and `baker` are proposed interfaces to build or supply through an adopted platform.
They are not commands installed by Firecracker.

**How to read the examples.**
Configuration fragments describe a contract.
Code marked as a sketch omits integration work deliberately.
The small complete functions can be checked independently, but a passing function does not prove the host is isolated.
Networking, jailer configuration, guest restore behavior, and failure recovery need tests on the actual Linux host.

**Version scope.**
The supplied research is “Self-Hosted Agent Sandboxes: Firecracker Plan,” revision 2, prepared on October 13, 2026.
This book retains its Firecracker-first architecture and Incus fallback.
It uses explicit artifact identities and compatibility tests instead of depending on a particular release being current.
The final chapter identifies the primary documentation checked while preparing this manuscript and the research claims that still require local evidence.

---

## Contents

1. [Define the job before choosing the runtime](#1-define-the-job-before-choosing-the-runtime)
2. [Choose the isolation boundary](#2-choose-the-isolation-boundary)
3. [Decide what to adopt and what to build](#3-decide-what-to-adopt-and-what-to-build)
4. [Separate execution from authority](#4-separate-execution-from-authority)
5. [Prepare the host and its resource budget](#5-prepare-the-host-and-its-resource-budget)
6. [Build a guest that can recover](#6-build-a-guest-that-can-recover)
7. [Give each task an independent repository](#7-give-each-task-an-independent-repository)
8. [Warm dependencies without sharing writable state](#8-warm-dependencies-without-sharing-writable-state)
9. [Bake and publish a consistent golden snapshot](#9-bake-and-publish-a-consistent-golden-snapshot)
10. [Restore into a new identity](#10-restore-into-a-new-identity)
11. [Make networking a host policy](#11-make-networking-a-host-policy)
12. [Keep credentials behind narrow services](#12-keep-credentials-behind-narrow-services)
13. [Build the broker as a durable state machine](#13-build-the-broker-as-a-durable-state-machine)
14. [Admit work the host can finish](#14-admit-work-the-host-can-finish)
15. [Preserve results before destroying the sandbox](#15-preserve-results-before-destroying-the-sandbox)
16. [Reconcile leaks and collect unused artifacts](#16-reconcile-leaks-and-collect-unused-artifacts)
17. [Measure the system the agents actually use](#17-measure-the-system-the-agents-actually-use)
18. [Upgrade deliberately and keep a fallback](#18-upgrade-deliberately-and-keep-a-fallback)
19. [Roll out through evidence](#19-roll-out-through-evidence)
20. [Appendix: contracts, drills, and verification notes](#20-appendix-contracts-drills-and-verification-notes)

---

## 1. Define the job before choosing the runtime

Start with a task that already works.

A developer asks an agent to change the validation behavior of a service.
The agent checks out a commit, installs dependencies, edits three files, runs the targeted tests, and produces a patch.
The developer reviews it.

The sandbox should improve the conditions around that task.
It should give the task a private filesystem.
It should keep repository credentials out of generated shell commands.
It should stop the work when its lease expires.
It should preserve the patch even when the model endpoint fails.

These are observable requirements.
“Run fifty agents” is a capacity target that comes later.

### 1.1 Write the operating contract

```yaml
service: personal-coding-sandboxes
owner: sandbox-maintainer
host_scope: one-linux-host

workloads:
  - repository checkout and dependency preparation
  - source edits and targeted tests
  - bounded builds and local development services

initial_capacity:
  active_leases: 10
  larger_goal: 50
  admission: measured resource classes

required_properties:
  - separate guest kernel for generated code
  - private writable root and workspace disks
  - exact task commit recorded
  - host-enforced network policy
  - credentials limited to the active lease
  - durable result export
  - cleanup after caller and broker failure

excluded_workloads:
  - mutually untrusted customer tenancy
  - production deployment from inside the guest
  - hardware passthrough
  - Android emulator and nested hypervisor workloads

recovery:
  templates: rebuild from reviewed inputs
  active_tasks: export periodically; restart if the host is lost
  broker_state: restore from tested backup
```

The exclusions are decisions about this system.
They are not claims that those workloads can never be supported.
Adding one changes the guest, the runtime requirements, or the threat model.
Give that change its own acceptance evidence.

### 1.2 Separate three clocks

There are three useful startup measurements:

| Measurement | Starts | Ends |
|---|---|---|
| Runtime startup | Launch or restore request reaches the runtime | Guest execution resumes |
| Sandbox readiness | Broker accepts a claim | Guest identity, policy, checkout, and dependencies are ready |
| Time to first passing test | Caller submits the task | The selected baseline test passes at the task commit |

The third is usually closest to what the engineer needs.
It includes queueing, preparation, and real repository work.

A passing baseline test is different from task success.
The baseline shows that the environment works before the agent edits anything.
The task still needs a reviewed change and its own verification.

Firecracker's published boot and memory-overhead figures describe a minimal guest under specified test conditions.[^firecracker-spec]
They do not promise that a Java service with a warm daemon, secrets, a fresh checkout, and a passing test will be ready in the same interval.

Make that distinction part of every benchmark report.

### 1.3 Choose success criteria before a capacity number

For the first implementation, require:

- Ten concurrent claims reach a usable checkout without manual repair.
- A change in one guest cannot alter another guest's repository or writable cache.
- Direct access to the host, sibling guests, and disallowed destinations fails.
- A caller retry does not allocate a second environment for the same request.
- Lease expiry revokes authority and eventually reclaims every owned resource.
- A malformed or incompatible snapshot has an observable failure and a tested recovery path.
- Every completed task has a result record outside the guest.

Set latency and throughput targets after measuring the current workflow.
An arbitrary target can encourage hiding preparation behind a narrower definition of ready.

### 1.4 Keep one host honest

One machine is an operating choice.
It can be easy to inspect, inexpensive to run, and sufficient for personal work.
It is also one failure domain.

A host reboot interrupts all active tasks.
A disk failure can remove both templates and private workspaces.
A broker backup stored only on that disk is not a recovery plan.

For this book, template rebuilds and broker recovery matter more than transparent task migration.
Active agents restart from preserved work.
That is a useful first contract because it can be tested.

### Lab: describe one completed task

Record the commit, dependency command, baseline test, expected result format, allowed network destinations, and maximum task duration for one repository.
Measure the current time to its first passing test.

**Done when:** someone can distinguish a fast runtime from a useful sandbox and can tell whether the task's work survived.

---

## 2. Choose the isolation boundary

The agent may be working on your own repository.
The code it runs still includes package scripts, downloaded tools, test fixtures, repository instructions, and commands generated from model output.

Ownership of the repository does not make every execution predictable.
Choose the boundary from what a failure could affect.

### 2.1 Name the assets

Our host holds:

- Source mirrors for several repositories.
- Credentials used by Git and model services.
- Immutable guest templates.
- Private workspace disks for active tasks.
- The broker's database and operation journal.
- Logs and exported patches.
- The host operating system.

A compromised guest must not receive those assets merely because it can run as root inside its own kernel.

This makes the host API, snapshot publisher, credential services, and artifact exporter part of the security design.
The VM boundary handles one part of the problem.

### 2.2 Compare boundaries by what they contain

| Approach | Main execution boundary | Use in this book | Additional work |
|---|---|---|---|
| Firecracker with jailer | Guest kernel plus constrained host VMM | Primary runtime | Images, guest channel, networking, leases, and cleanup |
| Incus VM | Guest kernel managed by Incus | Operational fallback | Broker contract, prewarming, policy, and result handling |
| gVisor with a container runtime | Userspace kernel mediation | Interim path when KVM is unavailable | Compatibility, policy, and the same task lifecycle |
| Ordinary rootless container | Host kernel with container isolation | Trusted build or development tier | Explicit acceptance of the shared kernel boundary |
| Process sandbox | Restricted process access | Optional inner layer | An outer execution environment and lifecycle |

Firecracker uses KVM and expects read/write access to `/dev/kvm`.[^firecracker-start]
gVisor's systrap platform uses syscall interception and does not require hardware virtualization.[^gvisor-platform]

If the target host cannot run KVM, the Firecracker path stops at that prerequisite.
Do not spend a week building images before discovering it.

### 2.3 Treat the jailer as part of the runtime

A VM escape should still encounter a constrained host process.
Use the matching Firecracker and jailer binaries, retain the runtime's seccomp policy, and verify the process's effective host identity and resource boundaries.

The jailer establishes a restricted filesystem view, drops to a selected UID and GID, and can join the network namespace you provide.[^firecracker-jailer]
Its inputs are trusted operator inputs.
A guest-controlled path should never become a jailer argument.

The recommendation here is one dedicated host identity per live sandbox, allocated from a managed range.
If you choose a shared identity, document how filesystem permissions and socket access still prevent one sandbox process from interfering with another.
Use observed permissions as evidence.

### 2.4 Restrict authority separately

A perfectly contained guest can still abuse a token you intentionally gave it.
It can call an allowed API with the wrong request.
It can publish confidential content to an allowed repository.
It can spend money through a legitimate model endpoint.

Execution containment and authority restriction need separate tests.

Our guest can edit a private checkout and produce results.
It can read the selected repository through a narrow mirror service.
It can use a bounded model capability.
It cannot deploy to production or acquire the host's Git identity.

The caller's claim does not authorize an arbitrary destination.
The broker chooses a named policy.

### 2.5 Use a different contract for stronger tenancy

This design addresses personal agents on one host.
Serving mutually untrusted customers adds tenant authentication, stronger resource accounting, isolation review, side-channel considerations, abuse response, and a larger recovery obligation.

Do not describe a personal sandbox implementation as a customer hosting platform because both use VMs.
The runtime may support stronger deployments.
The surrounding service still has to establish them.

### Lab: trace five escape routes

Try to explain how a guest could reach a sibling disk, the broker socket, the host Docker daemon, a cloud metadata endpoint, and a write-capable source credential.
Assign one enforcement point and one negative test to each route.

**Done when:** every important restriction has a host-side owner and an observable failure case.

---

## 3. Decide what to adopt and what to build

Firecracker supplies a runtime.
A sandbox service also needs image publication, task preparation, network policy, secrets, a guest execution channel, admission, logs, cleanup, and recovery.

Adopting those parts can be the right engineering decision.
Building them can be the right decision too.
Choose after exercising the contract.

### 3.1 Run an adoption spike

The research proposes CubeSandbox as the first candidate.
Its project documentation advertises KVM-based isolation, pooling and snapshot cloning, and compatibility with the E2B SDK.[^cubesandbox]

Those claims justify a spike.
They do not establish that the platform can publish your repository-specific templates, perform your restore barrier, or export your task results.

Use the TypeScript and Gradle repositories for the same experiment:

1. Build the environment from pinned inputs.
2. Prepare a repository-specific template.
3. Start ten environments concurrently.
4. Verify the exact commit and run a baseline test.
5. Attempt forbidden network access and cross-environment writes.
6. Interrupt the client and restart the platform controller.
7. Export the changes and reclaim the environments.
8. Repeat after changing a dependency lockfile.

Capture platform revision, configuration, workload inputs, failures, and timings.
An SDK method with a familiar name is only one part of compatibility.

### 3.2 Use a decision record

```yaml
decision: sandbox-platform
candidate: evaluated-project
candidate_revision: recorded-source-revision

required:
  golden_workspace: pass
  private_writable_storage: pass
  restore_identity_barrier: needs-extension
  host_egress_policy: pass
  result_export: pass
  interrupted_claim_recovery: fail

measurements:
  baseline: reports/current-workflow.jsonl
  candidate: reports/platform-spike.jsonl

outcome: extend-or-build
reason: interrupted claims require durable reconciliation
time_budget: bounded-engineering-spike
review_owner: sandbox-maintainer
```

The example outcome is illustrative.
It is not a report of a CubeSandbox deployment.

A useful record says which capability failed.
“Too young” is an opinion.
“A restart leaves a running VM with no discoverable owner” is something you can reproduce.

### 3.3 Keep the broker interface small

The service can expose a modest vocabulary:

```text
claim(repository, commit, resource_class, policy, request_key)
status(sandbox_id)
renew(sandbox_id, lease_token)
exec(sandbox_id, command_request)
export(sandbox_id, export_request)
release(sandbox_id, lease_token)
```

The external interface should remain stable when the runtime changes.
Keep it honest by exposing runtime capabilities explicitly:

```json
{
  "runtime": "firecracker",
  "capabilities": {
    "memory_template_restore": true,
    "guest_docker": true,
    "transparent_live_migration": false
  }
}
```

These values describe the supported implementation.
They are not derived from the runtime's marketing page.

### 3.4 Build the smallest useful control plane

For one host, start with a broker, a local durable database, a narrow privileged host helper, a guest controller, and an independent reaper.
Use service supervision for their processes.

Do not add a cluster scheduler until there is a scheduling problem the single host cannot solve.
Do not replace durable state with an in-memory map because the first demonstration has only one VM.

A small system can still have explicit failure semantics.
That is where its reliability comes from.

### Lab: decide with two repositories

Complete the same task preparation and interrupted-claim experiment on an adopted candidate and a minimal Firecracker prototype.
Write a decision record that includes the maintenance work you would own in each case.

**Done when:** the choice follows measured behavior and a named missing capability.

---

## 4. Separate execution from authority

The first architecture drawing should show trust boundaries.
A box called “agent” hides too many responsibilities.

### 4.1 Assign one owner to each kind of state

```text
Authenticated caller
        |
        v
Broker + durable operation database
        |
        +----> Narrow privileged host helper
        |          |
        |          +--> jailer, cgroups, disks, network slots
        |
        +----> Runtime adapter ----> Firecracker API socket
        |
        +----> Guest controller ---> private vsock channel
        |
        +----> Credential and egress services
        |
        +----> Result store

Trusted template publisher ---> immutable golden artifacts
Git refresher ---------------> repository mirror service

Independent reaper ---------> lease expiry and reconciliation
```

The drawing describes ownership.
It does not require a separate executable for every line.
You can combine components when their privilege and failure boundaries remain visible.

| State or operation | Owner | Guest access |
|---|---|---|
| Claim and lease | Broker database | Own status through mediated API |
| VM launch and destruction | Host helper and runtime adapter | None |
| Firecracker API socket | Runtime adapter | None |
| Template publication | Baker and publisher | Read indirectly through private restore |
| Workspace mutations | Task guest | Own disk only |
| Mirror updates | Git refresher | Read selected repository through service |
| API credentials | Credential service | Narrow leased capability |
| Network enforcement | Host policy service | No rule updates |
| Result publication | Exporter | Bounded untrusted payload |
| Cleanup | Reaper | Cannot prevent expiry |

### 4.2 Give root a closed vocabulary

The host helper may need authority to prepare namespaces, identities, cgroups, and jail resources.
Its request should select a known operation:

```json
{
  "operation": "prepare_sandbox",
  "sandbox_id": "sb-42",
  "generation": 1,
  "template_id": "service-template-17",
  "resource_class": "standard",
  "network_policy": "coding-default"
}
```

The helper resolves those identifiers against trusted records.
It chooses paths, binaries, limits, and rules itself.

Avoid an API that accepts a shell command or arbitrary root filesystem path.
That turns input validation into the host's entire protection.

The helper should reject unknown fields, invalid generations, retired templates, and unsupported policies before making changes.
Every allocated resource needs an ownership record before another component can use it.

### 4.3 Keep template builders out of the host trust boundary

A baker runs package installers and repository build scripts.
That code deserves the same containment as task code.

Run preparation inside a controlled guest.
Let a trusted publisher decide whether its outputs can become a template.
The publisher should collect known host files and verify their metadata.
It should not execute a shell script produced by the baker.

A template that passes tests can still contain unwanted startup behavior.
Build success is one publication check.
Credential hygiene, startup inventory, source approval, and restore tests are others.

### 4.4 Treat guest reports as untrusted

The guest can report a test exit code, a clock, a hostname, and a file inventory.
Those reports are useful.
They do not override the host's lease, resource accounting, or policy.

After task code starts, a compromised guest can also impersonate its own guest agent.
Use host observations for whether the VM is alive, which cgroup it occupies, whether its network capability is active, and which template it references.

A result manifest may contain a claimed test success.
A separate clean verification environment decides whether to trust the patch.

### Lab: remove one unnecessary privilege

Inspect every component's effective identity, sockets, and writable paths.
Choose one permission that exists for convenience and remove it.
Run the thin task again.

**Done when:** the system still works and the removed permission has a negative test.

---

## 5. Prepare the host and its resource budget

The host is the foundation of every boundary.
Use a supported Linux release and a reviewed kernel, then prove the capabilities the design needs.

The research names Ubuntu releases as candidates.
The acceptance record should name the release actually installed.
A distribution version alone does not establish KVM access, cgroup configuration, or storage cloning.

### 5.1 Record host capabilities

These are read-only inspection commands for the target Linux host:

```bash
uname -a
lscpu
stat /dev/kvm
findmnt -no FSTYPE /sys/fs/cgroup
findmnt -no SOURCE,FSTYPE,OPTIONS /srv/sandboxes
df -h /srv/sandboxes
```

Confirm virtualization by launching a small supported guest through the same helper and service identities the broker will use.
The presence of a device file is only a prerequisite.

Record:

```yaml
host:
  os_release: observed-release
  kernel_release: observed-kernel
  cpu_identity: observed-model-and-features
  virtualization_test: evidence/kvm-smoke.json
  cgroup_mode: v2
  sandbox_filesystem: observed-filesystem
  reflink_test: evidence/reflink-smoke.json
  free_space: measured
  runtime_binary_digest: verified-digest
  jailer_binary_digest: verified-digest
```

Store this record with host maintenance evidence.
Do not copy it into every task transcript as if it were user content.

### 5.2 Verify cloning on the actual volume

Use a dedicated scratch directory on the sandbox filesystem:

```bash
set -euo pipefail

probe_dir=$(mktemp -d /srv/sandboxes/reflink-probe.XXXXXX)
trap 'rm -rf -- "$probe_dir"' EXIT

printf 'original\n' > "$probe_dir/source"
cp --reflink=always -- "$probe_dir/source" "$probe_dir/clone"
printf 'changed\n' > "$probe_dir/clone"

test "$(cat "$probe_dir/source")" = original
test "$(cat "$probe_dir/clone")" = changed
```

GNU `cp --reflink=always` fails when copy-on-write cloning is unavailable; automatic mode can fall back to a full copy.[^cp-reflink]
Failing explicitly is useful when the hot path depends on this property.

This probe establishes that a small clone can be created and modified independently.
Measure multi-gigabyte image cloning separately.
Measure space growth during real builds too.

XFS with reflink is the reference storage choice from the research.
Another filesystem can satisfy the contract if it passes the same checks.
Do not provide a disk-formatting command as part of routine application bootstrap.
Storage provisioning belongs in the host's reviewed installation procedure.

### 5.3 Reserve capacity for the host

Do not allocate all host memory to guests.
The broker, proxy, logs, kernel, page cache, template pipeline, and recovery operations need room.

The first budget is conservative:

```text
usable guest budget
  = physical memory
  - host operating reserve
  - maintenance and baker reserve
  - measured runtime allowance
```

Use explicit units.
GiB and GB are different quantities.
Allocated guest RAM and host resident memory are different observations.

The research's ten-, twenty-, and fifty-agent hardware estimates are planning hypotheses.
The number of simultaneous compilers matters more than the number of model conversations.

### 5.4 Protect management capacity

An overloaded build fleet must not stop the reaper from running.
Reserve CPU and memory for host services.
Limit guest CPU and I/O.
Keep logs bounded.
Give the operator a management path that does not depend on guest networking.

Test this by starting enough builds to hit admission limits.
The broker should continue answering status requests.
Lease expiry should still progress.

If you cannot destroy a VM while the host is under pressure, the resource policy needs work before adding more agents.

### Lab: prove host prerequisites

Record one jailed guest launch, one cgroup limit observation, one reflink clone, and one cleanup under CPU pressure.
Keep the evidence with the host capability record.

**Done when:** the actual host satisfies the prerequisites through the production identities and paths.

---

## 6. Build a guest that can recover

A Docker image is a useful source of developer tools.
It is not automatically a bootable VM filesystem.

A container normally starts one configured process against the host kernel.
Our guest needs a compatible kernel, init behavior, disk mounts, network setup, and a controller that can perform restore preparation before task code begins.

### 6.1 Separate kernel, root, and workspace

Use three versioned inputs:

| Input | Contents | Changes when |
|---|---|---|
| Guest kernel | Drivers and operating-system execution support | Kernel patch, runtime compatibility, required feature |
| Base root filesystem | Tools, ordinary user, init, guest controller | Toolchain, controller, package, or policy change |
| Workspace filesystem | Independent checkout, dependencies, build caches | Repository inputs or preparation recipe change |

The base root disk is private and writable for each task in the reference design.
This makes identity updates and ordinary operating-system writes straightforward.
A later read-only base with a writable overlay can work, but its backing files and restore behavior need their own tests.

Mount the workspace at a stable path such as `/workspace`.
Put the task user's home and writable package caches on that disk deliberately.
A second disk does not help if Gradle still fills the root filesystem.

### 6.2 Build from reviewed inputs

The build recipe should identify:

```yaml
guest_recipe:
  kernel_artifact: approved-kernel
  kernel_config_digest: recorded-digest
  base_source_image: approved-image-digest
  toolchain_inputs: committed-lock-and-package-records
  guest_controller_digest: recorded-digest
  init_recipe_digest: recorded-digest
  task_uid: 1000
  workspace_mount: /workspace
  controller_port: 7000
```

Review the kernel features needed for virtio block and network, vsock, the selected filesystem, and clone notification.
If Docker runs in the guest, add and test its kernel, cgroup, filesystem, and networking requirements.
Do not infer those from Docker working on the host.

### 6.3 Export containers carefully

A controlled image pipeline can create a container without running repository code, export its filesystem, inspect the staging tree, and assemble an ext4 image.

Docker documents that `docker export` omits mounted volume contents.[^docker-export]
It also does not supply your VM's boot contract.
Inventory required files in the exported tree.

The assembly step can look like this:

```bash
# Assembly fragment, after trusted staging-tree validation.
# Paths are chosen by the image builder in a disposable build directory.
truncate -s 8G rootfs.ext4
mkfs.ext4 -F -d rootfs-tree rootfs.ext4
```

The fragment only creates a filesystem in the named image file.
The complete pipeline must preserve required ownership and permissions, provision init and mounts, and test boot.
Run privileged extraction or image manipulation in a disposable builder environment.

Include only the personal defaults needed by the workload.
Your shell aliases may be useful.
Your SSH private keys, Git credential helper state, cloud configuration, and editor authentication are not base-image inputs.

### 6.4 Boot into a control-only state

The guest controller starts before task code.
It listens on a fixed vsock port and initially accepts preparation and health operations.
It does not start the model client or repository command automatically.

Its states are:

```text
booting -> awaiting_prepare -> preparing -> prepared -> task_enabled
```

A restored controller must re-enter the preparation barrier.
A snapshotted flag called `prepared=true` is stale state.
The broker's new generation establishes readiness for the new task.

For the first version, stop repository daemons before snapshotting.
Warm files and installed dependencies remain useful.
Keeping processes alive can be added later when the process-specific restore behavior has been tested.

### 6.5 Define the guest command request

```json
{
  "protocol": 1,
  "operation": "exec",
  "request_id": "cmd-19",
  "sandbox_generation": 1,
  "argv": ["pnpm", "test", "--", "validation"],
  "cwd": "/workspace/repo",
  "environment_profile": "task-default",
  "timeout_seconds": 120,
  "output_limit_bytes": 1048576
}
```

The guest controller runs ordinary task commands as the task user.
It controls environment construction, deadlines, process-group cleanup, and output limits.
A shell remains available when explicitly requested as an argument vector.
The JSON boundary prevents accidental shell interpolation by the host controller.

Guest-side command restrictions are an inner layer.
They do not replace host resource or network enforcement.

### 6.6 Make the cold configuration inspectable

A cold guest configuration can use fixed paths inside its private jail:

```json
{
  "boot-source": {
    "kernel_image_path": "/images/vmlinux",
    "boot_args": "console=ttyS0 reboot=k panic=1 root=/dev/vda rw"
  },
  "drives": [
    {
      "drive_id": "root",
      "path_on_host": "/disks/root.ext4",
      "is_root_device": true,
      "is_read_only": false
    },
    {
      "drive_id": "workspace",
      "path_on_host": "/disks/workspace.ext4",
      "is_root_device": false,
      "is_read_only": false
    }
  ],
  "machine-config": {
    "vcpu_count": 4,
    "mem_size_mib": 6144,
    "smt": false
  },
  "network-interfaces": [
    {
      "iface_id": "task-net",
      "host_dev_name": "tap0",
      "guest_mac": "06:00:00:00:00:01"
    }
  ],
  "vsock": {
    "guest_cid": 3,
    "uds_path": "/run/control.vsock"
  }
}
```

This is a cold-boot configuration fragment for the pinned Firecracker interface.[^firecracker-api]
The helper supplies private writable disks, an approved kernel, the TAP, permissions, and the API socket.
The guest recipe mounts the second disk and configures the interface before starting the controller.
The serial console is useful during bring-up; decide its production logging policy separately.

The host adapter can submit the resources through the HTTP API and start the guest with `InstanceStart`.
A config-file startup path is another supported integration to check against the selected release.
Use the same resource identities in the operation journal whichever path you choose.

This fragment omits additional device and kernel setup for clone notification.
The published template must pass that check before it supports repeated memory restores.
Cold-boot success alone does not admit a template to the snapshot tier.

### Lab: cold boot before snapshotting

Boot the guest from the selected kernel and image.
Mount the workspace, connect to the guest controller, run a baseline test as the ordinary user, and stop the VM.
Repeat from the same inputs.

**Done when:** the cold path works independently of any memory snapshot.

---

## 7. Give each task an independent repository

A filesystem namespace is not the whole Git boundary.
Git has object storage, refs, configuration, hooks, and working-tree state.
Decide which of those a task may change.

The reference design gives each task a standalone clone on its private workspace disk.
A host mirror accelerates transport.
The mirror is never mounted into the guest.

### 7.1 Keep the mirror service read-only

The host refresher maintains one bare mirror per approved repository:

```text
/srv/sandbox-mirrors/
  typescript-service.git/
  java-service.git/
```

Only the refresher can mutate those mirrors.
The guest reads through an authenticated Git service that exposes approved repository identifiers.
It cannot choose an arbitrary host path or upload a new ref.

Do not expose a Git daemon with unrestricted repository discovery.
Do not run the read service under the credential identity used for upstream writes.

Refreshing a mirror and reading it are separate operations.
Keep the read service reliable during refresh and maintenance.
If you introduce aggressive pruning, verify that concurrent clients can still finish their fetches.
The guest retains its own objects after transfer.

### 7.2 Avoid hidden object dependencies

Git documents that `clone --shared` can leave the new repository dependent on the source repository's objects, with corruption possible after source maintenance removes them.[^git-clone]

For a network clone, avoid creating alternates at all.
For a trusted local preparation step, a standalone clone can use:

```bash
git clone --no-local /trusted/mirror.git /workspace/repo
test ! -e /workspace/repo/.git/objects/info/alternates
```

The trusted source path is a baker input.
It is not a guest-supplied host path.
The main property is that removing or collecting the mirror does not invalidate the published workspace.

A reflink of the workspace image gives later tasks separate writable blocks while retaining an independent repository logically.
It does not reintroduce Git alternates.

### 7.3 Keep worktrees inside one sandbox

Linked worktrees share repository state, including most refs, with their common repository.[^git-worktree]
That is useful within one developer's task.
It is an unsuitable default for isolating independent agents.

A guest can create extra worktrees inside its private disk.
That is its own state.
Two different guests should not receive host worktrees backed by the same writable repository.

The acceptance test should try both ordinary working-tree writes and changes to refs and configuration.
Checking that two source files live at different paths proves very little.

### 7.4 Prepare an exact commit

The caller names an approved repository and an exact commit.
The broker authorizes that pair.
It passes the accepted object identity to the guest controller as data.

In a disposable clone, a preparation fragment might be:

```bash
# The controller has validated task_commit as an object ID for this repository.
git fetch --no-tags origin
git cat-file -e "${task_commit}^{commit}"
git checkout --detach "$task_commit"
test "$(git rev-parse HEAD)" = "$task_commit"
```

Some servers do not advertise or permit fetching every historical object.
The mirror service must guarantee that accepted task commits are retrievable.
If it cannot, the claim fails with a source-availability error.

A branch name can move between request and fetch.
Record the resolved commit before preparation.
If the agent wants a branch name for its work, create a private task branch from that commit.

### 7.5 Reset warm state deliberately

A golden workspace starts at the baker's commit.
A task may request a newer commit with the same lockfiles.
Moving `HEAD` is only one preparation step.

Track:

| State | Preparation decision |
|---|---|
| Tracked source | Checkout exact accepted commit |
| Generated output | Keep only if cache validity is established |
| Dependency tree | Validate against dependency inputs and install policy |
| Git configuration | Apply task-specific identity and disable unintended helpers |
| Hooks | Establish reviewed policy before any command that could invoke them |
| Submodules | Pin and authorize each source |
| Large-file storage | Authorize its separate transport and verify availability |

A restored Git index lock or build lock should not be deleted blindly.
The baker should finish operations and stop relevant processes before publication.
A template that repeatedly requires unexplained lock removal is not healthy.

### Lab: cut the mirror loose

Prepare a workspace, stop its source mirror service, and run a local history query and baseline test.
Then create two task clones, alter a ref in one, and verify the other is unchanged.

**Done when:** the tasks own independent repository state and the template has no hidden object dependency.

---

## 8. Warm dependencies without sharing writable state

Repository preparation often dominates the useful startup time.
Memory restore can preserve initialized state.
It cannot make an invalid dependency graph correct.

Begin by warming files.
Installed dependencies, package archives, and build outputs are easier to reason about than live processes.

### 8.1 Separate cache roles

| Cache or artifact | Writer | Reader | Reference placement |
|---|---|---|---|
| Package archives | Baker and private task | Same task | Private workspace |
| Installed dependency tree | Baker and private task | Same task | Private workspace |
| Compiler incremental state | Baker and private task | Same task | Private workspace |
| Published trusted build cache | Trusted publisher | Authorized tasks | Separate service |
| Model credentials | Credential service | Active task capability | Outside template |

The word “cache” does not imply harmless data.
A package archive can become executable code.
A build cache can return an object file.
An installed package can contain a post-install script.

If one task can write shared entries consumed by another task, it can influence the other's execution.
Make that influence a deliberate trust decision.

### 8.2 Bake a private TypeScript environment

For the running service, record the Node and package-manager identities, the lockfile, workspace configuration, and preparation command.
Run an immutable or frozen install under the package manager's supported semantics.

Then run the baseline test and one representative build.
Keep the useful caches on the workspace disk.
At claim time, validate the dependency inputs before reusing the installed tree.

pnpm explicitly limits shared writable store use to mutually trusted users.[^pnpm-store]
Our default is one private store per task disk, initialized from the template.

That preserves warm downloads without giving the task a write path into another task's supply chain.
Measure its disk cost instead of weakening the boundary to save space prematurely.

### 8.3 Bake a private Gradle environment

Give the task user a defined `GRADLE_USER_HOME` on the private workspace disk.
Warm the dependency resolution and representative test.
Stop Gradle daemons before the first golden snapshot.

Gradle documents an optional read-only dependency cache through `GRADLE_RO_DEP_CACHE`, with requirements for independently preparing the cache.[^gradle-cache]
That can become a later optimization.

Firecracker does not offer the ordinary host bind-mount workflow.
A read-only shared artifact must reach the guest through supported storage or a service.
Choose an immutable disk or a controlled cache service and test the application's read behavior.

A host directory path in an environment variable does not make that directory available inside the VM.

### 8.4 Compute a preparation identity

A lockfile hash is a useful input.
It is not a complete template identity.

Include:

```json
{
  "repository": "typescript-service",
  "seed_commit": "resolved-full-object-id",
  "dependency_inputs": {
    "lockfile_digest": "sha256:...",
    "workspace_config_digest": "sha256:...",
    "install_recipe_digest": "sha256:..."
  },
  "toolchain_identity": "node-and-package-manager-record",
  "base_image_digest": "sha256:...",
  "guest_kernel_digest": "sha256:...",
  "resource_class": "standard",
  "controller_digest": "sha256:...",
  "runtime_digest": "sha256:...",
  "restore_policy_digest": "sha256:...",
  "build_environment_digest": "sha256:..."
}
```

Each listed field must have a defined producer.
“Build environment” should identify the small set of environment inputs allowed to affect preparation.
It should not be a dump of the baker's process environment.

A release can use a content identity built from a canonical encoding:

```python
import hashlib
import json


def preparation_identity(record: dict) -> str:
    """Complete for JSON records using this project's fixed encoding contract."""
    payload = json.dumps(
        record,
        sort_keys=True,
        separators=(",", ":"),
        ensure_ascii=True,
        allow_nan=False,
    ).encode("utf-8")
    return hashlib.sha256(payload).hexdigest()
```

This is a project-local encoding contract.
Use the same implementation for publishing and validation.
If independent languages must produce identical identities, adopt and test a canonicalization specification.

The digest establishes identity.
It does not establish that the publisher is trusted or the contents are safe.

### 8.5 State the invalidation rule

The broker can choose a template with an older source commit when dependency inputs match and the task preparation validates the new checkout.
Otherwise it selects another template or performs a private cold preparation.

Make the rule explicit:

```text
same approved toolchain and preparation recipe
AND matching dependency inputs at the task commit
AND compatible runtime and host
AND successful task preparation probes
=> template reuse permitted
```

Compiler caches have their own keys.
Do not delete every cache because a source file changed.
Do not assume every cache includes all relevant environment inputs either.
Exercise each build system's invalidation with a changed source, flag, toolchain, and dependency.

### 8.6 Add live daemons only with evidence

A warm daemon may preserve a useful heap.
It may also preserve file descriptors, locks, sockets, timestamps, random-generator state, and paths tied to the seed checkout.

Restarting it after restore sacrifices some latency.
It reduces the number of states the restore barrier must understand.
The first implementation chooses that tradeoff.

Later, add a named `warm-gradle-daemon` profile only after repeated restores, commit changes, clock changes, and concurrent clones produce correct builds.
Keep the process inventory in the template manifest.

### Lab: change one dependency input

Bake a template, change a lockfile, and request a task at the new commit.
Confirm that the broker rejects incompatible reuse or performs an explicit dependency preparation.
Measure the result and retained cache benefit.

**Done when:** warmth has a validity rule and a stale dependency tree cannot silently become ready.

---

## 9. Bake and publish a consistent golden snapshot

A golden snapshot is a release artifact.
It represents a prepared machine at a particular point in time.

The release includes memory, VM state, backing disks, and the manifest that explains them.
Publishing one component early can create a release that never existed as a coherent machine.

### 9.1 Use a trusted publication boundary

The baker receives approved inputs and produces a candidate.
The publisher verifies it and makes it discoverable.

```text
reviewed inputs
    -> isolated preparation guest
    -> completed build and quiesced processes
    -> paused VM and snapshot capture
    -> captured backing disks
    -> hashes and compatibility record
    -> restore tests
    -> immutable publication
```

A failed candidate remains outside the registry of usable templates.
Consumers should never discover it by scanning half-written directories.

### 9.2 Quiesce before pausing

For the reference baker:

1. Complete dependency installation and the baseline tests.
2. Stop application servers, build daemons, and external clients.
3. Confirm that no repository operation is in progress.
4. Remove baker credentials and task-specific outputs.
5. Flush guest filesystem writes.
6. Disconnect external services and enter the control-only state.
7. Pause the VM.
8. Capture the full memory and VM-state snapshot.
9. Capture both disks without resuming or allowing another writer.

The disk images remain frozen until their publication copies are complete.
The baker can be discarded after capture.

Firecracker's snapshot guidance requires backing disks to correspond to the captured machine state; memory and VM state do not replace disk capture.[^firecracker-snapshots]
The reference sequence is our implementation policy for maintaining that relationship.

A separate disk copy made while the guest continues building is not an acceptable shortcut.
A successful copy operation cannot tell you whether it matches the captured memory.

### 9.3 Keep the API example narrow

For a configured, running VM, the snapshot operations resemble:

```bash
# API fragment. The controller supplies a private socket and validated paths.
curl --fail --silent --show-error \
  --unix-socket "$api_socket" \
  -X PATCH http://localhost/vm \
  -H 'Content-Type: application/json' \
  -d '{"state":"Paused"}'

curl --fail --silent --show-error \
  --unix-socket "$api_socket" \
  -X PUT http://localhost/snapshot/create \
  -H 'Content-Type: application/json' \
  -d '{
    "snapshot_type": "Full",
    "snapshot_path": "/capture/vmstate",
    "mem_file_path": "/capture/memory"
  }'
```

These are paths visible to the jailed process.
The host adapter provisions their backing locations and permissions.
Check the exact API schema shipped with the pinned runtime.[^firecracker-api]

The fragment does not capture the disks, publish an artifact, or establish successful restore.
Those are separate controller steps.

### 9.4 Publish the manifest

```yaml
template_id: typescript-service-template-17
manifest_schema: 1
status: candidate

inputs:
  repository: typescript-service
  seed_commit: resolved-full-object-id
  preparation_identity: recorded-digest
  base_recipe: approved-recipe-4

compatibility:
  runtime_binary_digest: recorded-digest
  jailer_binary_digest: recorded-digest
  host_cpu_profile: approved-profile
  guest_kernel_digest: recorded-digest
  controller_protocol: 1
  restore_policy: restore-policy-3

machine:
  resource_class: standard
  vcpus: 4
  guest_memory_mib: 6144

files:
  memory: {path: memory, digest: "sha256:...", size_bytes: 0}
  vmstate: {path: vmstate, digest: "sha256:...", size_bytes: 0}
  root: {path: root.ext4, digest: "sha256:...", size_bytes: 0}
  workspace: {path: workspace.ext4, digest: "sha256:...", size_bytes: 0}

guest:
  workspace_mount: /workspace
  controller_port: 7000
  preserved_task_processes: []
  credentials_present: false

evidence:
  cold_boot: evidence/cold-boot.json
  restore_smoke: evidence/restore-smoke.json
  parallel_clone: evidence/parallel-clone.json
  secret_hygiene: evidence/secret-hygiene.json
```

The zeros and ellipses are placeholders.
A publisher rejects them.
`credentials_present: false` is an attestation backed by a process and evidence, not a fact established by writing the field.

Publish into an immutable version directory.
Make the complete files durable before adding the template to the broker registry.
Retain a publisher identity or signature suitable for your artifact trust model.

### 9.5 Verify restoration as a release check

Restore the candidate in a fresh jail with private disk clones.
Run the same barrier and baseline probes used for tasks.
Then restore several siblings concurrently.

Check:

- The controller starts in preparation mode.
- The exact repository and dependency identities are present.
- Writing one clone leaves the candidate and other clones unchanged.
- The identity and network policy differ for each claim.
- No baker credentials are usable.
- Release and cleanup succeed.

A template is ready only after these checks pass.
Returning success from snapshot creation establishes that capture succeeded according to the runtime.
It does not establish that your full release is usable.

### 9.6 Keep the memory artifact alive

File-backed restore can share clean pages and create private copies when guests write.[^firecracker-snapshots]
The backing memory file belongs to every VM that still relies on it.

Record a live reference before restore.
Hold it through teardown and reconciliation.
Do not overwrite a memory file in place to publish a newer template.

A private memory mapping is not permission to recycle the backing artifact while consumers are alive.
Immutability and lifetime are part of the release contract.
Expose the memory and state artifacts with read-only permissions to the runtime identity.
Keep the publisher's writable artifact paths outside the jail.
The runtime receives write access to its private disk clones only.

### Lab: break one candidate

Deliberately pair VM state with the wrong workspace disk in an isolated candidate.
Confirm that identity validation or restore probes reject it.
Keep the broken candidate out of the usable registry.

**Done when:** publication requires a coherent release and successful fresh restores.

---

## 10. Restore into a new identity

Every clone begins with copied state.
Some of that state is precisely what makes restore useful.
Other state must become unique before the task can act.

The restore barrier makes that distinction explicit.

### 10.1 Model the readiness barrier

```text
template selected and pinned
    -> private disks cloned
    -> host identity and network slot allocated
    -> VM restored
    -> fresh guest-control connection established
    -> entropy and identity preparation
    -> task commit and dependencies validated
    -> scoped capabilities issued
    -> readiness probes pass
    -> task execution enabled
```

The sandbox does not become ready when the VM resumes.
It becomes ready when the broker records successful preparation for the current generation.

Keep external egress closed until the barrier reaches the step that requires a specific service.
Preparation fetches should use preparation-only capabilities.
The model capability arrives when task execution is enabled.

### 10.2 Know what reseeding cannot fix

Firecracker documents the risk of copied random state and recommends guest support for VM Generation ID where available.[^firecracker-uniqueness]
Kernel support and configuration must be tested with the actual guest.

Updating the kernel random pool does not reach into every already-running application's PRNG.
Changing the hostname does not recreate TLS sessions.
Rotating an SSH host key does not change the address layout of existing processes.

For the first implementation:

- Keep credentials out of the template.
- Restart credential-consuming applications after preparation.
- Stop task daemons before capture.
- Use tested clone notification support in the guest kernel.
- Generate task identifiers on the host.
- Establish fresh external connections after restore.

If you preserve a process, write down which state it retains and which state it must reset.
“Reseed entropy” is too vague to be a complete acceptance check.

### 10.3 Use a fresh host-provided generation

The broker provides a sandbox identifier, generation, fresh nonce, wall-clock sample, and preparation policy.
The guest acknowledges the nonce through the newly established channel.

```json
{
  "operation": "prepare",
  "protocol": 1,
  "sandbox_id": "sb-42",
  "generation": 1,
  "challenge": "fresh-unpredictable-host-value",
  "repository": "typescript-service",
  "task_commit": "resolved-full-object-id",
  "identity_profile": "coding-task",
  "restore_policy": "restore-policy-3"
}
```

The nonce proves freshness of this exchange within the controller protocol.
It is not attestation that arbitrary guest code is trustworthy.
The host still owns authorization and the readiness transition.

Generate hostname, task identity, and any needed machine identity before dependent services start.
Avoid SSH entirely if the guest channel is sufficient.
If SSH is required, keep it disabled in the candidate and create host keys after restore.

### 10.4 Repair time without trusting guest expiry

An old snapshot may contain stale wall-clock and application time state.
Synchronize time through a tested guest mechanism before TLS clients and time-sensitive code start.
Observe the resulting guest clock.

Lease expiry remains a host decision.
Use host time for durable expiry and a monotonic timer for in-process elapsed deadlines.
Handle host clock corrections conservatively.

A guest reporting that its lease has not expired does not extend the lease.
A restarted broker reconstructs deadlines from durable host records.

Time-sensitive daemons should restart after clock preparation.
Their own timers may not respond to a wall-clock update in the way you expect.

### 10.5 Reconnect vsock explicitly

Firecracker's host-side vsock support uses a Unix-domain socket and a connection request that selects a guest port.[^firecracker-vsock]
The vsock backing path must belong to the current sandbox generation.

External host connections are not a useful thing to bake into a template.
Open a new host connection after restore.
Use framed messages with protocol version, request identity, generation, maximum size, and explicit completion.

The guest agent must tolerate connection loss during capture and teardown.
The broker must tolerate a reply lost after command completion.
That ambiguity is handled by command state and result lookup, not by blindly rerunning commands.

### 10.6 Test parallel uniqueness

Restore ten clones and collect:

```text
sandbox identifier and generation
guest hostname and required machine identity
fresh preparation challenge acknowledgement
post-restore random samples from relevant processes
clock observation
credential capability identity
task commit
network policy identity
```

Different sample output is a smoke check.
It is not proof of cryptographic independence.
The evidence also needs the supported clone-notification mechanism and a clear process-restart policy.

Record failures before enabling task execution.
An identity fixup that logs an error and continues creates exactly the state this barrier is meant to prevent.

### 10.7 Load into a fresh runtime process

The host helper stages the two disk clones at the paths recorded by the template:

```bash
# Host fragment. Both directories come from trusted broker records.
# The private destination is newly allocated and the VM has not started.
cp --reflink=always -- "$template_dir/root.ext4" \
  "$private_disk_dir/root.ext4"
cp --reflink=always -- "$template_dir/workspace.ext4" \
  "$private_disk_dir/workspace.ext4"
```

The adapter launches a fresh jailed Firecracker process.
It does not apply the cold-boot machine configuration first.
Snapshot loading reconstructs that configuration.[^firecracker-api]

Keep a stable view inside each jail:

| Resource inside jail or namespace | Backing resource for this generation |
|---|---|
| `/disks/root.ext4` | Private root clone |
| `/disks/workspace.ext4` | Private workspace clone |
| `/template/memory` | Read-only pinned golden memory file |
| `/template/vmstate` | Verified golden VM state |
| `/run/control.vsock` | Private socket in this jail |
| `tap0` | TAP in this sandbox's network namespace |

The host paths differ for every generation.
The jailed names match the captured device configuration.
Validate path and device restoration requirements against the selected release.[^firecracker-snapshots]

The load and resume fragments are:

```bash
curl --fail --silent --show-error \
  --unix-socket "$api_socket" \
  -X PUT http://localhost/snapshot/load \
  -H 'Content-Type: application/json' \
  -d '{
    "snapshot_path": "/template/vmstate",
    "mem_backend": {
      "backend_type": "File",
      "backend_path": "/template/memory"
    },
    "resume_vm": false
  }'

curl --fail --silent --show-error \
  --unix-socket "$api_socket" \
  -X PATCH http://localhost/vm \
  -H 'Content-Type: application/json' \
  -d '{"state":"Resumed"}'
```

The complete adapter checks the load response before requesting resume.
It configures runtime logging for the new process and retains the host resource journal.
After resume, it opens a new guest connection and begins preparation.
Task egress and model authority remain disabled.

Neither API success marks the sandbox ready.
Only the completed barrier for the current generation does.

### Lab: restore an old template

Use a candidate captured earlier, restore several clones, and exercise time-sensitive clients and identity probes.
Restart the host-side guest controller connection.
Verify that no guest becomes ready without the current preparation exchange.

**Done when:** copied state cannot bypass the new task's readiness and authority checks.

---

## 11. Make networking a host policy

The guest can run as root inside its own environment.
Assume it can replace its resolver, change routes, and open raw connections.
Host policy must still hold.

The reference network gives each sandbox a separate namespace and a controlled path to services.
It permits the same guest-side network configuration across restored clones while keeping their host-side resources independent.

### 11.1 Allocate a complete network slot

Each slot includes:

```yaml
network_slot:
  sandbox_id: sb-42
  generation: 1
  namespace: broker-generated-name
  tap: broker-generated-name
  host_veth: broker-generated-name
  namespace_veth: broker-generated-name
  guest_network_profile: template-network-1
  host_transit_addresses: allocated-pair
  policy_id: coding-default-7
  proxy_capability_id: lease-bound-identity
```

The guest's internal IP and MAC can match the template.
Unique namespace, interface, transit routing, and rule ownership separate the host side.
Linux network namespaces isolate network devices, protocol stacks, routes, and related network state.[^network-namespaces]

The jailer joins the allocated namespace.
The namespace contains the guest TAP and its controlled service path.
The host does not bridge all guest TAPs together into a shared unrestricted network.

Creating a namespace alone does not establish policy.
Forwarding, local input, NAT, DNS, and IPv6 need explicit handling.

### 11.2 Prefer an explicit allowed path

For the first implementation, permit only:

- The narrow host-to-guest control channel through vsock.
- The mirror endpoint for the approved repository.
- The credential-aware model endpoint.
- A controlled package or artifact proxy when preparation requires it.
- A bounded result upload endpoint.

Everything else is denied by the host.
The guest cannot contact the host's management address just because NAT exists.

A hostname environment variable such as `HTTPS_PROXY` is a convenience for applications.
The firewall enforces that applications cannot bypass it.
Test a command that ignores proxy environment variables.

### 11.3 Define destination policy precisely

```yaml
policy: coding-default-7

direct_egress: deny
host_management: deny
sibling_networks: deny

services:
  git:
    repository: typescript-service
    operations: [fetch]
  packages:
    profile: approved-typescript-registries
  models:
    profile: coding-model-budget
  results:
    sandbox_scope: current-lease

address_rules:
  private_and_local_ranges: deny_except_explicit_services
  link_local: deny
  ipv6: deny_until_supported_and_tested
  dns: controlled_resolver_only
```

The example avoids a general Internet gateway.
If your workload needs broader access, make it a separate policy and record the reason.

For a destination proxy, validate the final address used for each connection.
Apply the address policy after DNS resolution.
Handle redirects and rebinding.
Block loopback, private, link-local, multicast, and host-owned addresses unless a named service exception applies.

Choose DNS and IPv6 behavior deliberately.
An IPv4 firewall with unrestricted IPv6 is an incomplete policy.
An allowed hostname that resolves to a host service is another route around intent.

### 11.4 Do not confuse domain access with safe content

An allowed package registry still serves executable dependencies.
An allowed Git host can receive confidential data if the guest has a write credential.
An allowed model endpoint can process data you did not intend to send.

Enforce operations and identities at the service where possible.
Use the repository mirror for reads.
Use a model adapter with an approved upstream and a request budget.
Keep result uploads scoped to the task's destination.

A domain allowlist controls destinations.
It does not evaluate repository instructions or decide which content is appropriate.

### 11.5 Test policies with direct probes

Run the negative tests from a guest with administrative access:

| Probe | Expected result |
|---|---|
| Connect to host management port | Denied |
| Connect to another sandbox's service | Denied |
| Connect to link-local metadata address | Denied |
| Ignore proxy settings and connect directly | Denied |
| Resolve allowed name to disallowed address | Denied |
| Follow redirect to private address | Denied |
| Use unapproved IPv6 destination | Denied |
| Fetch authorized repository through mirror | Allowed |
| Fetch another repository through mirror | Denied |
| Use revoked lease capability | Denied |

Observe both the guest result and host logs.
A timeout can mean a policy deny, a broken route, or a dead service.
The test needs to distinguish those outcomes.

### Lab: bypass the convenience settings

Unset proxy environment variables, change the guest resolver, and attempt direct connections through every supported address family.
Then use the approved service paths.

**Done when:** host enforcement remains correct when the guest cooperates with none of the convenience configuration.

---

## 12. Keep credentials behind narrow services

Putting secrets into a golden snapshot creates a secret distribution system.
Every clone receives them.
Artifact backups preserve them.
Deletion becomes difficult to establish.

The cleanest first rule is simple:

> Templates contain tools and prepared state. Active leases receive capabilities.

### 12.1 Separate three credential uses

| Use | Identity | Lifetime | Location |
|---|---|---|---|
| Mirror refresh | Read-only upstream repository identity | Rotated host-service credential | Refresher service |
| Template preparation | Narrow dependency or source access | One bake | Preparation service or guest tmpfs |
| Agent execution | Task-specific model and service capability | Active lease | Host service or guest tmpfs |

The upstream Git refresher does not need a write token.
The task does not need the refresher's credential.
The template publisher does not need the task's model capability.

If a private dependency requires authentication, route its preparation through a narrowly scoped service.
Stop the client and remove its session state before capture.
Where possible, build the publishable candidate in a fresh guest from authenticated download artifacts rather than snapshotting a process that handled a broad credential.

### 12.2 Prefer a service adapter to a general tunnel

A credential-aware model service accepts an authenticated task request, validates the permitted upstream and request shape, applies budgets, and adds the upstream credential itself.

It should strip caller-supplied authorization headers.
It should prevent arbitrary URL selection.
It should scope the task capability to the active sandbox generation.
It should revoke the capability when the lease ends.

A plain HTTPS `CONNECT` proxy cannot add an HTTP authorization header inside end-to-end TLS without terminating or otherwise mediating that connection.
Design the adapter around the actual protocol.
Test the client compatibility and trust configuration.

For a few known providers, a narrow API adapter is easier to explain than a general credential-injection proxy.
The guest points its client at that adapter.
The real provider key stays in the host service.

### 12.3 Scope the capability

```json
{
  "capability_id": "cap-42-model",
  "sandbox_id": "sb-42",
  "generation": 1,
  "audience": "model-adapter",
  "upstream_profile": "coding-provider",
  "expires_at": "host-issued-expiry",
  "limits": {
    "requests": 200,
    "concurrent_requests": 2,
    "budget_units": 1000
  }
}
```

The example uses abstract budget units because providers bill differently.
The adapter defines the unit and enforces it atomically.
Budget accounting must tolerate concurrent requests and uncertain upstream completion.

The service checks current capability state as well as cryptographic validity.
A signed token that remains valid after lease destruction can leave authority behind.
Short expiry is useful but does not replace immediate revocation when the task ends.

### 12.4 Handle unavoidable guest secrets

Some tools require a local credential.
Inject the narrowest available credential after restore into a dedicated tmpfs with restrictive ownership.
Pass its location through a controlled environment profile.
Keep it out of command arguments, logs, and exported manifests.

tmpfs reduces disk persistence.
It does not hide the secret from a compromised guest process or prevent that process from copying it elsewhere.
The credential must be safe enough to expose to that guest.

Avoid snapshotting an active task that contains credentials.
Task checkpointing is a separate feature with encrypted storage, strict access, short retention, and a recovery policy.
It is not part of the reference golden-template workflow.

### 12.5 Review hygiene with more than a grep

Search memory and disks for known test credentials as one check.
Also inspect:

- Process environment and arguments.
- Git remotes and credential configuration.
- Package-manager authentication files.
- Shell and tool history.
- Controller logs and crash reports.
- Application state retained in memory.
- Exported build reports.

A string scan can miss transformed, fragmented, or encrypted state.
Absence of one literal token is not evidence that a process never handled a credential.

Keep a process inventory and a credential-handling inventory for the baker.
The publication review should explain how each credential-bearing process is excluded or restarted.

### Lab: revoke an active capability

Start an agent call through the model adapter, revoke the lease capability, and attempt another call.
Check concurrent requests, cached connections, and retry behavior.

**Done when:** new authority use stops promptly and no upstream secret appears in the guest, template, or task logs.

---

## 13. Build the broker as a durable state machine

The broker translates a request into resources.
That translation spans a database, files, processes, networking, and guest operations.
It cannot be one atomic transaction.

Failures between those steps are normal.
Design recovery before adding a pool.

### 13.1 Separate lifecycle from lease

Use lifecycle states such as:

```text
requested -> preparing -> ready -> running -> exporting
                                        \-> stopping -> destroyed

preparing -> failed -> stopping -> destroyed
```

Lease state is separate:

```text
valid -> expired
valid -> revoked
```

A VM can still be running after its lease becomes expired.
The reconciler must drive it toward destruction.
Calling it destroyed because authority expired hides a resource leak.

Record the desired state and observed state separately where host operations can be delayed.

### 13.2 Persist intent before allocating

A minimal SQLite schema can begin with:

```sql
PRAGMA foreign_keys = ON;

CREATE TABLE sandboxes (
    id TEXT PRIMARY KEY,
    caller_id TEXT NOT NULL,
    request_key TEXT NOT NULL,
    request_digest TEXT NOT NULL,
    generation INTEGER NOT NULL,
    template_id TEXT,
    task_commit TEXT NOT NULL,
    resource_class TEXT NOT NULL,
    policy_id TEXT NOT NULL,
    state TEXT NOT NULL,
    lease_expires_at INTEGER NOT NULL,
    created_at INTEGER NOT NULL,
    updated_at INTEGER NOT NULL,
    UNIQUE (caller_id, request_key)
);

CREATE TABLE operations (
    id TEXT PRIMARY KEY,
    sandbox_id TEXT NOT NULL REFERENCES sandboxes(id),
    generation INTEGER NOT NULL,
    kind TEXT NOT NULL,
    state TEXT NOT NULL,
    attempt INTEGER NOT NULL DEFAULT 0,
    error_code TEXT,
    created_at INTEGER NOT NULL,
    updated_at INTEGER NOT NULL
);

CREATE TABLE resources (
    kind TEXT NOT NULL,
    resource_id TEXT NOT NULL,
    sandbox_id TEXT NOT NULL REFERENCES sandboxes(id),
    generation INTEGER NOT NULL,
    desired_state TEXT NOT NULL,
    observed_state TEXT NOT NULL,
    PRIMARY KEY (kind, resource_id)
);

CREATE TABLE commands (
    sandbox_id TEXT NOT NULL REFERENCES sandboxes(id),
    generation INTEGER NOT NULL,
    request_id TEXT NOT NULL,
    request_digest TEXT NOT NULL,
    state TEXT NOT NULL,
    result_id TEXT,
    PRIMARY KEY (sandbox_id, generation, request_id)
);
```

This is a starting schema.
Production code also needs state validation, authorization, template references, capacity reservations, indexing, backups, and bounded retention.

SQLite WAL mode permits readers to coexist with a writer while retaining a single-writer model.[^sqlite-wal]
Keep transactions short.
Do not hold a write transaction open while waiting for a guest or copying an image.

Use the database through its supported backup mechanism.
Copying only the main database file from a live WAL deployment can miss committed state.

### 13.3 Make claim retries idempotent

The caller supplies a request key.
Scope it to the authenticated caller.
Compute a digest of the accepted request.

```text
new caller/key:
    persist accepted request and one sandbox identity

existing caller/key with same digest:
    return existing sandbox and operation status

existing caller/key with different digest:
    reject as request-key conflict
```

Do not silently change the commit, resource class, or policy on a retried key.
The caller can issue a new key for a new request.

If the claim times out, the caller looks up the existing operation.
It does not assume the host did nothing.

### 13.4 Use compare-and-set transitions

```sql
UPDATE sandboxes
SET state = 'ready', updated_at = :now
WHERE id = :id
  AND generation = :generation
  AND state = 'preparing'
  AND lease_expires_at > :now;
```

The application requires exactly one changed row.
Zero means the transition lost a race, expired, or addressed a stale generation.
It should inspect current state before deciding what to do.

Generation identifiers fence delayed callbacks.
An old restore completion must not enable a recycled sandbox identity.
Use generation checks in guest commands, host resource operations, and capabilities.

### 13.5 Journal the resource sequence

An operation can progress through:

```text
reserve capacity
pin template reference
allocate private paths and identity
clone disks
allocate network slot
launch jailed runtime
restore and prepare guest
issue capabilities
mark ready
```

Each step records intent and completion.
After a crash, reconciliation asks what exists and resumes or compensates.

Avoid treating “operation failed” as the end.
A failure after network allocation still needs network teardown.
A failure after issuing a capability still needs revocation.

### 13.6 Handle commands differently from allocation

The controller can make disk allocation repeatable.
It cannot make an arbitrary task command safe to repeat.

A command may have completed its external side effect before the connection dropped.
Record the command as `unknown` when completion cannot be established.
Offer result lookup and explicit caller retry policy.

Where possible, commands themselves use idempotent operation identities.
For general shell execution, preserve ambiguity rather than claiming exactly-once behavior.

A second invocation of “publish this package” is not a harmless recovery step.
This book keeps publication outside the guest's authority in the first place.

### 13.7 Keep renewal bounded

Renewal extends an existing valid lease within caller policy.
It does not resurrect a revoked lease or reset the task's maximum lifetime.

Check caller identity, lease token, generation, current state, and maximum expiry.
Persist the new expiry before reporting success.
The reaper and credential services use the durable record.

Test renewal at the expiry boundary.
A guest running when renewal arrives does not by itself establish permission to keep it.

### Lab: kill the broker between steps

Interrupt the broker after disk cloning, after VM launch, and after readiness probes.
Restart it and retry the same claim key.
Observe the operation journal and resource inventory.

**Done when:** each request has one accountable environment or a recorded failure whose resources are being reclaimed.

---

## 14. Admit work the host can finish

Ten waiting agents and ten compilers are different loads.
Agent count alone cannot drive admission.

Start with resource classes derived from observed repository behavior.
Add dynamic controls only when the measurements justify them.

### 14.1 Define classes

```yaml
resource_classes:
  small:
    guest_memory_mib: 2048
    vcpus: 2
    disk_growth_reserve_mib: 4096
    build_slot_cost: 1
  standard:
    guest_memory_mib: 6144
    vcpus: 4
    disk_growth_reserve_mib: 12288
    build_slot_cost: 2
  large:
    guest_memory_mib: 12288
    vcpus: 8
    disk_growth_reserve_mib: 24576
    build_slot_cost: 4
```

These are illustrative starting classes.
Replace them with measurements from the TypeScript and Gradle workloads.
The largest class may be inappropriate for the first host.

Bake and select a snapshot for the requested resource class.
Its recorded guest RAM and vCPU configuration must match the reservation.
The snapshot path does not turn a large captured machine into a small machine because the claim says `small`.
If you later support device-based memory changes, define and test that separate capability.

Reserve for bakers and pools too.
A warm VM consumes capacity even when no caller has claimed it.

### 14.2 Check memory and disk headroom

For a conservative first service, reserve the configured guest memory plus a measured runtime allowance for every admitted VM.
Shared clean pages can improve actual residency, but they should not be required to avoid host OOM.

The disk budget needs more than the apparent image size:

```text
physical disk use
  = retained template blocks
  + private changed blocks
  + memory artifacts
  + exports and logs
  + candidate bakes
  + maintenance reserve
```

Reflink clones start cheaply.
Builds dirty them.
Old templates can retain blocks even after a private clone is removed.
Measure filesystem usage and free space as well as per-file accounting.

### 14.3 Implement a pure admission check

```python
from dataclasses import dataclass


@dataclass(frozen=True)
class HostBudget:
    memory_limit_mib: int
    memory_reserved_mib: int
    disk_available_mib: int
    disk_floor_mib: int
    build_slots_total: int
    build_slots_reserved: int


@dataclass(frozen=True)
class ClaimCost:
    guest_memory_mib: int
    runtime_allowance_mib: int
    disk_growth_reserve_mib: int
    build_slot_cost: int


def admission_failures(budget: HostBudget, cost: ClaimCost) -> list[str]:
    """Complete pure check; reservation and allocation are separate steps."""
    values = (
        budget.memory_limit_mib,
        budget.memory_reserved_mib,
        budget.disk_available_mib,
        budget.disk_floor_mib,
        budget.build_slots_total,
        budget.build_slots_reserved,
        cost.guest_memory_mib,
        cost.runtime_allowance_mib,
        cost.disk_growth_reserve_mib,
        cost.build_slot_cost,
    )
    if any(type(value) is not int or value < 0 for value in values):
        raise ValueError("budget values must be nonnegative integers")
    if cost.guest_memory_mib == 0 or cost.build_slot_cost == 0:
        raise ValueError("a claim must reserve memory and build capacity")

    failures = []
    memory_cost = cost.guest_memory_mib + cost.runtime_allowance_mib
    if budget.memory_reserved_mib + memory_cost > budget.memory_limit_mib:
        failures.append("memory")
    if budget.disk_available_mib - cost.disk_growth_reserve_mib < budget.disk_floor_mib:
        failures.append("disk")
    if budget.build_slots_reserved + cost.build_slot_cost > budget.build_slots_total:
        failures.append("build_slots")
    return failures
```

The function makes the arithmetic inspectable.
It does not prevent concurrent over-admission by itself.
The broker must check and reserve against one authoritative ledger in a transaction.
Two claims cannot both spend the same free headroom.

Disk observations can change outside the ledger.
Retain a hard free-space floor and a separate circuit breaker.
Reconcile reservation estimates with actual growth.

### 14.4 Limit guest and host consumption

The guest memory size is part of VM configuration.
The host cgroup also needs room for the VMM and relevant charged memory.
Configure CPU and I/O controls for the runtime.

Linux cgroup v2 distinguishes `memory.high`, a pressure and throttling boundary, from `memory.max`, a hard limit that can lead to OOM handling.[^cgroup-v2]
Record the counters and outcomes.
Do not report every cgroup memory event as a guest application OOM.

A process limit on the host VMM cgroup does not count all tasks running inside the guest kernel.
Enforce guest process limits inside the guest if the workload needs them.
Host CPU and memory controls remain the outer bound.

Enable a supported balloon device only after testing its behavior with your workloads.
Treat ballooning as a measured optimization.
It does not justify ignoring the working-set peaks of simultaneous builds.

### 14.5 Separate conversation and build capacity carefully

A later broker can admit more waiting agents while restricting expensive builds through a build-slot service.
That can increase useful concurrency.

It requires the tasks to cooperate with build admission or the host to impose equivalent limits.
An arbitrary shell can start a compiler without calling your semaphore.
The first version therefore reserves a full resource class for the task and uses host controls.

Add cooperative build slots as an optimization when the task runner owns the build entry points.
Keep the outer resource budget conservative.

### 14.6 Add a pool after measuring demand

A pool can avoid preparation work on the claim path.
It also consumes memory, disk, and network resources.

Start with no restored pool.
Measure restore and preparation under realistic concurrency.
Add one or two control-only VMs for a hot repository if that measurably improves readiness.

Pooled VMs have no task credentials.
They pass the full restore and task preparation barrier when claimed.
They count against admission before allocation.
They have a maximum age and are discarded when the template or policy retires.

A pool of twenty obsolete environments is unused capacity.
Track hit rate, wasted preparation, and displacement of active work.

### Lab: reject the eleventh claim cleanly

Choose a budget that admits ten measured workloads.
Submit an eleventh while builds run.
Confirm it queues or fails with a resource reason, then proceeds after release.

**Done when:** admission protects the host and reports which budget prevented the claim.

---

## 15. Preserve results before destroying the sandbox

A sandbox is disposable.
The work may not be.

Deleting the VM before its patch leaves the guest turns cleanup into data loss.
Keeping the VM forever because it contains the only result turns a temporary environment into an accidental workstation.

Define an export contract.

### 15.1 Choose a result bundle

For a coding task, preserve:

```text
result/
  manifest.json
  tracked.patch
  untracked-inventory.json
  approved-untracked/
  test-results.json
  commands.jsonl
  transcript-reference.json
```

The result manifest records:

```json
{
  "result_id": "result-42",
  "sandbox_id": "sb-42",
  "generation": 1,
  "repository": "typescript-service",
  "base_commit": "accepted-task-commit",
  "template_id": "typescript-service-template-17",
  "task_state": "completed",
  "export_state": "verified",
  "verification": "guest-reported-tests",
  "files": [
    {"name": "tracked.patch", "digest": "sha256:...", "size_bytes": 0}
  ]
}
```

The verification field matters.
Tests reported by the same guest that produced the patch are useful task evidence.
They are not independent acceptance of the patch.

### 15.2 Export changes relative to the accepted base

A simple tracked patch can be produced with:

```bash
# Controller fragment for a validated base_commit in the task repository.
git diff --binary "$base_commit" -- . > /run/export/tracked.patch
```

This includes the current tracked working-tree result relative to the base, including changes the agent committed locally.
It does not include untracked files.
It does not preserve an entire commit history.

Inventory untracked files separately.
Select allowed files by the export policy and task requirements.
Apply size limits, path validation, and explicit handling of symlinks and special files.

If commit history is a required result, use a separately specified Git bundle workflow and validate it in a disposable receiver.
Do not assume one patch represents every useful artifact.

### 15.3 Treat exports as hostile input

The exporter must not unpack an arbitrary guest tarball into a privileged host directory.
It must not follow guest-selected symlinks or trust executable permission bits.

Prefer a bounded file protocol:

1. Guest declares normalized relative path, length, and digest.
2. Exporter validates policy and aggregate limits.
3. Exporter writes into a dedicated non-executable result area.
4. Exporter verifies length and digest.
5. Manifest publication occurs after all selected files are durable.

Reject absolute paths, parent traversal, duplicate paths, unsupported file types, and unexpected stream lengths.
Use safe filesystem APIs that resist symlink races.
The export root is chosen by the host.

Do not mount a task-controlled ext4 filesystem on the host simply to read a patch.
That exposes another parser and privileged filesystem path to guest-controlled input.
Use the guest channel or an unprivileged inspection environment.

### 15.4 Check the patch elsewhere

Apply the result to a clean clone at the base commit in a separate verification environment.
Run the relevant checks with the same ordinary limits and a fresh capability.

The reviewer receives:

```text
accepted base commit
exported change
guest task evidence
independent verification evidence
known gaps
```

Independent verification should not reuse the task's writable cache unquestioningly.
It can use a trusted template.
It should revalidate inputs and avoid outputs that the task could have poisoned.

### 15.5 Revoke authority before final teardown

The end sequence is:

```text
stop new task commands
revoke model and source-write capabilities
export through a bounded export-only capability
verify and publish the result
close the export capability
stop the VM
reclaim resources
mark destroyed after observation
```

The guest may keep running briefly for export.
It no longer has general task authority.
The export service only accepts the bounded result bundle.

At lease expiry, attempt a bounded best-effort export if policy permits.
Do not extend all authority indefinitely while waiting for a large upload.
Record missing or partial output honestly.

### 15.6 Preserve work during long tasks

For tasks that run for hours, export periodic checkpoints of code changes and command metadata.
Keep them separate from completed results.

A checkpoint needs its own base commit, timestamp, sandbox generation, and completeness status.
It may contain a half-written file.
It is useful for recovery, not automatically suitable for merging.

Measure the loss window.
If checkpointing runs every fifteen minutes, describe that as the intended maximum interval between attempted exports, then test whether congestion changes it.
Do not promise zero data loss from a timer configuration.

### Lab: lose the model endpoint

Disable the model service after the agent makes a source change.
Export the work, destroy the guest, and recover the patch in a clean clone.
Then submit a malicious export path and an oversized file.

**Done when:** useful work survives and the export boundary rejects unsafe payloads.

---

## 16. Reconcile leaks and collect unused artifacts

A caller should release its sandbox.
The service must work when it does not.

Cleanup is a control loop over desired and observed state.
It should run independently of the code path that created the resources.

### 16.1 Inventory owned resources

Track:

| Resource | Ownership identity | Teardown evidence |
|---|---|---|
| Jailed VMM process | Sandbox, generation, process start identity | Process and cgroup empty |
| API and vsock sockets | Private generation directory | Sockets absent |
| Private root and workspace disks | Resource ledger and private paths | Files absent after VM exit |
| Network namespace and interfaces | Allocated slot and generation | Devices and namespace users absent |
| Host policy entries | Policy handle and sandbox generation | Rules removed |
| Capabilities | Capability identifiers | Revoked and rejected |
| Capacity reservation | Broker ledger | Released after resource accounting |
| Template reference | Template and consumer generation | No live or pending consumer |
| Results | Result manifest | Retained under separate policy |

Do not identify a process by PID alone.
PIDs can be reused.
Compare process start identity and the owned cgroup, or use an appropriate process handle.

Do not delete directories merely because their names begin with `sandbox`.
Ownership must come from trusted records and verified location.

### 16.2 Reconcile in both directions

The reaper compares:

```text
database intent -> expected host resources
host resources -> accountable database owner
```

The first direction finds missing VMs.
The second finds orphans.
You need both.

For an expired lease:

1. Revoke capabilities.
2. Reject renewals and new commands.
3. Attempt policy-bounded export.
4. Stop the runtime gracefully within a deadline.
5. Terminate it if necessary.
6. Confirm that the runtime has exited.
7. Tear down networking and private files.
8. Release capacity and template references.
9. Mark destruction complete.

The sequence can be retried.
Absent resources are successful cleanup outcomes.
Unexpected resources or mismatched ownership produce an alert.

### 16.3 Treat host reboot as reconciliation

After a reboot, active VMs are gone in the reference service.
Private disks and ledger rows may remain.

The broker should start in reconciliation mode before enabling claims.
Revoke stale capabilities, classify incomplete tasks, retain available exports, clean orphaned runtime resources, and recompute capacity.

Do not automatically resume every old task.
The task's external state and authorization may have changed.
The caller can issue a new claim from preserved work.

### 16.4 Collect templates through a reference protocol

Template GC must serialize against claim selection:

```text
claim transaction:
    verify template is usable
    create consumer reference
    reserve capacity

GC transaction:
    verify no pending or live consumer references
    mark template retiring

after retirement:
    no new claims permitted
    delete only after reconciled consumers are gone
```

A separate live-resource scan checks the ledger.
The collector must not rely only on a reference count that a crashed allocation path could have failed to update.

Keep the template pinned through ambiguous teardown.
Releasing a memory artifact while a runtime still uses it is worse than retaining it a little too long.
Alert on stale references and reconcile them.

### 16.5 Make low disk space change behavior

Free space crossing a threshold should first stop new claims and new bakes that would worsen the condition.
Continue cleanup and preserve management capacity.

Retire unreferenced candidates and templates according to policy.
Do not delete the oldest file blindly.
It may be the backing memory for the oldest active task.

A template's retention policy differs from a task result's retention policy.
A reproducible template can be rebuilt.
An unreviewed patch may be the only copy of useful work.

### 16.6 Test cleanup repeatedly

A soak test should mix:

- Normal releases.
- Caller disconnects.
- Preparation failures.
- Expired leases.
- Broker restarts.
- Guest hangs.
- Export timeouts.
- Template retirement attempts.

Run hundreds of claim/release cycles initially.
The research proposes five hundred as an early exit criterion.
Keep that as a useful starting exercise, then choose longer tests from actual task rates.

Compare baseline and final owned resources.
Disk usage need not return byte-for-byte because logs, results, and retained templates legitimately grow.
Account for that growth explicitly.

### Lab: leave an orphan deliberately

Interrupt the creation path after host resources exist but before completion.
Let the independent reaper run.
Repeat with a missing ledger record in an isolated test setup.

**Done when:** the reconciler either reclaims the resource safely or produces a bounded, actionable ownership exception.

---

## 17. Measure the system the agents actually use

Benchmarking should explain where time and capacity go.
A single startup number cannot do that.

### 17.1 Record phase timings

```json
{
  "claim_id": "claim-42",
  "repository": "typescript-service",
  "commit": "accepted-task-commit",
  "template_id": "typescript-service-template-17",
  "runtime": "firecracker",
  "path": "snapshot",
  "concurrent_claims": 10,
  "durations_ms": {
    "queue": 0,
    "disk_clone": 0,
    "host_setup": 0,
    "runtime_restore": 0,
    "identity_barrier": 0,
    "source_update": 0,
    "dependency_validation": 0,
    "baseline_test": 0,
    "total_to_first_passing_test": 0
  },
  "outcome": "pass",
  "resource_record": "metrics/claim-42.json",
  "evidence_record": "evidence/claim-42.json"
}
```

Populate the measurements from a monotonic clock.
Record the exact definition of each boundary.
Queue time begins before the worker starts.
Failed attempts remain in the dataset.

### 17.2 Test cache states separately

Use a matrix:

| State | Purpose |
|---|---|
| Cold guest, cold workspace | Recovery baseline |
| Cold guest, baked workspace | Value of warmed files |
| Restored memory, baked workspace | Value of memory restore |
| Restored memory, new source with same dependencies | Typical task freshness |
| Changed dependency inputs | Invalidation and preparation cost |
| Warm pool hit | Value of pre-allocation |
| Pool miss at concurrency | Behavior under burst demand |

Distinguish host page-cache warmth from guest template warmth.
Repeatedly restoring the same memory file can make host I/O look cheaper.
That can be realistic for a hot repository.
It is not the same as the first restore after a host reboot.

Do not flush host caches on a shared production machine to produce a benchmark.
Use a dedicated test window or test host and document the procedure.

### 17.3 Measure memory after the build

Collect:

- Configured guest RAM.
- Host cgroup memory counters and events.
- VMM resident and proportional memory where available.
- Guest memory pressure and application OOM evidence.
- Host free memory and pressure.
- Private disk growth.
- Filesystem free space.

Adding VMM RSS values can count shared pages repeatedly.
Use proportional observations and host-level totals to explain sharing.
Reconcile them with cgroup accounting rather than presenting one measure as the whole truth.

A near-idle clone may share substantial memory.
A compiler that rewrites its heap changes that result.
Measure after a representative build and at peak concurrency.

### 17.4 Compare throughput fairly

Run the same commit, toolchain, tests, and network policy across:

```text
current container workflow
Firecracker cold path with baked files
Firecracker snapshot path
Incus fallback if implemented
```

Keep CPU allocation and storage placement comparable.
Report configurations where comparison is impossible.
A container allowed to use the entire host is not an equivalent baseline for a guest limited to four vCPUs.

Throughput is completed useful tasks per interval.
If a system starts fifty guests quickly and then takes twice as long to finish their builds, the startup chart has concealed the tradeoff.

### 17.5 Keep correctness beside latency

For every performance run, also record:

- Correct task commit.
- Successful dependency validation.
- Baseline test outcome.
- Policy probes.
- Export outcome.
- Cleanup outcome.
- Fallback use.

Do not exclude failed restores from latency analysis without showing their count.
Report successful latency distributions, failure rate, and end-to-end completion behavior.

A p95 without sample count and workload description is difficult to interpret.
Use the same test corpus when comparing revisions.

### 17.6 Observe service health

The operational dashboard can begin with:

```text
active leases by lifecycle state
pending claims and oldest queue age
host capacity reserved and observed
template use and retirement backlog
prepare and restore failure categories
leases past expiry with resources still alive
export failures and partial results
orphan resources
free disk space and memory pressure
model capability denials and budget exhaustion
```

Link an unhealthy row to the operation journal and relevant evidence.
A red “sandbox failed” badge is less useful than “restore succeeded; dependency identity mismatch; task never enabled.”

Keep high-volume command logs out of the main dashboard.
Preserve references and bounded summaries.
Redact credentials at the producer and again at collection.
Restrict transcript retention and access because source and task content can be sensitive.

### Lab: explain one slow claim

Choose a slow claim from the TypeScript repository.
Explain its queue, preparation, build, and export time from recorded evidence.
Repeat for Gradle.

**Done when:** a performance regression has an identified phase, comparable inputs, and a correctness outcome.

---

## 18. Upgrade deliberately and keep a fallback

The sandbox host runs code that may be malformed or adversarial.
Patch management is part of the operating model.

Pinned artifacts make behavior explainable.
They do not make old binaries safe forever.

### 18.1 Upgrade the compatibility unit

Treat these as a tested unit:

```text
host kernel and CPU compatibility record
Firecracker and jailer binaries
guest kernel
base root filesystem
guest controller protocol
template recipe and restore policy
snapshot artifacts
```

The conservative policy in this book accepts a template only under the recorded runtime binary and approved host CPU profile.
Even when upstream supports a broader compatibility path, widening that policy requires a local restore test.

Do not frame version compatibility as an assumption about semantic version numbers.
Read the pinned release's snapshot compatibility documentation and test it.[^firecracker-snapshots]

### 18.2 Rebuild alongside the active generation

An upgrade can proceed:

1. Verify the candidate runtime and guest inputs.
2. Build new templates with the candidate runtime.
3. Run cold, restore, policy, export, and cleanup checks.
4. Canary a small set of real tasks.
5. Stop new claims on the old generation.
6. Drain active tasks within a bounded period.
7. Retire old templates and binaries after consumers are gone.

Keep artifact paths generation-specific.
Do not replace the binary under an active runtime adapter's expected identity.
Do not rewrite old template files in place.

Firecracker's release policy describes supported branches and security-fix handling.[^firecracker-releases]
Use that policy and the project's advisories to set patch deadlines.
If a security issue requires immediate retirement, availability may yield to containment.
Record that decision and preserve task exports where possible.

### 18.3 Keep a cold path with the same boundaries

No usable snapshot can mean:

- New repository.
- Retired runtime.
- Corrupt artifact.
- Failed compatibility check.
- Changed dependency inputs.
- Publication failure.

The cold path boots approved kernel and disk inputs, prepares the repository privately, and passes the same readiness barrier.
It gets the same network policy, capabilities, result export, and teardown.

A fallback that drops the jailer or opens unrestricted egress is a different security tier.
Do not use it silently.

Cold fallback also needs admission.
It may consume more CPU, network traffic, and disk than a snapshot restore.
Queue its bake work separately rather than launching an unbounded rebuild for every miss.

### 18.4 Use Incus as an operational fallback

Incus VMs on ZFS are the research's plan B.
Their appeal is a managed VM lifecycle and storage API.
The same broker contract can drive an Incus runtime adapter.

Start by preparing versioned, disk-warm VM templates.
Clone them through Incus and wait for actual guest readiness.
Use a small prebooted pool only after measuring cold readiness.

Incus documents ZFS copy behavior and constraints involving clone descendants.[^incus-zfs]
Treat template generations as immutable and exercise deletion and retirement through Incus.
Do not mutate underlying ZFS objects behind the manager.

Incus also documents stateful VM operations through `migration.stateful`.[^incus-stateful]
Test them as a distinct capability.
Do not assume they provide the same many-clone memory-sharing behavior as the Firecracker file-backed restore design.

Incus's administrative access is powerful.[^incus-security]
Keep the guest and caller away from its management socket.
The adapter chooses approved profiles and resources.

### 18.5 Keep the fallback promise small

The fallback acceptance target is:

```text
same accepted repository and commit
same private workspace guarantee
same host-enforced service policy
same leased authority
same export format
same cleanup evidence
different readiness and capacity measurements
```

Runtime-specific capabilities can differ.
Expose the difference.

Do not promise that an active Firecracker task can migrate into an Incus VM.
For this book, fallback means a new task environment recovered from source and exported work.

### 18.6 Use gVisor as an explicit interim tier

If KVM is unavailable, a gVisor-backed container can offer an interim path.
Test application compatibility, filesystem behavior, syscall-sensitive tools, and the same service boundaries.

Remove the host Docker socket from task access.
Use private checkouts and caches.
Keep the broker's authority model unchanged.
Record the runtime tier in every result.

When Docker is required inside a Firecracker guest, run its daemon inside that guest.
Do not “solve” a compatibility problem by handing the guest the host's daemon socket.
The host boundary is the reason to use the VM.

### Lab: retire the primary path

Mark the current template unusable.
Request a task and verify cold fallback or the supported Incus adapter.
Restore the primary only through the normal publication process.

**Done when:** failure changes latency and capability honestly without bypassing the authority contract.

---

## 19. Roll out through evidence

The research proposes a short adoption spike followed by four implementation phases.
That is a useful sequence.
The dates are planning estimates.
The exit criteria decide whether the next phase starts.

### 19.1 Phase zero: baseline and adoption decision

Use two representative repositories.
Measure the current workflow and one candidate platform.
Write the operating contract and threat model.
Check the target host.

Deliver:

```text
baseline reports
host capability record
candidate platform evidence
adopt-or-build decision
initial authority and network policy
```

**Exit when:** the chosen path supports the required contract or the missing work is explicit.

If the candidate meets the contract, use its maintained lifecycle instead of rebuilding it for aesthetic control.
Keep the book's checks and result format.

### 19.2 Phase one: one cold guest and one complete task

Build the kernel and base image.
Launch through the jailer.
Connect the guest controller.
Fetch the exact commit, run a baseline test, export a small change, and destroy the guest.

Deliver:

```text
reviewed guest recipe
runtime and helper configuration
cold-path evidence
first result bundle
observed resource cleanup
```

**Exit when:** one task crosses every boundary with no manual recovery hidden in the procedure.

Do not begin with a warm pool.
A pool can conceal a broken startup path.

### 19.3 Phase two: published templates and parallel clones

Implement the baker and publisher.
Capture consistent memory and disks.
Record compatibility and input identities.
Implement restore preparation and network-slot allocation.

Run ten clones concurrently.
Use the same task commit first, then several newer commits and one changed dependency graph.

Deliver:

```text
immutable template release
parallel restore evidence
uniqueness and credential checks
cross-sandbox negative tests
cache invalidation report
```

**Exit when:** all clones produce correct baseline outcomes with private writable state and no copied authority.

### 19.4 Phase three: durable claims and bounded authority

Build the request-key handling, operation journal, reservation ledger, renewal, and expiry.
Add mirror, model, and export capabilities.
Exercise interruptions at every allocation step.

Deliver:

```text
claim/retry contract
command ambiguity policy
lease-bound capability tests
broker restart evidence
independent verification workflow
```

**Exit when:** a client disconnect or broker restart cannot create unaccountable resources or extend authority silently.

### 19.5 Phase four: soak, maintenance, and capacity

Run mixed claim/release cycles.
Add orphan detection and template GC.
Exercise low disk space and host reboot.
Measure peak build load.
Add a pool only if it improves the chosen metric.

Deliver:

```text
soak report
capacity and resource-class record
backup restore evidence
upgrade and rollback rehearsal
tested fallback
operator runbook
```

**Exit when:** maintenance and failures have a recorded path that does not depend on remembering how the system was built.

### 19.6 Grow from ten toward fifty

Raise concurrency one measured step at a time.
For each step, compare time to first passing test, completed-task throughput, peak pressure, queue age, failure rate, and cleanup lag.

If throughput flattens, identify the bottleneck.
It may be CPU, memory bandwidth, storage, a proxy, an upstream rate limit, or a build system that starts too many workers.

The remedy may be fewer simultaneous builds.
It may be larger classes for Gradle and smaller classes for editing tasks.
It may be a second host.

Do not protect the original one-host plan at the expense of the task's actual requirements.
The plan is useful while the evidence supports it.

### Lab: write the go/no-go record

After each phase, write what passed, what failed, which evidence supports the conclusion, and which restrictions remain.
Pick the next phase's acceptance owner.

**Done when:** increased concurrency follows demonstrated capacity and the service's limitations are visible to its users.

---

## 20. Appendix: contracts, drills, and verification notes

The following records make the implementation reviewable.
Keep their schemas versioned.
They are small enough to use before a dashboard exists.

### A. Claim request and response

```json
{
  "schema": 1,
  "request_key": "caller-generated-stable-key",
  "repository": "typescript-service",
  "commit": "resolved-full-object-id",
  "resource_class": "standard",
  "policy": "coding-default",
  "requested_lifetime_seconds": 1800
}
```

The authenticated caller identity is supplied by the service transport.
It is not trusted because it appears in the JSON body.

```json
{
  "schema": 1,
  "sandbox_id": "sb-42",
  "generation": 1,
  "operation_id": "op-42-prepare",
  "state": "preparing",
  "runtime": "firecracker",
  "template_id": "typescript-service-template-17",
  "accepted_commit": "resolved-full-object-id",
  "lease_expires_at": "host-issued-expiry",
  "status_reference": "service-defined-status-reference"
}
```

The accepted lifetime can be shorter than requested.
The response reports the durable decision.

### B. Readiness evidence

```yaml
readiness:
  sandbox_id: sb-42
  generation: 1
  template_digest_verified: true
  runtime_compatible: true
  private_disks_verified: true
  network_policy_installed: true
  fresh_control_exchange: true
  clone_identity_policy_passed: true
  clock_probe_passed: true
  exact_commit_verified: true
  dependency_inputs_verified: true
  baseline_probe_passed: true
  task_capabilities_issued: true
  lease_valid_at_transition: true
  evidence_record: evidence/sb-42-generation-1.json
```

Each boolean represents a specific probe or operation.
The record is emitted only after checking the current generation.
A copied template record cannot satisfy it.

### C. Failure taxonomy

| Category | Example | Expected response |
|---|---|---|
| Admission | Memory budget exhausted | Queue or reject without allocation |
| Source | Accepted commit unavailable | Fail preparation; preserve request evidence |
| Template | Hash mismatch | Quarantine candidate; do not restore |
| Compatibility | Unsupported host profile | Select tested alternative or fail |
| Restore | Runtime fails to load | Reclaim resources; bounded fallback |
| Identity | Preparation probe fails | Keep task disabled |
| Dependency | Lockfile incompatible | Private preparation or another template |
| Policy | Forbidden destination requested | Deny and record |
| Capability | Lease expired or budget exhausted | Reject authority use |
| Command | Connection lost after execution | Preserve unknown outcome; query result |
| Export | Size or path violation | Reject payload; retain bounded evidence |
| Cleanup | Runtime fails to stop | Escalate termination and reconcile |
| Host | Reboot or disk failure | Recover service; restart tasks from preserved work |

The taxonomy should distinguish a workload failure from an infrastructure failure.
A failing application test does not necessarily mean the sandbox is broken.
A dependency mismatch before the agent starts does not belong in the agent's code-quality score.

### D. Failure drills

| Drill | Invariant |
|---|---|
| Repeat identical claim key | One accepted allocation |
| Reuse key with changed commit | Conflict; original request preserved |
| Lose reply after claim acceptance | Existing operation discoverable |
| Expire lease during restore | Task authority never enabled |
| Kill broker after VM launch | Resource reconciled to durable intent |
| Corrupt a template file | Restore refused |
| Change dependency input | Stale installed tree not silently accepted |
| Restore clones concurrently | Independent writes and fresh identities |
| Bypass proxy environment | Direct egress still denied |
| Resolve approved name to local address | Final address policy enforced |
| Revoke model capability | New use denied |
| Export parent traversal path | Host result root preserved |
| Fill disk toward reserve floor | New allocations stop; cleanup continues |
| Retire a live template | Consumer pin prevents deletion |
| Lose a command completion reply | Outcome marked unknown |
| Reboot host | Stale authority revoked; resources reconciled |
| Break snapshot restore | Tested cold path or declared failure |

Run drills in an isolated test environment.
The destructive condition is the subject of the experiment.
It should not be improvised against active work.

### E. Maintenance record

```yaml
maintenance:
  change_id: runtime-generation-5
  owner: sandbox-maintainer
  reason: reviewed-update
  old_runtime_digest: recorded-digest
  new_runtime_digest: recorded-digest
  host_profile: approved-profile
  guest_kernel_digest: recorded-digest
  affected_templates: []
  evidence:
    cold_path: reports/cold.json
    restore: reports/restore.json
    isolation: reports/isolation.json
    export: reports/export.json
    cleanup: reports/cleanup.json
    soak: reports/soak.json
  canary_outcome: recorded-outcome
  drain_deadline: approved-deadline
  rollback_artifacts: retained-identities
  security_exception_expiry: null
```

Rollback requires compatible artifacts and an explicit security decision.
Retaining an old binary does not mean using it remains acceptable.

### F. Readiness checklist

#### Inputs and templates

- Kernel, base filesystem, controller, and runtime have recorded identities.
- The publisher admits only reviewed inputs and completed candidates.
- Repository preparation includes more than a lockfile name.
- Memory, VM state, and disks share one captured recovery point.
- Candidates pass cold and fresh restore tests.
- Immutable memory files remain pinned while consumers exist.

#### Identity and authority

- The restore barrier cannot be satisfied by copied guest state.
- Clone notification and application restart policies are tested.
- Templates contain no reusable task credential.
- Capabilities bind to sandbox, generation, service, budget, and expiry.
- Revocation is enforced by the service.
- Guest root cannot alter host policy.

#### Lifecycle

- Caller-scoped request keys have conflict semantics.
- Resource intent is durable before allocation.
- Every host resource has an accountable generation.
- Command ambiguity is preserved.
- Expiry and renewal races have tests.
- The reaper operates independently of the creation path.

#### Results and operations

- Export rejects unsafe paths and bounds size.
- Useful work exists outside the disposable guest.
- Guest-reported tests remain distinct from independent verification.
- Host capacity includes pools, bakers, and management reserve.
- Template GC serializes against claim selection.
- Backup restore, host reboot, patch rollout, and fallback are exercised.

### G. What came from the research

The supplied revision 2 plan provided the central direction:

- One Ubuntu host for roughly ten to fifty personal coding agents.
- Firecracker as the primary execution runtime.
- Per-repository warmed workspaces and golden memory snapshots.
- Private copy-on-write disks.
- Independent Git clones fed by host mirrors.
- Host-controlled networking and post-restore credentials.
- Incus VMs on ZFS as the operational fallback.
- An adoption spike before building a control plane.
- Measurement of real repository tests and repeated cleanup.

The book develops those ideas into explicit contracts, restore barriers, durable operations, capacity reservations, and failure drills.
Those are engineering recommendations for the reference design.
They are not a claim that this complete service has already been deployed.

Several details were deliberately narrowed.
The template identity includes kernel, runtime, controller, preparation, and policy inputs.
Snapshot publication captures consistent disks as well as memory.
The first templates preserve files but stop task daemons.
The broker treats ambiguous command completion as unknown.
Credential access is tied to the lease and generation.
Performance is evaluated through task readiness and correctness.

### H. What still needs deployment evidence

The research includes exact release dates, lifecycle claims, latency comparisons, platform overhead figures, advisories, hosted prices, and implementation schedules.
Those can become stale or depend on narrow test conditions.
They are not carried forward as capacity guarantees.

Use current official release and advisory records when installing.
Measure the host and repositories before choosing resource classes.
Validate the chosen CubeSandbox or other platform revision against the adoption contract.
Test Docker inside the actual guest kernel.
Verify the supported snapshot compatibility and clone-notification behavior of the installed runtime.

The shell fragments and service interfaces have not been run as a complete deployment on the target Ubuntu host.
The source checks below establish documented tool behavior.
They do not establish your network isolation, workload compatibility, or recovery performance.
The labs produce that evidence.

### I. Sources and verification notes

Primary documentation was checked while preparing this manuscript on October 13, 2026.
The list emphasizes runtime behavior, storage ownership, and lifecycle semantics.
Vendor performance claims are inputs to experiments.
They are not substitutes for local measurements.

[^firecracker-spec]: Firecracker, “Specification.”
    Defines the published minimal-guest boot and VMM memory-overhead targets and their test context.
    `https://github.com/firecracker-microvm/firecracker/blob/main/SPECIFICATION.md`

[^firecracker-start]: Firecracker, “Getting Started.”
    Documents KVM prerequisites, runtime configuration, and a basic guest launch.
    `https://github.com/firecracker-microvm/firecracker/blob/main/docs/getting-started.md`

[^gvisor-platform]: gVisor, “Platforms.”
    Describes the systrap and KVM platform boundaries and requirements.
    `https://gvisor.dev/docs/architecture_guide/platforms/`

[^firecracker-jailer]: Firecracker, “Jailer.”
    Documents filesystem confinement, identity changes, namespaces, resource options, and the jailer's trusted inputs.
    `https://github.com/firecracker-microvm/firecracker/blob/main/docs/jailer.md`

[^cubesandbox]: CubeSandbox project README.
    Describes its advertised isolation, pool, snapshot, and SDK-compatibility capabilities.
    These statements inform the adoption spike rather than proving suitability for the reference design.
    `https://github.com/TencentCloud/CubeSandbox`

[^cp-reflink]: GNU Coreutils manual, “cp invocation.”
    Defines reflink modes and the failure behavior of `--reflink=always`.
    `https://www.gnu.org/software/coreutils/manual/html_node/cp-invocation.html`

[^docker-export]: Docker CLI reference, “docker container export.”
    Documents filesystem export and the exclusion of volume contents.
    `https://docs.docker.com/reference/cli/docker/container/export/`

[^git-clone]: Git manual, “git-clone.”
    Describes local clone behavior, alternates, `--shared`, and the dependency risks of shared object storage.
    `https://git-scm.com/docs/git-clone`

[^git-worktree]: Git manual, “git-worktree.”
    Documents shared repository state and per-worktree state.
    `https://git-scm.com/docs/git-worktree`

[^pnpm-store]: pnpm settings reference, “storeDir.”
    States the trust requirement for sharing a writable store between users.
    `https://pnpm.io/settings/store`

[^gradle-cache]: Gradle user manual, “Dependency Caching.”
    Describes concurrent cache access and the separately prepared read-only dependency cache.
    `https://docs.gradle.org/current/userguide/dependency_caching.html`

[^firecracker-snapshots]: Firecracker, “Snapshot support.”
    Documents memory and VM-state capture, disk consistency, file-backed memory restore, artifact lifetime, and compatibility considerations.
    `https://github.com/firecracker-microvm/firecracker/blob/main/docs/snapshotting/snapshot-support.md`

[^firecracker-api]: Firecracker OpenAPI schema.
    Defines runtime operations including pause, full snapshot creation, and snapshot loading.
    Use the schema from the pinned release for implementation.
    `https://github.com/firecracker-microvm/firecracker/blob/main/src/firecracker/swagger/firecracker.yaml`

[^firecracker-uniqueness]: Firecracker, “Entropy for Clones.”
    Describes cloned-state risks, randomness, and VM Generation ID support.
    `https://github.com/firecracker-microvm/firecracker/blob/main/docs/snapshotting/random-for-clones.md`

[^firecracker-vsock]: Firecracker, “Virtio-vsock.”
    Documents the host Unix-socket connection protocol and guest-port selection.
    `https://github.com/firecracker-microvm/firecracker/blob/main/docs/vsock.md`

[^network-namespaces]: Linux man-pages, “network_namespaces(7).”
    Describes the network state isolated by a network namespace.
    `https://man7.org/linux/man-pages/man7/network_namespaces.7.html`

[^sqlite-wal]: SQLite, “Write-Ahead Logging.”
    Documents concurrency, WAL files, checkpoints, and backup considerations.
    `https://www.sqlite.org/wal.html`

[^cgroup-v2]: Linux kernel documentation, “Control Group v2.”
    Defines CPU, I/O, memory pressure, hard limits, and memory-event accounting.
    `https://docs.kernel.org/admin-guide/cgroup-v2.html`

[^firecracker-releases]: Firecracker, “Release Policy.”
    Defines supported releases and security-fix handling.
    Consult the current advisory and release records when selecting binaries.
    `https://github.com/firecracker-microvm/firecracker/blob/main/docs/RELEASE_POLICY.md`

[^incus-zfs]: Incus, “ZFS.”
    Documents storage-copy options and snapshot restoration constraints involving clone descendants.
    `https://linuxcontainers.org/incus/docs/main/reference/storage_zfs/`

[^incus-stateful]: Incus instance options, “migration.stateful.”
    Documents the option for stateful VM operations.
    `https://linuxcontainers.org/incus/docs/main/reference/instance_options/`

[^incus-security]: Incus, “About security.”
    Describes the authority associated with local administrative access.
    `https://linuxcontainers.org/incus/docs/main/explanation/security/`
