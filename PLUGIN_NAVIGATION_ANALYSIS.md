# K9s Plugin Navigation Enhancement Analysis

## Use Case: Seamless NodeClaim to Node Navigation

### Problem Statement

When working with Karpenter-managed clusters, users frequently need to navigate from NodeClaim resources to their associated Node resources. Currently, this requires a manual multi-step process:

1. View a NodeClaim in k9s (`:nodeclaims`)
2. Note or copy the associated node name from the NodeClaim status
3. Manually switch to nodes view (`:nodes`) 
4. Search/filter for the specific node name

This workflow is cumbersome and breaks the user's flow when investigating issues or understanding cluster state.

### Desired Behavior

**Goal**: Enable a single keypress (e.g., `Ctrl-N`) while viewing a NodeClaim to automatically:
1. Extract the associated node name from the NodeClaim's `status.nodeName` field
2. Navigate directly to the nodes view
3. Automatically filter/highlight the associated node
4. Provide seamless, contextual navigation between related Kubernetes resources

### Current Limitation

k9s plugins can only execute external commands and display output - they cannot programmatically control navigation or view switching within the k9s interface.

## Summary

This document analyzes the feasibility of extending k9s plugins to support navigation between resources, specifically to enable the NodeClaim-to-Node navigation described above. Based on codebase analysis, this enhancement is **definitely feasible** with minimal changes to the existing architecture.

## Current Plugin Architecture

### How Plugins Work Today

**File:** `internal/view/actions.go:124-178`

1. **Plugin Loading**: Plugins are loaded from `plugins.yaml` files via `config.NewPlugins()`
2. **Key Binding**: Plugin shortcuts are registered as key actions in the UI
3. **Execution**: When triggered, plugins execute as external shell commands
4. **Output**: Plugin output is displayed in a separate terminal view

**Key Functions:**
- `pluginActions()` - Registers plugin key bindings
- `pluginAction()` - Handles plugin execution (`internal/view/actions.go:180-245`)

### Plugin Configuration Structure

**File:** `internal/config/plugin.go:30-43`

```go
type Plugin struct {
    Scopes          []string `yaml:"scopes"`
    Args            []string `yaml:"args"`
    ShortCut        string   `yaml:"shortCut"`
    Override        bool     `yaml:"override"`
    Pipes           []string `yaml:"pipes"`
    Description     string   `yaml:"description"`
    Command         string   `yaml:"command"`
    Confirm         bool     `yaml:"confirm"`
    Background      bool     `yaml:"background"`
    Dangerous       bool     `yaml:"dangerous"`
    OverwriteOutput bool     `yaml:"overwriteOutput"`
}
```

### Current Limitations

- Plugins run as external shell commands with no direct UI control
- No mechanism for plugins to trigger navigation or view changes
- Output is isolated in terminal view, cannot affect main k9s interface

## Navigation System Analysis

### How k9s Handles Navigation

**Core Flow:**
1. **Entry Point**: `App.gotoResource()` (`internal/view/app.go:766`)
2. **Command Processing**: `Command.run()` (`internal/view/command.go:161`)
3. **Component Creation**: `componentFor()` creates appropriate view components
4. **UI Update**: `app.inject()` pushes new component to UI stack

**Key Files:**
- `internal/view/app.go:766` - Main navigation entry point
- `internal/view/command.go:161` - Command processing and execution
- `internal/view/cmd/interpreter.go` - Command parsing and interpretation

### Navigation Command Flow

```
User Input (:nodes) 
    ↓
App.gotoResource(cmd, path, clearStack, pushCmd)
    ↓  
Command.run(interpreter, fqn, clearStack, pushCmd)
    ↓
viewMetaFor() - Resolve resource type
    ↓
componentFor() - Create view component
    ↓
app.inject() - Push to UI stack
```

## Proposed Enhancement Approaches

### Option 1: Output-Based Navigation (Recommended)

**Concept**: Extend plugin execution to parse special output commands for navigation.

**Implementation Location**: `internal/view/actions.go:200-245` (pluginAction function)

**Changes Required:**

1. **Modify Plugin Execution Handler**:
```go
// After plugin execution, check for navigation commands
if strings.HasPrefix(output, "NAVIGATE:") {
    navCmd := strings.TrimPrefix(output, "NAVIGATE:")
    parts := strings.Fields(navCmd)
    if len(parts) >= 1 {
        resource := parts[0]
        filter := ""
        if len(parts) > 1 {
            filter = parts[1]
        }
        r.App().gotoResource(resource, "", false, true)
        // Optionally apply filter to new view
    }
}
```

2. **Plugin Script Enhancement**:
```bash
# Plugin outputs navigation command
echo "NAVIGATE:nodes $NODE_NAME"
```

**Advantages:**
- Minimal code changes
- Backward compatible
- Uses existing navigation infrastructure
- Simple plugin interface

### Option 2: Environment Variable Communication

**Concept**: Plugins set environment variables that k9s reads for post-execution actions.

**Implementation**: Check environment variables after plugin execution:
```go
if navResource := os.Getenv("K9S_NAVIGATE"); navResource != "" {
    filter := os.Getenv("K9S_NAVIGATE_FILTER")
    r.App().gotoResource(navResource, "", false, true)
    os.Unsetenv("K9S_NAVIGATE")
    os.Unsetenv("K9S_NAVIGATE_FILTER")
}
```

### Option 3: Configuration-Based Navigation

**Concept**: Extend plugin configuration to support navigation directives.

**New Plugin Fields:**
```go
type Plugin struct {
    // ... existing fields
    NavigateAfter   string   `yaml:"navigateAfter"`   // Resource to navigate to
    NavigateFilter  string   `yaml:"navigateFilter"` // Filter expression
}
```

## Recommended Implementation Plan

### Phase 1: Output-Based Navigation (Option 1)

**Files to Modify:**
- `internal/view/actions.go` - Plugin execution handler
- `internal/config/plugin.go` - Optional: Add navigation field to Plugin struct

**Implementation Steps:**

1. **Extend pluginAction() function** (`internal/view/actions.go:180-245`):
   - Capture plugin stdout
   - Parse for navigation commands
   - Execute navigation via existing `gotoResource()`

2. **Add navigation parsing**:
   - Support format: `NAVIGATE:<resource> [filter]`
   - Examples: `NAVIGATE:nodes`, `NAVIGATE:pods myapp`

3. **Update plugin output handling**:
   - Strip navigation commands from displayed output
   - Execute navigation after plugin completes

### Phase 2: Enhanced Configuration (Optional)

1. **Extend Plugin struct** with navigation fields
2. **Add UI indicators** for navigation-enabled plugins
3. **Support complex navigation scenarios** (namespace switching, etc.)

## Example: NodeClaim to Node Navigation

### Current Plugin (Manual Navigation)
```yaml
plugins:
  nodeclaim-to-node:
    shortCut: Ctrl-N
    description: Jump to Node from NodeClaim
    scopes: [nodeclaims]
    command: sh
    args: [-c, |
      NODE_NAME=$(kubectl get nodeclaim $NAME -o jsonpath='{.status.nodeName}')
      echo "Associated node: $NODE_NAME"
      echo "Navigate manually with: :nodes"
    ]
```

### Enhanced Plugin (Automatic Navigation)
```yaml
plugins:
  nodeclaim-to-node:
    shortCut: Ctrl-N
    description: Jump to Node from NodeClaim
    scopes: [nodeclaims]
    command: sh
    args: [-c, |
      NODE_NAME=$(kubectl get nodeclaim $NAME -o jsonpath='{.status.nodeName}')
      if [ -n "$NODE_NAME" ]; then
        echo "Navigating to node: $NODE_NAME"
        echo "NAVIGATE:nodes $NODE_NAME"
      else
        echo "No associated node found"
      fi
    ]
```

## Technical Considerations

### Error Handling
- Invalid navigation commands should be logged but not crash k9s
- Graceful fallback when target resource doesn't exist
- User feedback for navigation failures

### Security
- Validate navigation commands to prevent injection
- Restrict navigation to valid k8s resource types
- Maintain existing plugin security model

### Performance
- Minimal overhead for existing plugins (no navigation commands)
- Efficient output parsing for navigation-enabled plugins
- No impact on k9s startup or resource loading

### Backward Compatibility
- Existing plugins continue to work unchanged
- New navigation feature is opt-in via special output format
- No breaking changes to plugin configuration

## Conclusion

The plugin navigation enhancement is **highly feasible** with the output-based approach (Option 1). The implementation requires minimal changes to existing code, maintains backward compatibility, and leverages k9s's existing robust navigation infrastructure.

**Key Benefits:**
- Seamless navigation between related resources
- Enhanced user workflow efficiency  
- Minimal implementation complexity
- Strong backward compatibility

**Estimated Implementation Effort:** 
- Core functionality: ~50 lines of code
- Testing and edge cases: ~100 lines
- Documentation updates: Minimal

This enhancement would significantly improve the k9s user experience for complex Kubernetes environments with related resources like NodeClaims and Nodes.