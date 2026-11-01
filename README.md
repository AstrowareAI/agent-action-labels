# Agent Action Labels

Six labels for what an AI agent's action can do if it goes wrong.

Give every tool action an agent can take (send an email, run a command, issue a refund, delete a file) six labels. The labels describe the worst the action can do. They never say whether to allow it: that is for a profile, which each team writes for itself.

| Label | Least risky → most risky |
|---|---|
| mutation | read → create → update → delete |
| scope | single_record → bulk → account_wide |
| reversibility | reversible → reversible_with_effort → irreversible |
| egress | none → internal → external |
| financial | none → indirect → direct |
| returns_class | first_party_structured → third_party_content |

Full text, rules and worked examples: [SPEC.md](SPEC.md). Machine-readable examples: [examples/](examples/).

Use it to label the tools your agents can call, to compare tools, or to write rules for what an agent may do without asking a person.

Maintained by [Astroware](https://astroware.ai). Licensed under [CC BY 4.0](LICENSE): copy, adapt and use it, including commercially, with credit and a note of any changes. Disagreements and proposals are welcome as issues.
