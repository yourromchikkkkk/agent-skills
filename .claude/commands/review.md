Run `git remote -v` to detect the platform, then invoke the matching review skill:

- If any remote URL contains `github.com` → invoke the `github-pr-review` skill, passing any PR number from the user's message.
- If any remote URL contains `gitlab` → invoke the `gitlab-mr-review` skill, passing any MR IID from the user's message.
- If the remote is missing or ambiguous → ask the user which platform using AskUserQuestion ("GitHub" / "GitLab"), then invoke the matching skill.
