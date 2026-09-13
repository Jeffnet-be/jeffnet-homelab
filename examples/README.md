# Examples

Self-contained patterns lifted out of a private infrastructure repository, with estate-specific
names generalised. Each one exists because of a fault written up in
[docs/findings](../docs/findings/).

| Pattern | What it is for |
|---|---|
| [service-contract](ansible/service-contract/) | Declare a service once; derive the proxy route, dashboard card, uptime check and metric targets |
| [compose-stack-guard](ansible/compose-stack-guard/) | Prove that variables declared in a stack's `.env` actually reached the container |

Both are reconstructions rather than copies. They are the right shape and the right reasoning; they
are not wired to anything.
