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
