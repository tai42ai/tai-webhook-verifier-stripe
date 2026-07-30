# Contributing to tai42-webhook-verifier-stripe

`tai42-webhook-verifier-stripe` is a per-provider **webhook-signature verifier**
plugin for the TAI ecosystem: it authenticates each inbound Stripe delivery by
its `Stripe-Signature` HMAC-SHA256 before the payload is parsed or dispatched.
The hard rule (the plugin rule): **it depends on `tai42-contract` only and never
imports the skeleton.** It registers through the `tai42_app` handle from
`tai42_contract.app` and is loaded by the host from the manifest's
`webhook_verifier_modules` field by dynamic import — there is no import edge to
the skeleton in either direction.

> A webhook door binds a named verifier to a topic; this plugin supplies the
> `stripe` verifier.

## Ground rules

- **No skeleton import — ever.** The package is contract-facing; the ban is
  enforced by ruff (`flake8-tidy-imports`), so a stray import fails lint:
  ```bash
  grep -rn "tai42_skeleton" src/   # must be empty
  ```
- **Fails closed.** A misconfigured secret (a missing `secret_env` key, a
  missing environment variable, or an empty secret value) and a non-positive or
  non-int `tolerance_seconds` raise loudly (`KeyError` / `ValueError`) rather
  than being treated as an ordinary signature failure — a misconfigured door is
  never a silently-unauthenticated one.
- **Loud errors.** No swallowed exceptions, silent fallbacks, or silent
  truncation. A missing header, an unparsable `Stripe-Signature`, a stale
  timestamp, or no matching `v1` digest raises `WebhookVerificationError`; an
  operator misconfiguration raises.
- **A malformed `v1` is skipped, not fatal.** A `v1` of the wrong length or
  non-hex is dropped from the candidate set; verification fails only when no
  well-formed candidate matches. A delivery mid-rotation carries several `v1`
  signatures and one unusable entry must not reject a delivery that also carries
  a good one.
- **The freshness window is one-sided.** A stale timestamp
  (`now - t > tolerance_seconds`) is rejected; a future one is not — sender
  clock skew is not an attack signal.
- **The secret never leaves the environment.** It is read from `os.environ` at
  verify time and never carried in the per-binding `config`, a fixture, or a
  test. The only signing material in the tree is a locally computed example
  vector.
- **Constant-time compare.** The final digest check uses `hmac.compare_digest`;
  never replace it with a plain `==`.
- **Typed package** (`py.typed`). Pyright runs clean.

## Layout

- `src/tai42_webhook_verifier_stripe/__init__.py` — the import-only registration
  side effect: `tai42_app.webhook_verifiers.register("stripe", StripeWebhookVerifier())`.
- `src/tai42_webhook_verifier_stripe/verifier.py` — `StripeWebhookVerifier` and its
  private header-parsing helpers.
- `tests/` mirrors `src/`.

## Naming

PyPI is a flat namespace with no owner in the path, so distributions carry the
`tai42-` prefix. GitHub repositories keep their `tai-` names, because the
`tai42ai` organisation already namespaces them. Import packages follow the
distribution.

| Surface | Form |
| --- | --- |
| Distribution — PyPI, `pip install`, dependency pins | `tai42-<name>` |
| Import package | `tai42_<name>` |
| GitHub repository | `tai-<name>` |

So a dependency is declared as `tai42-<name>` while its repository is named
`tai-<name>`, and both spellings are correct in their own context.

Some surfaces are deliberately neither, and must not be renamed: the `tai` CLI
command (`tai42` is an alias), the Prometheus metric namespace (`tai_tool_*`),
`TAI_*` environment variables, and the `tai-plugin.yml` descriptor filename.

## Dev

```bash
uv venv --python 3.13
uv pip install --no-sources --group dev --editable .
uv run --no-sync pytest --cov --cov-report=term-missing
uv run --no-sync ruff check .
uv run --no-sync ruff format --check .
uv run --no-sync pyright
```

`make dev` installs the sibling `tai-contract` repo as an editable install for local cross-repo development.

Before any commit, run a secret scan over `src/` and `tests/` (e.g.
`detect-secrets scan`).

## Dependency resolution

`uv.lock` pins the `tai42-*` siblings to their released index versions while `[tool.uv.sources]` points them at local `../tai-*` checkouts. The two disagree deliberately: CI sets `UV_NO_SOURCES=1` and asserts the lock with `uv sync --locked`, so it resolves the artifacts a user installs. A bare `uv lock` beside sibling checkouts re-couples the lock to editable path entries, which then fails that `--locked` check — run `uv lock --no-sources` instead. See [How dependencies resolve](https://tai42.ai/contributing#how-dependencies-resolve).

## License

By contributing you agree your contributions are licensed under Apache-2.0.
