---
description: List operators that support automated upgrades
argument-hint: [--search-path PATH]
---

## Name
operator-upgrade:list-supported

## Synopsis
```
/operator-upgrade:list-supported [--search-path PATH]
```

## Description
The `operator-upgrade:list-supported` command searches for OpenShift operators that have upgrade automation scripts and can be upgraded using `/operator-upgrade:upgrade-versions`.

This helps with **discoverability** - you can find which operators support automated upgrades.

## Arguments
- `$1` (optional): Search path (default: `~/openshift_working`)
  - Format: `--search-path /path/to/search`
  - Searches recursively for operator repositories

## Implementation

```bash
# Parse arguments
SEARCH_PATH="${1:-$HOME/openshift_working}"

# Handle --search-path flag
if [[ "$1" == "--search-path" ]]; then
  SEARCH_PATH="$2"
fi

# Validate search path exists
if [[ ! -d "$SEARCH_PATH" ]]; then
  echo "❌ Error: Search path not found: $SEARCH_PATH"
  exit 1
fi

echo "=========================================="
echo "Searching for Operators with Upgrade Automation"
echo "=========================================="
echo ""
echo "Search path: $SEARCH_PATH"
echo ""
echo "Looking for: hack/upgrade-automation/scripts/upgrade.sh"
echo ""

# Find all upgrade scripts
FOUND_COUNT=0
SUPPORTED_OPERATORS=()

# Search for upgrade scripts
while IFS= read -r -d '' script; do
  # Get the repository root (3 levels up from script)
  REPO_DIR=$(dirname "$(dirname "$(dirname "$(dirname "$script")")")")

  # Skip if not a git repository
  if [[ ! -d "$REPO_DIR/.git" ]]; then
    continue
  fi

  # Extract operator name from go.mod
  OPERATOR_NAME="unknown"
  MODULE_PATH="unknown"
  if [[ -f "$REPO_DIR/go.mod" ]]; then
    MODULE_PATH=$(grep '^module' "$REPO_DIR/go.mod" | awk '{print $2}')
    OPERATOR_NAME=$(echo "$MODULE_PATH" | sed 's/.*\///')
  fi

  # Check if script is executable
  EXECUTABLE="✅"
  if [[ ! -x "$script" ]]; then
    EXECUTABLE="⚠️  (not executable)"
  fi

  # Store info
  SUPPORTED_OPERATORS+=("$OPERATOR_NAME|$REPO_DIR|$EXECUTABLE")
  FOUND_COUNT=$((FOUND_COUNT + 1))

done < <(find "$SEARCH_PATH" -type f -path "*/hack/upgrade-automation/scripts/upgrade.sh" -print0 2>/dev/null)

# Display results
if [[ $FOUND_COUNT -eq 0 ]]; then
  echo "No operators with upgrade automation found in $SEARCH_PATH"
  echo ""
  echo "To add upgrade automation to an operator:"
  echo "  https://github.com/openshift-eng/ai-helpers/tree/main/plugins/operator-upgrade#for-operator-developers"
else
  echo "Found $FOUND_COUNT operator(s) with upgrade automation:"
  echo ""
  printf "%-40s %-60s %s\n" "OPERATOR" "PATH" "STATUS"
  printf "%-40s %-60s %s\n" "--------" "----" "------"

  for entry in "${SUPPORTED_OPERATORS[@]}"; do
    IFS='|' read -r name path status <<< "$entry"
    printf "%-40s %-60s %s\n" "$name" "$path" "$status"
  done

  echo ""
  echo "Usage:"
  echo "  cd <path-to-operator>"
  echo "  /operator-upgrade:upgrade-versions <go-version> <k8s-version>"
  echo ""
  echo "Or:"
  echo "  /operator-upgrade:upgrade-versions <go-version> <k8s-version> --repo-path <path-to-operator>"
fi

exit 0
```

## Return Value

**Success (exit 0):**
- Lists all operators found with upgrade scripts
- Shows operator name, path, and executable status

**Output format:**
```
==========================================
Searching for Operators with Upgrade Automation
==========================================

Search path: /home/user/openshift_working

Looking for: hack/upgrade-automation/scripts/upgrade.sh

Found 2 operator(s) with upgrade automation:

OPERATOR                                 PATH                                                         STATUS
--------                                 ----                                                         ------
multiarch-tuning-operator                /home/user/openshift_working/multiarch-tuning-operator       ✅
cluster-version-operator                 /home/user/openshift_working/cluster-version-operator        ✅

Usage:
  cd <path-to-operator>
  /operator-upgrade:upgrade-versions <go-version> <k8s-version>

Or:
  /operator-upgrade:upgrade-versions <go-version> <k8s-version> --repo-path <path-to-operator>
```

## Examples

**Search default location:**
```bash
/operator-upgrade:list-supported
```

**Search custom location:**
```bash
/operator-upgrade:list-supported --search-path ~/code
```

**Search entire home directory:**
```bash
/operator-upgrade:list-supported --search-path ~
```

## Notes

- **Discovers operators dynamically** - No hardcoded list
- **Validates executable status** - Warns if script isn't executable
- **Shows paths** - Easy to cd to the operator
- **Fast** - Only searches for specific file pattern

## See Also

- [/operator-upgrade:upgrade-versions](upgrade-versions.md) - Upgrade an operator
- [operator-upgrade plugin README](../../README.md)
