# Version Control Guide for Claude Instances

## Quick Reference

**When starting a new feature:**
```bash
git checkout -b feature/descriptive-name
```

**When feature is working:**
```bash
git checkout main
git merge feature/descriptive-name
git tag -a v0.X.0 -m "Feature: [description] working"
git push origin main --tags
```

**Emergency rollback to last working version:**
```bash
git tag  # List all tags
git checkout v0.X.0  # Go to specific working version
```

---

## Tagging Guidelines

### When to Tag
- ✅ After completing a feature successfully
- ✅ Before starting risky/experimental work
- ✅ When user says "this works well now"
- ✅ End of productive coding session

### Tag Format
- **Stable features:** `v0.X.0` (semantic versioning)
- **Quick rollback points:** `working-feature-name`
- **Before experiments:** `stable-YYYY-MM-DD`

### Example Tag Messages
```bash
git tag -a v0.3.0 -m "Combat system working with all animations"
git tag -a working-inventory -m "Inventory system stable before UI redesign"
git tag -a stable-2025-11-09 -m "All systems working before multiplayer experiment"
```

---

## Branch Strategy

### Main Branch
- Always keep main branch in working state
- Only merge completed features
- Tag after each successful merge

### Feature Branches
```bash
# Examples:
git checkout -b feature/player-movement
git checkout -b feature/enemy-ai
git checkout -b fix/collision-bug
git checkout -b experiment/new-renderer
```

### Branch Naming Convention
- `feature/` - New functionality
- `fix/` - Bug fixes
- `experiment/` - Risky/exploratory work
- `refactor/` - Code cleanup

---

## Rollback Scenarios

### "Go back to when X feature worked"
```bash
git log --oneline --decorate  # Find the tag
git checkout working-X
```

### "Undo everything since last tag"
```bash
git reset --hard $(git describe --tags --abbrev=0)
```

### "Create a new branch from old working version"
```bash
git checkout -b recovery-branch v0.2.0
```

---

## Claude Instance Responsibilities

1. **Before starting work:** Check current branch with `git branch`
2. **When creating new feature:** Always create feature branch
3. **When feature works:** Create pull request OR merge to main + tag
4. **Before risky changes:** Tag current working state
5. **Document tags:** Update CHANGELOG.md or version notes in README

---

## Project-Specific Notes

[Add project-specific versioning decisions here]
- Current version:
- Next planned version:
- Known stable tags:
