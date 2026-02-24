Review all changes on this branch compared to main before creating a PR.

1. Run `git diff main...HEAD` and `git diff` to see all committed and uncommitted changes
2. For each changed file, check for:
   - Bugs or logic errors
   - Missing error handling or edge cases
   - Leftover console.logs or debug code
   - TypeScript issues (`any` types, missing types, type assertions)
   - Security concerns (unsanitized input, exposed secrets, SQL injection)
   - API response shape violations
   - Business logic in controllers (should be in services)
3. Flag issues by severity: **critical** / **warning** / **nit**
4. If clean, confirm the changes are ready for PR
