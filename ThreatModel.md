## Scope
Hyperliquid non-validator node on EKS (2 AZs, Tokyo), GitOps via ArgoCD, ESO for secrets.

## Key Assets
- Signing keys (if ever run validator), API/Slack keys
- Node binaries integrity (hl-visor/hl-node)
- Live L1/EVM data & local snapshots
- Cluster network paths to peers and APIs

## Threats & Mitigations
1) Supply chain / binary tampering
   - **Mitigation (Idea)**: GPG-verify `hl-visor` at startup; `hl-visor` verifies `hl-node`. Pin key/fingerprint when published; use immutable artifact URLs. *(InitContainer code included.)*  
   - Further: pin container base digests, enable image scanning in CI (e.g., Trivy GitHub Action).

2) Excessive network exposure / lateral movement
   - **Mitigation**: Namespace-scoped default-deny **NetworkPolicy**, no public Service, allow ingress only from monitoring if needed.
3) Secret sprawl in Git/CI
   - **Mitigation**: Use External Secrets Operator (ESO) with AWS Secrets Manager via IRSA; ArgoCD never sees cleartext; enforce least-privilege IAM for ESO.

4) ArgoCD pipeline & Git integrity
   - **Mitigation**: Use ArgoCD Projects, repo-server without service account token, signed commits/tags, branch protections; ArgoCD declarative setup. 

5) Data loss / IO saturation
   - **Mitigation**: Right-size gp3 IOPS/throughput; rotate & archive logs; consider dedicated storage-optimized instances if you ever move off EBS. Monitor NVMe/EBS stats.
