---
description: Upgrade OpenShift operator to target OCP version
argument-hint: <ocp-version> [go-version] [k8s-version] [--repo-path PATH]
---

## Name
operator-upgrade:upgrade-versions

## Synopsis
```
/operator-upgrade:upgrade-versions <ocp-version> [go-version] [k8s-version] [--repo-path PATH]
```

## Description
The `operator-upgrade:upgrade-versions` command is a **generic operator upgrade orchestrator** that works with ANY OpenShift operator that follows the upgrade automation convention.

**How it works:**
1. Looks for `hack/upgrade-automation/scripts/upgrade.sh` in the operator repository
2. Calls that script with the target OCP version (and optionally Go and K8s versions)
3. The operator's script discovers compatible versions and handles all operator-specific logic

**This is a universal wrapper** - it works with any operator as long as the operator provides its own upgrade scripts.

## Arguments
- `$1` (required): Target OCP version (e.g., `4.20`, `4.21`)
- `$2` (optional): Target Go version in format `X.Y` (e.g., `1.24`) - auto-discovered if not specified
- `$3` (optional): Target Kubernetes version in format `X.Y.Z` (e.g., `1.34.1`) - auto-discovered if not specified
- `$4` (optional): Repository path (default: current working directory)
  - Format: `--repo-path /path/to/repo`
  - If not specified, assumes you're already in the operator repository

## Convention

For this command to work, an operator repository must provide:

**Required:**
```
<operator-repo>/
└── docs/
    └── upgrade-automation/
        └── scripts/
            └── upgrade.sh    # Executable script that accepts <ocp-version> [go-version] [k8s-version]
```

**The operator's `upgrade.sh` script is responsible for:**
- Validating it's in the correct repository
- Discovering compatible Go and K8s versions from the target OCP version
- Updating files (Dockerfiles, Makefiles, go.mod, etc.)
- Running tests
- Creating commits
- Providing post-upgrade guidance

**Recommended structure** (but not required):
```
<operator-repo>/
└── docs/
    └── upgrade-automation/
        ├── README.md                    # Documentation
        ├── version-matrix.yaml          # Version discovery config (optional)
        ├── file-update-patterns.yaml    # File patterns config (optional)
        ├── upgrade-workflow.yaml        # Workflow config (optional)
        └── scripts/
            ├── upgrade.sh               # Main entry point (REQUIRED)
            ├── 01-update-base-images.sh # Step implementations (optional)
            ├── 02-update-go-mod.sh
            ├── 03-update-vendor.sh
            ├── 04-update-tools.sh
            ├── 05-run-code-generation.sh
            ├── 06-run-tests.sh
            └── lib/                     # Helper functions (optional)
                ├── version-discovery.sh
                ├── file-updates.sh
                └── validations.sh
```

## Implementation

```bash
# Parse arguments
OCP_VERSION="$1"
GO_VERSION="${2:-}"
K8S_VERSION="${3:-}"
REPO_PATH="${4:-.}"  # Default to current directory

# Handle --repo-path flag
if [[ "$2" == "--repo-path" ]]; then
  REPO_PATH="$3"
  GO_VERSION=""
  K8S_VERSION=""
elif [[ "$3" == "--repo-path" ]]; then
  REPO_PATH="$4"
  K8S_VERSION=""
elif [[ "$4" == "--repo-path" ]]; then
  REPO_PATH="$5"
fi

# Validate arguments
if [[ -z "$OCP_VERSION" ]]; then
  echo "Usage: /operator-upgrade:upgrade-versions <ocp-version> [go-version] [k8s-version] [--repo-path PATH]"
  exit 1
fi

# Validate repository exists
if [[ ! -d "$REPO_PATH" ]]; then
  echo "❌ Error: Repository not found at $REPO_PATH"
  exit 1
fi

# Look for the upgrade script
UPGRADE_SCRIPT="$REPO_PATH/hack/upgrade-automation/scripts/upgrade.sh"
if [[ ! -f "$UPGRADE_SCRIPT" ]]; then
  echo "❌ Error: Upgrade script not found"
  echo "   Expected: $UPGRADE_SCRIPT"
  echo "   Current directory: $(pwd)"
  echo ""
  echo "This operator doesn't have upgrade automation scripts yet."
  echo ""

  # Try to detect operator name for helpful error message
  OPERATOR_NAME="unknown"
  if [[ -f "$REPO_PATH/go.mod" ]]; then
    OPERATOR_NAME=$(grep '^module' "$REPO_PATH/go.mod" | awk '{print $2}' | sed 's/.*\///')
  fi

  echo "Operator: $OPERATOR_NAME"
  echo ""
  echo "To add upgrade automation to this operator:"
  echo "  1. Create the directory:"
  echo "     mkdir -p hack/upgrade-automation/scripts"
  echo ""
  echo "  2. Add an upgrade script:"
  echo "     # See: https://github.com/openshift/multiarch-tuning-operator/tree/main/hack/upgrade-automation/scripts"
  echo "     # for a reference implementation"
  echo ""
  echo "  3. Make it executable:"
  echo "     chmod +x hack/upgrade-automation/scripts/upgrade.sh"
  echo ""
  echo "See plugin README for more details:"
  echo "https://github.com/openshift-eng/ai-helpers/tree/main/plugins/operator-upgrade"
  exit 1
fi

# Validate script is executable
if [[ ! -x "$UPGRADE_SCRIPT" ]]; then
  echo "❌ Error: Upgrade script is not executable"
  echo "   Run: chmod +x $UPGRADE_SCRIPT"
  exit 1
fi

# Detect operator name for display
OPERATOR_NAME="unknown"
if [[ -f "$REPO_PATH/go.mod" ]]; then
  OPERATOR_NAME=$(grep '^module' "$REPO_PATH/go.mod" | awk '{print $2}' | sed 's/.*\///')
fi

# Change to operator repository
cd "$REPO_PATH" || exit 1

# Call the operator's upgrade script
echo "=========================================="
echo "Operator Upgrade: $OPERATOR_NAME"
echo "=========================================="
echo ""
echo "Repository: $(pwd)"
echo "Script: $UPGRADE_SCRIPT"
echo "OCP version: $OCP_VERSION"
if [[ -n "$GO_VERSION" ]]; then
  echo "Go version: $GO_VERSION (override)"
fi
if [[ -n "$K8S_VERSION" ]]; then
  echo "K8s version: $K8S_VERSION (override)"
fi
echo ""

# Execute the operator's upgrade script with all provided arguments
if [[ -n "$K8S_VERSION" ]]; then
  exec "$UPGRADE_SCRIPT" "$OCP_VERSION" "$GO_VERSION" "$K8S_VERSION"
elif [[ -n "$GO_VERSION" ]]; then
  exec "$UPGRADE_SCRIPT" "$OCP_VERSION" "$GO_VERSION"
else
  exec "$UPGRADE_SCRIPT" "$OCP_VERSION"
fi
```

That's it! ~50 lines of pure delegation.

## Supported Operators

Any operator that provides `hack/upgrade-automation/scripts/upgrade.sh` will work.

**Currently known to support this convention:**
- [`multiarch-tuning-operator`](https://github.com/openshift/multiarch-tuning-operator)

**To add your operator:**
1. Create `hack/upgrade-automation/scripts/upgrade.sh` in your operator repo
2. Make it executable: `chmod +x hack/upgrade-automation/scripts/upgrade.sh`
3. Implement your operator-specific upgrade logic
4. Document it (optional but recommended)

## Examples

**Upgrade to OCP 4.21 (auto-discover Go and K8s):**
```bash
cd ~/code/multiarch-tuning-operator
/operator-upgrade:upgrade-versions 4.21
```

**Upgrade to OCP 4.20 with specific Go version:**
```bash
cd ~/code/multiarch-tuning-operator
/operator-upgrade:upgrade-versions 4.20 1.24
```

**Upgrade with all versions specified:**
```bash
cd ~/code/cluster-version-operator
/operator-upgrade:upgrade-versions 4.21 1.24 1.34.1
```

**Upgrade any operator with --repo-path:**
```bash
/operator-upgrade:upgrade-versions 4.21 --repo-path ~/code/my-operator
```

**Run operator's script directly (no plugin):**
```bash
cd ~/code/my-operator
./hack/upgrade-automation/scripts/upgrade.sh 4.21
```

## Return Value

**Success (exit 0):**
- Operator's upgrade script completed successfully
- Delegated entirely to operator's script

**Failure (exit 1):**
- Repository not found
- Upgrade script not found in repository
- Upgrade script not executable
- Any error from the operator's upgrade script (delegated)

**Output:**
All output comes from the operator's `upgrade.sh` script.

## For Operator Developers

### Adding Upgrade Automation to Your Operator

To make your operator work with this generic command:

**Minimum requirement:**
```bash
# In your operator repository root
mkdir -p hack/upgrade-automation/scripts
cat > hack/upgrade-automation/scripts/upgrade.sh << 'EOF'
#!/bin/bash
set -euo pipefail

OCP_VERSION="$1"
GO_VERSION="${2:-}"
K8S_VERSION="${3:-}"

# Your operator-specific upgrade logic here
echo "Upgrading to OCP $OCP_VERSION"

# If Go/K8s not provided, discover them from OCP version
if [[ -z "$K8S_VERSION" ]]; then
  # Discover K8s from openshift/api release-$OCP_VERSION
  K8S_VERSION="..." # Your discovery logic
fi

if [[ -z "$GO_VERSION" ]]; then
  # Discover Go from openshift/api release-$OCP_VERSION
  GO_VERSION="..." # Your discovery logic
fi

# Validate repository
# Update files
# Run tests
# Create commits
# etc.

EOF

chmod +x hack/upgrade-automation/scripts/upgrade.sh
```

**Reference implementation:**

See [`multiarch-tuning-operator`](https://github.com/openshift/multiarch-tuning-operator/tree/main/hack/upgrade-automation) for a complete example with:
- Version discovery functions
- File update utilities
- Validation helpers
- Step-by-step scripts
- Configuration files

**Your script should:**
1. Validate it's in the correct repository (check go.mod module name, etc.)
2. Accept one required argument `<ocp-version>` and two optional arguments `[go-version] [k8s-version]`
3. Auto-discover Go and K8s versions from the OCP version when not provided
4. Exit with code 0 on success, non-zero on failure
5. Provide clear output and error messages
6. Handle all operator-specific logic (file updates, tests, commits, etc.)

## Benefits of This Approach

✅ **Truly generic** - Works with any operator
✅ **Zero coupling** - Plugin has no operator-specific code
✅ **Operator ownership** - Each operator fully controls its upgrade process
✅ **Scales infinitely** - New operators "just work" if they follow convention
✅ **Portable** - Operators can be upgraded with or without this plugin
✅ **Testable** - Run operator scripts directly for testing
✅ **Maintainable** - Update operator repos, never touch the plugin

## Notes

- **Convention over configuration**: Operators must provide `hack/upgrade-automation/scripts/upgrade.sh`
- **No operator registry**: Plugin doesn't know or care which operators exist
- **Pure delegation**: Plugin is just a convenient wrapper around operator scripts
- **Minimal plugin code**: ~40 lines total, no operator-specific logic

## See Also

- [multiarch-tuning-operator upgrade scripts](https://github.com/openshift/multiarch-tuning-operator/tree/main/hack/upgrade-automation/scripts) - Reference implementation
- [operator-upgrade plugin README](../../README.md)
- [Adding upgrade automation to your operator](#for-operator-developers)
