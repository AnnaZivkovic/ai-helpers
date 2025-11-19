---
description: Upgrade Go and Kubernetes versions following official OCP release process
argument-hint: <go-version> <k8s-version> [--repo-path PATH]
---

## Name
operator-upgrade:multiarch-tuning-operator-upgrade-versions

## Synopsis
```
/operator-upgrade:multiarch-tuning-operator-upgrade-versions <go-version> <k8s-version> [--repo-path PATH]
```

## Description
The `operator-upgrade:multiarch-tuning-operator-upgrade-versions` command automates the complete Go and Kubernetes version upgrade process for the multiarch-tuning-operator repository, following the official 9-step process documented in `docs/ocp-release.md`.

This command implements ALL steps from the official release process including:
1. Base image updates (Dockerfile, Makefile BUILD_IMAGE, .tekton checks)
2. Special validation for `getCorrectHostmountAnyUIDSCC` function
3. go.mod updates with proper verification (download, tidy, verify)
4. Vendor folder refresh
5. Tool version updates (kustomize, controller-tools, setup-envtest, golangci-lint)
6. Code generation (make generate, make manifests, make bundle)
7. Full test suite (make docker-build, make build, make test)
8. Structured commits following example log pattern
9. Prow config update detection and warnings

**IMPORTANT**: This command enforces version compatibility:
- Validates that K8s version aligns with OpenShift version from BUILD_IMAGE
- Verifies Go version matches k8s.io/api requirements
- Ensures Universal Base Images (UBI) are used in Dockerfiles
- Prevents accidental version downgrades or over-upgrades

## Implementation

### Prerequisites Check

1. **Verify repository location**
   - Default: `~/openshift_working/multiarch-tuning-operator`
   - Can override with `--repo-path` flag
   - Verify git repository is clean (no uncommitted changes)

2. **Extract current versions**
   ```bash
   cd ~/openshift_working/multiarch-tuning-operator

   # Current Go version
   grep '^go ' go.mod

   # Current K8s version
   grep 'k8s.io/api' go.mod | head -1

   # Current OpenShift version from BUILD_IMAGE
   grep 'BUILD_IMAGE' Makefile | grep -oP 'openshift-\K[0-9]+\.[0-9]+'
   ```

3. **Validate target versions**
   - Extract OCP version from BUILD_IMAGE in Makefile
   - Map K8s version to required OCP version:
     - K8s 1.29.x → OCP 4.16
     - K8s 1.30.x → OCP 4.17
     - K8s 1.31.x → OCP 4.18
     - K8s 1.32.x → OCP 4.19
     - K8s 1.33.x → OCP 4.20
     - K8s 1.34.x → OCP 4.21
   - **FAIL if target K8s version doesn't match OCP version**
   - **WARN if target Go version doesn't match intended OCP Go version**

4. **Validate base images exist**
   - Before making any changes, verify all target images exist and are accessible
   - Check the following images:

   a. **Dockerfile golang image**:
   ```bash
   # Check Docker Hub golang image exists
   docker manifest inspect golang:<go-version> > /dev/null 2>&1
   ```
   - If fails: Try with patch version (e.g., `1.23.0`, `1.23.1`, etc.)
   - If still fails: **FAIL** with error listing attempted versions

   b. **BUILD_IMAGE from Makefile**:
   ```bash
   # Extract target BUILD_IMAGE
   TARGET_IMAGE="registry.ci.openshift.org/ocp/builder:rhel-9-golang-<go-version>-openshift-<ocp-version>"

   # Check if image exists (may require VPN/registry access)
   skopeo inspect docker://${TARGET_IMAGE} > /dev/null 2>&1
   ```
   - If fails: Try alternative tag format `rhel-9-golang-<go-version>-builder-multi-openshift-<ocp-version>`
   - If still fails: **WARN** (may require VPN) and ask user to verify manually

   c. **Brew registry images (.tekton, konflux)**:
   ```bash
   # Check brew registry image
   skopeo inspect docker://brew.registry.redhat.io/rh-osbs/openshift-golang-builder:rhel_9_<go-version> > /dev/null 2>&1
   ```
   - If fails: **FAIL** with error - these images must exist before proceeding
   - Note: Requires Red Hat VPN and registry authentication

   d. **CI operator image (.ci-operator.yaml)**:
   ```bash
   # Check CI operator image
   TARGET_TAG="rhel-9-golang-<go-version>-openshift-<ocp-version>"
   skopeo inspect docker://registry.ci.openshift.org/ocp/builder:${TARGET_TAG} > /dev/null 2>&1
   ```
   - If fails: Try alternative format with `-builder-multi-`
   - If still fails: **WARN** and list attempted image tags

5. **Image validation retry logic**
   - If image validation fails for version `X.Y`, automatically try:
     1. `X.Y.0`
     2. `X.Y.1`
     3. `X.Y.2`
     4. Latest patch version from upstream Go releases
   - For each attempted version, re-check all images
   - Present user with:
     - ✅ Images that were found
     - ❌ Images that failed
     - Suggested version to use based on what's available
   - Ask user: "Continue with version X.Y.Z? [Y/n]"

### Step 1: Update Go version in base images

**Commit message**: `Update Makefile and Dockerfiles to use the new Golang version base image to <go-version>`

**IMPORTANT**: Before making changes, validate Go/OCP version compatibility:
- Determine target OCP version from K8s version (see Prerequisites Check step 3)
- Verify Go version is compatible with target OCP version
- Both Go version AND OCP version will be updated in this step

1. **Update .ci-operator.yaml**
   - Regex pattern: `(tag:\s+rhel-9-golang-)(\d+\.\d+)((?:-builder-multi)?-openshift-)(\d+\.\d+)`
   - Capture groups:
     - Group 1: `rhel-9-golang-` (prefix)
     - Group 2: Go version (e.g., `1.22`)
     - Group 3: Middle separator (e.g., `-builder-multi-openshift-` or `-openshift-`)
     - Group 4: OCP version (e.g., `4.17`)
   - Replace with: `$1<go-version>$3<ocp-version>`
   - Example: `rhel-9-golang-1.22-builder-multi-openshift-4.17` → `rhel-9-golang-1.23-openshift-4.19`
   - **Validate**: Ensure OCP version aligns with K8s version (4.19 → K8s 1.32.x)

2. **Update .tekton YAML files**
   - Search for files in `.tekton/` directory:
     - `.tekton/*-integration-tests.yaml`
     - `.tekton/*-pull-request.yaml`
     - `.tekton/*-push.yaml`
   - Regex pattern: `(brew\.registry\.redhat\.io/rh-osbs/openshift-golang-builder:rhel_9_)(\d+\.\d+)`
   - Capture groups:
     - Group 1: Full image path prefix
     - Group 2: Go version (e.g., `1.22`)
   - Replace with: `$1<go-version>`
   - Example: `rhel_9_1.22` → `rhel_9_1.23`
   - **Update all matching files** in the commit
   - Note: .tekton files only have Go version, not OCP version

3. **Update Dockerfile**
   - Regex pattern: `(FROM\s+golang:)(\d+\.\d+)`
   - Capture groups:
     - Group 1: `FROM golang:` (prefix)
     - Group 2: Go version (e.g., `1.22`)
   - Replace with: `$1<go-version>`
   - Example: `FROM golang:1.22` → `FROM golang:1.23`
   - Note: This uses Docker Hub golang images, not UBI (repository-specific pattern)

4. **Update Makefile BUILD_IMAGE**
   - Regex pattern: `(BUILD_IMAGE\s*\?=\s*registry\.ci\.openshift\.org/ocp/builder:rhel-9-golang-)(\d+\.\d+)((?:-builder-multi)?-openshift-)(\d+\.\d+)`
   - Capture groups:
     - Group 1: Full path prefix ending in `golang-`
     - Group 2: Go version (e.g., `1.22`)
     - Group 3: Middle separator (`-builder-multi-openshift-` or `-openshift-`)
     - Group 4: OCP version (e.g., `4.17`)
   - Replace with: `$1<go-version>$3<ocp-version>`
   - Example: `rhel-9-golang-1.22-builder-multi-openshift-4.17` → `rhel-9-golang-1.23-openshift-4.19`
   - **Validate**: OCP version must match the one from .ci-operator.yaml

5. **Update bundle.konflux.Dockerfile**
   - Regex pattern: `(FROM\s+brew\.registry\.redhat\.io/rh-osbs/openshift-golang-builder:rhel_9_)(\d+\.\d+)`
   - Capture groups:
     - Group 1: Full image path prefix
     - Group 2: Go version (e.g., `1.22`)
   - Replace with: `$1<go-version>`
   - Example: `rhel_9_1.22` → `rhel_9_1.23`

6. **Update konflux.Dockerfile**
   - Regex pattern: `(FROM\s+brew\.registry\.redhat\.io/rh-osbs/openshift-golang-builder:rhel_9_)(\d+\.\d+)`
   - Capture groups:
     - Group 1: Full image path prefix
     - Group 2: Go version (e.g., `1.22`)
   - Replace with: `$1<go-version>`
   - Example: `rhel_9_1.22` → `rhel_9_1.23`

7. **Validate getCorrectHostmountAnyUIDSCC function**
   - Search for `getCorrectHostmountAnyUIDSCC` in codebase
   - Read the function implementation
   - Verify Kubernetes version logic matches target K8s version
   - Check that OCP version mapping is correct
   - **WARN** if function needs updates for version alignment
   - Explain: "This function relies on K8s version to determine SCC for podplacementconfig pod's hostPath mounts"

8. **Commit changes**
   ```bash
   git add .ci-operator.yaml .tekton/ Dockerfile Makefile bundle.konflux.Dockerfile konflux.Dockerfile
   git commit -m "Update Makefile and Dockerfiles to use the new Golang version base image to <go-version>"
   ```

### Step 2: Update go.mod

**Commit message**: `pin K8S API to v<k8s-version> and set go minimum version to <go-version>`

1. **Update go directive in go.mod**
   - Edit go.mod to set: `go <go-version>`
   - Example: Change `go 1.22` to `go 1.23`

2. **Extract ALL direct dependencies from go.mod**
   ```bash
   # Extract all direct dependencies (not indirect/commented ones)
   ALL_DEPS=$(grep -E '^\s+[a-zA-Z0-9]' go.mod | grep -v '^\s*//' | awk '{print $1}' | sort -u)

   # Separate into categories for targeted updates
   K8S_DEPS=$(echo "$ALL_DEPS" | grep '^k8s\.io/')
   SIGS_DEPS=$(echo "$ALL_DEPS" | grep '^sigs\.k8s\.io/')
   OPENSHIFT_DEPS=$(echo "$ALL_DEPS" | grep '^github\.com/openshift/')
   OTHER_DEPS=$(echo "$ALL_DEPS" | grep -v '^k8s\.io/' | grep -v '^sigs\.k8s\.io/' | grep -v '^github\.com/openshift/')
   ```
   - This captures ALL dependencies in go.mod
   - Categories:
     - **K8s dependencies**: Need specific version pinning to K8s version
     - **Sigs dependencies**: Need alignment with K8s/controller-runtime versions
     - **OpenShift dependencies**: Need alignment with OCP version
     - **Other dependencies**: Will be updated to latest compatible versions

3. **Update all k8s.io dependencies to target version**
   ```bash
   # Update all k8s.io dependencies
   for dep in $(grep '^\s*k8s\.io/' go.mod | awk '{print $1}' | sort -u); do
     echo "Updating ${dep} to v<k8s-version>..."
     go get ${dep}@v<k8s-version> || echo "⚠️  ${dep}@v<k8s-version> not available, will use compatible version"
   done
   ```
   - **IMPORTANT**: Not all k8s.io packages may have a release at the exact version
   - If a specific package doesn't exist at v<k8s-version>, `go get` will use the nearest compatible version
   - This is expected behavior for packages like `k8s.io/klog/v2` and `k8s.io/utils`

4. **Update controller-runtime and related sigs.k8s.io dependencies**
   ```bash
   # Determine compatible controller-runtime version for K8s version
   # K8s 1.29.x → controller-runtime v0.17.x
   # K8s 1.30.x → controller-runtime v0.18.x
   # K8s 1.31.x → controller-runtime v0.19.x
   # K8s 1.32.x → controller-runtime v0.20.x
   # K8s 1.33.x → controller-runtime v0.21.x
   # K8s 1.34.x → controller-runtime v0.22.x

   # Update controller-runtime to compatible version
   go get sigs.k8s.io/controller-runtime@<controller-runtime-version>

   # Let go.mod automatically resolve other sigs.k8s.io dependencies
   ```
   - **NOTE**: controller-runtime versions must align with K8s versions
   - Reference: https://github.com/kubernetes-sigs/controller-runtime/blob/main/README.md#compatibility

5. **Update OpenShift API dependencies (if present)**
   ```bash
   # Check if OpenShift dependencies exist
   if grep -q 'github.com/openshift/api' go.mod; then
     # Update OpenShift API to compatible version with OCP version
     # OCP 4.16 → release-4.16
     # OCP 4.17 → release-4.17
     # OCP 4.18 → release-4.18
     # OCP 4.19 → release-4.19
     # OCP 4.21 → release-4.21

     for dep in $(grep 'github.com/openshift' go.mod | awk '{print $1}' | sort -u); do
       echo "Updating ${dep} to release-<ocp-version>..."
       go get ${dep}@release-<ocp-version> || echo "⚠️  ${dep}@release-<ocp-version> not available"
     done
   fi
   ```

6. **Update ALL other dependencies to latest compatible versions**
   ```bash
   # Extract all direct dependencies from the first require block (excluding k8s, sigs, and openshift which are handled separately)
   DEPS=$(awk '/^require \\($/,/^\\)$/ {print}' go.mod | \
          grep -E '^\\s+[a-zA-Z0-9]' | \
          grep -v '// indirect' | \
          awk '{print $1}' | \
          grep -v '^k8s\\.io/' | \
          grep -v '^sigs\\.k8s\\.io/controller-runtime' | \
          grep -v '^github\\.com/openshift/' | \
          sort -u)

   TOTAL=$(echo "$DEPS" | wc -l)
   echo "Found $TOTAL direct dependencies to update"
   echo ""

   COUNTER=0
   while IFS= read -r dep; do
       COUNTER=$((COUNTER + 1))
       echo "[$COUNTER/$TOTAL] Updating $dep..."

       # Update to latest compatible version
       if go get -u "$dep" 2>&1 | grep -v 'no required module provides'; then
           echo "  ✅ Updated successfully"
       else
           echo "  ⚠️  Could not update, keeping current version"
       fi
       echo ""
   done <<< "$DEPS"
   ```
   - **CRITICAL**: This updates EVERY direct dependency, not just k8s ones
   - Uses `-u` to upgrade to latest compatible minor/patch versions (important for Go version upgrades)
   - This ensures all dependencies are compatible with the new Go version
   - Includes dependencies like:
     - `github.com/BurntSushi/toml`
     - `github.com/cilium/ebpf`
     - `github.com/containers/image/v5`
     - `github.com/onsi/ginkgo/v2`
     - `github.com/onsi/gomega`
     - `golang.org/x/*` packages
     - `google.golang.org/grpc`
     - All other direct dependencies
   - **NOTE**: `go mod tidy` afterward will resolve any indirect dependency conflicts

7. **Run go mod tidy to resolve all transitive dependencies**
   ```bash
   go mod tidy
   go mod verify
   ```
   - **Document output**: Capture and log any warnings or errors
   - **FAIL** if `go mod verify` fails
   - This ensures all indirect dependencies are properly resolved
   - **NOTE**: May encounter dependency conflicts (e.g., opentelemetry packages)
   - If conflicts occur, downgrade conflicting packages to compatible versions:
   ```bash
   # Example: If opentelemetry v1.38 conflicts, downgrade to v1.35
   go get go.opentelemetry.io/otel/sdk@v1.35.0 go.opentelemetry.io/otel@v1.35.0
   go mod tidy
   go mod verify
   ```

8. **Verify version consistency**
   ```bash
   # Check that all k8s.io dependencies are at the same minor version
   echo "Checking k8s.io dependency versions:"
   grep 'k8s.io/' go.mod | grep -v '//' | awk '{print $1, $2}'

   # Count how many different k8s.io minor versions exist
   VERSIONS=$(grep '^\s*k8s\.io/' go.mod | grep -v '//' | awk '{print $2}' | grep -oP 'v\d+\.\d+' | sort -u | wc -l)

   if [ "$VERSIONS" -gt 1 ]; then
     echo "⚠️  WARNING: Multiple k8s.io minor versions detected!"
     echo "Expected: v<k8s-major>.<k8s-minor>"
     echo "Found:"
     grep '^\s*k8s\.io/' go.mod | grep -v '//' | awk '{print $1, $2}' | grep -oP 'v\d+\.\d+' | sort -u
   fi
   ```
   - **WARN** if any k8s.io library is at different minor version
   - **Explain**: Some packages (like k8s.io/klog, k8s.io/utils) may have different versioning schemes
   - **Action**: Review and ensure compatibility

9. **Document dependency changes**
   ```bash
   # Show summary of what changed
   echo "Dependency version summary:"
   echo "- Go: <old-version> → <go-version>"
   echo "- Kubernetes APIs: $(grep 'k8s.io/api ' go.mod | awk '{print $2}')"
   echo "- controller-runtime: $(grep 'sigs.k8s.io/controller-runtime ' go.mod | awk '{print $2}')"
   if grep -q 'github.com/openshift/api' go.mod; then
     echo "- OpenShift API: $(grep 'github.com/openshift/api ' go.mod | awk '{print $2}')"
   fi
   ```

10. **Preserve go.mod structure and formatting**
   ```bash
   # CRITICAL: Consolidate indirect dependency blocks
   # Go 1.24+ may create multiple indirect blocks - consolidate to 2 blocks total

   # Run go mod edit -fmt to ensure proper formatting
   go mod edit -fmt

   # Count require blocks
   REQUIRE_BLOCKS=$(grep -c '^require (' go.mod)
   echo "Found $REQUIRE_BLOCKS require blocks in go.mod"

   # If we have more than 2 blocks, consolidate them using Python
   if [ "$REQUIRE_BLOCKS" -gt 2 ]; then
     echo "Consolidating indirect dependency blocks..."

     python3 << 'PYEOF'
import re

# Read go.mod
with open('go.mod', 'r') as f:
    content = f.read()

# Split into sections
sections = re.split(r'\n(require \()', content)

# Find all require blocks
require_blocks = []
i = 0
while i < len(sections):
    if i > 0 and sections[i] == 'require (':
        block_content = sections[i+1] if i+1 < len(sections) else ''
        end_idx = block_content.find('\n)')
        if end_idx != -1:
            deps = block_content[:end_idx]
            require_blocks.append(deps)
    i += 1

# Separate direct and indirect dependencies
direct_deps = []
indirect_deps = []

for block in require_blocks:
    for line in block.strip().split('\n'):
        line = line.strip()
        if not line:
            continue
        if '// indirect' in line:
            if line not in indirect_deps:
                indirect_deps.append(line)
        else:
            if line not in direct_deps:
                direct_deps.append(line)

# Sort dependencies
direct_deps.sort()
indirect_deps.sort()

# Reconstruct go.mod
lines = content.split('\n')
output = []
in_require = False
require_count = 0

for line in lines:
    if line.startswith('require ('):
        require_count += 1
        if require_count == 1:
            output.append('require (')
            for dep in direct_deps:
                output.append(f'\t{dep}')
            output.append(')')
            output.append('')
            output.append('require (')
            for dep in indirect_deps:
                output.append(f'\t{dep}')
            output.append(')')
            in_require = True
        continue
    elif line.strip() == ')' and in_require:
        in_require = False
        continue
    elif in_require:
        continue
    elif line.startswith('module ') or line.startswith('go '):
        output.append(line)
    elif require_count == 0:
        output.append(line)

# Write back
with open('go.mod', 'w') as f:
    f.write('\n'.join(output))

print("✅ Consolidated go.mod to 2 require blocks")
PYEOF

     go mod edit -fmt
     REQUIRE_BLOCKS=$(grep -c '^require (' go.mod)
     echo "Final count: $REQUIRE_BLOCKS require blocks"
   fi

   # Verify with go mod tidy
   go mod tidy
   go mod verify

   echo "✅ go.mod structure validated"
   echo "- Block 1: Direct dependencies"
   echo "- Block 2: Indirect dependencies"
   ```
   - **IMPORTANT**: The go.mod file has a specific two-block structure:
     - First `require ()` block: Direct dependencies (no `// indirect` comments)
     - Second `require ()` block: Indirect dependencies (with `// indirect` comments)
   - `go mod tidy` automatically maintains this structure
   - `go mod edit -fmt` ensures consistent formatting
   - The grouping and ordering within blocks may change, but the two-block structure remains

11. **Commit changes**
   ```bash
   git add go.mod go.sum
   git commit -m "pin K8S API to v<k8s-version> and set go minimum version to <go-version>"
   ```

### Step 3: Update vendor folder

**Commit message**: `go mod vendor`

1. **Clean and regenerate vendor**
   ```bash
   rm -rf vendor/
   go mod vendor
   ```

2. **Commit changes**
   ```bash
   git add vendor/
   git commit -m "go mod vendor"
   ```

### Step 4: Update tools in Makefile

**Commit message**: `Update tools in Makefile`

**IMPORTANT FOR AI AGENTS**: The bash scripts below are written in multi-line format for readability. When executing these scripts via the Bash tool, you MUST convert them to single-line commands using semicolons (;) to separate statements, as the Bash tool cannot handle multi-line scripts with newlines.

1. **Determine compatible tool versions automatically**

   a. **KUSTOMIZE_VERSION** - Get latest compatible with both K8s and Go versions:
   ```bash
   # Fetch all recent kustomize v5.x releases
   KUSTOMIZE_RELEASES=$(curl -s https://api.github.com/repos/kubernetes-sigs/kustomize/releases | grep '"tag_name"' | grep 'kustomize/v5' | sed -E 's/.*"kustomize\/v([^"]+)".*/\1/')

   # Find the latest version compatible with Go <go-version>
   # Check each release's go.mod to find Go requirement
   KUSTOMIZE_VERSION=""
   GO_MINOR=$(echo "<go-version>" | cut -d. -f2)

   for version in $KUSTOMIZE_RELEASES; do
     # Fetch go.mod for this version
     GO_REQ=$(curl -s "https://raw.githubusercontent.com/kubernetes-sigs/kustomize/kustomize/v${version}/kustomize/go.mod" | grep '^go ' | awk '{print $2}' | cut -d. -f2)

     if [ -n "$GO_REQ" ] && [ "$GO_REQ" -le "$GO_MINOR" ]; then
       KUSTOMIZE_VERSION="$version"
       echo "Found kustomize v${version} (requires Go 1.${GO_REQ}) - compatible with Go <go-version>"
       break
     fi
   done

   # Fallback to current version if no compatible version found
   if [ -z "$KUSTOMIZE_VERSION" ]; then
     echo "⚠️  Could not find compatible kustomize version, using current version from Makefile"
     KUSTOMIZE_VERSION=$(grep 'KUSTOMIZE_VERSION' Makefile | grep -oP 'v\K[0-9]+\.[0-9]+\.[0-9]+')
   fi

   # Validate compatibility with K8s version
   # Kustomize v5.x supports K8s 1.27+ (including 1.34+)
   # Kustomize v4.x supports K8s 1.25-1.29
   # Reference: https://github.com/kubernetes-sigs/kustomize/blob/master/README.md
   KUSTOMIZE_MAJOR=$(echo "$KUSTOMIZE_VERSION" | cut -d. -f1)
   if [ "$KUSTOMIZE_MAJOR" -lt 5 ]; then
     echo "⚠️  WARNING: Kustomize v${KUSTOMIZE_VERSION} may not support K8s <k8s-version>"
     echo "   Recommend Kustomize v5.0.0+ for K8s 1.27+"
     echo "   Proceeding with selected version"
   fi

   echo "Kustomize version: v${KUSTOMIZE_VERSION}"
   echo "  ✓ Compatible with K8s <k8s-version> and Go <go-version>"
   ```

   **Single-line version for Bash tool execution:**
   ```bash
   KUSTOMIZE_RELEASES=`curl -s https://api.github.com/repos/kubernetes-sigs/kustomize/releases | grep '"tag_name"' | grep 'kustomize/v5' | sed -E 's/.*"kustomize\/v([^"]+)".*/\1/'` && KUSTOMIZE_VERSION="" && GO_MINOR=24 && for version in $KUSTOMIZE_RELEASES; do GO_REQ=`curl -s "https://raw.githubusercontent.com/kubernetes-sigs/kustomize/kustomize/v${version}/kustomize/go.mod" | grep '^go ' | awk '{print $2}' | cut -d. -f2`; if [ -n "$GO_REQ" ] && [ "$GO_REQ" -le "$GO_MINOR" ]; then KUSTOMIZE_VERSION="$version"; echo "Found kustomize v${version} (requires Go 1.${GO_REQ})"; break; fi; done && if [ -z "$KUSTOMIZE_VERSION" ]; then KUSTOMIZE_VERSION=`grep 'KUSTOMIZE_VERSION' Makefile | grep -oP 'v\K[0-9]+\.[0-9]+\.[0-9]+'`; fi && echo "KUSTOMIZE_VERSION=v${KUSTOMIZE_VERSION}"
   ```

   b. **CONTROLLER_TOOLS_VERSION** - Get latest compatible with K8s and Go versions:
   ```bash
   # Fetch all controller-tools releases
   CONTROLLER_TOOLS_RELEASES=$(curl -s https://api.github.com/repos/kubernetes-sigs/controller-tools/releases | grep '"tag_name"' | grep -E '"v0\.' | sed -E 's/.*"v([^"]+)".*/\1/')

   # Extract K8s minor version from go.mod (e.g., v0.34.2 → 34)
   K8S_MINOR=$(grep 'k8s.io/apimachinery' go.mod | grep -v '//' | awk '{print $2}' | grep -oP 'v0\.\K[0-9]+')
   GO_MINOR=$(echo "<go-version>" | cut -d. -f2)

   # Find compatible version by checking each release's go.mod
   # Reference: https://github.com/kubernetes-sigs/controller-tools?tab=readme-ov-file#compatibility
   CONTROLLER_TOOLS_VERSION=""

   for version in $CONTROLLER_TOOLS_RELEASES; do
     # Fetch go.mod for this version
     GOMOD=$(curl -s "https://raw.githubusercontent.com/kubernetes-sigs/controller-tools/v${version}/go.mod")

     # Extract k8s.io/apimachinery version (e.g., v0.34.0 → 34)
     CT_K8S_MINOR=$(echo "$GOMOD" | grep 'k8s.io/apimachinery' | awk '{print $2}' | grep -oP 'v0\.\K[0-9]+' | head -1)

     # Extract Go requirement (e.g., 1.24.0 → 24)
     CT_GO_MINOR=$(echo "$GOMOD" | grep '^go ' | awk '{print $2}' | cut -d. -f2)

     # Check if this version matches both K8s and Go requirements
     if [ -n "$CT_K8S_MINOR" ] && [ -n "$CT_GO_MINOR" ]; then
       if [ "$CT_K8S_MINOR" -eq "$K8S_MINOR" ] && [ "$CT_GO_MINOR" -le "$GO_MINOR" ]; then
         CONTROLLER_TOOLS_VERSION="$version"
         echo "Found controller-tools v${version} (k8s.io v0.${CT_K8S_MINOR}, Go 1.${CT_GO_MINOR})"
         echo "  Matches: k8s.io v0.${K8S_MINOR} and Go 1.${GO_MINOR}"
         break
       fi
     fi
   done

   # Fallback to current version if no compatible version found
   if [ -z "$CONTROLLER_TOOLS_VERSION" ]; then
     echo "⚠️  Could not find compatible controller-tools version, using current version from Makefile"
     CONTROLLER_TOOLS_VERSION=$(grep 'CONTROLLER_TOOLS_VERSION' Makefile | grep -oP 'v\K[0-9]+\.[0-9]+\.[0-9]+')
   fi

   echo "Controller-tools version: v${CONTROLLER_TOOLS_VERSION}"
   echo "  ✓ Compatible with K8s v0.${K8S_MINOR} and Go <go-version>"
   ```

   **Single-line version for Bash tool execution:**
   ```bash
   CONTROLLER_TOOLS_RELEASES=`curl -s https://api.github.com/repos/kubernetes-sigs/controller-tools/releases | grep '"tag_name"' | grep -E '"v0\.' | sed -E 's/.*"v([^"]+)".*/\1/'` && K8S_MINOR=`grep 'k8s.io/apimachinery' go.mod | grep -v '//' | awk '{print $2}' | grep -oP 'v0\.\K[0-9]+'` && GO_MINOR=24 && CONTROLLER_TOOLS_VERSION="" && for version in $CONTROLLER_TOOLS_RELEASES; do GOMOD=`curl -s "https://raw.githubusercontent.com/kubernetes-sigs/controller-tools/v${version}/go.mod"`; CT_K8S_MINOR=`echo "$GOMOD" | grep 'k8s.io/apimachinery' | awk '{print $2}' | grep -oP 'v0\.\K[0-9]+' | head -1`; CT_GO_MINOR=`echo "$GOMOD" | grep '^go ' | awk '{print $2}' | cut -d. -f2`; if [ -n "$CT_K8S_MINOR" ] && [ -n "$CT_GO_MINOR" ]; then if [ "$CT_K8S_MINOR" -eq "$K8S_MINOR" ] && [ "$CT_GO_MINOR" -le "$GO_MINOR" ]; then CONTROLLER_TOOLS_VERSION="$version"; echo "Found controller-tools v${version} (k8s.io v0.${CT_K8S_MINOR}, Go 1.${CT_GO_MINOR})"; break; fi; fi; done && if [ -z "$CONTROLLER_TOOLS_VERSION" ]; then CONTROLLER_TOOLS_VERSION=`grep 'CONTROLLER_TOOLS_VERSION' Makefile | grep -oP 'v\K[0-9]+\.[0-9]+\.[0-9]+'`; fi && echo "CONTROLLER_TOOLS_VERSION=v${CONTROLLER_TOOLS_VERSION}"
   ```

   c. **SETUP_ENVTEST_VERSION** - Must match controller-runtime minor version:
   ```bash
   # Fetch all controller-runtime releases
   CONTROLLER_RUNTIME_RELEASES=$(curl -s https://api.github.com/repos/kubernetes-sigs/controller-runtime/releases | grep '"tag_name"' | grep -E '"v0\.' | sed -E 's/.*"v([^"]+)".*/\1/')

   # Extract K8s minor version from go.mod (e.g., v0.34.2 → 34)
   K8S_MINOR=$(grep 'k8s.io/apimachinery' go.mod | grep -v '//' | awk '{print $2}' | grep -oP 'v0\.\K[0-9]+')
   GO_MINOR=$(echo "<go-version>" | cut -d. -f2)

   # Find compatible version by checking each release's go.mod
   # Reference: https://github.com/kubernetes-sigs/controller-runtime/blob/main/README.md#compatibility
   CONTROLLER_RUNTIME_VERSION=""

   for version in $CONTROLLER_RUNTIME_RELEASES; do
     # Fetch go.mod for this version
     GOMOD=$(curl -s "https://raw.githubusercontent.com/kubernetes-sigs/controller-runtime/v${version}/go.mod")

     # Extract k8s.io/apimachinery version (e.g., v0.34.0 → 34)
     CR_K8S_MINOR=$(echo "$GOMOD" | grep 'k8s.io/apimachinery' | awk '{print $2}' | grep -oP 'v0\.\K[0-9]+' | head -1)

     # Extract Go requirement (e.g., 1.24.0 → 24)
     CR_GO_MINOR=$(echo "$GOMOD" | grep '^go ' | awk '{print $2}' | cut -d. -f2)

     # Check if this version matches both K8s and Go requirements
     if [ -n "$CR_K8S_MINOR" ] && [ -n "$CR_GO_MINOR" ]; then
       if [ "$CR_K8S_MINOR" -eq "$K8S_MINOR" ] && [ "$CR_GO_MINOR" -le "$GO_MINOR" ]; then
         CONTROLLER_RUNTIME_VERSION="$version"
         echo "Found controller-runtime v${version} (k8s.io v0.${CR_K8S_MINOR}, Go 1.${CR_GO_MINOR})"
         echo "  Matches: k8s.io v0.${K8S_MINOR} and Go 1.${GO_MINOR}"
         break
       fi
     fi
   done

   # Fallback to current version if no compatible version found
   if [ -z "$CONTROLLER_RUNTIME_VERSION" ]; then
     echo "⚠️  Could not find compatible controller-runtime version, extracting from go.mod"
     CONTROLLER_RUNTIME_VERSION=$(grep 'sigs.k8s.io/controller-runtime' go.mod | grep -v '//' | awk '{print $2}' | grep -oP 'v0\.\K[0-9]+')
   fi

   # Setup envtest uses release-0.X format matching controller-runtime v0.X.Y
   # Extract minor version from version string (e.g., "0.22.4" → "22")
   CONTROLLER_RUNTIME_MINOR=$(echo "$CONTROLLER_RUNTIME_VERSION" | cut -d. -f2)
   SETUP_ENVTEST_VERSION="release-0.${CONTROLLER_RUNTIME_MINOR}"

   echo "Controller-runtime version: v0.${CONTROLLER_RUNTIME_MINOR}.x"
   echo "Setup-envtest version: ${SETUP_ENVTEST_VERSION}"

   # Verify this release exists
   ENVTEST_EXISTS=$(curl -s "https://api.github.com/repos/kubernetes-sigs/controller-runtime/git/refs/tags/${SETUP_ENVTEST_VERSION}" | grep '"ref"')
   if [ -z "$ENVTEST_EXISTS" ]; then
     echo "⚠️  WARNING: ${SETUP_ENVTEST_VERSION} tag not found in controller-runtime"
     echo "   This may cause setup-envtest installation to fail"
     echo "   Recommend checking: https://github.com/kubernetes-sigs/controller-runtime/tags"
   else
     echo "  ✓ Compatible with K8s v0.${K8S_MINOR} and Go <go-version>"
   fi
   ```

   **Single-line version for Bash tool execution:**
   ```bash
   CONTROLLER_RUNTIME_RELEASES=`curl -s https://api.github.com/repos/kubernetes-sigs/controller-runtime/releases | grep '"tag_name"' | grep -E '"v0\.' | sed -E 's/.*"v([^"]+)".*/\1/'` && K8S_MINOR=`grep 'k8s.io/apimachinery' go.mod | grep -v '//' | awk '{print $2}' | grep -oP 'v0\.\K[0-9]+'` && GO_MINOR=24 && CONTROLLER_RUNTIME_VERSION="" && for version in $CONTROLLER_RUNTIME_RELEASES; do GOMOD=`curl -s "https://raw.githubusercontent.com/kubernetes-sigs/controller-runtime/v${version}/go.mod"`; CR_K8S_MINOR=`echo "$GOMOD" | grep 'k8s.io/apimachinery' | awk '{print $2}' | grep -oP 'v0\.\K[0-9]+' | head -1`; CR_GO_MINOR=`echo "$GOMOD" | grep '^go ' | awk '{print $2}' | cut -d. -f2`; if [ -n "$CR_K8S_MINOR" ] && [ -n "$CR_GO_MINOR" ]; then if [ "$CR_K8S_MINOR" -eq "$K8S_MINOR" ] && [ "$CR_GO_MINOR" -le "$GO_MINOR" ]; then CONTROLLER_RUNTIME_VERSION="$version"; echo "Found controller-runtime v${version} (k8s.io v0.${CR_K8S_MINOR}, Go 1.${CR_GO_MINOR})"; break; fi; fi; done && if [ -z "$CONTROLLER_RUNTIME_VERSION" ]; then CONTROLLER_RUNTIME_VERSION=`grep 'sigs.k8s.io/controller-runtime' go.mod | grep -v '//' | awk '{print $2}' | grep -oP 'v0\.\K[0-9]+'`; fi && CONTROLLER_RUNTIME_MINOR=`echo "$CONTROLLER_RUNTIME_VERSION" | cut -d. -f2` && SETUP_ENVTEST_VERSION="release-0.${CONTROLLER_RUNTIME_MINOR}" && echo "SETUP_ENVTEST_VERSION=${SETUP_ENVTEST_VERSION}"
   ```

   d. **ENVTEST_K8S_VERSION** - Extract from controller-runtime's k8s.io/api version:
   ```bash
   # Extract K8s version from controller-runtime's go.mod (already fetched in step 4.1.c)
   # This ensures envtest uses the exact K8s version that controller-runtime was built with
   # Reference: https://github.com/kubernetes-sigs/controller-runtime/blob/main/go.mod

   # Reuse the GOMOD variable from step 4.1.c (already contains controller-runtime's go.mod)
   # Extract k8s.io/api version (e.g., v0.34.0)
   ENVTEST_K8S_VERSION=$(echo "$GOMOD" | grep 'k8s.io/api ' | awk '{print $2}')


   echo "Controller-runtime k8s.io/api version: ${ENVTEST_K8S_VERSION}"
   echo "  ✓ Matches controller-runtime v${CONTROLLER_RUNTIME_VERSION}"
   ```

   **Single-line version for Bash tool execution:**
   ```bash
   ENVTEST_K8S_VERSION=`echo "$GOMOD" | grep 'k8s.io/api ' | awk '{print $2}'` && echo "ENVTEST_K8S_VERSION=${ENVTEST_K8S_VERSION}"
   ```

   e. **GOLINT_VERSION** - Get latest compatible with Go version:
   ```bash
   # Fetch all golangci-lint releases
   GOLINT_RELEASES=$(curl -s https://api.github.com/repos/golangci/golangci-lint/releases | grep '"tag_name"' | sed -E 's/.*"v([^"]+)".*/\1/')

   # Find the latest version compatible with Go <go-version>
   # Check each release's go.mod to find Go requirement
   GOLINT_VERSION=""
   GO_MINOR=$(echo "<go-version>" | cut -d. -f2)

   for version in $GOLINT_RELEASES; do
     # Fetch go.mod for this version
     GO_REQ=$(curl -s "https://raw.githubusercontent.com/golangci/golangci-lint/v${version}/go.mod" | grep '^go ' | awk '{print $2}' | cut -d. -f2)

     if [ -n "$GO_REQ" ] && [ "$GO_REQ" -le "$GO_MINOR" ]; then
       GOLINT_VERSION="$version"
       echo "Found golangci-lint v${version} (requires Go 1.${GO_REQ}) - compatible with Go <go-version>"
       break
     fi
   done

   # Fallback to current version if no compatible version found
   if [ -z "$GOLINT_VERSION" ]; then
     echo "⚠️  Could not find compatible golangci-lint version, using current version from Makefile"
     GOLINT_VERSION=$(grep 'GOLINT_VERSION' Makefile | grep -oP 'v\K[0-9]+\.[0-9]+\.[0-9]+')
   fi

   echo "Golangci-lint version: v${GOLINT_VERSION}"
   echo "  ✓ Compatible with Go <go-version>"
   ```

   **Single-line version for Bash tool execution:**
   ```bash
   GOLINT_RELEASES=`curl -s https://api.github.com/repos/golangci/golangci-lint/releases | grep '"tag_name"' | sed -E 's/.*"v([^"]+)".*/\1/'` && GOLINT_VERSION="" && GO_MINOR=24 && for version in $GOLINT_RELEASES; do GO_REQ=`curl -s "https://raw.githubusercontent.com/golangci/golangci-lint/v${version}/go.mod" | grep '^go ' | awk '{print $2}' | cut -d. -f2`; if [ -n "$GO_REQ" ] && [ "$GO_REQ" -le "$GO_MINOR" ]; then GOLINT_VERSION="$version"; echo "Found golangci-lint v${version} (requires Go 1.${GO_REQ})"; break; fi; done && if [ -z "$GOLINT_VERSION" ]; then GOLINT_VERSION=`grep 'GOLINT_VERSION' Makefile | grep -oP 'v\K[0-9]+\.[0-9]+\.[0-9]+'`; fi && echo "GOLINT_VERSION=v${GOLINT_VERSION}"
   ```

   f. **Version compatibility summary**:
   ```bash
   echo ""
   echo "Tool versions to be set in Makefile:"
   echo "- KUSTOMIZE_VERSION: v${KUSTOMIZE_VERSION}"
   echo "- CONTROLLER_TOOLS_VERSION: v${CONTROLLER_TOOLS_VERSION}"
   echo "- SETUP_ENVTEST_VERSION: ${SETUP_ENVTEST_VERSION} (matches controller-runtime v0.${CONTROLLER_RUNTIME_VERSION})"
   echo "- ENVTEST_K8S_VERSION: ${ENVTEST_K8S_VERSION}"
   echo "- GOLINT_VERSION: v${GOLINT_VERSION}"
   ```

2. **Update Makefile variables with detected versions**
   ```bash
   # Update each variable in the Makefile

   # Update KUSTOMIZE_VERSION
   sed -i "s/^KUSTOMIZE_VERSION ?= .*/KUSTOMIZE_VERSION ?= v${KUSTOMIZE_VERSION}/" Makefile

   # Update CONTROLLER_TOOLS_VERSION
   sed -i "s/^CONTROLLER_TOOLS_VERSION ?= .*/CONTROLLER_TOOLS_VERSION ?= v${CONTROLLER_TOOLS_VERSION}/" Makefile

   # Update SETUP_ENVTEST_VERSION
   sed -i "s/^SETUP_ENVTEST_VERSION ?= .*/SETUP_ENVTEST_VERSION ?= ${SETUP_ENVTEST_VERSION}/" Makefile

   # Update ENVTEST_K8S_VERSION
   sed -i "s/^ENVTEST_K8S_VERSION = .*/ENVTEST_K8S_VERSION = ${ENVTEST_K8S_VERSION}/" Makefile

   # Update GOLINT_VERSION
   sed -i "s/^GOLINT_VERSION = .*/GOLINT_VERSION = v${GOLINT_VERSION}/" Makefile

   echo "✅ Makefile tool versions updated"
   ```

3. **Verify Makefile changes**
   ```bash
   # Display what was updated
   echo ""
   echo "Updated Makefile variables:"
   grep -E '(KUSTOMIZE_VERSION|CONTROLLER_TOOLS_VERSION|SETUP_ENVTEST_VERSION|ENVTEST_K8S_VERSION|GOLINT_VERSION)' Makefile
   ```

4. **Commit changes**
   ```bash
   git add Makefile
   git commit -m "Update tools in Makefile"
   ```

### Step 5: Run code generation and bundle

**Commit message**: `Update code after Golang, pivot to k8s <k8s-version> and dependencies upgrade`

1. **Run generation commands**
   ```bash
   make generate
   make manifests
   make bundle
   ```

2. **Capture and analyze output**
   - Log all warnings and errors
   - Check for deprecation warnings
   - **WARN** about any deprecated APIs or functions
   - Reference controller-runtime repository for deprecation PR context

3. **Commit all generated changes**
   ```bash
   git add -A
   git commit -m "Update code after Golang, pivot to k8s <k8s-version> and dependencies upgrade"
   ```

### Step 6: Run tests and build

**No commit** - Validation step only

1. **Run full test suite**
   ```bash
   make docker-build
   make build
   ./hack/golangci-lint.sh
   make unit
   ```

2. **Analyze test results**
   - **FAIL** if any tests fail
   - Log deprecation warnings for user review
   - Check for controller-runtime deprecation warnings
   - Suggest reviewing controller-runtime PRs for migration guidance

3. **If tests fail**
   - Present errors to user
   - Ask if they want to proceed with fixes or abort
   - If fixes needed, commit as: `Update code after Golang, pivot to k8s <k8s-version> and dependencies upgrade`

### Step 7: Detect Prow config needs

**No commit** - Detection and warning only

1. **Check if Prow config may need updates**
   - Analyze if Go version change affects CI
   - Check if K8s version change affects test environments
   - Reference example: https://github.com/openshift/release/pull/55728/commits/707fa080a66d8006c4a69e452a4621ed54f67cf6

2. **Generate warning message**
   ```
   ⚠️  PROW CONFIG UPDATE MAY BE REQUIRED

   The Go/K8s version upgrade may require updates to Prow configuration:
   - Repository: https://github.com/openshift/release
   - Path: ci-operator/config/openshift/multiarch-tuning-operator/

   Example PR: https://github.com/openshift/release/pull/55728

   Check if CI configuration needs updates for:
   - Go version: <go-version>
   - K8s version: <k8s-version>
   ```

### Step 8: Create summary and next steps

1. **Generate upgrade summary**
   ```
   ✅ UPGRADE SUMMARY

   Versions:
   - Go: <old-version> → <new-version>
   - Kubernetes: <old-version> → <new-version>
   - OpenShift: <ocp-version> (validated alignment)

   Commits created (in order):
   1. <sha1> Update go version in base images to <go-version>
   2. <sha2> pin K8S API to v<k8s-version> and set go minimum version to <go-version>
   3. <sha3> go mod vendor
   4. <sha4> Update tools in Makefile
   5. <sha5> Update code after Golang, pivot to k8s <k8s-version> and dependencies upgrade
   6. <sha6> Add info in the ocp-release.md doc about k8s and golang upgrade

   Files updated:
   - Dockerfile (UBI base image)
   - Makefile (BUILD_IMAGE, tool versions)
   - go.mod, go.sum (Go and K8s versions)
   - vendor/ (dependencies)
   - Generated code (CRDs, manifests, bundle)
   - docs/ocp-release.md (documentation)

   Warnings:
   [List any warnings about .tekton, getCorrectHostmountAnyUIDSCC, Prow config, etc.]

   Next steps:
   1. Review git log to verify commit structure
   2. Create PR with changes: gh pr create --title "Upgrade to Go <go-version> and K8s <k8s-version>"
   3. Check if Prow config needs updates (see warning above)
   4. Ensure all CI checks pass
   ```

2. **Offer to create PR**
   - Ask user if they want to create PR immediately
   - If yes, use gh CLI:
     ```bash
     gh pr create --title "Upgrade to Go <go-version> and K8s <k8s-version>" \
       --body "$(cat <<'EOF'
     ## Summary
     Upgrades Go and Kubernetes versions following official release process.

     - Go: <old-version> → <new-version>
     - Kubernetes: <old-version> → <new-version>
     - OpenShift: <ocp-version> (validated)

     ## Changes
     - Updated base images to UBI with Go <go-version>
     - Updated K8s dependencies to v<k8s-version>
     - Refreshed vendor directory
     - Updated tool versions in Makefile
     - Regenerated code, manifests, and bundle
     - All tests passing

     ## Validation
     - ✅ K8s version aligns with OCP <ocp-version>
     - ✅ UBI base images used
     - ✅ `getCorrectHostmountAnyUIDSCC` reviewed
     - ✅ Tests pass: make test
     - ✅ Build succeeds: make build

     Follows process: docs/ocp-release.md#pin-to-a-new-golang-and-k8s-api-version

     🤖 Generated with [Claude Code](https://claude.com/claude-code)
     EOF
     )"
     ```

## Arguments

- **$1** (`<go-version>`): Target Go version (e.g., "1.23", "1.23.6")
  - Format: Major.minor or Major.minor.patch
  - Must be compatible with target K8s version

- **$2** (`<k8s-version>`): Target Kubernetes version (e.g., "1.32.3")
  - Format: Major.minor.patch
  - **MUST** align with OpenShift version from BUILD_IMAGE

- **$3** (`--repo-path`): Optional path to repository
  - Default: `~/openshift_working/multiarch-tuning-operator`
  - Must be a valid git repository

## Return Value

- **Exit 0**: Upgrade completed successfully, all tests pass
- **Exit 1**: Version validation failed (K8s/OCP mismatch, invalid versions)
- **Exit 2**: Tests failed after upgrade
- **Exit 3**: Code generation failed
- **Exit 4**: Docker file uses non-UBI base image

## Examples

### Basic upgrade to Go 1.23 and K8s 1.32.3

```bash
/go-updater:multiarch-tuning-operator:upgrade-versions 1.23 1.32.3
```

Expected behavior:
1. Validates K8s 1.32.3 matches OCP version in BUILD_IMAGE
2. Updates all base images to UBI with Go 1.23
3. Updates go.mod to Go 1.23 and K8s 1.32.3
4. Regenerates vendor, runs code generation
5. Runs full test suite
6. Creates 6 structured commits
7. Generates summary with PR creation option

### Upgrade with custom repository path

```bash
/go-updater:multiarch-tuning-operator:upgrade-versions 1.24 1.33.0 --repo-path /custom/path/multiarch-tuning-operator
```

### Version validation failure example

```bash
/go-updater:multiarch-tuning-operator:upgrade-versions 1.23 1.30.4
```

Expected output:
```
❌ VERSION VALIDATION FAILED

Current OpenShift version: 4.19 (from BUILD_IMAGE)
Expected Kubernetes version: 1.32.x
Requested Kubernetes version: 1.30.4

ERROR: Kubernetes version 1.30.4 does not align with OpenShift 4.19
OpenShift 4.19 requires Kubernetes 1.32.x

Correct version alignment:
- OCP 4.16 → K8s 1.29.x
- OCP 4.17 → K8s 1.30.x
- OCP 4.18 → K8s 1.31.x
- OCP 4.19 → K8s 1.32.x
- OCP 4.20 → K8s 1.33.x

Please use: /go-updater:multiarch-tuning-operator:upgrade-versions 1.23 1.32.3
```

## Error Handling

### Image validation failures

If base images cannot be found during Prerequisites Check step 4:

```
❌ IMAGE VALIDATION FAILED

Attempted Go version: 1.23
Target OCP version: 4.19

Image validation results:
✅ golang:1.23 (Docker Hub) - FOUND
❌ registry.ci.openshift.org/ocp/builder:rhel-9-golang-1.23-openshift-4.19 - NOT FOUND
   Tried: rhel-9-golang-1.23-openshift-4.19
   Tried: rhel-9-golang-1.23-builder-multi-openshift-4.19
❌ brew.registry.redhat.io/rh-osbs/openshift-golang-builder:rhel_9_1.23 - NOT FOUND

Retrying with patch versions...

Attempting 1.23.0:
❌ registry.ci.openshift.org/ocp/builder:rhel-9-golang-1.23.0-openshift-4.19 - NOT FOUND

Attempting 1.23.1:
✅ golang:1.23.1 (Docker Hub) - FOUND
✅ registry.ci.openshift.org/ocp/builder:rhel-9-golang-1.23.1-openshift-4.19 - FOUND
✅ brew.registry.redhat.io/rh-osbs/openshift-golang-builder:rhel_9_1.23.1 - FOUND

RECOMMENDATION: Use Go version 1.23.1

Continue with Go 1.23.1? [Y/n]
```

**Retry behavior:**
- If user-provided version (e.g., `1.23`) fails, automatically try patch versions
- Try up to patch versions 0-3, or check golang.org/dl for latest
- If any patch version has all images available, suggest it to user
- If no patch versions work, FAIL and recommend user check:
  - VPN connection (for Red Hat internal registries)
  - Registry authentication (`podman login brew.registry.redhat.io`)
  - Whether the OCP/Go version combination exists yet

### Non-UBI Dockerfile

If Dockerfile contains `FROM golang:` instead of UBI:
```
❌ DOCKERFILE VALIDATION FAILED

Found: FROM golang:1.23
Required: FROM registry.access.redhat.com/ubi9/go-toolset:1.23

The multiarch-tuning-operator MUST use Universal Base Images (UBI).
Docker Hub golang images are not acceptable for OpenShift operators.

Please manually update Dockerfile to use UBI before proceeding.
```

### .tekton directory detected

If `.tekton/` directory exists:
```
⚠️  KONFLUX .TEKTON CONFIGURATION DETECTED

The following files may need Go version updates:
- .tekton/multiarch-tuning-operator-1-0-pull-request.yaml
- .tekton/multiarch-tuning-operator-1-0-push.yaml

Check for 'build-args' parameters with base image references.
Update base image to match Go <go-version>.

This command does NOT automatically update .tekton files.
Manual review required.
```

### getCorrectHostmountAnyUIDSCC validation warning

If function needs updates:
```
⚠️  KUBERNETES VERSION VALIDATION REQUIRED

Function: getCorrectHostmountAnyUIDSCC
Location: <file>:<line>

This function maps Kubernetes versions to OpenShift SCC for hostPath mounts.
After upgrading to K8s <k8s-version>, verify the version logic is correct.

Current K8s version: <current-version>
Target K8s version: <target-version>
OpenShift version: <ocp-version>

Please review and update if necessary.
```

### Test failures

If `make test` fails:
```
❌ TEST SUITE FAILED

The following tests failed after version upgrade:
<test output>

Common causes:
1. Deprecated API usage (check controller-runtime deprecation PRs)
2. Breaking changes in K8s dependencies
3. Tool version incompatibilities

Next steps:
1. Review controller-runtime repository for deprecation warnings
2. Check K8s changelog for breaking changes
3. Update code to fix failures

Would you like to:
- [A] Abort and rollback changes
- [C] Continue and commit fixes manually
- [H] Get help analyzing failures
```

## Notes

- This command implements the complete official process from `docs/ocp-release.md`
- All 9 steps are enforced in the correct order
- Commit structure follows the example log pattern from the official docs
- Version compatibility is strictly validated
- UBI image requirements are enforced
- Special validations (getCorrectHostmountAnyUIDSCC) are performed
- Prow config needs are detected and warned about
- See example PRs: #225, #542

## See Also

- Official process: `docs/ocp-release.md#pin-to-a-new-golang-and-k8s-api-version`
- Example PR #225: https://github.com/openshift/multiarch-tuning-operator/pull/225
- Example PR #542: https://github.com/openshift/multiarch-tuning-operator/pull/542
- Prow config example: https://github.com/openshift/release/pull/55728