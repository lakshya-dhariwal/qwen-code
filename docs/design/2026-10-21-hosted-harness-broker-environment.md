# Hosted Harness Broker Environment and Fixed Port

[English](2026-10-21-hosted-harness-broker-environment.md) | [简体中文](2026-10-21-hosted-harness-broker-environment.zh-CN.md)

Status: proposed and implemented in the same change.

## Problem

The Hosted Harness reaches the Runtime Broker only through `--managed-runtime-broker-url` and `--managed-runtime-broker-token`. These options have no environment fallback. The shipped systemd unit therefore expands `QWEN_HOSTED_HARNESS_BROKER_ARGS` into the command line, where any local user can read the Broker token from `/proc/<pid>/cmdline`. The unit and its environment template both carry a warning to keep the tool path to single-tenant hosts.

The Broker token is also stored twice. The Harness gets it through `harness.env`, the control plane through `QWEN_MANAGED_AGENT_RUNTIME_BROKER_TOKEN` in `server.env`, and every rotation must change both.

Separately, `qwen serve` moves to the next free port when the requested one is busy. For the Hosted Harness this is never what the operator wants. The control plane names one fixed Harness URL, so a moved Harness starts cleanly while every Session fails until someone edits `server.env`. The deployment guide and systemd README document this trap instead of preventing it.

## Goals

- The Hosted Harness profile accepts the Broker URL and token from its environment, and no shipped file puts the token on a command line.
- The token does not reach the processes the Harness starts.
- `harness.env` holds the Broker token once for both processes, building on the Harness credentials file.
- A busy Harness port fails startup instead of moving.

## Non-goals

- Changing the default profile's port fallback.
- Changing how the Broker authenticates, or its token format.
- Removing the existing command-line options. They keep working and take precedence.

## Design

### Environment fallback

Under `--profile hosted-harness`, `runQwenServe` resolves:

- `managedRuntimeBrokerUrl` from `--managed-runtime-broker-url`, otherwise `QWEN_HOSTED_HARNESS_BROKER_URL`;
- `managedRuntimeBrokerToken` from `--managed-runtime-broker-token`, otherwise `QWEN_HOSTED_HARNESS_BROKER_TOKEN`.

This mirrors the existing `QWEN_HOSTED_HARNESS_CAPABILITY_DIGEST` fallback. It happens before `validateHostedHarnessProfile`, so the existing rule still requires both values or neither. Outside the profile the variables are ignored, as the digest variable already is.

### Child environment

`QWEN_HOSTED_HARNESS_BROKER_TOKEN` joins `SCRUBBED_CHILD_ENV_KEYS`, the list the ACP bridge removes from every `qwen --acp` child it spawns. That list already holds `QWEN_SERVER_TOKEN` for the same reason: the agent speaks to the daemon over stdio and never needs the credential. A prompt-injected agent with the Broker token could otherwise call the Broker directly. In the Hosted Harness profile the model loop runs inside the Harness process, and Workspace tools run in Runtime workers that the control plane starts. The profile serves no shells, channels or WebSocket tunnels, so today the Harness starts no model-directed process at all. The scrub keeps the token out of an ACP agent child should any path start one, exactly as it already does for `QWEN_SERVER_TOKEN`.

### Fixed port

Under `--profile hosted-harness`, `EADDRINUSE` on the requested port fails startup with a message naming the port. The control plane's `QWEN_MANAGED_AGENT_HARNESS_BASE_URL` points at it, so moving the Harness would break every Session. `--port 0` keeps its ephemeral meaning.

### Shared Broker token

The Harness credentials file gains an optional third key. When `harness.credentials-file` is set and the file contains `QWEN_HOSTED_HARNESS_BROKER_TOKEN`, the control plane takes `runtime-broker.token` from it. As with the other two keys, a file that sets the key together with a direct `QWEN_MANAGED_AGENT_RUNTIME_BROKER_TOKEN` fails startup. A file without the key leaves the direct setting in charge, so text-only deployments are unaffected. Its line follows the same rules as the other two: no whitespace around the `=`, and a value of only letters, digits and `._:/+=-`. Binding is atomic: on a conflict, none of the three values is bound.

### Templates and documentation

- The bootstrap writes the generated Broker token to `harness.env` as `QWEN_HOSTED_HARNESS_BROKER_TOKEN`, with `QWEN_HOSTED_HARNESS_BROKER_URL=http://127.0.0.1:4182` beside it. `server.env` no longer carries the token.
- `harness.env.example` lists both variables for the tool path.
- The Harness unit drops `$QWEN_HOSTED_HARNESS_BROKER_ARGS` from `ExecStart`, along with the single-tenant warning that existed because of it.
- The deployment guide, the systemd README (English and Chinese) and the server README describe the variables and the fixed port. They also drop the instruction to update the base URL when the port moves.

## Compatibility and risks

- Deployments that pass the command-line options are unchanged; an option still wins over its variable.
- A Hosted Harness that previously moved to a free port now fails to start. That failure is the intended signal; the previous behaviour produced a running Harness that no Session could reach.
- A deployment that sets the Broker token both in `harness.env` and directly in `server.env` fails at startup until one copy is removed.

## Validation plan

- CLI unit tests:
  - both variables resolve under the profile;
  - an option overrides its variable;
  - the variables are ignored outside the profile;
  - a URL without a token, or a token without a URL, still fails validation;
  - the ACP child environment omits the token while the parent environment carries it.
- A CLI test that a busy port fails the Hosted Harness profile, while the default profile still moves to the next port.
- Java unit tests for the optional Broker token key: resolved from the file, a conflict with the direct setting fails and binds nothing, an absent key leaves the direct value, and the key's line is held to the credential rules.
- Template tests: `server.env` carries no Broker token, `harness.env` carries both variables, and the unit's `ExecStart` has no Broker arguments.
- End to end with the bootstrap files:
  - the Harness starts from `harness.env` with no Broker arguments;
  - the Broker token is absent from the Harness's `/proc/<pid>/cmdline`, and no child process of the Harness carries it;
  - the tool-path server starts and completes a Turn;
  - a second Harness on the same port fails startup.

## Acceptance criteria

- No shipped file places the Broker token on a command line, and it is stored once.
- The Harness's ACP children do not inherit the token.
- A busy Hosted Harness port stops startup with a message naming it.
- Existing command-line configuration behaves as before.

## Follow-up work

None planned within the deployment-ease series beyond publishing the first reference prerelease.
