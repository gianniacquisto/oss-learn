# :material-flag: Done Checklist — Milestone 1

[:material-arrow-left: Back to Steps](steps.md){: .md-button }&nbsp;&nbsp;[:octicons-arrow-right-24: Next: Messages & Agent State →](../02-messages-and-agent/README.md){: .md-button .md-button--primary }

---

If everything works, you should be able to do the following:

## Verification Steps

1. Run `npm run build` and see **no TypeScript errors** (warnings are fine for now)
2. Run `npm start` and see `Hello agent` printed to your terminal
3. Delete or rename `.agentrc.json` (if you created one) and verify the app still runs using environment variable fallback
4. Set a fake API key via `AGENT_API_KEY=fake npm start` — it should run without errors

---

## What Should Work

??? success "Expected behavior"
    - `npm install` completes without errors
    - `npm run build` compiles all TypeScript files to `dist/` with no type errors
    - `npm start` prints `Hello agent` and exits cleanly (exit code 0)
    - Missing API key produces a clear error message: `Error: No API key found. Set AGENT_API_KEY or add it to .agentrc.json`

## What Should NOT Work (Yet)

??? failure "Not expected yet"
    - Talking to an LLM — that comes in Milestone 4
    - Parsing CLI commands beyond the default "run" behavior
    - Loading config from multiple files or merging overrides
    - Hot-reloading config changes

---

## Troubleshooting

??? bug "'npm run build' fails with module resolution errors"
    - Make sure `"type": "module"` is set in package.json
    - Check that `moduleResolution` is `"NodeNext"` in tsconfig.json
    - Verify all your imports use `.js` extensions (TypeScript requires this for ESM)

??? bug "'npm start' crashes with 'Cannot find module'"
    - Did you run `npm run build` first? The CLI runs from `dist/`, not `src/`
    - Check that all files in `src/` compiled successfully — look at the `dist/` directory to see what was generated

??? bug "Config loading throws an error about missing API key"
    - This is expected if you haven't set `AGENT_API_KEY` or created `.agentrc.json`. Set one of them and try again.
    - For now, the actual key value doesn't need to be valid — just present. You'll add format validation later.

??? bug "TypeScript complains about 'fs' not existing"
    - Make sure you have `@types/node` installed: `npm install -D @types/node`
    - The error might say "Cannot find module 'fs' or its corresponding type declarations"
