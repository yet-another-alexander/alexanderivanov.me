---
title: "Spec First in a Software Factory"
layout: post
description: When AI agents write most of the code, the hard part is no longer writing it but knowing it does what was asked. A spec-first workflow, enforced by CI rather than by good intentions, is how a software factory keeps its output honest.
tags:
  - AI
  - coding
---
Agents now write most of the code in many teams, mine included. That moved the bottleneck. Typing code is cheap now. The expensive part is knowing that the code does what was asked, and reviewing the dozens of pull requests a day they produce.

The industry has a name for this kind of setup: the software factory. The term is decades older than coding agents, but it has found a new meaning. A ticket comes in at one end, working software comes out at the other, and in between there is a repeatable process: planning, specification, implementation, verification, delivery. Agents do most of the work on the line. Humans decide what gets built and own the quality gates.

Every real factory has two things before anything gets built: a blueprint and quality control. Without them you just have a very fast machine making the wrong thing.

A factory is also clear about where people spend their attention. They check the blueprint carefully, once, before production starts. They don't inspect every part by hand. Instead they put automated checks on the line that reject a bad part, and they maintain those checks. They step in only when a check fails. In this post the blueprint is the spec, the checks are CI, and the parts are pull requests.

This post is about the blueprint and the checks: the spec-first workflow. I packed it into a .NET template you can try: [spec-first-dotnet-template](https://github.com/yet-another-alexander/spec-first-dotnet-template).

#### Two principles

Everything below follows from two sentences. The first:

> A guardrail the agent can edit is not a guardrail.

Agent rules, `CLAUDE.md` files and "constitutions" are advice. The agent reads them, mostly follows them, and sometimes "improves" them. CI is enforcement. So anything we actually care about has to fail a build, not just be written down somewhere.

The second is just as simple:

> The specification is the source of truth.

Code conforms to it, and CI checks that it did.

#### Write requirements a machine can check

Acceptance criteria in a ticket usually die on merge. Nobody reads them again, and agents fill the gaps by making things up. So requirements go into the repository, in a fixed format called [EARS](https://en.wikipedia.org/wiki/Easy_Approach_to_Requirements_Syntax) (Easy Approach to Requirements Syntax). The rule: one sentence, one `shall`, one testable behaviour.

```markdown
### REQ-006

When a client confirms an order in status Submitted, the system shall
change the status to Confirmed and respond 204.

### REQ-010

If a client cancels an order in status Confirmed, then the system shall
respond 409 with a problem details body.
```

EARS has five patterns, and the nice part is that each one tells you how to test it:

| Pattern | Form | Tested by |
|---|---|---|
| Ubiquitous | The system shall … | Property test |
| Event-driven | When X, the system shall … | Example test |
| State-driven | While X, the system shall … | Example test |
| Unwanted behaviour | If X, then the system shall … | Negative test |
| Optional | Where X, the system shall … | Feature-gated test |

If a requirement needs an "and" between two actions, it is two requirements. If no test could ever fail because of it ("we use RabbitMQ"), it is not a requirement at all. It is a decision, and it goes into an ADR.

Every requirement gets a stable ID: `REQ-nnn` for behaviour a client can see, `TECH-nnn` for behaviour the system can verify about itself (migrations run at startup, errors never leak stack traces). Ticket keys name the *change*, not the requirement. Requirements outlive tickets.

#### Tie every test to a requirement

A requirement without a test is a wish. A test without a requirement is someone's guess. So every test that proves a requirement says which one, by ID and by a hash of its text:

```gherkin
@REQ-006:4377a453
Scenario: Confirming a submitted order
  Given a submitted order with lines
    | sku   | quantity | unitPrice |
    | APPLE | 1        | 1.00      |
  When the client confirms the order
  Then the response status is 204
  And reading the order shows status "confirmed"
```

For plain xUnit tests, the same thing is an attribute: `[Requirement("REQ-001", "7f6cdb52")]`.

The hash is the first eight characters of a SHA-256 over the normalised requirement text:

```csharp
public static string Of(string text)
{
    var normalised = Whitespace().Replace(text.Trim(), " ");
    return Convert.ToHexStringLower(SHA256.HashData(Encoding.UTF8.GetBytes(normalised)))[..8];
}
```

Why bother? Because requirements change. When someone rewords REQ-006, its hash changes, the test still carries the old one, and the build goes red. A human then has to make a decision: either the test still proves the new wording (update the hash), or it doesn't (send the test back to work in progress). The change can't slip through quietly.

A set of guardrail tests enforces all of this on every run. The build fails when:

- a testable requirement has no test;
- a test points to an ID that doesn't exist;
- a hash is stale;
- an ID is duplicated, or a requirement has zero or two `shall`s;
- the spec still contains a `[NEEDS CLARIFICATION]` marker.

They are ordinary xUnit facts, so they run locally in seconds too:

```csharp
[Fact]
public void Every_testable_requirement_has_a_test()
{
    var tested = Tests.Markers.Select(marker => marker.Id).ToHashSet();
    var untested = Requirements
        .Where(requirement => requirement.IsTestable && !tested.Contains(requirement.Id))
        .ToList();

    untested.ShouldBeEmpty();
}
```

#### Two pull requests, two sessions

Spec and tests in one PR, implementation in another.

The **spec PR** holds everything that defines what "correct" means: requirement text, contract changes (OpenAPI, database schema, messages) and acceptance tests that fail for now, tagged `@wip`. A human or an agent can write it, but a different human reviews it carefully. This is where human attention goes. It's one page of requirements and a few scenarios, not two thousand lines of generated code.

**Implementation PRs** come from agents, one task per PR. They may not touch the spec or the spec tests. The only change they are allowed to make there is deleting an `@wip` line once the scenario passes. Everything else is reviewed in CI: by the guardrails, and by an AI reviewer that reads the diff. A human may still take a look, but the line doesn't wait for one.

Just as important: the agent session that writes the tests is never the session that writes the code. One session doing both writes tests that fit its own implementation rather than the spec. That's not malice, just the path of least resistance.

The rule is enforced at three layers, and only the last one really counts:

- **Claude hooks** stop an implementation session the moment it tries to edit a protected file. Fast feedback, but only for Claude.
- **Husky** git hooks format, scan for secrets and check commit messages for anyone who commits.
- **CI** checks the pull request itself. This is the one that can't be skipped:

```bash
diff=$(git diff "origin/$BASE...HEAD" -- docs/specs tests/SpecFirst.Service.Tests.Spec)
added=$(printf '%s\n' "$diff" | grep -E '^\+' | grep -vE '^\+\+\+ ' || true)
removed=$(printf '%s\n' "$diff" | grep -E '^-' | grep -vE '^--- ' \
  | grep -vE '^-[[:space:]]*@wip[[:space:]]*$' || true)
[ -z "$added" ] && [ -z "$removed" ] || exit 1
```

Added lines fail. Removed lines fail unless they are exactly `@wip`. That's the whole check, and it runs on every PR without the `spec` label.

A requirement therefore has a short and boring life: written → work in progress → covered. Agents only perform the second step. A nightly job lists `@wip` markers older than 14 days, so "work in progress" can't become a parking lot.

#### Contracts that code can't outgrow

Requirements describe behaviour. The service also has a shape: endpoints, tables, messages. That shape is declared in contract files under `docs/specs/`: `openapi.yaml`, `schema.sql` and `asyncapi.yaml`.

On every PR, CI regenerates the real shape from the code and compares it with the contracts. The trick is that the comparison goes one way only: everything in the code must be in the contract. The reverse, a contract element with no code yet, is just pending work and passes.

This one detail makes the check independent of PR order:

1. The spec PR adds a cancel endpoint to `openapi.yaml`. Code hasn't changed, so it passes.
2. The implementation PR adds the endpoint to code. It's already in the contract, so it passes.
3. An agent adds a "helpful" extra endpoint nobody asked for. It fails.

The mechanics differ per contract, and none of them are exotic:

- **API**: the build writes the generated OpenAPI to `spec.json`, and `oasdiff` compares it with the contract. Spectral lints the contract itself.
- **Database**: an integration test applies migrations to a fresh Postgres in Testcontainers, runs `pg_dump --schema-only`, and checks every statement exists in `schema.sql`.
- **Messages**: a test generates a JSON Schema from every message type in code and checks its topic and properties against `asyncapi.yaml`.

#### Invariants get property tests

Ubiquitous requirements ("the system shall always…") are invariants, and a few hand-picked examples prove very little about an invariant. So they get property tests with CsCheck, which throws random data at them:

```csharp
[Fact]
[Requirement("REQ-001", "7f6cdb52")]
public void The_total_is_additive_over_line_sets() =>
    Gen.Select(Lines, Lines)
        .Sample((first, second) =>
            OrderTotal.Of(first.Concat(second))
                .ShouldBe(OrderTotal.Of(first) + OrderTotal.Of(second)));
```

Random inputs find the combinations nobody would think to write down by hand.

#### The assembly line

Once a spec PR is merged, the implementation part of the line can run mostly unattended. The change folder (`docs/specs/changes/<ticket>/`) holds a `spec.md`, a `plan.md` and a `tasks.md` where one task equals one PR.

The template ships `/specify`, `/clarify`, `/plan`, `/tasks` and `/implement` commands for Claude Code, plus an `implement-change` workflow that turns the two-sessions rule into a mechanism: a fresh implementer agent for each task, an independent verifier agent that never saw the implementer's work, one repair round, and a hard stop at the first task that doesn't verify.

If the implementer finds a test that looks wrong, it doesn't fix the test. It stops and reports it. A wrong spec goes back to a spec PR, not around it.

#### Why not Spec Kit or OpenSpec?

[GitHub Spec Kit](https://github.com/github/spec-kit) and [OpenSpec](https://github.com/Fission-AI/OpenSpec) already exist and are widely used. Why write custom commands?

You can use either of them. The approach stays the same. Only the folder structure changes a little.

Both tools, like my own commands, are markdown prompts run by the coding agent, plus a folder convention. They help you *write* a spec and break it into tasks. None of them verifies that the code matches the spec. That part is the guardrails: the hashes, the traceability tests, the one-way contract checks and the protected paths. They work the same no matter which tool produced the markdown.

What changes is mostly where things live:

- **Spec Kit** works per feature, in `specs/NNN-feature/spec.md`. Requirements would move out of that file into a central requirements file, and `spec.md` becomes an index of IDs. Its `clarify` and `plan → tasks` commands map well onto the workflow. Technical changes with no user story fit less naturally.
- **OpenSpec** works per change, with `ADDED` / `MODIFIED` / `REMOVED` deltas against per-capability specs in `openspec/`. That is very close to the two-PR model: the delta is the spec PR, and archiving the change closes it. The main adaptation is adding EARS and IDs to its format, and computing the hash from the delta while a change is still open.

Both tools hardcode their root folder, so the guardrails point there instead. For example, in the hook that protects the spec:

```python
PROTECTED = ("docs/specs/", "tests/SpecFirst.Service.Tests.Spec/")  # custom commands
PROTECTED = ("openspec/", "tests/SpecFirst.Service.Tests.Spec/")    # with OpenSpec
```

The same change goes into the CI check and the traceability tests. Nothing else changes: requirements are still EARS with IDs, tests still carry hashes, spec and implementation still travel in separate PRs from separate sessions.

I went with custom commands in the template because they took about a day to write and fit the layout exactly. The price is maintaining them myself, with no upstream to follow. If your team already uses Spec Kit or OpenSpec, keep it, and add the guardrails around it. The tool helps the factory draw the blueprint. The guardrails make sure the line builds what the blueprint says.

#### And the rest of the factory floor

None of this works on a codebase with no other guardrails, so the template sets them up at commit zero: warnings as errors, analyzers, NsDepCop for namespace rules, CSharpier, gitleaks, a 90% coverage gate, weekly mutation testing with Stryker, OpenTelemetry, and a Docker smoke test. Once an agent has produced twenty files with no enforcement, the repository is already brownfield.

#### The price

This isn't free. You write requirements in a fixed form. You write tests before code. You open two PRs instead of one. For a one-line fix, that can feel like ceremony. Small changes follow the same rule with smaller PRs: a bug fix starts with a failing test tagged with the requirement it violates, and if no requirement covers the case, the bug just revealed a gap in the spec.

For me it's worth it. Humans review one page of spec and a handful of tests, once. After that, CI does the reviewing. Everything the agent made up is visible and blocked. And for any line of behaviour, the requirement, the test that proves it and the change that introduced it are one search away.

#### Try it

```sh
git clone https://github.com/yet-another-alexander/spec-first-dotnet-template.git
cd spec-first-dotnet-template
dotnet new install ./templates/SpecFirstApi
cd ..
dotnet new spec-first-api -n Acme.Orders
cd Acme.Orders
dotnet test --filter "Category!=wip"   # needs Docker
```

The template comes with a small Orders feature. Order cancellation (REQ-008 to REQ-011) is specified and tagged `@wip` but not implemented. That's exactly the state a merged spec PR leaves behind. Run `/implement EXAMPLE-1-orders 4` and watch the line move.

A software factory is only as good as its blueprints and its quality control. Agents are getting faster every month. The spec is what keeps "fast" from turning into "fast at building the wrong thing".
