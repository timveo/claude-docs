Create a pull request for the current branch against main.

1. Run `git log main..HEAD --oneline` and `git diff main...HEAD --stat` to understand all changes
2. Push the branch to origin if not already pushed
3. Create the PR using `gh pr create` with:
   - A concise title under 70 characters
   - A body with a `## Summary` section (1-3 bullet points) and a `## Test plan` section
4. Return the PR URL
