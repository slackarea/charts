# OpenLDAP High Availability Helm Chart

A production-ready Helm chart for deploying OpenLDAP in high availability mode with multi-master replication on Kubernetes.

## Features

- **High Availability**: Multi-master replication with configurable replica count
- **TLS/SSL Support**: Automatic certificate management via cert-manager
- **phpLDAPadmin**: Optional web-based administration interface
- **Persistence**: Persistent storage for data and configuration
- **Pod Disruption Budget**: Ensures minimum availability during maintenance
- **Anti-Affinity**: Spreads pods across different nodes
- **Traefik Integration**: LDAPS external access via IngressRouteTCP

## Prerequisites

- Kubernetes 1.19+
- Helm 3.0+
- cert-manager (for TLS certificates)
- Traefik ingress controller (for LDAPS and phpLDAPadmin ingress)
- Persistent Volume provisioner support in the underlying infrastructure

## Installation

### Quick Start

```bash
# Add the repository (if hosted)
helm repo add openldap-ha https://slackarea.github.io/charts/
helm repo update

# Install with default values
helm install my-ldap openldap-ha/openldap-ha --namespace authspace --create-namespace
```

### Installation with Custom Values

Create a `custom-values.yaml` file:

```yaml
namespace: production

openldap:
  organization: "MyCompany"
  domain: "mycompany.com"
  adminPassword: "StrongPassword123!"
  configPassword: "ConfigPassword456!"
  
  replication:
    replicas: 5
  
  tls:
    certificate:
      issuerName: "letsencrypt-prod"

phpldapadmin:
  ingress:
    hostname: "ldap.mycompany.com"
```

Install with custom values:

```bash
helm install my-ldap openldap-ha/openldap-ha \
  --namespace production \
  --create-namespace \
  -f custom-values.yaml
```

## Configuration

### Essential Parameters

| Parameter | Description | Default |
|-----------|-------------|---------|
| `namespace` | Kubernetes namespace | `authspace` |
| `openldap.organization` | LDAP organization name | `MyOrganization` |
| `openldap.domain` | LDAP domain | `example.com` |
| `openldap.adminPassword` | Admin password | `changeme` |
| `openldap.configPassword` | Config password | `changeme` |
| `openldap.replication.replicas` | Number of replicas | `3` |

### Advanced Configuration

#### Port Configuration

You can customize LDAP ports if needed:

```yaml
openldap:
  ports:
    ldap: 389
    ldaps: 636

service:
  ldapPort: 389
  ldapsPort: 636
```

#### Storage Configuration

```yaml
openldap:
  persistence:
    data:
      enabled: true
      size: 20Gi
      storageClass: "fast-ssd"
    config:
      enabled: true
      size: 2Gi
      storageClass: "fast-ssd"
```

#### Resource Limits

```yaml
openldap:
  resources:
    requests:
      memory: "2Gi"
      cpu: "1000m"
    limits:
      memory: "4Gi"
      cpu: "4000m"
```

#### TLS Configuration

```yaml
openldap:
  tls:
    enabled: true
    enforce: true  # Require TLS for all connections
    certificate:
      enabled: true
      issuerName: "letsencrypt-prod"
      issuerKind: "ClusterIssuer"
      duration: "2160h"
      renewBefore: "360h"
      subject:
        organizations:
          - "MyCompany Inc"
```

#### Disable phpLDAPadmin

```yaml
phpldapadmin:
  enabled: false
```

## Usage Examples

### Connect from Application

```yaml
# Example application configuration
ldap:
  url: "ldap://my-ldap-openldap-ha.production.svc.cluster.local:389"
  baseDN: "dc=mycompany,dc=com"
  bindDN: "cn=admin,dc=mycompany,dc=com"
  bindPassword: "StrongPassword123!"
```

### LDAP Search Command

```bash
ldapsearch -x \
  -H ldap://my-ldap-openldap-ha.production.svc.cluster.local:389 \
  -b "dc=mycompany,dc=com" \
  -D "cn=admin,dc=mycompany,dc=com" \
  -w "StrongPassword123!"
```

### Add User Entry

```bash
ldapadd -x \
  -H ldap://my-ldap-openldap-ha.production.svc.cluster.local:389 \
  -D "cn=admin,dc=mycompany,dc=com" \
  -w "StrongPassword123!" \
  -f user.ldif
```

## Upgrading

```bash
helm upgrade my-ldap openldap-ha/openldap-ha \
  --namespace production \
  -f custom-values.yaml
```

## Uninstalling

```bash
# This will NOT delete PVCs by default
helm uninstall my-ldap --namespace production

# To also delete PVCs
kubectl delete pvc -n production -l app.kubernetes.io/instance=my-ldap
```

## Troubleshooting

### Check Pod Status

```bash
kubectl get pods -n production -l app=openldap
```

### View Pod Logs

```bash
kubectl logs -n production my-ldap-openldap-ha-0 -f
```

### Check Replication Status

```bash
kubectl exec -n production my-ldap-openldap-ha-0 -- \
  ldapsearch -Y EXTERNAL -H ldapi:/// -b "cn=config" "(olcSyncrepl=*)"
```

### Test Connection

```bash
kubectl run ldap-test --rm -it --restart=Never \
  --image=osixia/openldap:1.5.0 -- \
  ldapsearch -x -H ldap://my-ldap-openldap-ha.production.svc.cluster.local:389 \
  -b "dc=mycompany,dc=com" -D "cn=admin,dc=mycompany,dc=com" -w "password"
```

### Common Issues

**Pods not starting**: Check storage class availability and PVC binding
```bash
kubectl get pvc -n production
```

**Certificate issues**: Verify cert-manager is installed and ClusterIssuer exists
```bash
kubectl get clusterissuer
kubectl get certificate -n production
```

**Replication not working**: Check network policies and service discovery
```bash
kubectl exec -n production my-ldap-openldap-ha-0 -- \
  nslookup my-ldap-openldap-ha-1.my-ldap-openldap-ha-headless.production.svc.cluster.local
```

## Multi-Domain Deployment

You can deploy multiple independent LDAP instances for different domains:

```bash
# Domain 1
helm install ldap-domain1 openldap-ha/openldap-ha \
  --namespace domain1 --create-namespace \
  --set openldap.domain=domain1.com \
  --set openldap.organization="Domain1 Corp"

# Domain 2
helm install ldap-domain2 openldap-ha/openldap-ha \
  --namespace domain2 --create-namespace \
  --set openldap.domain=domain2.com \
  --set openldap.organization="Domain2 Inc"
```

## Security Considerations

1. **Change default passwords** immediately in production
2. **Enable TLS enforcement** for production environments
3. **Use strong passwords** and consider external secret management (e.g., Sealed Secrets, External Secrets Operator)
4. **Restrict network access** using NetworkPolicies
5. **Regular backups** of persistent volumes
6. **Monitor access logs** for suspicious activity

## Contributing

Contributions are welcome! Please open an issue or submit a pull request.

## License

GNU General Public License v3.0

## Support

For issues and questions:
- GitHub Issues: [https://github.com/vcnngr/helm-openldap]
- Documentation: [https://github.com/vcnngr/helm-openldap]