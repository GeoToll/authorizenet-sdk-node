# authorizenet-sdk-node

GeoToll's fork of [AuthorizeNet/sdk-node](https://github.com/AuthorizeNet/sdk-node), the
Authorize.Net payment gateway SDK for Node.js (npm package name `authorizenet`). The fork exists
to keep the SDK's dependencies patched (mainly `axios`). It is not published separately:
`geotoll-system` (`api/`, `azure/elk-sync/`) installs `authorizenet` from the npm registry and
forces a patched `axios` through `overrides`. Keep changes minimal so upstream stays mergeable.

## Stack

- Plain CommonJS JavaScript, Node 22/24 (CI matrix); no build step, no TypeScript.
- Runtime deps: `axios` (HTTP, pinned exact), `https-proxy-agent`, `winston` + `winston-daily-rotate-file` (logging).
- Dev: `mocha` (tests), `chai`, `eslint` (`.eslintrc.yml`), `jsonix` (schema code generation).
- No `package-lock.json` is committed (library; upstream convention).

## Layout

- `lib/authorizenet.js` - entry point; exports `APIContracts`, `APIControllers`, `Constants`.
- `lib/apicontracts.js`, `lib/apicontrollers.js` - generated request/response types and controllers (large).
- `lib/apicontrollersbase.js` - HTTP call via axios, optional proxy agent.
- `lib/config.js` (timeout, logger, proxy), `lib/constants.js` (sandbox/production endpoints), `lib/logger.js`, `lib/utils.js`.
- `scripts/` - `masterUpdate.sh` + `generate*FromSchema.{sh,js}` regenerate `lib/` from the Authorize.Net XSD (`mappings/Schema.js`, jsonix).
- `test/` - mocha suites per API area; `test/constants.js` holds the sandbox credentials used by the suites.
- `sample/test.js` - minimal usage example. `sample-code-node/` - git submodule (upstream sample code, usually empty).

## Commands

- Install: `npm install`
- Test: `npm test` (= `mocha`; single file: `npx mocha test/<file>.js`). The suites call the live
  Authorize.Net sandbox (`apitest.authorize.net`), so they need network access and valid sandbox
  credentials. `npx mocha --dry-run` checks that all suites load without network.
- Lint: `npx eslint lib` (no npm script; the generated `lib/` files do not pass the current rules).
- Regenerate SDK from schema: `bash scripts/masterUpdate.sh` (from repo root).
- Pack: `npm pack`.

## Environments & config

No environment variables. Credentials are passed in code via `MerchantAuthenticationType`
(`setName`, `setTransactionKey`). Endpoint is chosen with `ctrl.setEnvironment(Constants.endpoint.sandbox|production)`.
Proxy, timeout and logging come from `lib/config.js` or a `config` object passed to a controller.

## CI/CD

`.github/workflows/nodejs-workflow.yml` (push, pull_request, manual): on Node 22.x and 24.x it runs
`npm install && npm test`, packs the SDK, installs it into a checkout of
`authorizenet/sample-code-node` and runs its `test-runner.js`. `.travis.yml` is a leftover from upstream.
The sandbox account is shared, so runs are serialized (`max-parallel: 1`, per-ref `concurrency`, push only on `master`);
parallel runs trigger duplicate-transaction errors. `test-paypalexpresscheckout.js` and the Capture Only test are
`describe.skip` because the sandbox rejects them (errors 19/63 and 67).
No release/publish pipeline.

## Dependencies & security

- `.github/dependabot.yml`: weekly npm (`/`) and github-actions (`/`) updates; minor+patch grouped.
- Node is pinned to the 24 LTS major (GeoToll/backlog#268): Dependabot ignores `@types/node` majors.
- `axios` is pinned to an exact version; bump it to the first patched release when alerts appear,
  and keep the matching `overrides.authorizenet.axios` in `geotoll-system` in sync.
- `overrides.xmldom` maps the abandoned `xmldom` (used by dev-only `jsonix`) to `@xmldom/xmldom`.
- Patching: `npm install`, `npm audit fix` (no `--force`), then `overrides` for leftover transitive alerts.

## Conventions

- Default branch: `master`.
- Branches: `{user}/issue-{n}-{desc}`, e.g. `obalyuk87/issue-347-security-patching`.
- Commit / PR titles end with `| GeoToll/backlog#N`; PR body starts with `Relates to GeoToll/backlog#N`.
