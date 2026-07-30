# MCPs and Java standardisation

Convenor: Drazen Nikolic

The session was motivated by the release candidate of the new Model Context Protocol (MCP)
specification, the biggest update to MCP since its launch. The lead maintainers
[froze the release candidate](https://blog.modelcontextprotocol.io/posts/2026-07-28-release-candidate/)
on 21 May 2026, with the final specification arriving on 28 July 2026, during the week of the
conference.

The most significant changes in `2026-07-28` are:

- **A stateless core** — the `initialize`/`initialized` handshake and `Mcp-Session-Id` are gone, so
  requests no longer need sticky routing or a shared session store.
- **First-class extensions** — server-rendered UIs through MCP Apps, and long-running work through
  Tasks, which moved out of the core.
- **Authorization hardening**, aligned with how OAuth 2.0 and OpenID Connect are actually deployed.
- **Deprecation of Roots, Sampling and Logging** — Sampling in favour of integrating with LLM
  provider APIs directly, which is a real change for clients that had built on it.
- **A formal feature lifecycle policy**, giving every feature a defined path from Active through
  Deprecated to Removed.

---

We started from the observation that most MCP servers today still run on developers' laptops, while
the expectation is that more and more MCP servers will be consumed remotely over the internet.

From there the discussion moved to tooling. Spring AI and Quarkus both ship MCP support, yet one of
the jCreteans shared their experience of having to build a custom MCP server implementation in
order to cover more complex enterprise requirements. That raised the questions which occupied most
of the session:

- Are these frameworks fast enough to adopt new MCP specification revisions?
- Are they, in their current state, flexible enough and at an adequate level of abstraction to
  support future enterprise needs?
- How easily can they support migrating an existing implementation to the new specification, and to
  whatever changes come after it?

One business requirement was discussed at
some length: the agent logic calling an MCP server should only be able to access a subset of that
server's resources, tools and prompts, determined by the role of the end user driving the agent and
by the point the user has reached in the business domain workflow.

Versioning and deprecation management of the tools a server exposes was another thread. One idea we
explored was to let a deprecated tool still answer a call, but return an instruction back to the
agent telling it to use the replacement tool instead. We concluded that this is a poor foundation
for backward compatibility: there is no guarantee the agent follows the redirect, so compatibility
becomes non-deterministic, and it normalises models acting on imperative text arriving in tool
output — eroding the boundary that lets an agent treat results from less trusted servers as data
rather than as instructions. This approach also raises security concerns, like prompt injection, etc.

We also noted the ten-week window between the frozen release candidate and the final specification,
which gives SDK maintainers, client implementers and companies time to validate and adapt. Java
enters that window a revision behind: MCP Java SDK 2.0.0 targets the 2025-11-25 specification, so
the ecosystem's answer to "are these frameworks fast enough" will become visible over the coming
months.

There were no concrete conclusions. The general feeling was that the new specification should
simplify things — the move from stateful to stateless, the deprecations, and the new lifecycle
policy all point in that direction — but whether the Java tooling keeps pace with it remains an open
question.

## Links

- [The 2026-07-28 MCP Specification Release Candidate](https://blog.modelcontextprotocol.io/posts/2026-07-28-release-candidate/)
- [MCP Java SDK](https://github.com/modelcontextprotocol/java-sdk)
- [Spring AI 2.0.0 GA announcement](https://spring.io/blog/2026/06/12/spring-ai-2-0-0-GA-available-now/)
- [Quarkus MCP Server extension](https://github.com/quarkiverse/quarkus-mcp-server)

