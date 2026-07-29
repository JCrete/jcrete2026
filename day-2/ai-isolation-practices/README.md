Sourced from https://docs.google.com/document/u/0/d/1xxKwApvL_RXu5vUIvShsUOWN0IbZqqxtZHpUhAtEIv0

# JCrete 2026: AI isolation practices

Local AI agents are useful because they can do real work: read files, run commands, install dependencies, call APIs, edit a project, and occasionally make a decision you would have preferred they did not make.

That was the main thread in the JCrete session on AI isolation. If we let agents act autonomously, the boundary has to live outside the prompt. A prompt can ask the model to behave. A sandbox removes the option not to. We likely need both, but only one of them still works after the model reads a malicious markdown file.

## The risk model

Simon Willison's [lethal trifecta](https://simonwillison.net/2025/Jun/16/the-lethal-trifecta/) is still a good way to frame this. An agent becomes dangerous when it has three things at once:

- access to private data;
- exposure to untrusted content;
- a way to communicate externally.

For a coding agent, this is not exotic. Private data can be the source code, .env files, SSH keys, internal docs, git history, or credentials in the shell environment. Untrusted content can be an issue description, README, webpage, dependency metadata, generated file, or MCP tool output. External communication can be curl, package managers, GitHub, Slack, telemetry, or just a convenient HTTP client.

No sandbox escape required. The agent only needs to use its normal access and permissions to do the .

## What to control

The sandbox surface is not very mysterious. There are three things to care about first to limit the blast radius: 

- what the agent can read, write, delete, or discover on the filesystem;
- which hosts, services, registries, and internal endpoints it can reach;
- whether credentials are exposed inside the workload, scoped down, proxied at the boundary, or withheld completely.

If those are open, the rest is mostly theater. If those are controlled, the agent can still be productive, but the blast radius has a shape we can reason about.

## Ecosystem overview

We mentioned a few projects in the session. You can think of this as a map of where people are putting the isolation boundary.

### Local and VM-style sandboxes

- [Docker Sandboxes](https://docs.docker.com/ai/sandboxes/) takes the local microVM route. Each sandbox gets its own Docker daemon, filesystem, and network, so the agent can build containers, install packages, and modify files without touching the host directly.

- [isx](https://github.com/Sanne/incus-spawn), formerly incus-spawn, comes at the problem through Incus-powered system containers, with optional KVM VMs for untrusted code. The useful direction is familiar: give the agent a machine-shaped environment and make review and extraction explicit.

- [Bromure Agentic Coding](https://bromure.io/en/agentic-coding) runs coding agents inside a hardware VM and adds an opinionated host-side layer for credential brokering, egress control, supply-chain scanning, prompt-injection detection, traces, and agent workflow.

### Container sandboxes and technologies

- [NVIDIA OpenShell](https://docs.nvidia.com/openshell/about/overview) is a runtime for autonomous agents in sandboxed environments. Its policy model covers filesystem, network, process, and inference routing controls.

- [Alibaba OpenSandbox](https://open-sandbox.ai/) is a general-purpose sandbox platform for AI applications. It covers coding agents, browser automation, remote development, and safe execution of model-generated code, with Docker and Kubernetes runtimes.

- [Dev Containers](https://containers.dev/) define a repeatable development environment: tools, runtimes, settings, Features, and `devcontainer.json`. They can describe what should be available inside a sandbox or VM, but they are not the isolation boundary themselves (normal containers). 

### Proxy and egress-control layer

- [Bifrost Proxy](https://bifrost-proxy.github.io/) gives you HTTP/HTTPS/SOCKS5 proxying, TLS interception, request rewriting, scripting, inspection, replay, and a web UI.

- [iron-proxy](https://docs.iron.sh/) is aimed at untrusted workloads: outbound allowlists, credentials held outside the workload, real secrets swapped in at egress, and structured request logs.

This category deserves more attention than it usually gets. If the sandbox only sees a placeholder token, then leaking that token is annoying rather than catastrophic. The real credential never entered the agent's access boundary.

### Governance layer

- [Archestra](https://github.com/archestra-ai/archestra) is working around MCP governance: LLM gateway, MCP gateway, private MCP registry, virtual API keys, routing, cost limits, tool-call guardrails, identity, and related policy controls.
- Docker has an [AI Governance product](https://www.docker.com/products/ai-governance/) that builds on top of sbx as implementation and offers enterprise controls for everything AI. 

This is a different layer from filesystem or process isolation, but it belongs in the same conversation. Once agents start using MCP servers, model APIs, and internal tools, the question expands from where the process runs to which tools exist, who approved them, which identity they run as, and how their calls are audited.

### Cloud sandboxes

- [E2B](https://e2b.dev/docs/use-cases/coding-agents) provides cloud sandboxes for coding agents with terminal, filesystem, git, and package-manager access.

- [Daytona](https://www.daytona.io/docs/en/sandboxes/) provides composable computers for AI agents: isolated runtime environments with their own filesystem, network stack, and allocated compute.

Cloud sandboxes are useful, but interactive local coding has different tradeoffs. Latency, file locality, editor integration, and trust boundaries all feel different when the agent is not next to the project. Sometimes that is fine. Sometimes it is exactly the wrong place to put the boundary.

## A practical baseline

Any real sandbox beats a guardrail-only setup.

For a coding agent, the baseline should look roughly like this:

- run the agent somewhere isolated;
- give it only the workspace it needs;
- default-deny the network;
- keep real secrets outside the sandbox;
- use a proxy when the agent needs controlled access to external services;
- log what the agent tried to read, write, and call.

This will not make agents perfectly safe. Nothing interesting is perfectly safe.

But it changes the failure mode from "the model saw a malicious instruction and had access to everything" to "the model saw a malicious instruction and hit a wall". Which is a much better incident report to write.

In the end, we should treat AI agents less like clever chat windows and more like untrusted workloads with unusually good autocomplete. The prompt can ask nicely. The sandbox should say no.

