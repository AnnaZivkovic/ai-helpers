# operator-upgrade Plugin

Generic automation for upgrading Go and Kubernetes versions in any OpenShift operator.

## Overview

The `operator-upgrade` plugin is a **universal wrapper** that works with ANY OpenShift operator that provides upgrade automation scripts.

**How it works:**
1. You call `/operator-upgrade:upgrade-versions <go> <k8s>`
2. Plugin looks for `hack/upgrade-automation/scripts/upgrade.sh` in the operator repo
3. Plugin calls that script - the operator handles everything else

**Architecture:**
- **Pure delegation**: Plugin has ZERO operator-specific code
- **Convention-based**: Operators provide scripts at a standard location
- **Operator ownership**: Each operator fully controls its upgrade process
- **Scales infinitely**: New operators "just work" if they follow the convention
- **Discoverable**: Use `/operator-upgrade:list-supported` to find compatible operators

**Currently supported operators:**
- `multiarch-tuning-operator` ✅

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

## Commands

### /operator-upgrade:upgrade-versions

**Generic command** that works with any operator.

```bash
cd ~/code/my-operator
/operator-upgrade:upgrade-versions 1.23 1.32.3
```

Or:
```bash
/operator-upgrade:upgrade-versions 1.23 1.32.3 --repo-path ~/code/my-operator
```

**See:** [Full documentation](commands/upgrade-versions.md)

### /operator-upgrade:list-supported

**Discovery command** that finds operators with upgrade automation.

```bash
/operator-upgrade:list-supported
```

Searches your `~/openshift_working` directory (or custom path) and lists all operators that support automated upgrades.

**See:** [Full documentation](commands/list-supported.md)


## Supported Operators

Any operator can be supported by adding upgrade automation scripts. Currently known to work:

### multiarch-tuning-operator ✅

**Repository:** https://github.com/openshift/multiarch-tuning-operator

**Usage:**
```bash
cd ~/code/multiarch-tuning-operator
/operator-upgrade:upgrade-versions 1.23 1.32.3
```

**Features:**
- Dynamic version discovery (no hardcoded versions)
- Validates K8s↔OCP alignment
- Updates all dependencies
- Runs code generation and tests
- Creates structured commits

**See:** [upgrade automation scripts](https://github.com/openshift/multiarch-tuning-operator/tree/main/hack/upgrade-automation)

## Structure

The plugin provides generic commands that work with any operator:

```
plugins/operator-upgrade/
├── commands/
│   ├── upgrade-versions.md       # Generic upgrade command
│   └── list-supported.md         # Discovery command
└── README.md
```

The plugin is intentionally minimal - all operator-specific logic lives in each operator's repository.

## Adding New Operators

**Good news: You don't need to modify this plugin at all!**

To add upgrade automation to your operator:

1. **In your operator repository**, create the upgrade script:
   ```bash
   mkdir -p hack/upgrade-automation/scripts
   ```

2. **Add your upgrade logic:**
   ```bash
   # Create the main upgrade script
   cat > hack/upgrade-automation/scripts/upgrade.sh << 'EOF'
   #!/bin/bash
   set -euo pipefail

   GO_VERSION="$1"
   K8S_VERSION="$2"

   # Your operator-specific upgrade logic here
   # - Validate repository
   # - Update files
   # - Run tests
   # - Create commits
   EOF

   chmod +x hack/upgrade-automation/scripts/upgrade.sh
   ```

3. **Test it:**
   ```bash
   cd /path/to/your-operator
   ./hack/upgrade-automation/scripts/upgrade.sh 1.23 1.32.3
   ```

4. **Use the generic command:**
   ```bash
   cd /path/to/your-operator
   /operator-upgrade:upgrade-versions 1.23 1.32.3
   ```

**That's it!** The plugin automatically works with your operator.

**Reference implementation:**
- See [multiarch-tuning-operator/hack/upgrade-automation](https://github.com/openshift/multiarch-tuning-operator/tree/main/hack/upgrade-automation) for a complete example

## Why Convention-Based?

**The generic command delegates everything to each operator**, allowing operators to handle their own unique requirements:

| Aspect | Example (multiarch-tuning-operator) | How It Works |
|--------|-------------------------------------|--------------|
| Base images | Must use specific UBI images | Operator's script handles this |
| K8s version | Must align with OCP version | Operator validates alignment |
| Special validation | getCorrectHostmountAnyUIDSCC function | Operator-specific logic in script |
| Konflux | .tekton directory checks | Operator handles if needed |
| Commit structure | Specific 6-commit pattern | Operator creates commits |
| Documentation | Updates docs/ocp-release.md | Operator updates its own docs |

**The plugin doesn't need to know any of this** - it just calls the operator's script. This is much cleaner than a complex generic tool with operator-specific flags and conditionals.

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

- [Generic upgrade command documentation](commands/upgrade-versions.md)
- [Discovery command documentation](commands/list-supported.md)
- [Plugin Development Guide](../../CLAUDE.md)
- [multiarch-tuning-operator upgrade scripts](https://github.com/openshift/multiarch-tuning-operator/tree/main/hack/upgrade-automation) - Reference implementation
- [OpenShift Operator Conventions](https://docs.openshift.com/container-platform/latest/operators/index.html)

## License

This plugin is part of the ai-helpers repository.
