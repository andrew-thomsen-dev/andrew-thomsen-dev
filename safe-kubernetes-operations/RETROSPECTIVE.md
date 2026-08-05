# Lessons Learned: Rake Console Launcher

**Companion to:** [Case Study](./case-study-rake-console.md) and [Architecture Concepts](./architecture-concepts-rake-console.md)

This is a retrospective on how the tool actually got built — including the wrong turns, the assumptions that had to be corrected by the people actually using it, and the bugs that only showed up once the tool was pushed past a happy-path demo. Most of the real lessons here aren't about bash; they're about how a small internal tool's requirements only become fully clear once real users start pushing on it.

## The friction I optimized for wasn't the friction people actually had

Partway through, a request came in to reduce how often the tool made people re-authenticate. My first instinct was to assume that meant AWS SSO login — that's the credential prompt that appears most often in a kubectl-based workflow, so it seemed like the obvious target. I restructured the loop so cluster selection and login happened once per session instead of once per task, and left it there.

That was solving the wrong problem. The actual complaint was about re-entering the Datadog audit-logging API key, not the SSO login — and re-prompting for SSO on every region switch was explicitly fine with the team. The fix ended up being a locally cached, permissioned credential file for the audit key, which is a meaningfully different piece of engineering than what I'd built. The lesson wasn't "test more" — it was that when someone describes friction in a multi-step flow, don't assume which step it is; ask, or wait for the correction, before optimizing.

## "Convenience" and "security concern" were about two different things

Early on, the idea of a shared team alias came up purely as a UX convenience — one shortcut, less to remember. It took a teammate's infosec-minded pushback to separate that into two genuinely different questions: is the *shell alias* a problem (no — it's cosmetic, purely local, and doesn't touch identity at all), and is a *shared credential/identity* a problem (yes — that's what actually breaks individual accountability for a security review).

Once that distinction was clear, the real gap was easy to name precisely: standard Kubernetes audit logging captures that an exec happened and who initiated it, but not what was typed once inside an interactive shell. That's a narrower, more specific problem than "we need better logging," and it's what the Datadog audit event was actually built to close. The broader lesson: when a security concern is raised against a convenience feature, resist the urge to just make the convenience feature "more secure" — figure out which underlying capability is actually missing first.

## An edge case that only showed up once looping was added

Adding the "run another task without restarting" loop introduced a subtle bug: the menu-picker function looped on invalid input by design, but it never checked whether `read` itself had failed — only whether the value it got back was a valid choice. Under normal interactive use that distinction never mattered. It only surfaced once the input stream could be exhausted mid-flow (in this case, during scripted testing with piped input), where it turned into a silent infinite loop instead of a clean failure. It was a good reminder that "loop until valid input" and "loop until *any* input" are different guarantees, and the difference only shows up under conditions a normal interactive session doesn't create.

## Real user feedback beat my first guess at good-enough UX

The first version of the API key prompt used a completely silent password-style read — nothing echoed back at all. That felt like the "secure" default. Feedback from an actual user was that it was genuinely hard to tell whether a paste had gone through at all, let alone correctly. The fix (asterisk-per-character masking, plus a last-four-characters confirmation after entry) is a small thing, but it only happened because someone using the tool said the silent version was confusing — my first pass at "secure-feeling" input didn't account for "confidence that something actually happened," which turned out to matter more in practice.

## Holding a security boundary even when it would have been easier not to

At one point I was offered a real Datadog API key to hardcode directly into the script, and separately asked whether the same key could be reused inside a browser extension for click tracking. Both were reasonable-sounding shortcuts, and both were declined for related reasons: a key typed into a chat transcript or committed into a script destined for a shared repo is a key that's now exposed far beyond its intended use, and — confirmed by checking Datadog's own documentation rather than assuming — API keys are explicitly not meant to run in browser-side code at all, since anything shipped to a browser is inherently visible to whoever's using it. Neither of these was a hard technical constraint; both were "this would work, but it trades away a security property for convenience," which is exactly the kind of shortcut that's easiest to take under time pressure and hardest to walk back later.

## Tooling failures need a decision point, not indefinite retries

A GitHub connector needed for pushing the script to the team's repo reported as unavailable in every session, despite appearing connected in the user-facing settings UI — almost certainly a session-sync issue rather than a real configuration problem. It got rechecked more than once before landing on the right call: stop retrying the same broken path and explicitly hand the task back as a manual workaround. Tool or integration failures that don't resolve on a reasonable retry are a decision point ("route around this manually") rather than something to keep re-attempting hoping the next check behaves differently.

## "No hardcoded secrets" and "safe to publish" are not the same bar

The script was security-reviewed for credential handling from day one — no hardcoded keys, local-only caching, masked input — and that held up. But when the time came to actually put a copy in a personal public repo, a separate pass turned up direct links to internal documentation, a production admin URL, internal cluster/AWS naming conventions, and internal service codenames baked into the task list, none of which are "secrets" in the credential sense but all of which are internal, proprietary detail that had no business in a public repo. The lesson: a tool being credential-safe says nothing about whether it's safe to publish — those are two different reviews, and the second one only became obvious once publishing was actually on the table, not while the tool was being built for internal use.
