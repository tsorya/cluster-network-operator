# DPU Host Full Mode Extension

This document describes the extension of DPU Host Control Plane Mode to also affect regular "full mode" OVN Kubernetes nodes when DPU hosts are present in the cluster.

## Overview

Building on the DPU Host Control Plane Mode feature, this extension ensures that when DPU hosts are detected in the cluster, not only the control plane but also all regular (full mode) nodes have the same features disabled to maintain consistency across the entire cluster.

## Problem Statement

Previously:
1. DPU host nodes (`OVN_NODE_MODE=dpu-host`) had certain features disabled
2. Control plane was updated to disable the same features when DPU hosts are present
3. Regular nodes (`OVN_NODE_MODE=full`) continued to operate with all features enabled

This created an inconsistency where regular nodes might try to use features that aren't supported when DPU hosts are present in the cluster.

## Solution

Extend the existing logic to also disable DPU-incompatible features on regular nodes when `DpuHostControlPlaneMode=true` (i.e., when DPU hosts are present).

## Implementation

### Script Logic Updates (`008-script-lib.yaml`)

Modified the conditional checks in the startup script to consider both the node mode AND the DpuHostControlPlaneMode flag:

#### Core Feature Disabling
```bash
# Previous logic: only check OVN_NODE_MODE
if [ "${OVN_NODE_MODE}" == "dpu-host" ]; then
  # disable features
fi

# New logic: check both node mode AND DPU host control plane mode
if [ "${OVN_NODE_MODE}" == "dpu-host" ] || [ "{{.DpuHostControlPlaneMode}}" == "true" ]; then
  if [ "${OVN_NODE_MODE}" == "dpu-host" ]; then
    # DPU host specific settings (gateway interface, node mode)
    gateway_interface="derive-from-mgmt-port"
    ovnkube_node_mode="--ovnkube-node-mode dpu-host"
  fi
  # Common feature disabling for both DPU hosts and when DPU hosts are present
  egress_features_enable_flag=""
  enable_multicast_flag=""
  init_ovnkube_controller=""
  multi_external_gateway_enable_flag=""
fi
```

#### Feature Flag Conditions
Updated all conditional feature flags to include the DpuHostControlPlaneMode check:

```bash
# Multi-network support
if [[ "{{.OVN_MULTI_NETWORK_ENABLE}}" == "true" && "${OVN_NODE_MODE}" != "dpu-host" && "{{.DpuHostControlPlaneMode}}" != "true" ]]; then
  multi_network_enabled_flag="--enable-multi-network"
fi

# Network segmentation
if [[ "{{.OVN_NETWORK_SEGMENTATION_ENABLE}}" == "true" && "${OVN_NODE_MODE}" != "dpu-host" && "{{.DpuHostControlPlaneMode}}" != "true" ]]; then
  network_segmentation_enabled_flag="--enable-network-segmentation"
fi

# Multi-network policy
if [[ "{{.OVN_MULTI_NETWORK_POLICY_ENABLE}}" == "true" && "${OVN_NODE_MODE}" != "dpu-host" && "{{.DpuHostControlPlaneMode}}" != "true" ]]; then
  multi_network_policy_enabled_flag="--enable-multi-networkpolicy"
fi

# Admin network policy
if [[ "{{.OVN_ADMIN_NETWORK_POLICY_ENABLE}}" == "true" && "${OVN_NODE_MODE}" != "dpu-host" && "{{.DpuHostControlPlaneMode}}" != "true" ]]; then
  admin_network_policy_enabled_flag="--enable-admin-network-policy"
fi
```

### Configuration Files

The INI configuration files (`004-config.yaml`) that were already updated for control plane also apply to nodes since both use the same `ovnkube-config` ConfigMap.

## Features Affected

When DPU hosts are present in the cluster, the following features are disabled on ALL nodes (not just DPU host nodes):

### Core Features
- ❌ Egress IP and related features
- ❌ Multicast support  
- ❌ Multi-external gateway support
- ❌ OVN controller initialization (for non-DPU nodes)

### Advanced Features
- ❌ Multi-network support
- ❌ Network segmentation
- ❌ Multi-network policies
- ❌ Admin network policies

### Behavior by Node Type

| Node Type | Without DPU Hosts | With DPU Hosts Present |
|-----------|------------------|------------------------|
| Regular (`full`) | All features enabled | DPU-incompatible features disabled |
| DPU Host (`dpu-host`) | DPU-incompatible features disabled | DPU-incompatible features disabled |
| Smart NIC (`smart-nic`) | All features enabled | DPU-incompatible features disabled |

## Benefits

1. **Consistency**: All nodes in the cluster have the same feature set when DPU hosts are present
2. **Compatibility**: Prevents feature conflicts between DPU hosts and regular nodes
3. **Automatic**: No manual configuration required - automatically detected and applied
4. **Safe**: Ensures cluster-wide compatibility with DPU host limitations

## Files Modified

- `bindata/network/ovn-kubernetes/common/008-script-lib.yaml` - Core script logic updates
- Previous control plane and config changes already covered both control plane and nodes

## Backward Compatibility

This change is fully backward compatible:
- Clusters without DPU hosts continue to operate with full feature support
- DPU host nodes continue to work as before
- Only affects behavior when DPU hosts are present, ensuring cluster-wide consistency

## Testing

To verify the implementation:

1. **Without DPU hosts**: All nodes should have full features enabled
2. **With DPU hosts**: All nodes (including regular nodes) should have DPU-incompatible features disabled
3. **Logs**: Check for the "DPU host control plane mode enabled" message in operator logs when DPU hosts are present
