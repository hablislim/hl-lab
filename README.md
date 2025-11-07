# Hyperliquid node on Kubernetes (GitOps)

## Prereqs
- EKS (Tokyo `ap-northeast-1`) with two worker nodes in two AZs
- ArgoCD installed
- (Optional) External Secrets Operator (ESO) for secrets
- Build & push hyperlequid image (from officel github repo) to Aneralabs image registry.

## Deploy
<code> kubectl apply -n argocd -f argocd/app-hyperliquid-node.yaml </code>

This installs a StatefulSet with 2 replicas using the Helm libraries  **bjw-s-labs/common**.

## Storage
Each pod gets a 2Tb `gp3` volume (`statefulset.volumeClaimTemplates`). Increase if enabling verbose `--write-*` flags (logs can be ~100 GB/day). For highest throughput, tune gp3 IOPS/throughput.

## Performance
- AZ spreading: `topologySpreadConstraints` + `podAntiAffinity`
- Node pinning: `nodeAffinity` to Tokyo AZs
- CPU/RAM reserved via `Guaranteed QoS (requests == limits)`
- Use NLB for Hyperliquid’s P2P TCP ports (4001–4002) to get lower latency, fewer hops, and true end-to-end peer connections.
- Preserves client IP (externalTrafficPolicy: Local)

  
## Ops & Monitoring
- Liveness/readiness probes
- EVM/Info server on :3001 (local-only Service by default)
- Scrape logs for consensus/latency signals as per Hyperliquid docs
- Consider CloudWatch Agent / node-exporter for ENA & disk stats

## Security
- Pod runs as non-root, read-only root FS
- NetworkPolicy: default-deny ingress, minimal allow
- Use ESO to pull secrets (API keys) from AWS Secrets Manager (ArgoCD doesn’t see cleartext)
- Bonus: add an Init containers securely fetch seed peers via the Hyperliquid Info API, download the hl-visor binary, and verify its GPG signature on startup. Ensuring a fully trusted execution chain.

## Tuning tips (AWS/EKS)
- Prefer AWS VPC CNI for lowest overhead (no overlay)
- Use **gp3** with sufficient IOPS/throughput; pre-warm volumes if needed
- Ensure EKS service controller creates NLB (not ALB) (service.beta.kubernetes.io/aws-load-balancer-type: "nlb")
- Run worker nodes on ENA-enabled high-network-performance instances (e.g., c6i, m6i, r6i, c7i)
- Keep pods on the same region; cross-AZ adds latency (trade-off vs resilience)
- Place nodes in a cluster placement group (same rack / network switch) for lower inter-AZ latency trade-offs 
- Bonus:
        
        
        1- Enable TCP keep-alive
        Keeps long-lived TCP peer connections active even when idle.
        Prevents the connection from timing out or being dropped by NAT/firewalls.
        sysctl:
            net.ipv4.tcp_keepalive_time = 60
            net.ipv4.tcp_keepalive_intvl = 30
            net.ipv4.tcp_keepalive_probes = 5

            => Sends a small keep-alive packet every 30–60 s to verify peers are alive.


        2- Tune OS buffers
        Increase kernel socket send/receive buffers for high-throughput and stable latency:
        sysctl:
            net.core.rmem_max = 134217728
            net.core.wmem_max = 134217728
            net.ipv4.tcp_rmem = 4096 87380 134217728
            net.ipv4.tcp_wmem = 4096 65536 134217728

            => Prevents throttling under heavy peer traffic.

        3- Disable unnecessary packet inspection / firewall hops
        Avoid inline systems like NAT gateways, IDS/IPS, or deep packet inspection between nodes and the Internet.


## Uninstall
argocd app delete hyperliquid-node -n argocd
