# DPU Host ConfigMap Approach - Final Implementation

This document describes the final architecture for handling DPU host mode that centralizes feature disabling in ConfigMaps while maintaining consistency across the cluster.

## Architecture Overview

The implementation follows a clean layered approach:

1. **Go Code**: Detects DPU hosts and sets `DpuHostControlPlaneMode` flag
2. **ConfigMaps**: Centrally handle ALL feature enabling/disabling based on DPU host presence  
3. **Control Plane Templates**: Handle command-line argument features based on DPU host presence
4. **Node Scripts**: Handle ONLY node-specific settings for DPU host nodes (no feature disabling logic)

## Implementation Details

### 1. DPU Host Detection (Go Code)

```go
// pkg/network/ovn_kubernetes.go
// Enable DPU host mode for control plane if any DPU host nodes are present
ovnConfigResult.DpuHostControlPlaneMode = len(ovnConfigResult.DpuHostModeNodes) > 0
```

The Go code only:
- Detects DPU host nodes in the cluster
- Sets the `DpuHostControlPlaneMode` boolean flag
- Passes this flag to all templates via `data.Data["DpuHostControlPlaneMode"]`

### 2. ConfigMap Feature Disabling

**Managed ConfigMap** (`managed/004-config.yaml`):
```ini
[ovnkubernetesfeature]
{{- if ne .DpuHostControlPlaneMode true }}
enable-egress-ip=true
enable-egress-firewall=true
enable-egress-qos=true
enable-egress-service=true
{{- end }}

{{- if and .OVN_MULTI_NETWORK_ENABLE (ne .DpuHostControlPlaneMode true) }}
enable-multi-network=true
{{- end }}

{{- if and .OVN_NETWORK_SEGMENTATION_ENABLE (ne .DpuHostControlPlaneMode true) }}
enable-network-segmentation=true
{{- end }}

{{- if and .OVN_MULTI_NETWORK_POLICY_ENABLE (ne .DpuHostControlPlaneMode true) }}
enable-multi-networkpolicy=true
{{- end }}
```

**Self-Hosted ConfigMap** (`self-hosted/004-config.yaml`):
```ini
[ovnkubernetesfeature]
{{- if ne .DpuHostControlPlaneMode true }}
enable-egress-ip=true
enable-egress-firewall=true
enable-egress-qos=true
enable-egress-service=true
enable-multicast=true
{{- end }}

{{- if and .OVN_NETWORK_SEGMENTATION_ENABLE (ne .DpuHostControlPlaneMode true) }}
enable-network-segmentation=true
{{- end }}

{{- if and .OVN_MULTI_NETWORK_POLICY_ENABLE (ne .DpuHostControlPlaneMode true) }}
enable-multi-networkpolicy=true
{{- end }}
```

### 3. Control Plane Template Features

Features passed as command-line arguments to the control plane:

```yaml
# Both managed and self-hosted control plane templates
{{- if ne .DpuHostControlPlaneMode true }}
--enable-egress-ip=true \
--enable-egress-firewall=true \
--enable-egress-qos=true \
--enable-egress-service=true \
--enable-multicast \
--enable-multi-external-gateway=true \
{{- end }}
```

Script variables for control plane:
```bash
if [[ "{{.OVN_MULTI_NETWORK_ENABLE}}" == "true" && "{{.DpuHostControlPlaneMode}}" != "true" ]]; then
  multi_network_enabled_flag="--enable-multi-network"
fi

if [[ "{{.OVN_MULTI_NETWORK_POLICY_ENABLE}}" == "true" && "{{.DpuHostControlPlaneMode}}" != "true" ]]; then
  multi_network_policy_enabled_flag="--enable-multi-networkpolicy"
fi
```

### 4. Node Script Logic (Simplified)

**DPU Host Node-Specific Settings Only**:
```bash
# Handle ONLY DPU host node-specific settings (no feature disabling)
if [ "${OVN_NODE_MODE}" == "dpu-host" ]; then
  # DPU host specific gateway configuration
  gateway_interface="derive-from-mgmt-port"
  ovnkube_node_mode="--ovnkube-node-mode dpu-host"
  # Disable features not supported on DPU host hardware
  egress_features_enable_flag=""
  enable_multicast_flag=""
  init_ovnkube_controller=""
  multi_external_gateway_enable_flag=""
fi
```

**Feature Flags (Read from ConfigMap)**:
```bash
# These flags simply read the already-processed values from ConfigMaps
# ConfigMaps have already disabled features when DpuHostControlPlaneMode=true
multi_network_enabled_flag=
if [[ "{{.OVN_MULTI_NETWORK_ENABLE}}" == "true" && "${OVN_NODE_MODE}" != "dpu-host" ]]; then
  multi_network_enabled_flag="--enable-multi-network"
fi

network_segmentation_enabled_flag=
if [[ "{{.OVN_NETWORK_SEGMENTATION_ENABLE}}" == "true" && "${OVN_NODE_MODE}" != "dpu-host" ]]; then
  network_segmentation_enabled_flag="--enable-network-segmentation"
fi

multi_network_policy_enabled_flag=
if [[ "{{.OVN_MULTI_NETWORK_POLICY_ENABLE}}" == "true" && "${OVN_NODE_MODE}" != "dpu-host" ]]; then
  multi_network_policy_enabled_flag="--enable-multi-networkpolicy"
fi

admin_network_policy_enabled_flag=
if [[ "{{.OVN_ADMIN_NETWORK_POLICY_ENABLE}}" == "true" && "${OVN_NODE_MODE}" != "dpu-host" ]]; then
  admin_network_policy_enabled_flag="--enable-admin-network-policy"
fi
```

**Key Change**: Scripts no longer check `DpuHostControlPlaneMode` - they just read the flags that ConfigMaps have already processed!

## Feature Distribution

| Feature | ConfigMap | Control Plane Args | Node Scripts |
|---------|-----------|-------------------|--------------|
| Egress IP/Firewall/QoS/Service | ✅ | ✅ | ✅ |
| Multicast | ✅ (self-hosted only) | ✅ | ✅ |
| Multi-external Gateway | ❌ | ✅ | ✅ |
| Multi-network | ✅ | Script var | Script var |
| Network Segmentation | ✅ | Script var | Script var |
| Multi-network Policy | ✅ | Script var | Script var |
| Admin Network Policy | ❌ | Script var | Script var |

## Benefits of This Approach

### 1. **True Centralization**
- **Go Code**: Single place for DPU host detection
- **ConfigMaps**: Single place for ALL feature enabling/disabling decisions
- **Scripts**: No longer contain complex conditional logic

### 2. **Clean Layer Separation**
- **ConfigMap**: Controls ALL feature flags (single source of truth)
- **Templates**: Handle deployment-specific features  
- **Scripts**: Handle ONLY node-specific settings (gateway, mode, etc.)

### 3. **Simplified Logic Flow**
1. Go detects DPU hosts → sets `DpuHostControlPlaneMode=true`
2. ConfigMaps see flag → disable incompatible features in INI config
3. Scripts read already-processed flags → no conditional logic needed
4. Result: Consistent cluster behavior automatically

### 4. **Maintainability**
- **Adding new features**: Update ConfigMap templates only
- **No duplicate logic**: Feature disabling logic exists in one place
- **Easy debugging**: Check ConfigMap to see what features are enabled
- **Clean scripts**: Scripts focus on node settings, not feature decisions

## Cluster Behavior Matrix

| Scenario | Control Plane | DPU Host Nodes | Regular Nodes | Result |
|----------|--------------|----------------|---------------|---------|
| No DPU hosts | All features enabled | N/A | All features enabled | ✅ Full feature cluster |
| DPU hosts present | DPU-incompatible disabled | DPU-incompatible disabled | DPU-incompatible disabled | ✅ Consistent cluster |

## Features Disabled When DPU Hosts Present

- ❌ **Egress Features**: IP, Firewall, QoS, Service
- ❌ **Multicast Support**
- ❌ **Multi-external Gateway Support**
- ❌ **Multi-network Support**
- ❌ **Network Segmentation**
- ❌ **Multi-network Policies**
- ❌ **Admin Network Policies**

This implementation ensures cluster-wide consistency while maintaining clean architectural boundaries and making the system easily maintainable.
