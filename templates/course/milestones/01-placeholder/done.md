# Done Checklist — Milestone N: <Title>

If everything works, you should be able to do the following:

## Verification Steps

<!-- Concrete, testable checks. Each one should have a clear pass/fail outcome. -->

1. <!-- e.g., Run `npm run build` and see no errors -->
2. <!-- e.g., Run `./myapp --help` and see the expected output -->
3. <!-- e.g., Test with `node test/fixtures/valid.json` and verify it prints `{ ... }` -->

## What Should Work

<!-- Describe the expected behavior in plain language -->
- <!-- e.g., The CLI should accept a config file path as an argument -->
- <!-- e.g., Invalid configs should produce a clear error message (not a crash) -->
- <!-- e.g., Running without arguments should show usage help -->

## What Should NOT Work (Yet)

<!-- Set expectations — these are intentionally out of scope for this milestone -->
- <!-- e.g., Environment variable overrides (coming in Milestone X) -->
- <!-- e.g., Hot-reloading config changes (not needed for MVP) -->
- <!-- e.g., Multiple config file formats (JSON only for now) -->

## If Something Is Broken

<!-- Common issues and how to troubleshoot them. Be specific. -->

### "It crashes with X error"
- Check that you've created all files in the right locations
- Verify your imports match the module structure described in context.md

### "The output doesn't match"
- Compare against the reference pattern — are you returning the same shape?
- Check if you're handling edge cases (empty input, missing fields, etc.)
