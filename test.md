# Prometheus Adapter Helm Chart Security Review Report

## Executive Summary

This report presents the findings of a comprehensive security review of the prometheus-adapter Helm chart. The review focused on identifying potential security vulnerabilities, evaluating RBAC configurations, and providing actionable recommendations to improve the security posture of the deployment.

## Review Methodology

The security review followed a systematic approach, examining all aspects of the Helm chart that could impact the security of deployed resources:

1. Chart structure and metadata analysis
2. Container security settings assessment
3. RBAC permissions evaluation
4. Secret management practices review
5. Network security configuration analysis
6. Pod security policy verification
7. Certificate management assessment
8. High availability and resilience controls review

## Security Review Checklist

| # | Check | Status | Notes |
|---|-------|--------|-------|
| 1 | Chart version and dependencies verification | ✅ | v0.12.0, no vulnerable dependencies identified |
| 2 | Container runs as non-root | ✅ | Uses UID 10001 |
| 3 | ReadOnlyRootFilesystem enabled | ✅ | Properly configured in securityContext |
| 4 | Privilege escalation controls | ✅ | allowPrivilegeEscalation: false |
| 5 | Capability restrictions | ✅ | capabilities.drop: ["ALL"] |
| 6 | Security context configuration | ✅ | Both pod and container security contexts set |
| 7 | Secure TLS implementation | ⚠️ | TLS disabled by default; insecureSkipTLSVerify: true when disabled |
| 8 | RBAC principle of least privilege | ⚠️ | Uses wildcard resources in some roles |
| 9 | Resource limits defined | ⚠️ | Not set by default, empty resources: {} |
| 10 | Pod Security Policy | ✅ | Available but disabled by default (psp.create: false) |
| 11 | Secret management | ✅ | Properly stores TLS credentials, read-only mounts |
| 12 | Network security | ⚠️ | hostNetwork disabled by default, but HTTP used for Prometheus URL |
| 13 | PodDisruptionBudget | ✅ | Available but disabled by default |
| 14 | Certificate management | ⚠️ | cert-manager integration available but long certificate duration |
| 15 | ServiceAccount controls | ✅ | Custom ServiceAccount with appropriate permissions |
| 16 | Health probes | ✅ | Properly configured liveness and readiness probes |
| 17 | Seccomp profile | ✅ | Set to RuntimeDefault |

## Detailed Findings

### Container Security

The prometheus-adapter container configuration demonstrates strong security practices:

```yaml
securityContext:
  allowPrivilegeEscalation: false
  capabilities:
    drop: ["ALL"]
  readOnlyRootFilesystem: true
  runAsNonRoot: true
  runAsUser: 10001
  seccompProfile:
    type: RuntimeDefault
```

These settings ensure the container:
- Runs as a non-root user (UID 10001)
- Cannot escalate privileges
- Has all capabilities dropped
- Uses a read-only root filesystem
- Utilizes the default seccomp profile

### RBAC Configuration

The chart creates several roles and cluster roles with specific permissions. Most follow the principle of least privilege, but some use wildcards that should be reviewed:

```yaml
rbac:
  externalMetrics:
    resources: ["*"]
  customMetrics:
    resources: ["*"]
```

| Role/ClusterRole Name | Type | Permissions | Purpose |
|---|---|---|---|
| system:auth-delegator | ClusterRole | Auth delegation | Authentication delegation |
| extension-apiserver-authentication-reader | Role/ClusterRole | Read auth configurations | Authentication configuration access |
| resource-reader | ClusterRole | get, list, watch on namespaces, pods, services, configmaps | Resource reading for adapter |
| custom-metrics-server-resources | ClusterRole | All operations on custom.metrics.k8s.io resources | Custom metrics management |
| external-metrics | ClusterRole | list, get, watch on external.metrics.k8s.io resources | External metrics access |
| metrics | ClusterRole | get, list, watch on pods, nodes, nodes/stats | Resource metrics access |
| psp | ClusterRole | use podsecuritypolicies | PSP usage when enabled |

### TLS and Certificate Management

The chart provides three options for TLS:

1. Disabled (default) - Uses insecureSkipTLSVerify: true
2. Manual TLS configuration - Requires providing certificate, key, and CA
3. cert-manager integration - Automatically generates certificates

When using cert-manager, certificate duration is set to 8760h (1 year) by default, which may be too long for some security policies.

### Pod Security Policy

The chart includes a Pod Security Policy that is disabled by default:

```yaml
psp:
  create: false
  annotations: {}
```

When enabled, it enforces:
- User must run as UID 1024-65535
- Restricted volume types
- Appropriate seLinux, fsGroup, and supplementalGroups settings

### Network Security

- Default deployment uses ClusterIP service type
- hostNetwork setting is disabled by default
- Default prometheus URL uses HTTP, not HTTPS

## Security Recommendations

Based on the review findings, the following recommendations are provided to enhance the security posture:

### Critical Recommendations

1. **Enable TLS or cert-manager**
   - Set `tls.enable: true` or `certManager.enabled: true`
   - Eliminate the use of `insecureSkipTLSVerify: true`

2. **Define Explicit Resource Limits**
   - Set appropriate CPU and memory limits and requests
   - Prevents resource exhaustion attacks

3. **Use HTTPS for Prometheus URL**
   - Update `prometheus.url` to use HTTPS instead of HTTP
   - Ensures secure communication with Prometheus

### Important Recommendations

4. **Restrict RBAC Wildcards**
   - Replace `resources: ["*"]` with specific resources in custom and external metrics roles
   - Follows principle of least privilege

5. **Reduce Certificate Duration**
   - Set shorter durations for certificates (e.g., 2160h for 90 days)
   - Ensures regular certificate rotation

6. **Enable Pod Security Policy** (if your cluster supports it)
   - Set `psp.create: true`
   - Enhances pod-level security controls

### Additional Recommendations

7. **Enable PodDisruptionBudget for Production**
   - Set `podDisruptionBudget.enabled: true`
   - Ensures availability during cluster maintenance

8. **Implement NetworkPolicy Resources**
   - Add NetworkPolicy objects to restrict traffic
   - Only allow necessary communication paths

9. **Review and Set Appropriate Probe Parameters**
   - Adjust timeouts and initial delays based on application behavior
   - Prevents false positives in health checks

10. **Consider Setting runAsGroup**
    - Add explicit runAsGroup to match runAsUser
    - Provides additional control over process permissions

## Implementation Example

The following values.yaml snippet demonstrates implementing the key recommendations:

```yaml
# Security-enhanced values.yaml
securityContext:
  allowPrivilegeEscalation: false
  capabilities:
    drop: ["ALL"]
  readOnlyRootFilesystem: true
  runAsNonRoot: true
  runAsUser: 10001
  seccompProfile:
    type: RuntimeDefault

# Enable TLS with cert-manager
certManager:
  enabled: true
  certDuration: 2160h0m0s  # 90 days

# Restrict resources
rbac:
  externalMetrics:
    resources: ["pods", "deployments", "statefulsets"]  # Specific resources instead of "*"
  customMetrics:
    resources: ["pods", "deployments", "statefulsets"]  # Specific resources instead of "*"

# Set resource limits
resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 200m
    memory: 256Mi

# Enable Pod Disruption Budget
podDisruptionBudget:
  enabled: true
  minAvailable: 1

# Use HTTPS for Prometheus
prometheus:
  url: https://prometheus.default.svc
  port: 9090
```

## Conclusion

The prometheus-adapter Helm chart provides a solid foundation for secure deployment with numerous security controls available. However, many security features are disabled by default, requiring explicit configuration to achieve a robust security posture. By implementing the recommendations in this report, organizations can significantly enhance the security of their prometheus-adapter deployments.

Regular review of the chart's security settings is recommended as new versions are released and as security best practices evolve.
