# Case Study: Making Kubernetes-Gated Support Operations Safe for Non-Kubernetes Engineers

**Role:** Sole designer/builder. **Context:** Internal tool built for a Support organization at Firstup. **Note:** this case study describes the problem, design decisions, and outcome only — the underlying script is proprietary to Firstup and is not included here.

## The problem

Our Support team regularly needs to run a set of routine operational tasks — rebuilding a corrupted data snapshot, restoring users who were deleted in error, freeing up an email address for reuse, issuing a manual password reset — that only exist as Rails rake tasks or console snippets, runnable only by connecting into the correct pod on one of our production Kubernetes clusters.

That created a real adoption problem. Running any of these tasks meant knowing which of several regional clusters to target, authenticating through AWS SSO, finding the right pod out of dozens with `kubectl get pods`, exec-ing into it, and then typing an exact rake command or Rails snippet from memory or from a wiki page open in another tab. None of that is Kubernetes-specific tribal knowledge Support engineers had a reason to build up, and the team was — reasonably — reluctant to invest in learning kubectl just to run what was, from their point of view, a fill-in-the-blanks operation.

The risk wasn't just friction. Several of these tasks are destructive and irreversible (e.g. erasing all content for a workspace), and copy-pasting a rake command from documentation into a live production shell is exactly the kind of manual step where a stray character or wrong ID does real damage.

## Constraints that shaped the design

A few things ruled out the obvious shortcuts:

Teaching kubectl to the whole team wasn't the ask — the team wanted to keep doing their jobs, not become infrastructure engineers. Any solution had to meet them at "I know which task I need to run," not "I understand Kubernetes contexts."

A shared team credential was explicitly off the table. Early in the project, a security-minded teammate flagged that using a shared alias or shared identity for convenience would make it impossible to attribute who actually ran a given task — a real concern for an infosec review, since some of these tasks touch customer data directly. Whatever got built needed individual accountability preserved, not traded away for ease of use.

Nothing could touch the database directly. One task (restoring a user deleted in error) has a secondary step that requires direct database write access. That's a deliberately higher-trust operation than "run this rake task," and folding it into a script that Support runs unsupervised would have expanded the blast radius of a compromised laptop or a copy-paste mistake into direct data mutation. That step needed to stay a manual, human-gated action — the script's job was to make sure nobody forgot it needed doing, not to do it for them.

## The solution

The result is a menu-driven bash script that walks an engineer through: picking a region/cluster, authenticating via the AWS SSO flow they already use, choosing a task from a plain-language menu, having the script find the right pod automatically, and then landing them on a command line with the exact command already typed out — placeholders and all — so they only ever have to fill in an ID and press Enter, never copy-paste raw text into a live shell.

A few design decisions did most of the work of making this actually safe to hand to a non-Kubernetes team, rather than just convenient:

**Pre-fill instead of copy-paste.** The script builds a small helper script on the fly, base64-encodes it, and writes it into the target pod via a here-string (so it doesn't consume the interactive terminal), then uses bash's readline pre-fill (`read -e -i`) to put the exact command on an editable line inside the pod. The engineer sees and can edit the real command before running it — there's no clipboard step where a partial paste or an extra character can slip in unnoticed.

**Individual audit logging instead of a shared identity.** Every run of a task sends a log event — who ran it (from the AWS identity already tied to their SSO login), when, which cluster, which pod, and which task — to a central logging pipeline, before the script ever execs into the pod. This is what actually resolved the infosec concern: the team gets the operational convenience of a documented, repeatable workflow without losing the ability to trace any individual action back to a person. It's deliberately a best-effort, non-blocking log call — a failed log send never blocks the actual task, since audit visibility is a nice-to-have layered on top of the workflow, not a gate on it.

**A typed confirmation, not just a keypress, for irreversible actions.** Every task shows a preview of the exact command and a general "this can't be undone" warning before connecting. But the single most destructive task in the list — one that permanently erases all content for a workspace — goes further: it shows a loud, unmissable warning banner and requires typing an exact confirmation phrase, not just pressing Enter, before the script will connect. A single keypress is too low-friction a gate for something with no undo; making someone type out what they're about to do is a deliberate, small speed bump aimed at catching "wait, am I sure?" before it's too late to matter.

**A visible reminder instead of a hidden automation for the database-write step.** Rather than trying to safely automate the one task that needs direct database access, the script does the safe part (the rake task) and then prints a clear, hard-to-miss alert once that's done, spelling out exactly what manual step is still outstanding and who needs to be looped in to finish it. The goal was to keep a genuinely higher-trust action in human hands without letting it quietly fall through the cracks either.

**Credentials that never touch the script or a shared repo.** The audit-logging key is never hardcoded anywhere. The script checks environment variables first, then a locally cached file scoped to the individual's own machine (readable only by them), and falls back to an interactive, masked prompt — with an explicit menu to swap or remove a saved key if it's ever entered wrong. Nothing secret ever needs to live in source control for this to work across a whole team.

## Iterating from real usage

The early version of the script re-prompted for both cluster selection and the audit-logging key on every single task, which made it just as tedious as the kubectl workflow it was replacing. Feedback from alpha testing — and a few false starts on my part diagnosing the actual friction — led to two changes: the workflow now loops back to "run another task" without restarting the script or re-authenticating, and the audit key is cached locally after the first successful entry so it's a one-time setup per machine rather than a per-task tax. A separate piece of feedback (picking the wrong region with no way back except restarting) led to adding a "switch region" escape hatch directly on the task menu, not just after a task finished.

## Outcome

The team moved from a documentation-and-tribal-knowledge process — where running one of these tasks meant finding the right wiki page, remembering or re-deriving the exact pod and command, and hoping nothing was mistyped along the way — to a guided, menu-driven flow that a Support engineer with no Kubernetes background can run correctly on the first try. The destructive-task confirmation gate and the database-write alert both directly target the two places a mistake would have been hardest to walk back. And the shift from "shared team alias" to "individually attributed audit log" closed the specific infosec gap that could have blocked adoption of this kind of tooling altogether — the team got a faster, safer workflow without trading away the accountability a security review would rightly ask about.

## What I'd revisit

A teammate later asked about wiring a Chrome extension into the same audit pipeline for a related workflow. I recommended against reusing the same server-side API key for that — Datadog's own guidance is explicit that API keys aren't meant to run in browser-side code, since anything shipped to a browser is inherently exposed to whoever's using it. A follow-on version of that idea would need a separate, purpose-built client-side token rather than reusing this script's credential, which is a good example of a decision that looked like a shortcut but wasn't actually the same risk profile as the original problem.
