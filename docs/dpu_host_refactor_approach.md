# DPU Host Feature Flag Refactor

This document describes the refactored approach for handling DPU host mode that centralizes the logic in Go code rather than spreading it across templates and scripts.

## Problem with Previous Approach

The initial implementation added complex conditional logic in script templates:

```bash
# COMPLEX: Multiple conditions in templates
if [[ "{{.OVN_MULTI_NETWORK_ENABLE}}" == "true" && "${OVN_NODE_MODE}" != "dpu-host" && "{{.DpuHostControlPlaneMode}}" != "true" ]]; then
  multi_network_enabled_flag="--enable-multi-network"
fi
```

This approach had several issues:
- **Complex template logic**: Hard to maintain and error-prone
- **Scattered logic**: DPU host handling spread across Go code, templates, and scripts
- **Template variable propagation**: Required passing `DpuHostControlPlaneMode` to all templates

## Improved Approach

### Centralized Feature Flag Management

Instead of checking `DpuHostControlPlaneMode` in templates, we now disable the feature flags directly in the Go code when DPU hosts are present:

```go
// pkg/network/ovn_kubernetes.go
dpuHostControlPlaneMode := bootstrapResult.OVN.OVNKubernetesConfig.DpuHostControlPlaneMode

// Disable DPU-incompatible features when DPU hosts are present
data.Data["OVN_ADMIN_NETWORK_POLICY_ENABLE"] = featureGates.Enabled(apifeatures.FeatureGateAdminNetworkPolicy) && !dpuHostControlPlaneMode
data.Data["OVN_NETWORK_SEGMENTATION_ENABLE"] = featureGates.Enabled(apifeatures.FeatureGateNetworkSegmentation) && !dpuHostControlPlaneMode

// Disable multi-network features when DPU hosts are present
if dpuHostControlPlaneMode {
    data.Data["OVN_MULTI_NETWORK_ENABLE"] = false
    data.Data["OVN_MULTI_NETWORK_POLICY_ENABLE"] = false
}
```

### Simplified Script Logic

Scripts now only need to check `OVN_NODE_MODE` for DPU host specific settings:

```bash
# SIMPLE: Only check node mode for DPU host specific settings
if [[ "{{.OVN_MULTI_NETWORK_ENABLE}}" == "true" && "${OVN_NODE_MODE}" != "dpu-host" ]]; then
  multi_network_enabled_flag="--enable-multi-network"
fi
```

## Benefits of Refactored Approach

### 1. **Centralized Logic**
- All DPU host feature disabling happens in one place (Go code)
- No need to duplicate conditions across multiple templates
- Single source of truth for feature state

### 2. **Simplified Templates**
- Scripts only handle node-specific settings (gateway interface, node mode)
- No complex multi-condition checks in bash scripts
- Easier to read and maintain

### 3. **Cleaner Architecture**
- **Go Code**: Handles feature flag logic based on cluster state
- **Templates**: Use pre-calculated feature flags 
- **Scripts**: Handle runtime node-specific configuration

### 4. **Better Maintainability**
- Adding new DPU-incompatible features only requires changes in Go code
- No need to update multiple template files
- Reduced chance of inconsistencies

## Implementation Details

### Feature Flags Affected

When `DpuHostControlPlaneMode = true` (DPU hosts present):

```go
// Disabled via boolean logic
OVN_ADMIN_NETWORK_POLICY_ENABLE = featureGateEnabled && !dpuHostControlPlaneMode
OVN_NETWORK_SEGMENTATION_ENABLE = featureGateEnabled && !dpuHostControlPlaneMode

// Disabled via direct assignment  
OVN_MULTI_NETWORK_ENABLE = false
OVN_MULTI_NETWORK_POLICY_ENABLE = false
```

### Template Usage

Templates consume the pre-calculated flags without additional logic:

```yaml
# Control Plane Template
{{- if .OVN_MULTI_NETWORK_ENABLE }}
multi_network_enabled_flag="--enable-multi-network"
{{- end }}

# Config Template  
{{- if .OVN_NETWORK_SEGMENTATION_ENABLE }}
enable-network-segmentation=true
{{- end }}
```

### Node-Specific Logic

Scripts still handle DPU host node-specific settings:

```bash
if [ "${OVN_NODE_MODE}" == "dpu-host" ]; then
  gateway_interface="derive-from-mgmt-port"
  ovnkube_node_mode="--ovnkube-node-mode dpu-host"
  # Disable features not supported on DPU host nodes
  egress_features_enable_flag=""
  enable_multicast_flag=""
fi
```

## Result

The refactored approach provides:
- ✅ **Single source of truth** for DPU host feature logic
- ✅ **Simplified templates** without complex conditional logic  
- ✅ **Cleaner separation of concerns** between Go code, templates, and scripts
- ✅ **Easier maintenance** when adding new DPU-incompatible features
- ✅ **Consistent behavior** across all cluster components

This architectural improvement makes the codebase more maintainable while achieving the same functional result: when DPU hosts are present, incompatible features are disabled across the entire cluster.
