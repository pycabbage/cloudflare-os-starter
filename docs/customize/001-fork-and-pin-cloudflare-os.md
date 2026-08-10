# Forking upstream `cloudflare-os` and re-pinning the submodule

## Why

Two planned changes need code that the wrapper boundary (`deployment.jsonc`, `/admin`,
`packages/custom-gatekeeper`) can't express, per [Code extensions](../customization.md#code-extensions):

- **AI Gateway BYOK for additional providers.** `AiModelProvider` in
  `packages/workshop-shared/src/api.ts` is a closed union
  (`"openai" | "anthropic" | "google" | "cloudflare" | "ollama"`), and
  `gatewayNativeModel()` in `packages/workshop-backend/src/ai-models.ts` only has cases for
  those four gateway-routed providers. Adding AWS Bedrock, OpenRouter, and Z.ai (GLM) as
  first-class model options — on top of BYOK keys already stored on the Cloudflare-managed AI
  Gateway — requires extending both. OpenRouter and Z.ai are OpenAI-compatible and can likely
  reuse the existing `openai-completions`/`openai-responses` pi-ai API handling; AWS Bedrock's
  native shape is unconfirmed and may need more care.
- **Google Chat.** Not yet decided whether this needs upstream changes. If it ships as a new,
  independent custom Gatekeeper (own OAuth client, own Worker, alongside the existing "Google"
  vendor), it fits entirely inside `packages/custom-gatekeeper` and needs no upstream fork. It
  only requires touching `packages/gatekeeper-google` upstream if Chat should share the *same*
  connected-Google-account session as Gmail/Calendar/Docs/Sheets instead of being a separate
  connection.

Since at least one of the two needs upstream edits, we forked rather than patching the
submodule checkout locally, so the changes are reviewable commits instead of an untracked
overlay.

## What changed

- Forked `cloudflare/cloudflare-os` to
  [`pycabbage/cloudflare-os`](https://github.com/pycabbage/cloudflare-os).
- Re-pointed the submodule in `.gitmodules`:
  `https://github.com/cloudflare/cloudflare-os.git` → `git@github.com:pycabbage/cloudflare-os.git`.
- Updated the pinned commit. `cloudflare-os-starter` had been pinned to
  `bf7f762d7fa73553284d731ab6a978d3ea17be24`, which was already behind upstream `main`. Fetching
  from the fork and checking out `main` landed on the fork's tip, `b2a51b5426398c8353d9d4dd984bd525121ab5f2`
  — effectively an upstream upgrade, not just a URL swap.
- Committed as `8cf7154 Update cloudflare-os submodule to b2a51b5`.

## Upgrade review (`bf7f762..b2a51b5`, 12 commits)

Reviewed before committing, per the [Upgrade checklist](../customization.md#upgrade):

- No changes to the Gatekeeper contract (`packages/workshop-shared/src/gatekeeper.ts` interfaces
  are unchanged; only a `stripTrailingSlashes` helper was added and reused across several
  gatekeeper packages, including `gatekeeper-google`).
- No breaking changes to Wrangler bindings, env vars, or migrations. `workshop-backend/wrangler.jsonc`
  gained an additive `observability.traces` key, which doesn't affect this starter since
  `scripts/deploy.mjs` generates that block from `deployment.jsonc` instead of copying upstream's.
- `scripts/release/manifest-lib.mjs` gained `PREINSTALL`/`SINGLETON` handling, used only by
  upstream's own hosted-deploy release pipeline — not read by this starter's `scripts/deploy.mjs`.
- Everything else was CI/repo hygiene, regenerated `worker-configuration.d.ts` files, observability
  logging additions, and backward-compatible bug fixes — including a Google OAuth insufficient-scope
  fix in `ObserverConfigModal.tsx` (#52) that's directly relevant to the Google Chat work above.

## Rollback

If this needs to be undone:

```sh
git submodule set-url cloudflare-os https://github.com/cloudflare/cloudflare-os.git
git submodule sync cloudflare-os
cd cloudflare-os && git checkout bf7f762d7fa73553284d731ab6a978d3ea17be24 && cd ..
git add .gitmodules cloudflare-os
git commit -m "Revert cloudflare-os submodule to upstream bf7f762"
```
