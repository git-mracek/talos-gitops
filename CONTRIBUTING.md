# 🤝 Contribution Guidelines

To keep the repository history clean and readable, we follow the **Conventional Commits** specification.

## 📝 Commit Message Format

Each commit message consists of a **type**, an optional **scope**, and a **subject**:

```text
<type>(<scope>): <subject>
```

### Types:
- **feat**: A new feature or application (e.g., `feat(n8n): add persistence`)
- **fix**: A bug fix (e.g., `fix(ingress): correct host header`)
- **chore**: Maintenance tasks, version updates (e.g., `chore(deps): update talos to v1.9`)
- **refactor**: Code change that neither fixes a bug nor adds a feature
- **docs**: Documentation changes only
- **style**: Changes that do not affect the meaning of the code (white-space, formatting)
- **perf**: A code change that improves performance

### Rules:
1. **Always Sign Your Commits:** Unsigned commits will be rejected by GitHub.
2. **Use Imperative Mood:** "add feature" instead of "added feature".
3. **No Period at the End:** Keep the subject line concise.
4. **Squash on Merge:** All feature branch commits will be squashed into a single clean commit on `main`.

---
*Note: This repository is managed via GitOps. Every merge to `main` triggers an automatic sync in ArgoCD.*
