# pid-path

IMPLEMENTATION IN PROGRESS

Get the executable path for a specified process ID (PID).

## Usage

```ts
import { pidPath } from "@silicon-soldier/pid-path";

pidPath(process.pid);
// For Linux/macOS...
// /usr/bin/node
// For Windows...
// C:\\Program Files\\nodejs\\node.exe

pidPath(process.ppid);
// For Linux/macOS...
// /usr/bin/bash
// For Windows...
// C:\\Windows\\System32\\WindowsPowerShell\\v1.0\\powershell.exe
```

## But Why?

To enrich errors thrown by a wrapper around `node:child_process` with the resolved executable, or at least a candidate (API calls such as `exec` allow process replacement while retaining the PID, _and_ it is possible that by the time the lookup is performed the target process has died and the PID reallocated to a new process... FUN!).

The why behind the why is this extra information can aid debugging in certain scenarios, like the following hypothetical...

```ts
import * as child_process from "node:child_process";

// ...

const result = child_process.spawnSync("node", [...], { timeout: 30_000 });
// Has started erroring
// result = {
//   error: Error: spawnSync node ETIMEDOUT
//       at ... {
//     errno: -110,
//     code: 'ETIMEDOUT',
//     syscall: 'spawnSync node',
//     path: 'node',
//     spawnargs: [ ... ],
//   },
//   status: null,
//   output: [ null, <Buffer >, <Buffer > ],
//   pid: 31504,
//   stdout: <Buffer >,
//   stderr: <Buffer >
// }

// ...
```

A specific codepath has started timing out. We know where, we know it is a child process taking longer to complete or getting stuck, we know the arguments. With a little luck this information, additional context like stacktraces and identity of the system having the problem lead to the development of what _should_ be reproduction steps.

We attempt to reproduce... but we can't. The problem does not reproduce in the test environment. Investigations continue, several hours and an incident escalation later the problem is discovered. It turns out the production environment access token used by the infra deployment script had expired 2 weeks ago and the CI pipeline responsible for running this had been silently failing. Meanwhile a NodeJS upgrade had been performed, code changes made to take advantage of said optimisations, the timeout reduced to a level that at the time all environments were happy with, and load increased in tandem with userbase growth.

Mitigations are performed, the access token renewed, and silent failure fixed to be loud. Perhaps the incident itself could have been handled better, and clearly monitoring wasn't good enough.

But in this hypothetical, software dependencies are managed with Nix. That means the NodeJS executable was not at `/usr/bin/node` like in conventional Linux setups.
* Expected: `/nix/store/{hash}-nodejs-18.16.0/bin/node`
* Actual: `/nix/store/{hash}-nodejs-14.18.0/bin/node`

As this was a major NodeJS upgrade it had previously been announced to engineers, had the error included the resolved executable the version mismatch could have been caught early in the incident.

## Implementation Specifics

- Uses `/proc/<pid>/exe` on Linux.
- Uses `libproc`'s `proc_pidpath` function (via [`@silicon-soldier/darwin-libproc`](https://github.com/Silic0nS0ldier/darwin-libproc-nodejs)) on macOS.
- Uses `PSAPI`'s `GetModuleFileNameExA` function (via [`@silicon-soldier/windows-psapi`](https://github.com/Silic0nS0ldier/windows-psapi-nodejs)) on Windows.
