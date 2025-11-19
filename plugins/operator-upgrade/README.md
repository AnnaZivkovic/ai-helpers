# operator-upgrade Plugin

Operator-specific Go and Kubernetes version upgrade automation for OpenShift operators following official release procedures.

## Overview

The `operator-upgrade` plugin provides operator-specific commands for automating Go and Kubernetes version upgrades in OpenShift operator repositories. Each operator has its own tailored upgrade workflow that follows the operator's official release documentation and conventions.

**Philosophy:**
- **Operator-specific, not generic**: Each operator has unique requirements, build processes, and validation steps
- **Follows official procedures**: Implements the exact steps from each operator's release documentation
- **Extensible**: New operators can be added with their own specific workflows
- **Currently supported**: multiarch-tuning-operator

## Installation

### From Claude Code Marketplace

```bash
# Add the marketplace
/plugin marketplace add openshift-eng/ai-helpers

# Install the plugin
/plugin install operator-upgrade@ai-helpers
```

### Manual Installation (Cursor)

```bash
mkdir -p ~/.cursor/commands
git clone git@github.com:openshift-eng/ai-helpers.git
ln -s ai-helpers ~/.cursor/commands/ai-helpers
```

## Supported Operators

### multiarch-tuning-operator

Automates the complete 9-step upgrade process from `docs/ocp-release.md`.

**Command:**
```bash
/operator-upgrade:multiarch-tuning-operator:upgrade-versions <go-version> <k8s-version> [--repo-path PATH]
```

**Example:**
```bash
# Upgrade to Go 1.23 and Kubernetes 1.32.3
/operator-upgrade:multiarch-tuning-operator:upgrade-versions 1.23 1.32.3
```

**What it does:**
1. ✅ Updates Go version in base images (Dockerfile, Makefile BUILD_IMAGE)
2. ✅ Checks .tekton directory for Konflux configuration
3. ✅ Validates `getCorrectHostmountAnyUIDSCC` function for K8s compatibility
4. ✅ Updates go.mod with proper verification (download, tidy, verify)
5. ✅ Refreshes vendor directory
6. ✅ Updates tool versions (kustomize, controller-tools, envtest, golangci-lint)
7. ✅ Runs code generation (make generate, make manifests, make bundle)
8. ✅ Runs full test suite (make docker-build, make build, make test)
9. ✅ Creates structured commits following official pattern
10. ✅ Warns about Prow config update requirements
11. ✅ Updates documentation (docs/ocp-release.md)

**Version validation:**
- Enforces OpenShift/Kubernetes alignment (e.g., OCP 4.19 → K8s 1.32.x)
- Requires Universal Base Images (UBI) in Dockerfiles
- Prevents version mismatches and downgrades

**See:** [Full documentation](commands/multiarch-tuning-operator/upgrade-versions.md)

## Structure

The plugin is organized by operator:

```
plugins/operator-upgrade/
├── commands/
│   ├── multiarch-tuning-operator/
│   │   └── upgrade-versions.md        # MTO-specific upgrade workflow
│   └── <future-operator>/
│       └── upgrade-versions.md        # Future operator workflows
└── README.md
```

Each operator directory contains commands specific to that operator's upgrade procedures.

## Adding New Operators

To add a new operator to this plugin:

1. **Create operator directory:**
   ```bash
   mkdir -p plugins/operator-upgrade/commands/<operator-name>
   ```

2. **Create upgrade command:**
   ```bash
   touch plugins/operator-upgrade/commands/<operator-name>/upgrade-versions.md
   ```

3. **Implement operator-specific workflow:**
   - Follow the operator's official release documentation
   - Include operator-specific validations
   - Use operator-specific make targets and tools
   - Enforce operator-specific version requirements

4. **Document in this README:**
   - Add operator to "Supported Operators" section
   - Include command syntax and examples
   - List what the command does

5. **Test and validate:**
   ```bash
   make lint
   ```

### Example: Adding hypothetical "cluster-ingress-operator"

```markdown
### cluster-ingress-operator

Automates version upgrades following cluster-ingress-operator conventions.

**Command:**
```bash
/operator-upgrade:cluster-ingress-operator:upgrade-versions <go-version> <k8s-version>
```

**What it does:**
1. Updates go.mod and dependencies
2. Updates operator-specific Dockerfiles
3. Runs operator-specific tests
4. etc.
```

## Why Operator-Specific?

Different operators have different requirements:

| Aspect | multiarch-tuning-operator | Generic Approach |
|--------|---------------------------|------------------|
| Base images | Must use UBI | May use any golang image |
| K8s version | Must align with OCP version | No alignment required |
| Special validation | getCorrectHostmountAnyUIDSCC | None |
| Konflux | .tekton directory checks | Not applicable |
| Commit structure | Specific 6-commit pattern | Generic commits |
| Documentation | Updates docs/ocp-release.md | No doc updates |

A generic upgrade tool can't handle these operator-specific requirements without becoming overly complex with flags and conditionals. Operator-specific commands are cleaner, more maintainable, and more accurate.

## Common Patterns

While each operator has unique requirements, commands typically:

1. **Validate versions** - Check compatibility and alignment
2. **Update base images** - Dockerfiles, Makefiles, CI configs
3. **Update dependencies** - go.mod, go.sum, vendor
4. **Update tools** - Operator-specific tooling versions
5. **Generate code** - CRDs, manifests, deepcopy, etc.
6. **Run tests** - Operator-specific test suites
7. **Create commits** - Following operator conventions
8. **Document changes** - Update operator documentation

## OpenShift Version Alignment

Many OpenShift operators must align their Kubernetes dependencies with OpenShift versions:

| OpenShift Version | Kubernetes Version | Typical Go Version |
|-------------------|-------------------|-------------------|
| 4.16 | 1.29.x | 1.22+ |
| 4.17 | 1.30.x | 1.22+ |
| 4.18 | 1.31.x | 1.22+ |
| 4.19 | 1.32.x | 1.23+ |
| 4.20 | 1.33.x | 1.23+ |

Operator-specific commands enforce these alignments when applicable.

## Prerequisites

### General Requirements
- Go toolchain installed
- Git repository
- Make (for build targets)

### Operator-Specific Requirements
Each operator may have additional requirements. Check the operator's command documentation for details.

## Troubleshooting

### "Command not found"

Ensure the plugin is installed:
```bash
/plugin list
```

If not listed, reinstall:
```bash
/plugin install operator-upgrade@ai-helpers
```

### "Version validation failed"

The command detected incompatible versions. Read the error message for details on:
- Required version alignment (e.g., K8s → OpenShift)
- Version ordering (e.g., can't downgrade)
- Operator-specific requirements

### "Test failures after upgrade"

Review the test output for:
- Deprecated API usage
- Breaking changes in dependencies
- Operator-specific validation failures

The command will provide suggestions for common issues.

## Best Practices

1. **Use the correct operator command** - Don't try to use a command meant for a different operator
2. **Review changes before committing** - Always check the diffs
3. **Run locally first** - Test the upgrade before creating a PR
4. **Follow operator conventions** - Trust the operator-specific workflow
5. **Read the warnings** - The commands provide important context about manual steps

## Contributing

### Reporting Issues

Found a bug or have a suggestion? Open an issue at:
https://github.com/openshift-eng/ai-helpers/issues

### Adding Operators

Want to add support for your operator? See "Adding New Operators" above, then:

1. Fork the repository
2. Create operator-specific command following the structure
3. Test thoroughly with your operator
4. Submit a pull request

Please include:
- Link to operator's release documentation
- Description of operator-specific requirements
- Test results showing successful upgrades

## See Also

- [Plugin Development Guide](../../CLAUDE.md)
- [multiarch-tuning-operator upgrade documentation](commands/multiarch-tuning-operator/upgrade-versions.md)
- [OpenShift Operator Conventions](https://docs.openshift.com/container-platform/latest/operators/index.html)

## License

This plugin is part of the ai-helpers repository.
