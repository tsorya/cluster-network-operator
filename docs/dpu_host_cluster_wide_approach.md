# DPU Host Mode Cluster-Wide Feature Disabling - Current Implementation

This document describes the current implementation for disabling DPU-incompatible features cluster-wide when DPU host mode is enabled anywhere in the cluster.

## Overview

When DPU (Data Processing Unit) hosts are detected **anywhere** in the cluster, certain networking features that are incompatible with DPU hardware must be disabled **cluster-wide** to ensure consistent network behavior across all nodes.

## Architecture

The implementation uses a single detection point and consolidated control approach:

1. **Detection**: Go code detects if any DPU nodes exist in the cluster using `DpuModeNodes`
2. **Control Flag**: Sets `DPU_HOST_MODE_ENABLED = true` when DPU hosts present
3. **Feature Disabling**: Go code directly controls all DPU-incompatible feature flags
4. **Template Rendering**: ConfigMaps and scripts use the processed flags for rendering

## Implementation Details

### 1. DPU Host Detection (Go Code)

```go
// pkg/network/ovn_kubernetes.go (bootstrapOVNConfig function)

// Detect if DPU nodes are present in cluster  
ovnConfigResult.DpuHostModeEnabled = len(ovnConfigResult.DpuModeNodes) > 0
if ovnConfigResult.DpuHostModeEnabled {
    klog.Infof("DPU host mode enabled - DPU-incompatible features will be disabled cluster-wide (%d DPU nodes found)", len(ovnConfigResult.DpuModeNodes))
}
```

### 2. Feature Flag Control (Go Code)

```go
// pkg/network/ovn_kubernetes.go (renderOVNKubernetes function)

// Detect if DPU host mode is enabled cluster-wide
dpuHostModeEnabled := bootstrapResult.OVN.OVNKubernetesConfig.DpuHostModeEnabled

// Single flag to control all DPU-incompatible features in templates
data.Data["DPU_HOST_MODE_ENABLED"] = dpuHostModeEnabled

// Set default feature gate values
data.Data["OVN_ADMIN_NETWORK_POLICY_ENABLE"] = featureGates.Enabled(apifeatures.FeatureGateAdminNetworkPolicy)
data.Data["DNS_NAME_RESOLVER_ENABLE"] = featureGates.Enabled(apifeatures.FeatureGateDNSNameResolver)
data.Data["OVN_NETWORK_SEGMENTATION_ENABLE"] = featureGates.Enabled(apifeatures.FeatureGateNetworkSegmentation)
data.Data["OVN_OBSERVABILITY_ENABLE"] = featureGates.Enabled(apifeatures.FeatureGateOVNObservability)
data.Data["OVN_ROUTE_ADVERTISEMENTS_ENABLE"] = c.RouteAdvertisements == operv1.RouteAdvertisementsEnabled
data.Data["OVN_PRE_CONF_UDN_ADDR_ENABLE"] = featureGates.Enabled(apifeatures.FeatureGatePreconfiguredUDNAddresses)
data.Data["OVN_MULTICAST_ENABLE"] = true // Default enabled

// Disable all DPU-incompatible features when DPU host mode enabled
if dpuHostModeEnabled {
    // Disable feature gates that are incompatible with DPU
    data.Data["OVN_ADMIN_NETWORK_POLICY_ENABLE"] = false
    data.Data["OVN_NETWORK_SEGMENTATION_ENABLE"] = false
    data.Data["OVN_MULTI_NETWORK_ENABLE"] = false
    data.Data["OVN_MULTI_NETWORK_POLICY_ENABLE"] = false
    data.Data["OVN_MULTICAST_ENABLE"] = false
}
```

### 3. ConfigMap Template Usage

**Both Managed and Self-Hosted ConfigMaps** (`004-config.yaml`):

```ini
[ovnkubernetesfeature]
{{- if not .DPU_HOST_MODE_ENABLED }}
enable-egress-ip=true
enable-egress-firewall=true
enable-egress-qos=true
enable-egress-service=true
enable-multi-external-gateway=true
{{- end }}
{{- if .ReachabilityNodePort }}
egressip-node-healthcheck-port={{.ReachabilityNodePort}}
{{- end }}
{{- if .OVN_MULTI_NETWORK_ENABLE }}
enable-multi-network=true
{{- end }}
{{- if .OVN_NETWORK_SEGMENTATION_ENABLE }}
{{- if not .OVN_MULTI_NETWORK_ENABLE }}
enable-multi-network=true
{{- end }}
enable-network-segmentation=true
{{- end }}
{{- if .OVN_PRE_CONF_UDN_ADDR_ENABLE }}
enable-preconfigured-udn-addresses=true
{{- end }}
{{- if .OVN_MULTI_NETWORK_POLICY_ENABLE }}
enable-multi-networkpolicy=true
{{- end }}
{{- if .OVN_ADMIN_NETWORK_POLICY_ENABLE }}
enable-admin-network-policy=true
{{- end }}
{{- if .DNS_NAME_RESOLVER_ENABLE }}
enable-dns-name-resolver=true
{{- end }}
```

### 4. Script Template Usage

**Node Script** (`008-script-lib.yaml`):

```bash
# Multicast flag controlled by Go-level feature flag
enable_multicast_flag=""
if [[ "{{.OVN_MULTICAST_ENABLE}}" == "true" ]]; then
  enable_multicast_flag="--enable-multicast"
fi

# Gateway interface logic for DPU host nodes
gateway_interface=br-ex
if [ "${OVN_NODE_MODE}" == "dpu-host" ]; then
  # DPU host mode configuration
  gateway_interface="derive-from-mgmt-port"
  ovnkube_node_mode="--ovnkube-node-mode dpu-host"
fi

# Use the processed flags in ovnkube command
exec /usr/bin/ovnkube \
  ${enable_multicast_flag} \
  --gateway-interface ${gateway_interface} \
  # ... other arguments
```

## Bootstrap Type Definition

```go
// pkg/bootstrap/types.go
type OVNConfigBoostrapResult struct {
    // ...
    // DpuHostModeEnabled indicates whether DPU host mode is enabled cluster-wide.
    // When true, DPU-incompatible features will be disabled cluster-wide.
    DpuHostModeEnabled bool
    
    // DpuModeNodes contains list of DPU nodes detected in the cluster
    DpuModeNodes []string
    // ...
}
```

## Cluster Behavior

| Scenario | Result | Affected Components |
|----------|--------|-------------------|
| **DPU host mode disabled** | All features enabled | ✅ All nodes have full features |
| **DPU host mode enabled** | DPU-incompatible features disabled | ❌ **All nodes** lose DPU-incompatible features |

### Why Cluster-Wide Disabling?

1. **Network Consistency**: Ensures uniform network behavior across all nodes
2. **Traffic Flow**: Prevents traffic routing through paths that don't support certain features
3. **Operational Simplicity**: Avoids complex per-node feature negotiation
4. **Reliability**: Eliminates potential failures from feature mismatches

## Features Disabled When DPU Host Mode Enabled

When `DpuHostModeEnabled = true`, the following features are disabled **cluster-wide**:

- ❌ **Egress Features**: IP, Firewall, QoS, Service
- ❌ **Multi-external Gateway**
- ❌ **Multicast Support**
- ❌ **Admin Network Policy**
- ❌ **Network Segmentation**  
- ❌ **Multi-network Support**
- ❌ **Multi-network Policies**

## Detection Logic

DPU nodes are identified by the `DpuModeLabel` which is typically configured as:

```bash
# Default DPU mode label selector
kubectl get nodes -l feature.node.kubernetes.io/network-sriov.capable=true
```

The exact label is configurable via:
- `OVN_NODE_SELECTOR_DEFAULT_DPU` environment variable (for DPU mode nodes)
- Bootstrap configuration in `DpuModeLabel` field

## Logging

When DPU nodes are detected:

```
INFO DPU host mode enabled - DPU-incompatible features will be disabled cluster-wide (2 DPU nodes found)
```

## Summary

This implementation ensures that:

1. **Single Detection Point**: DPU nodes detected once during bootstrap using `DpuModeNodes`
2. **Centralized Control**: Go code controls all feature flags in one consolidated `if` block  
3. **Template Flag**: `DPU_HOST_MODE_ENABLED` controls ConfigMap egress features
4. **Feature Flag Control**: Individual `OVN_*_ENABLE` flags control their respective features
5. **Cluster-Wide Impact**: All nodes affected when DPU host mode enabled
6. **Consistent Behavior**: Network operates uniformly across entire cluster
7. **Comprehensive Coverage**: All DPU-incompatible features controlled (egress, multicast, multi-network, network segmentation, admin network policy)

This approach guarantees reliable network operation in mixed clusters containing both regular nodes and DPU nodes, with comprehensive test coverage validating all implemented features.

## Architecture Notes

This document reflects the **current implementation** as of the final DPU host mode feature. Previous architectural approaches explored different methods:

- **Pure Go Approach** (`dpu_host_pure_go_approach.md`): Early exploration of Go-only feature control
- **ConfigMap Approach** (`dpu_host_configmap_approach.md`): ConfigMap-centric feature disabling
- **Control Plane Mode** (`dpu_host_control_plane_mode.md`): Control plane specific implementations

The **current cluster-wide approach** combines the best aspects of these explorations:
- **Go code** handles detection and feature flag control (from Pure Go approach)
- **ConfigMaps** handle template rendering of processed flags (from ConfigMap approach)  
- **Cluster-wide scope** ensures consistent behavior across all components (from cluster-wide requirement)

This hybrid approach provides the maintainability benefits of centralized Go control while leveraging the template system for clean separation of concerns.
