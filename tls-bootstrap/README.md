# TLS Bootstrap (cert-manager)

Automates cert-manager installation, IRSA annotation, ClusterIssuer, and wildcard certificate creation.

## Prerequisites

- EKS cluster with OIDC provider configured
- `ingress-nginx` namespace exists in the cluster
- IRSA role for cert-manager created via Terraform in the `terraform-infra` repo (`cert-manager.tf`)

## Configuration

### Files to update (per domain)

| File | What to change |
|---|---|
| `clusterissuer.yaml` | `email` — your email for Let's Encrypt notifications |
| `wildcard-cert.yaml` | `dnsNames`, `secretName`, `metadata.name` — match your domain |

The `hostedZoneID` in `clusterissuer.yaml` uses a placeholder (`HOSTED_ZONE_PLACEHOLDER`) that gets replaced at runtime by the bootstrap script.

### GitHub repository variables

| Variable | Source | Description |
|---|---|---|
| `HOSTED_ZONE_ID` | `terraform output hosted_zone_id` | Route 53 hosted zone ID |
| `CERT_MANAGER_ROUTE53_ROLE_ARN` | `terraform output cert_manager_role_arn` | IRSA role ARN from terraform-infra repo |

### Workflow env block

```yaml
- name: Bootstrap TLS
  env:
    CLUSTERISSUER_NAME: letsencrypt-prod-dns
    HOSTED_ZONE_ID: ${{ vars.HOSTED_ZONE_ID }}
    CERT_MANAGER_ROUTE53_ROLE_ARN: ${{ vars.CERT_MANAGER_ROUTE53_ROLE_ARN }}
    CLUSTER_ISSUER_MANIFEST: tls-bootstrap/clusterissuer.yaml
    WILDCARD_CERT_MANIFEST: tls-bootstrap/wildcard-cert.yaml
    WILDCARD_SECRET_NS: ingress-nginx
    WILDCARD_SECRET_NAME: wildcard-team312-tls-secret
    WILDCARD_CERT_NAME: wildcard-team312-tls-cert
  run: ./tls-bootstrap/bootstrap-tls.sh
```

## What the script does

1. Installs cert-manager if not present
2. Annotates the cert-manager ServiceAccount with the IRSA role ARN
3. Creates the ClusterIssuer (DNS-01 challenge via Route 53)
4. Creates the wildcard Certificate and waits for it to be issued

The script is idempotent — safe to run multiple times.
