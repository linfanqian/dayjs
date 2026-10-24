# Day.js

Day.js is a small date library. Its API is largely compatible with Moment.js, and its objects are immutable.
The core stays small and has a size limit. Extra features come as optional plugins and locales.
There is no app or dev server.

## Architecture
- Entry point: `src/index.js`. It defines the `Dayjs` class and exports the `dayjs()` function, which carries
  `dayjs.extend` and `dayjs.locale`. Shared helpers are in `src/utils.js` (plugins get them with
  `this.$utils()`) and `src/constant.js`.
- Plugins: `src/plugin/<camelName>/index.js`. Each exports one function that `dayjs.extend()` calls:
  `export default (o, c, d) => {}` (option, Dayjs class, `dayjs`).
  - Most add new methods to Day.js objects or wrap existing ones
    (`const oldAdd = proto.add` ... `oldAdd.bind(this)(...)`).
  - A few (e.g. `minMax`, `isMoment`, `updateLocale`) only add functions to `dayjs` itself.
- Locales: `src/locale/<lowercase-code>.js`. The first line is a comment `// <Language name> [<code>]`.
  The file then does `import dayjs from 'dayjs'`, `dayjs.locale(locale, null, true)`, `export default locale`.
  - `test/locale/keys.test.js` fails if the comment is missing or `name` differs from the lowercase filename.
    `build/index.js` also reads the comment.
  - Exception: `en.js` is built into the core. It has the comment but no `dayjs` import or `dayjs.locale` call.
- Types: `types/` (`index.d.ts` for the core, `plugin/<name>.d.ts`, `locale/`). They are hand-written, and no
  script type-checks them. Update them by hand when the JS API changes.
  - If a plugin adds API, its type file extends `declare module 'dayjs'`.
- Tests: `test/` (Jest): core in `test/*.test.js`, plus `test/plugin/`, `test/locale/`, `test/issues/`.
  `test/__mocks__/dayjs.js` makes `import dayjs from 'dayjs'` load `src/`.
- Build scripts: `build/`. These are source files, not build output. Lint checks them too.
- Build output is not in git. Commands create it in the repo root:
  - `npm run build`: `plugin/`, `locale/`, `dayjs.min.js`, `index.d.ts`, `locale.json`
  - `npm run babel`: `esm/`
  - Jest with coverage: `coverage/`
  - Root `plugin/` and `locale/` are not the source folders `src/plugin/` and `src/locale/`.

## Commands (repo root)
- Install: `npm ci`. It installs from the lockfile and adds a git pre-commit hook that runs lint.
- One test file: `npx jest test/plugin/isToday.test.js --coverage=false`. This does not check the coverage limit.
- All tests: `npm test`. It runs several timezone stages (`test-tz` and `test-tz-plugin`; the `test` script lists
  the timezones), then all tests with a 100% line-coverage limit.
- Lint: `npm run lint`. It runs ESLint on `src/* test/* build/*`. CI runs lint and `npm test` on ubuntu.
- Build: `npm run build`. It bundles the code, copies `types/` to the repo root, then runs `npm run size` to check
  the size of `dayjs.min.js`. Skip it for test-only changes; run it when the core or bundling changes.
- Smoke test after a build (should print `2018-08-09`):
  `node -e "const d=require('./dayjs.min.js');console.log(d('2018-08-08').add(1,'day').format('YYYY-MM-DD'))"`

Notes (any OS):
- Some tests depend on the timezone. The final coverage stage can fail in a non-UTC timezone and pass with
  `TZ=UTC`. When you report a full test run, run that stage in the host timezone first. Report a `TZ=UTC`
  rerun separately, not as a replacement. That stage alone is not `npm test`.
- `npm run size` uses webpack 4, which hashes with MD4. Node 17 and later use OpenSSL 3, which disables MD4,
  so the size check fails with `ERR_OSSL_EVP_UNSUPPORTED`. After a build, rerun it with
  `npx cross-env NODE_OPTIONS=--openssl-legacy-provider npm run size`.

Windows (npm runs scripts in cmd.exe by default; use Git Bash):
- `npm run lint` fails in cmd.exe: cmd.exe can't run `./node_modules/.bin/eslint`.
- Lint: `npm run lint --script-shell="<Git for Windows dir>/bin/bash.exe"`.
- Tests: run each stage of the `test` script separately, using cross-env to set `TZ`, e.g.
  `npx cross-env TZ=Pacific/Auckland node node_modules/jest/bin/jest.js test/timezone.test --coverage=false`
  Final stage:
  `node node_modules/jest/bin/jest.js --coverage --coverageThreshold='{ "global": { "lines": 100} }'`
  For the UTC rerun, put `node node_modules/cross-env/dist/bin/cross-env.js TZ=UTC` in front of it.
- Don't run test scripts through bash, and don't put `TZ=...` before a command. Git Bash drops `TZ` before
  Node starts, so timezone tests run in the wrong timezone.

## Conventions
- Style (airbnb-base + `.eslintrc.json`): no semicolons, no trailing commas, no file extensions in imports,
  2-space indent, LF line endings. Prettier only formats `docs/**/*.md`.
- Day.js objects are immutable. `set`, `add`, and other date-changing methods return a new object.
  `$set` is the internal method that changes an object in place. Plugin methods must not
  change the object they are called on. The `badMutable` plugin breaks this on purpose.
- A new plugin needs three files: `src/plugin/<name>/index.js`, `test/plugin/<name>.test.js`, and
  `types/plugin/<name>.d.ts`. `test/plugin/declarations.test.js` fails if the type file is missing.
- Tests usually import `dayjs` and plugins/locales from `src/` with a relative path (`../src` in `test/`,
  `../../src` in subfolders). Tests that use a plugin call `dayjs.extend(...)` at the top of the file.
- Most tests that use the current time keep it constant: `MockDate.set(new Date())` in `beforeEach`,
  `MockDate.reset()` in `afterEach`.
- Many tests compare results with Moment.js (a pinned dev dependency) instead of writing the expected value by
  hand. Keep this style when the behavior should match Moment.js.
- Commit messages use semantic prefixes, e.g. `fix(plugin): ...`, `docs: ...`.

## Guardrails
- Don't hand-edit build output (listed above), `CHANGELOG.md` (semantic-release writes it at release time), or
  `version` in `package.json` (a placeholder, `0.0.0-development`).
- Don't modify `test/browser.spec.js`; its own comment says so.
- Don't fix `linebreak-style` errors by converting line endings across the repo, running `--fix` on the whole
  repo, or turning off the rule. Report them instead.
- Change these only when the task requires it:
  - `package-lock.json`: change it through npm as part of a dependency change.
  - `patches/` (patches for release tooling), `.github/workflows/`, `.releaserc`, and `build/`.
