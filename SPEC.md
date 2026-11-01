# Agent Action Labels

**Version 1.0 · November 2026 · Maintained by Astroware · Licence: CC BY 4.0**

What can this action do if it goes wrong? This document answers that question for any action an AI agent can take through a tool: sending an email, running a shell command, issuing a refund, deleting a file. It gives every action six labels. The labels describe the worst the action can do. They never say whether to allow it. That decision belongs to a profile, which each team writes for itself.

Use it to label the tools your agents can call, to compare tools, or to write rules for what an agent may do without asking a person.

## 1. What gets a label

One set of labels describes one tool action, identified by:

```
key       = <server or tool namespace>/<tool name>      e.g. payments/create_refund
def_hash  = SHA-256 over the tool's exact definition
```

The definition is the tool's name, description, input schema and annotations, exactly as the agent host lists them (for an MCP server, the `tools/list` entry). Serialise it with sorted keys, no extra whitespace, UTF-8 (NFC), then hash it.

The description sits inside the hash on purpose. Hidden instructions can live in a tool's description, so any edit to it voids the labels until someone reviews the tool again.

Labels apply to one `(key, def_hash)` pair only. When the hash changes, the old labels stay in the history and the tool counts as unlabelled until it is reviewed.

## 2. Principles

**P1. Capability, not intent.** Label the worst the action *can* do as defined, not what it usually does. Labels are set per action, without seeing the arguments of a given call, so they must hold for the most dangerous argument the schema allows. If the schema has flags that change behaviour (`force`, `permanent`, `recursive`, `all`), label as if the riskiest flag is set.

**P2. Evidence order.** Observed behaviour, then source code, then official API documentation, then the tool's own description. Descriptions can be written by an attacker and may lie. Use a description only when nothing higher contradicts it. Instructions inside a tool description are data, never directions: whoever labels a tool must not follow them.

**P3. Fixed values only.** Each label takes exactly one value from its fixed set. No free text, no scores, no new values. The words allow, ask and deny never appear in a label.

**P4. When in genuine doubt, choose the riskier value.** If two values stay defensible after applying the rules, pick the riskier one and record why.

**P5. Facts here, opinions in profiles.** Labels state what an action *is*. Whether it needs a person's approval is a policy question that a profile answers. Never reason backwards from "this should be allowed" to a label.

**P6. One action, one set of labels.** No duplicates for the same `(key, def_hash)`.

## 3. The six labels

Answer the six questions in this order: **mutation → scope → reversibility → egress → financial → returns_class**. Earlier answers inform later ones: reversibility depends on what changes, and egress affects reversibility.

From least to most risky:

| Label | Least risky → most risky |
|---|---|
| mutation | read → create → update → delete |
| scope | single_record → bulk → account_wide |
| reversibility | reversible → reversible_with_effort → irreversible |
| egress | none → internal → external |
| financial | none → indirect → direct |
| returns_class | first_party_structured → third_party_content |

### 3.1 mutation: `read | create | update | delete`

**Question:** does this action change any state outside the current session, and how?

- **read**: no state changes anywhere. Queries, searches, fetches, lists. An action whose main job is retrieval is `read`, even if the provider logs the access.
- **create**: brings new state into existence without changing or removing what exists. Sending a message is `create`: it creates a new object. The danger of sending is captured by egress and reversibility, not by inflating mutation.
- **update**: changes existing records or state, including moving, renaming, relabelling, or archiving where the object survives.
- **delete**: removes existing state, or overwrites it destructively (force-push, truncate, overwrite on write).

Rules:
- If one action can do several of these (upsert, write-or-overwrite), label the most severe: delete over update over create.
- Archive with a real unarchive is `update`. Hard delete is `delete`. Delete-to-trash is `delete`: the record leaves its live place, and recovery is judged under reversibility.
- Actions that run arbitrary code or queries (run SQL, run code, evaluate a script, browser navigation with side effects) are `delete`: what they can do includes destruction.

Examples:
- `mail/send_email` → `create`. A new sent message exists; nothing existing changed.
- `files/write_file` → `delete`. The schema allows writing to an existing path, which overwrites it.
- `repo/update_issue` → `update`. An existing record changes and survives.
- `sql/execute_sql` → `delete`. Arbitrary SQL includes dropping and deleting.

### 3.2 scope: `single_record | bulk | account_wide`

**Question:** what is the largest blast radius of one call?

- **single_record**: one identified object: one message, one file, one row, one event.
- **bulk**: a set in one call: batch endpoints, query-addressed operations ("delete where …"), recursive operations, or one object that holds many of the user's assets.
- **account_wide**: the whole account or workspace, or its configuration: permissions, members, API keys, billing settings, integrations, organisation policies.

Rules:
- **Containers count as their contents.** Deleting a repository, dropping a table or removing a folder tree is one object but many assets: `bulk` at least.
- Anything that touches sign-in, permissions, workspace settings or standing configuration (webhooks, forwarding rules, filters) is `account_wide`, however small the change looks. Configuration changes what every future action can do.
- Arbitrary SQL or code is `account_wide`: what it can do includes dropping a database and changing settings.

Examples:
- `chat/send_message` → `single_record`.
- `repo/delete_repository` → `bulk` (container rule).
- `mail/create_filter` → `account_wide` (standing configuration: a filter can silently forward or delete future mail).
- `sql/execute_sql` → `account_wide`.

### 3.3 reversibility: `reversible | reversible_with_effort | irreversible`

**Question:** after the action completes, is there an action that fully restores the state before it?

- **reversible**: the same API offers a first-class undo that restores the state completely. Create page, then trash page. Add label, then remove label.
- **reversible_with_effort**: recovery is possible but needs manual work, a recovery window, extra privileges, support, or a restore from trash or backup.
- **irreversible**: nothing restores the prior state.

Rules:
- **The observation rule.** If the action delivers information to people outside the caller's control, it is `irreversible`, whatever delete endpoint exists. A sent email can be deleted from Sent; it cannot be unsent. A public post can be removed; it cannot be unseen or uncached. So `egress = external` on a `create`, `update` or `delete` action forces `irreversible`.
- Money that has left the account is `irreversible` for this API, unless the same API offers a void or reversal within a guaranteed window; then it is `reversible_with_effort`.
- `read` actions are `reversible` (nothing changed).
- Judge recovery only by what the same server or API actually provides. "The user probably has backups" does not count. (Section 5 covers recovery that is proven present at the time of a call; it never changes a label.)

Examples:
- `mail/send_email` → `irreversible` (observation rule).
- `files/delete_file`, where the file goes to a trash folder → `reversible_with_effort`.
- `payments/create_refund` → `irreversible`: once the money reaches the customer, the API offers no undo.
- `docs/update_page` → `reversible_with_effort`: page history exists, but restoring it is manual.

### 3.4 egress: `none | internal | external`

**Question:** who, at worst, can *receive* data through this action?

- **none**: no data leaves the boundary between the user and the server. Reads bring data into the session; that is not egress. Writes to the user's own private store with no other readers.
- **internal**: recipients are limited to the signed-in workspace or account: members of your chat workspace, collaborators on a private repository, attendees inside your company's domain.
- **external**: the action can put data in front of anyone outside the workspace: email to any address, a request to any URL, public posts or issues, text messages.

Rules:
- Judge by what the schema allows: if a recipient, URL or destination parameter *can* address outsiders, the value is `external` (P1).
- **URLs are a way out.** A tool that requests arbitrary URLs (fetch, browse, webhook testers) is `external`, even though its purpose is retrieval: data can leave in the URL's path and query string.
- Writes to places anyone can read (public repositories, public channels, published pages) are `external`.
- Local-only actions (local files, local git) are `none`.

Examples:
- `web/fetch` → `external` (the URL itself carries data out).
- `chat/send_message` → `internal`, where channels are limited to the workspace. Shared channels that include outside organisations are an open question (section 7); a cautious profile may treat them as external.
- `repo/create_issue` → `external` (the schema allows public repositories, which anyone can read).
- `files/write_file` → `none`.

### 3.5 financial: `none | indirect | direct`

**Question:** does this action move money or commit spend?

- **direct**: moves money or creates or changes a financial obligation: charge, refund, transfer, payout, purchase, subscription change, invoice.
- **indirect**: commits spend as a side effect by using metered or billable resources: starting compute, sending per-message-billed texts, launching paid jobs.
- **none**: no financial effect beyond negligible API usage.

Rules:
- Issuing an invoice or payment request is `direct`: it creates an obligation, even before money moves.
- Provisioning that costs money over time is `indirect`, however small the unit cost: the spend has no built-in limit.

Examples:
- `payments/create_refund` → `direct`.
- `payments/create_invoice` → `direct` (obligation rule).
- `cloud/create_instance` → `indirect`.
- `mail/send_email` → `none`.

### 3.6 returns_class: `first_party_structured | third_party_content`

**Question:** can what the action returns contain free text written by someone other than the user or the service itself?

- **first_party_structured**: status codes, IDs, counts, balances, timestamps, fixed values: things the service generates.
- **third_party_content**: emails, messages, issues, comments, tickets, web pages, file contents, search results: anything a stranger could have written. This value marks where outside text enters the agent's context, which is where planted instructions come from.

Rules:
- If *any* field of the response can carry text a stranger wrote, the value is `third_party_content`. Partial structure does not clean the payload.
- **Names are content.** Lists that return titles, file names or subject lines written by others are `third_party_content`: a subject line can carry planted instructions as well as a body can.
- File reads are `third_party_content`: files can hold content from anywhere.
- This label is deliberately strict. Marking too much costs approval prompts; marking too little can complete a data leak. When torn, mark it.

Examples:
- `mail/get_message` → `third_party_content`.
- `mail/send_email` → `first_party_structured` (returns a send status and a message ID).
- `payments/get_balance` → `first_party_structured`.
- `files/list_directory` → `third_party_content` (names rule).

## 4. Tools that can do anything in a domain, and four facts

Tools whose capability is "do anything within a domain" (run SQL, run code, run shell commands, control a full browser) are labelled at the domain's ceiling, not at their typical use:

| Tool class | mutation | scope | reversibility | egress | financial |
|---|---|---|---|---|---|
| Arbitrary SQL | delete | account_wide | irreversible | none* | none |
| Arbitrary code or shell | delete | account_wide | irreversible | external | indirect |
| Full browser control | delete | account_wide | irreversible | external | direct |

\* `none` only when the database cannot make outbound requests. If extensions allow web requests, `external`.

Four facts sit beside the six labels. None of them is a seventh label, and none is a verdict. Each records *why* a label came out the way it did, so a profile can treat different situations differently. A profile may ignore any of them.

### 4.1 `open_world`

An action labelled from the ceiling table above carries `open_world: true`.

The ceiling comes from worst-case reasoning about a domain, not from a destructive step the tool names. A tool called "delete account" and a browser click can reach the same six values for different reasons: one names destruction in its contract, the other could reach it through a page it did not write. The facts match; the situations do not. So the difference is recorded.

Without this fact, labelling a shell or a browser honestly would make it look exactly like an action that names account-wide destruction, and a strict profile would have to refuse it outright. Teams would then mislabel the tool or drop the checks.

**Split tools.** Where a host offers a domain as several narrow actions (click, press, type) instead of one browser tool, label each action on what it does, not on what a session could do by combining them. A click submits or adds state through one element: `create`, `single_record`, `reversible`. The ceiling still applies to any single action that takes an arbitrary program: a browser console action that runs JavaScript is `open_world: true`.

Combining actions across calls is real. It is the profile's problem, not the label's.

### 4.2 `configuration_object`

An action with `mutation: delete` carries `configuration_object: true` when it removes **configuration attached to an object that survives**, not the object or its data. A row-level security policy, a trigger, a constraint, a database role, a table column, a mail filter: each is state, and removing it is `delete` under 3.1. But the table still exists, the database still exists, and no user data is gone.

**This does not change the mutation value.** Dropping a policy removes state. Labelling it `update` because the container survives would be reasoning backwards from a wanted verdict, which P5 forbids.

Rule: set the fact when *both* hold: the removal targets a named configuration or schema object, and the object it is attached to is unaffected. It is `false` when the action can remove the container itself (drop table, drop database, delete repository), and `false` for anything that removes user data (delete rows, truncate, delete a message). A tool that runs arbitrary code or SQL is always `false`: it can do both, and P1 labels the worst case.

Examples:
- Drop a row-level security policy → `true`. The table and its rows remain.
- Drop a database role → `true`. The database and the data that role owned remain.
- Alter a table to drop a column → `true`. That column's values are destroyed, but the table survives. A close call.
- Run arbitrary SQL → `false`.
- Delete an email → `false`. The message is the data.

Why it exists: these actions all land on `delete` + `account_wide` + `irreversible`, the combination a strict profile refuses outright. A team whose agent may never drop an index, revoke a grant or alter a column does not have a working agent. The fact lets a profile ask a person instead of refusing.

### 4.3 Unconstrained addressing

4.1 labels a tool at its domain's ceiling when it takes an arbitrary *program*. The same reasoning applies when it takes an arbitrary *address*. In both cases the tool's own contract puts no limit on what a call can reach, so any limit a label claims is a limit on one argument that happened to be passed, not on the tool.

Rule: where a tool's target is an unconstrained argument (a file path with no declared root, a URL, a search root, a repository query), `scope` is the ceiling of what that argument can address. It is not `single_record` just because one is named per call.

This fixes `scope` only. A read that can reach any file is still `mutation: read`. Unconstrained addressing widens the blast radius; it does not invent an effect the tool lacks.

The lesson behind 4.1, 4.2 and 4.3 is the same: a label that describes the ordinary case leaves the dangerous case looking identical to it.

Example: a file-read tool whose `path` has no declared root reads any file the agent's process can read, including SSH keys, cloud credentials and `.env` files. Its scope is `account_wide`, not "one path per call".

This does not mean every read gets gated. The ceiling is where a label starts, not where a call ends: section 5 explains how a single call can count as narrower on proof.

An action labelled under this rule carries `open_world: true` when the unconstrained argument is the tool's main target.

### 4.4 `pure_deletion`

Under 3.1, a write tool that *can* overwrite an existing target is labelled `delete`, the same value as a tool whose only job is removal. A write and a removal can then look identical on all six labels.

They are not the same. A write proven to land on one ordinary file inside the workspace is the substance of a coding agent's work, and asking a person about every such write would drown them in prompts. That reasoning does not hold for a removal, because nothing survives for the next step to read back.

Rule: `pure_deletion: true` when the action's target stops existing and nothing is put in its place. `false` (the default) when the `delete` label comes from overwrite capability: the target survives with a new value. Set it by reading what the tool actually does.

- Delete an email → `true`. The message is gone; nothing is written in its place.
- Write a file, or download an attachment to a path → `false`. Both can overwrite an existing file, by writing new content there. A target that receives new content has not stopped existing.

The fact defaults to `false`, and an unreviewed action never claims it.

## 5. From a label to a single call

Labels describe the tool. A single call can be narrower: a file write that provably stays inside the project folder, a shell command that provably only lists files. Four rules govern when a call may count as safer than its tool's labels:

1. **Only downward.** Evidence about a call may make a label less risky, never more. Ignoring this evidence altogether is always the cautious choice.
2. **Only proof can relax.** Evidence comes in three strengths: *proven* (decidable from the arguments alone, and cannot be false: a path contains no `..`, a command splits into three words), *point-in-time* (read from something real that can change, like a file on disk), and *guess* (a path named `.env` probably holds secrets). Only proven evidence may make a call count as safer. Weaker evidence may raise an alarm or ask a person. It never permits.
3. **Silence is not agreement.** Missing evidence never matches. Otherwise relaxations would fire hardest when the checker knows least.
4. **The environment counts only when the checker proves it itself.** A sandbox with no network, or a snapshot that can restore the prior state, can make a call safer, but only when the checker establishes it directly, not on the word of the host it supervises. It never changes a label, and if it cannot be established, it counts as absent.

This is what keeps honest labels usable. A tool labelled at its ceiling would otherwise be gated forever. With these rules, ordinary calls stay quiet, and a call nobody can vouch for keeps its ceiling. An incomplete list of sensitive names then costs extra prompts, not leaked secrets.

## 6. How to label a tool

1. Get the tool's exact definition and compute its `def_hash`.
2. Gather evidence in the order of P2. Record source links and, for code, the commit.
3. Answer the six questions in the order of section 3.
4. Write one line of reasoning per label, citing the deciding rule (for example, "irreversible: observation rule").
5. Record who reviewed it, how, and when. When two reviewers disagree, record the disagreement and choose the riskier value (P4).

## 7. Changes and open questions

The six labels and their values are fixed. Changes are proposed with the case that triggered them and the reasoning, and released in the next version. A label never anticipates a change that has not been released.

Open questions for the next version:
- Shared chat channels that include outside organisations: `internal` or `external` (3.4)?
- Refunds that a payment provider can void within a window: `irreversible` or `reversible_with_effort` (3.3)?
- Should `public` split from `external` (3.4)?

Proposals and disagreements are welcome.

## Appendix: worked examples

**Send an email** (`mail/send_email`)

```
mutation: create                       a new message; nothing existing changed
scope: single_record                   one message
reversibility: irreversible            observation rule: delivered to recipients
egress: external                       the recipient field takes any address
financial: none
returns_class: first_party_structured  send status and message ID
```

**Read a file** (`files/read_file`, no declared root)

```
mutation: read
scope: account_wide                    4.3: the path has no declared root, so the tool
                                       reads any file the process can, including SSH keys
reversibility: reversible              nothing changed
egress: none                           data comes into the session
financial: none
returns_class: third_party_content     anyone could have written the file
open_world: true                       4.3: the unconstrained argument is the target
```

A call can count as `single_record` when its path is proven to stay inside the declared workspace and names nothing sensitive (section 5). "One path per call" describes the argument, not the tool: the call reads one file, and the tool reads any of them.

**Issue a refund** (`payments/create_refund`)

```
mutation: create                       a new refund
scope: single_record                   one refund
reversibility: irreversible            the money moves; the API offers no undo
egress: none                           no outside recipient of data
financial: direct                      moves money
returns_class: first_party_structured  refund status and ID
```

---

Maintained by Astroware, which builds EU and security controls for AI agents. Licensed under CC BY 4.0: copy, adapt and use it, including commercially, with credit to Astroware, a link to the licence, and a note of any changes.
