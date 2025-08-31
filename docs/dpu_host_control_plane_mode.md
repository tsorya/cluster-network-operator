# DPU Host Control Plane Mode

This feature automatically configures the OVN Kubernetes control plane to disable features that are not supported in DPU host environments when DPU host nodes are detected in the cluster.

## Overview

When DPU (Data Processing Unit) host nodes are present in the cluster, certain networking features cannot be supported on those nodes. Previously, these features were only disabled on the DPU host nodes themselves via the daemonset configuration, but the control plane continued to operate in "full" mode.

This enhancement introduces automatic detection of DPU host nodes and configures the control plane to disable the same features that are disabled on DPU host nodes, ensuring consistency across the cluster.

## Features Disabled in DPU Host Control Plane Mode

When DPU host control plane mode is enabled, the following features are automatically disabled:

### Egress Features
- Egress IP (`--enable-egress-ip=false`)
- Egress Firewall (`--enable-egress-firewall=false`) 
- Egress QoS (`--enable-egress-qos=false`)
- Egress Service (`--enable-egress-service=false`)

### Network Features  
- Multicast support (not enabled)
- Multi-external gateway support (not enabled)
- Multi-network support (`--enable-multi-network` not set)
- Network segmentation (`--enable-network-segmentation` not set)
- Multi-network policies (`--enable-multi-networkpolicy` not set)
- Admin network policies (`--enable-admin-network-policy` not set)

## Implementation Details

### Bootstrap Configuration

The `DpuHostControlPlaneMode` flag is added to the `OVNConfigBoostrapResult` struct and automatically set to `true` when any DPU host nodes are detected in the cluster:

```go
// Enable DPU host mode for control plane if any DPU host nodes are present
ovnConfigResult.DpuHostControlPlaneMode = len(ovnConfigResult.DpuHostModeNodes) > 0
```

### Template Modifications

Both self-hosted and managed (hypershift) control plane deployments are updated to conditionally disable features:

#### Command Line Flags
```bash
{{- if ne .DpuHostControlPlaneMode true }}
--enable-egress-ip=true \
--enable-egress-firewall=true \
--enable-egress-qos=true \
--enable-egress-service=true \
--enable-multicast \
--enable-multi-external-gateway=true \
{{- end }}
```

#### Script Variables
```bash
if [[ "{{.OVN_MULTI_NETWORK_ENABLE}}" == "true" && "{{.DpuHostControlPlaneMode}}" != "true" ]]; then
  multi_network_enabled_flag="--enable-multi-network"
fi
```

#### INI Configuration
```ini
{{- if and .OVN_MULTI_NETWORK_ENABLE (ne .DpuHostControlPlaneMode true) }}
enable-multi-network=true
{{- end }}
```

### Control Plane Regeneration

When DPU host nodes are detected, the control plane deployment will be regenerated with the appropriate feature flags disabled, ensuring that:

1. The control plane doesn't attempt to configure unsupported features
2. Cluster behavior is consistent between control plane and DPU host nodes
3. No conflicts arise from mismatched feature configurations

### Logging

The system logs when DPU host control plane mode is enabled:

```
DPU host control plane mode enabled due to presence of N DPU host nodes
```

## Files Modified

- `pkg/bootstrap/types.go` - Added `DpuHostControlPlaneMode` field
- `pkg/network/ovn_kubernetes.go` - Added logic to detect DPU hosts and enable control plane mode
- `bindata/network/ovn-kubernetes/managed/ovnkube-control-plane.yaml` - Template updates for managed clusters
- `bindata/network/ovn-kubernetes/self-hosted/ovnkube-control-plane.yaml` - Template updates for self-hosted clusters  
- `bindata/network/ovn-kubernetes/managed/004-config.yaml` - INI config updates for managed clusters
- `bindata/network/ovn-kubernetes/self-hosted/004-config.yaml` - INI config updates for self-hosted clusters

## Compatibility

This feature is backward compatible and only activates when DPU host nodes are present. Clusters without DPU host nodes continue to operate with full feature support as before.

The feature works with both:
- Self-hosted OpenShift clusters
- Managed (HyperShift) clusters
