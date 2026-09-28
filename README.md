# pnpm 12.4.2+: `pnpm dedupe` is not idempotent

Right after `pnpm dedupe`, `pnpm dedupe --check` reports more changes, and a second
`pnpm dedupe` rewrites the lockfile again. It settles after the second run.

No settings in `pnpm-workspace.yaml` beyond `packages`. Two workspace projects and one
local `file:` dependency, all other packages public.

```
local-ui         deps: styled-components@5.3.11   peer: antd@^5.2.0
packages/app-a   deps: local-ui (file:), react@18.3.1, react-dom@18.3.1, react-date-range@^2
                 devDeps: @types/react-date-range@^1
packages/app-b   deps: local-ui (file:), react@18.3.1, react-date-range@^2,
                       react-dates@^21, @mui/x-date-pickers@^6
```

## Reproduce

```sh
rm -f pnpm-lock.yaml
corepack pnpm@12.8.1 install --lockfile-only
corepack pnpm@12.8.1 dedupe --lockfile-only
corepack pnpm@12.8.1 dedupe --check      # ERR_PNPM_DEDUPE_CHECK_ISSUES
corepack pnpm@12.8.1 dedupe --lockfile-only   # changes the lockfile again
corepack pnpm@12.8.1 dedupe --check      # clean
```

## What changes on the second run

After the first `dedupe`, the lockfile is internally inconsistent: `styled-components`
is keyed with `@babel/core@7.29.7(supports-color@5.5.0)`, while its child
`babel-plugin-styled-components` still refers to plain `@babel/core@7.29.7`. The second
run adds the missing `(supports-color@5.5.0)` suffix:

```diff
-  '@babel/plugin-syntax-jsx@7.29.7(@babel/core@7.29.7)':
+  '@babel/plugin-syntax-jsx@7.29.7(@babel/core@7.29.7(supports-color@5.5.0))':
-  babel-plugin-styled-components@2.3.0(@babel/core@7.29.7)(styled-components@5.3.11(@babel/core@7.29.7)(react-is@19.3.0)(react@18.3.1))(supports-color@5.5.0):
+  babel-plugin-styled-components@2.3.0(@babel/core@7.29.7(supports-color@5.5.0))(styled-components@5.3.11(@babel/core@7.29.7(supports-color@5.5.0))(react-is@19.3.0)(react@18.3.1))(supports-color@5.5.0):
```

## Versions

| pnpm | `dedupe --check` after one `dedupe` |
|---|---|
| 10.34.5 | clean |
| 11.28.2 | clean |
| 12.0.0 – 12.4.1 (every release) | clean |
| 12.4.2 | **fails** |
| 12.6.0 | **fails** |
| 12.8.1 | **fails** |

First bad release: **12.4.2**.

## Notes

- The shape matters: moving `local-ui`'s `styled-components` and `antd` directly into
  the apps makes it stable, and so does turning `local-ui` into a `workspace:` package.
  A `file:` directory or tarball dependency reproduces.
- Reduced mechanically from a large private monorepo, where `dedupe` on 12.x also needs
  several runs to settle.

---

# `pnpm11-lockfile/`: `pnpm dedupe` on a lockfile written by pnpm 11

A second fixture, still reproducing with the fix from pnpm/pnpm#16359. It is not
idempotent on **every** pnpm 12 release, starting from a lockfile that pnpm 11 wrote.

Settings: `injectWorkspacePackages: true`, `linkWorkspacePackages: deep`. Six workspace
packages; the only registry packages are `winston` and `is-number`.

```
a   deps: f (workspace:^)
b   peer: winston@^3
c   peer: f@^1.0.0
d   peer: b@^1.0.0
e   deps: is-number@^7          peer: c@^1.0.0
f   deps: b (workspace:^)       peer: is-number@>=7
```

`pnpm-lock.pnpm11.yaml` is the output of `pnpm@11.28.2 install --lockfile-only`: every
workspace dependency is recorded as `link:`.

```sh
cd pnpm11-lockfile
cp pnpm-lock.pnpm11.yaml pnpm-lock.yaml
corepack pnpm@12.8.1 dedupe --lockfile-only
corepack pnpm@12.8.1 dedupe --check          # ERR_PNPM_DEDUPE_CHECK_ISSUES (packages/e -> @repro/c)
corepack pnpm@12.8.1 dedupe --lockfile-only  # changes the lockfile again
corepack pnpm@12.8.1 dedupe --check          # clean
```

The first `dedupe` rewrites three importer entries to peer-suffixed `file:` copies
(`a → f`, `c → f`, `e → c`). On 12.4.2 and later the second turns `e → c` back into
`link:`; on earlier 12.x the count stays at 3 but the lockfile still changes.

`file:` importer entries:

| pnpm | fresh `install` | pnpm-11 lockfile → `dedupe` | `--check` | → `dedupe` again |
|---|---|---|---|---|
| 11.28.2 | 0 | 0 | clean | 0 |
| 12.0.0 | 3 | 3 | **fails** | 3 (lockfile changed) |
| 12.4.1 | 3 | 3 | **fails** | 3 (lockfile changed) |
| 12.4.2 | 3 | 3 | **fails** | 2 |
| 12.6.0 | 3 | 3 | **fails** | 2 |
| 12.8.1 | 3 | 3 | **fails** | 2 |

The `fresh install` column is pnpm/pnpm#16354: pnpm 11 injects nothing here.

Reduced mechanically from the same private monorepo, using a CI build of #16359.
