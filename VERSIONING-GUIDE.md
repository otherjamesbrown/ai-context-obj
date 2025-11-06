# Documentation Versioning Guide

This guide defines how to version the constitution, PRD, and specs in this project.

---

## 🎯 Philosophy

**Living Documents + Git History = Version Control**

- Documents evolve as you build (they're not frozen artifacts)
- Git provides complete change history
- Version numbers communicate significance of changes
- CHANGELOGs provide human-readable summaries

---

## 📖 Constitution & PRD (Foundational Documents)

### Versioning Format

Use **Semantic Versioning** in the document header:

```markdown
# Inference-as-a-Service Constitutional Principles

**Version**: 1.1
**Last Updated**: 2025-11-06
```

### Version Bumping Rules

- **MAJOR (1.0 → 2.0)**: Fundamental architectural changes
  - Example: Switch from microservices to monolith
  - Example: Change from PostgreSQL to MongoDB
  
- **MINOR (1.0 → 1.1)**: New principles or significant additions
  - Example: Add GitOps management section
  - Example: Add new architectural mandate
  
- **PATCH (1.1 → 1.1.1)**: Clarifications, typos, small refinements
  - Example: Fix typo in command example
  - Example: Clarify ambiguous wording

### CHANGELOG Format

Keep a `/memory/CHANGELOG.md` for high-level version history:

Update summaries (detailed changes) go in `/memory/updates/YYYY-MM-DD-vX.Y-description.md`

```markdown
# Constitution Changelog

## [1.1] - 2025-11-06

### Added
- GitOps Management section
- CLI-Only Features section

### Changed
- Updated scale targets to 1,000 orgs
- RBAC model clarified as V1 fixed roles

### Deprecated
- (none)

### Removed
- (none)
```

### What NOT to Do

❌ Don't create `constitution-v1.md`, `constitution-v2.md`  
❌ Don't create dated copies like `constitution-2025-11-06.md`  
✅ Git history shows all previous versions

---

## 📋 Specs (Implementation Artifacts)

### Versioning Format

Add status and version metadata at the top of each `spec.md`:

```markdown
# Spec 005: User & Organization Service

**Status**: In Progress
**Version**: 1.1
**Last Updated**: 2025-11-06

## Purpose
...
```

### Status Values

- **Draft**: Spec is being written, not ready for implementation
- **In Progress**: Spec is complete, implementation underway
- **Complete**: Implementation finished and deployed
- **Deprecated**: Spec is obsolete (replaced or removed)

### Version Bumping Rules

- **MAJOR (1.0 → 2.0)**: Breaking changes to contracts
  - Example: Change API endpoint structure
  - Example: Incompatible database schema change
  
- **MINOR (1.0 → 1.1)**: New features or endpoints
  - Example: Add new API endpoints
  - Example: Add new database tables
  
- **PATCH (1.1 → 1.1.1)**: Bug fixes, clarifications
  - Example: Fix typo in SQL schema
  - Example: Update example code

### CHANGELOG Per Spec

Each spec gets its own CHANGELOG: `/specs/NNN-feature-name/CHANGELOG.md`

```markdown
# Spec 005: User & Organization Service - Changelog

## [1.1] - 2025-11-06

### Added
- GitOps Management API endpoints
- Password reset endpoint

### Changed
- Updated service architecture diagram

## [1.0] - 2025-10-15
- Initial release
```

### What NOT to Do

❌ Don't create `specs/005-v1-user-service/` and `specs/005-v2-user-service/`  
❌ Don't rename spec directories  
✅ Keep the same directory, update the contents, track in CHANGELOG

---

## 🔄 When to Bump Versions

### Constitution / PRD

**Immediate Bump**: Any change that affects how developers build the system

- New architectural principles
- New technology mandates
- Clarifications that change interpretation
- Scale target updates

**No Bump**: Typo fixes, formatting, dead link fixes

### Specs

**Immediate Bump**: Any change to contracts (API specs, schemas, CLI commands)

- New API endpoints
- New database columns
- New CLI commands
- Changed behavior

**No Bump**: Documentation updates, example improvements, typo fixes

---

## 🏷️ Git Tagging Strategy

### Tag Constitutional Milestones

```bash
git tag -a constitution-v1.1 -m "Add GitOps management and CLI password reset"
git push origin constitution-v1.1
```

### Tag Implementation Milestones (Optional)

```bash
git tag -a spec-005-v1.1 -m "User service with GitOps support"
git push origin spec-005-v1.1
```

### Tag Releases

```bash
git tag -a v0.1.0 -m "MVP: Core services deployed"
git push origin v0.1.0
```

---

## 📊 Version Coordination

### Keep Versions Aligned

When updating the constitution:

1. **Bump constitution version** (e.g., 1.0 → 1.1)
2. **Update CHANGELOG** in `/memory/CHANGELOG.md`
3. **Create update summary** in `/memory/updates/YYYY-MM-DD-vX.Y-description.md`
4. **Review all specs** for needed updates
5. **Bump affected spec versions** (e.g., 005 v1.0 → v1.1)
6. **Update spec CHANGELOGs**
7. **Git commit with clear message**: `chore: bump constitution to v1.1 - add GitOps support`
8. **Git tag**: `git tag -a constitution-v1.1 -m "..."`

### Example Commit Message

```
chore: bump constitution to v1.1 and update specs

Constitution v1.1:
- Add GitOps management (hybrid mode)
- Add CLI password reset
- Update scale targets to 1K orgs

Updated specs:
- 003-database-schemas v1.1 (GitOps tables)
- 005-user-org-service v1.1 (GitOps reconciler)
- 009-admin-cli v1.1 (new commands)
```

---

## 🔍 Finding Version History

### View Document Version History

```bash
# See all changes to constitution
git log -p memory/constitution.md

# See constitution at specific version
git show constitution-v1.0:memory/constitution.md

# Compare versions
git diff constitution-v1.0 constitution-v1.1 -- memory/constitution.md
```

### View Spec Version History

```bash
# See all changes to a spec
git log -p specs/005-user-org-service/spec.md

# View at specific tag
git show spec-005-v1.0:specs/005-user-org-service/spec.md
```

---

## ✅ Checklist: Updating a Document

### When Updating Constitution

- [ ] Update version number in header
- [ ] Update "Last Updated" date
- [ ] Add entry to `/memory/CHANGELOG.md`
- [ ] Create detailed summary in `/memory/updates/YYYY-MM-DD-vX.Y-description.md`
- [ ] Review specs for needed updates
- [ ] Commit with descriptive message
- [ ] Create git tag: `constitution-vX.Y`

### When Updating a Spec

- [ ] Update version number in header
- [ ] Update "Last Updated" date
- [ ] Update "Status" if changed
- [ ] Add entry to spec's `CHANGELOG.md`
- [ ] Update contracts if needed
- [ ] Commit with descriptive message
- [ ] Create git tag (optional): `spec-NNN-vX.Y`

---

## 🎓 Examples from This Project

### Constitution v1.0 → v1.1 (2025-11-06)

**Changes**:
- Added GitOps Management section
- Added CLI-Only Features section
- Updated scale targets
- Clarified RBAC forward compatibility

**Affected Specs**: 003, 005, 009

**Commit**: `chore: bump constitution to v1.1 - add GitOps support`

**Tag**: `constitution-v1.1`

---

## 📚 Additional Resources

- **Semantic Versioning**: https://semver.org/
- **Keep a Changelog**: https://keepachangelog.com/
- **Spec-Kit Methodology**: https://github.com/github/spec-kit

---

## ❓ FAQ

**Q: Do I create a new file for each version?**  
A: No. Edit the existing file and let Git track changes.

**Q: When should I create a CHANGELOG entry?**  
A: For any version bump (MAJOR, MINOR). Not needed for typos.

**Q: Can I skip PATCH versions?**  
A: Yes. Go straight from 1.1 to 1.2 for the next minor change.

**Q: Should specs and constitution have the same version?**  
A: No. They version independently. Constitution v1.1 might update spec 005 to v1.1, but spec 003 stays at v1.0 if unchanged.

**Q: How do I know what version a spec was at a given time?**  
A: Use Git history: `git log --follow specs/005-user-org-service/spec.md`

---

**This guide is itself a living document. Update it as versioning practices evolve!**

