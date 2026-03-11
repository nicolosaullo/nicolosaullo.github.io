---
layout: post
title: claude code - agents that verify their own work
---

## the problem

AI made writing code cheap. Verification is now the bottleneck.

Did the code actually work end to end? Did it break something else? The agent has no idea unless you give it a feedback loop.
If we are meant to treat AI agents as independent engineers, the minimum we should expect is that the code passes a happy path test locally.

tl;dr: give the agent a way to (1) start the local services, (2) run the same API test collection your team already uses, (3) parse the results, and (4) retry until it passes.

**Give the agent a way to verify its own work.**

An agent that writes code, runs it against a live system, reads the result, and fixes what's broken is possible. This applies at every stage:

- **Unit tests**: logic in isolation + regressions
- **Code review**: quality, security, standards
- **A live end-to-end test loop**: prepare the local environment and verify the implementation end to end

Each is a different kind of feedback loop. Chain them together and the agent gets all three before anything ships.

## the key idea: API test collections as the verification layer

For simple APIs, `curl` and `jq` are enough - the agent can fire requests and parse responses directly from the shell.

But most real systems have multi-step auth flows: login, extract a session token, set cookies, carry state across requests. Wiring that up in raw `curl` every time is tedious and fragile.

Most teams already have this solved. They have API test collections - Bruno, Postman, Insomnia, or similar. Sequences of HTTP requests with assertions: login, call an endpoint, check the response shape, extract values for the next request.

The auth is already wired. These collections are usually run by humans clicking buttons in a clunky GUI, or in CI after the code is already merged.

But they can also be run from the command line. And if they can be run from the command line, an agent can run them. The test infrastructure already exists. The agent just needs a way to reach it.

In my case, I use [Bruno](https://www.usebruno.com/) - an open-source API client where collections are plain text files stored in git (which makes it that much better than Postman).

Bruno has a CLI (`bru run`) that executes requests, runs inline test assertions, and outputs JSON results. The agent can:

1. **Edit the request body** - Bruno files are just text, so the agent writes the test message directly into the `.bru` file
2. **Run the collection** - `bru run "Login" "Post Message.bru" --env local -o ./result.json -f json`
3. **Read the output** - parse the JSON to extract the response, check which tools were used, verify the data

The general flow looks like this:

```
agent edits the request body in a test file
         ↓
agent runs the test from the command line
         ↓
the system under test processes the request end to end
         ↓
agent runs follow-up request(s) to fetch the result
         ↓
agent parses the output
         ↓
agent checks: did the system behave as expected? is the response data correct?
```

In my case I'm building tools for an AI chatbot, so the verification is: did the chatbot choose to use the new tool, and does the response contain data that could only have come from it?

Bruno CLI doesn't persist cookies or environment variables between separate `bru run` invocations. So the agent has to include the login step in every call and pass extracted values explicitly via `--env-var`.

The [skill](https://code.claude.com/docs/en/skills) encodes these quirks so the agent handles them automatically.

This same approach works with any API client that has a CLI. Postman has `newman`. Insomnia has `inso`.

The principle is the same: if your team already has a collection of API tests, you can give your agent access to them and let it verify its own changes against a running system.

## the other piece: service lifecycle

Spinning up all the local services and making sure that everything is up and running is one of the most annoying things to me as a software engineer. The things I want to avoid the most are context switching and using my mouse.

If it can be done in a terminal, the agent can do it too.

API tests are useless if there's nothing running to test against. The agent needs to manage the services too.

Claude Code can run processes in the background. The skill uses this to start each microservice, then polls their health endpoints until they're ready:

```bash
# example (bash-like syntax)
# start services in the background
uvicorn mcp_server:app --port 2001 &
uvicorn agent_server:app --port 8000 &

# poll until healthy
curl http://localhost:2001/healthz/readiness  # → {"status": "ready"}
curl http://localhost:8000/healthz            # → {"status": "ready"}
```

The skill encodes the operational knowledge: which ports each service runs on, which health endpoints to check, how long to wait before giving up, and environment-specific quirks (like setting `PYTHONIOENCODING=utf-8` on Windows to prevent emoji in log statements from crashing the process).

When something fails, the skill knows where to look. Log paths are baked in - so when the live test returns unexpected results, the agent reads the server logs, diagnoses the issue, and fixes it without the human pointing at a stack trace.

After testing, the skill restores any modified test files (like the Bruno request body) back to their defaults to avoid dirty diffs in the test repo.

This is the part that replaced the most manual work for me.

Not the code generation - the pulling up terminals, starting services, waiting for them, running a test, reading logs, stopping services, fixing something, starting them again.

That loop, automated.

## the full pipeline

I wired this together using [Claude Code skills](https://docs.anthropic.com/en/docs/claude-code/slash-commands) - markdown files that define a sequence of instructions the agent follows when you invoke them as `/slash-commands`.

One orchestrating skill chains sub-skills into a build-test-review-ship pipeline. One command, one input, three phases.

The input is a story describing what the tool should do. Everything else is automated.

<details markdown="1">
<summary>view structure</summary>

```
Phase 1: Build
  → find/build backend methods
  → create models, tool, registry entry, tests
  → run unit tests
  ↓ feedback loop: tests must pass

Phase 2: Live Test (Bruno CLI)
  → stop old services, start new ones in background
  → poll health endpoints until ready
  → edit request body in .bru file
  → run login + post message sequence
  → wait for processing
  → run login + get conversation sequence
  → parse JSON output, verify tool usage and response data
  → restore modified test files to avoid dirty diffs
  ↓ feedback loop: if fail → read server logs → diagnose → fix → retry

Phase 3: Review, Fix, Ship
  → parallel review of repos via sub-agents
  → address critical and high issues
  → re-run unit tests
  → branch, commit, push
  → output final pass/fail summary
  ↓ feedback loop: issues trigger fixes, fixes trigger re-verification
```

</details>

### phase 1: build

A sub-skill handles the implementation:

1. Searches the backend codebase for the relevant service methods
2. Creates response models matching the backend DTOs
3. Writes the tool with auth, service calls, and response transforms
4. Registers the tool across repositories
5. Writes unit tests and runs them

14 unit tests, all passing. But unit tests only prove the transform logic is correct - they say nothing about whether these calls will actually succeed in the running system.

### phase 2: live test - verification via Bruno

The skill restarts all services, waits for health checks, then runs the Bruno verification flow described above.

The success criteria isn't "did the service return 200." It's: did the AI agent choose to use the new tool, and did its response contain data that could only have come from that tool?

The skill parses the JSON output from Bruno and checks both.

The first attempt failed. One of the service endpoints returned a 404. The mocks said the call would work. The real system said otherwise.

The skill diagnosed it from the server logs, found that the service contract path didn't match the expected endpoint, switched to an alternative method, updated the response model, fixed the tests, and retried.

Second attempt passed. This is exactly the kind of failure that unit tests can't catch. In practice, the live test often surfaces something the mocks missed.

### phases 3: review, fix, ship

Multiple review agents — [Claude Code sub-agents](https://code.claude.com/docs/en/sub-agents) with no knowledge of the main context window — run in parallel, one per repository. They check correctness, security, performance, maintainability, and standards compliance.

The most notable find: `not {}` evaluates to `True` in Python — empty dicts are falsy. An auth guard using `not token.claims` silently passes for an empty claims dict and blows up downstream instead of returning a clean auth error.

| Priority | Issue |
|----------|-------|
| High | Auth guard used `not token.claims` — falsy for empty dict `{}` |
| High | No error handling test for partial failure in parallel calls |
| Medium | Dead model left behind after refactor |
| Medium | Test mocks relied on positional call ordering — fragile |
| Low | Import ordering (stdlib after local) |

The agent wrote the code, a separate review process found problems in it, the agent fixed those problems, re-ran the tests, then branched, committed, and pushed — write, review, fix, ship, without a human in between.

Every phase either produces a pass/fail signal or a list of issues. Failures trigger diagnosis and retry. Issues trigger fixes and re-verification. The agent never moves forward without confirmation that the previous step actually worked.

## what this changes


Without verification loops, AI is reduced to a code generator: fast, but blind. With a verification loop, the agent closes its own feedback cycle. And once an agent can verify its own work, you can run several in parallel — each on a different story, each self-checking. The bottleneck shifts from the AI to the human pilot's ability to hold context across all of them.

The core enabler in this case is mundane: an API test collection with a CLI. Most teams already have this.

The agent doesn't need special infrastructure or custom test harnesses - it needs access to the same tools the team already uses to test manually, just invoked from the command line instead of a GUI.

Three things made this work:

1. **API tests as agent-accessible verification.** Bruno collections are text files with a CLI. The agent can edit them, run them, and parse the results. Postman/newman, Insomnia/inso - same idea. For simpler APIs, raw `curl` and `jq` work fine. If your tests can run from a terminal, your agent can use them.

2. **Skills/sub-agents as composable units.** One skill builds. Another tests against a live system via Bruno. Another reviews. The orchestrating skill chains them. The next workflow I build can reuse all of them.

3. **Self-review as a separate pass.** The agent that wrote the code is not the same instance that reviews it. Fresh context, fresh analysis. It catches things the builder missed because it's looking at the code without the tunnel (or context window) vision of having just written it.

This isn't foolproof. The agent can't always diagnose a failure from logs alone. A passing live test doesn't guarantee correctness - it guarantees the specific scenario tested worked. The agent misses things that require domain knowledge it doesn't have.

The human still reviews, still tests edge cases, still makes the judgment calls. But the mechanical verification loop - start, test, read, fix, retry - is no longer manual.

The constraint is still the human in the loop - I pick the story, approve the push, and still test it.

But the mechanical work between those decisions is gone, and the verification that used to be manual is now built into the pipeline itself.
