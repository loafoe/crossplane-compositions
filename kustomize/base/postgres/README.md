# Postgres Crossplane Composition

This Crossplane v2 composition provides a complete solution for **PostgreSQL** databases, supporting both **AWS RDS** and **CloudNativePG** on Kubernetes. It abstracts the underlying provider details, allowing for seamless switching between cloud-managed and Kubernetes-native databases.

## Features

- ✅ **Namespace-scoped** composite resources (no claims required)
- ✅ **Multi-Provider Support**: Switch between `aws` and `cnpg` backends
- ✅ **T-Shirt Sizing**: Simple `xsmall`, `small`, `medium`, `large` abstraction
- ✅ **Create new AWS RDS/Aurora** instances with full configuration
- ✅ **Deploy CloudNativePG Clusters** on Kubernetes
- ✅ **Multi-AZ deployment** support (AWS Multi-AZ / CNPG Replicas)
- ✅ **Automatic backups** with configurable retention
- ✅ **Encryption at rest** enforced by default
- ✅ **Connection secret output** with endpoint, port, username and password
- ✅ **Resource tagging** support
- ✅ **VPA Support** (CNPG): Optional Vertical Pod Autoscaler for automatic resource right-sizing

## Prerequisites

### 1. Crossplane Providers and Functions

The composition requires the following Crossplane providers and functions:

- `function-environment-configs` v0.4.0+
- `function-go-templating` v0.11.0+
- `function-patch-and-transform` v0.9.1+
- `function-auto-ready` v0.5.1+
- `provider-aws-rds` v2.2.0+

### 2. Environment Configuration

The composition expects an `EnvironmentConfig` with the following fields:

```yaml
apiVersion: apiextensions.crossplane.io/v1beta1
kind: EnvironmentConfig
metadata:
  name: hsp-addons
  labels:
    config: hsp-addons
data:
  accountId: "123456789012"
  region: "us-east-1"
  clusterName: "my-cluster"
  eks:
    oidcProvider: "oidc.eks.us-east-1.amazonaws.com/id/EXAMPLE"
    oidcProviderArn: "arn:aws:iam::123456789012:oidc-provider/oidc.eks.us-east-1.amazonaws.com/id/EXAMPLE"
```

## Usage Examples

### Connection Secret Schema

Both `aws` and `cnpg` providers verify and populate the same connection secret schema, ensuring application portability.

| Key | Description | Example |
|-----|-------------|---------|
| `host` | Database hostname | `myapp-db.cluster-ro.abcdef.us-east-1.rds.amazonaws.com` |
| `port` | Database port | `5432` |
| `username` | Connection username | `postgres` or `app_user` |
| `password` | Connection password | `secure_random_string` |
| `database` | Database name | `app` |
| `sslmode` | SSL connection mode | `require` |
| `endpoint` | Hostname + Port | `myapp-db...:5432` |

### Example 4: Unified Application Deployment

Since the secret structure is identical for both providers, you can use the same application manifest for both.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  containers:
  - name: app
    image: my-app:latest
    env:
    - name: PGHOST
      valueFrom:
        secretKeyRef:
          name: myapp-db-connection
          key: host
    - name: PGPORT
      valueFrom:
        secretKeyRef:
          name: myapp-db-connection
          key: port
    - name: PGUSER
      valueFrom:
        secretKeyRef:
          name: myapp-db-connection
          key: username
    - name: PGPASSWORD
      valueFrom:
        secretKeyRef:
          name: myapp-db-connection
          key: password
    - name: PGDATABASE
      valueFrom:
        secretKeyRef:
          name: myapp-db-connection
          key: database
    - name: PGSSLMODE
      valueFrom:
        secretKeyRef:
          name: myapp-db-connection
          key: sslmode
```

### Database Creation Prerequisites

Before creating a new database, you need:

1. **Environment Configuration**: An `EnvironmentConfig` resource with VPC and security group details (automatically provided by the platform)

The composition automatically extracts VPC configuration from the `EnvironmentConfig`:
- Database subnet IDs from `servicesVpc.subnetGroups.database.subnet_ids`
- Database security group from `servicesVpc.securityGroups.database.id`

### Example 1: Create Development Database

```yaml
apiVersion: dip.io/v1alpha1
kind: Postgres
metadata:
  name: myapp-dev-db
  namespace: myapp
spec:
  database:
    identifier: myapp-development-db
    engineVersion: "18.1"
    databaseName: myapp_dev
    
    # Instance configuration
    size: small
    allocatedStorage: 20
    storageType: gp3
    
    # Master credentials
    masterUsername: postgres
    
    # Backup and HA
    backupRetentionPeriod: 7
    multiAz: false
  
  tags:
    Environment: development
    Team: platform
```

### Example 2: Create Production Database

```yaml
apiVersion: dip.io/v1alpha1
kind: Postgres
metadata:
  name: myapp-prod-db
  namespace: myapp
spec:
  database:
    identifier: myapp-prod-db
    engineVersion: "18.1"
    databaseName: myapp_production
    
    # Production instance
    size: large
    allocatedStorage: 100
    storageType: gp3
    
    # Master credentials
    masterUsername: postgres
  
    # Production settings
    backupRetentionPeriod: 30
    multiAz: true

  writeConnectionSecretToRef:
    name: myapp-prod-db-connection
  
  tags:
    Environment: production
    CostCenter: engineering
```

### Example 3: Create Aurora PostgreSQL Cluster

```yaml
apiVersion: dip.io/v1alpha1
kind: Postgres
metadata:
  name: myapp-aurora
  namespace: myapp
spec:
  database:
    identifier: myapp-aurora-prod
    type: aurora-cluster
    engineVersion: "18.1"
    databaseName: myapp_data

    # Master credentials
    masterUsername: postgres
 
    # Backup settings
    backupRetentionPeriod: 14
  
  writeConnectionSecretToRef:
    name: myapp-aurora-connection
  
```

### Configuration Options

| Parameter | Required | Default | Description |
|-----------|----------|---------|-------------|
| `identifier` | Yes | - | Name of the database resource |
| `provider` | No | `aws` | Backend provider: `aws` or `cnpg` |
| `size` | No | `small` | T-shirt size: `xsmall`, `small`, `medium`, `large` |
| `engineVersion` | Yes | - | PostgreSQL engine version (e.g., '18.1', '16.1') |
| `allocatedStorage` | Yes | - | Storage in GB (minimum 20 for AWS, 5 for CNPG) |
| `storageType` | No | `gp3` | Storage type: gp2, gp3, io1 |
| `masterUsername` | Yes | - | Master database username |
| `masterPasswordSecretRef` | No | - | Reference to password secret (auto-generated if not provided) |
| `backupRetentionPeriod` | No | `7` | Backup retention days (0-35) |
| `multiAz` | No | `false` | Enable Multi-AZ deployment |
| `vpa.enabled` | No | `false` | Enable VPA for automatic resource right-sizing (CNPG only) |
| `vpa.updateMode` | No | `Off` | VPA update mode: `Off`, `Initial`, or `Auto` |

**Note:** VPC configuration (subnets and security groups) is automatically extracted from the `environmentConfig` resource. The composition uses the database subnet group and database security group defined in the environment configuration.

### T-Shirt Sizing

The `size` parameter provides a simple abstraction for resource allocation. Choose based on your workload requirements:

| Size | CPU Request | CPU Limit | Memory Request | Memory Limit | Use Case |
|------|-------------|-----------|----------------|--------------|----------|
| `xsmall` | 100m | 500m | 128Mi | 256Mi | Low-traffic databases, dev/test, sidecars (Grafana DB, Dex DB) |
| `small` | 500m | 1000m | 512Mi | 1Gi | Standard workloads, small applications |
| `medium` | 2 | 2 | 4Gi | 4Gi | Medium traffic, production workloads |
| `large` | 4 | 4 | 16Gi | 16Gi | High traffic, large datasets, analytics |

**AWS RDS Mapping**: For AWS provider, sizes map to instance classes:
- `small` → `db.t3.small`
- `medium` → `db.t3.medium`
- `large` → `db.m5.large`

### Vertical Pod Autoscaler (VPA) Support

For CNPG databases, you can enable VPA to automatically right-size your database resources based on actual usage:

```yaml
apiVersion: dip.io/v1alpha1
kind: Postgres
metadata:
  name: myapp-db
  namespace: default
spec:
  crossplane:
    compositionSelector:
      matchLabels:
        provider: cnpg
  parameters:
    size: xsmall
    masterUsername: postgres
    identifier: myapp
    vpa:
      enabled: true
      updateMode: "Off"  # Recommendations only
```

**Update Modes:**
| Mode | Behavior |
|------|----------|
| `Off` | VPA provides recommendations only; no automatic changes |
| `Initial` | VPA sets resources only when pods are created |
| `Auto` | VPA automatically updates pod resources (may cause restarts) |

**Recommendation:** Start with `updateMode: "Off"` to observe recommendations before enabling automatic updates. View recommendations with:

```bash
kubectl get vpa -n <namespace> -o custom-columns='NAME:.metadata.name,TARGET_MEM:.status.recommendation.containerRecommendations[0].target.memory,TARGET_CPU:.status.recommendation.containerRecommendations[0].target.cpu'
```

## Connecting to the Database

### Using Golang (pgx)

#### Standard Connection (Unified)
If you rely on the connection secret (as shown in the [Unified Application Deployment](#example-4-unified-application-deployment) example), `pgx` will automatically use the standard environment variables (`PGHOST`, `PGUSER`, `PGPASSWORD`, etc.).

```go
package main

import (
	"context"
	"fmt"
	"os"

	"github.com/jackc/pgx/v5"
)

func main() {
    // Connect using environment variables (PGHOST, PGUSER, PGPASSWORD, etc.)
    conn, err := pgx.Connect(context.Background(), "") // Empty string = use env vars
    if err != nil {
        fmt.Fprintf(os.Stderr, "Unable to connect to database: %v\n", err)
        os.Exit(1)
    }
    defer conn.Close(context.Background())

    var version string
    err = conn.QueryRow(context.Background(), "SELECT version()").Scan(&version)
    if err != nil {
        fmt.Fprintf(os.Stderr, "QueryRow failed: %v\n", err)
        os.Exit(1)
    }

    fmt.Println(version)
}
```

### Status Fields

After creating a Postgres resource, you can check its status:

```bash
kubectl get postgres myapp-db-access -n myapp -o yaml
```

Status fields include:
- `dbInstanceIdentifier`: RDS instance or Aurora cluster identifier
- `dbResourceId`: RDS resource ID
The connection secret will contain:
- `endpoint`: Database endpoint hostname
- `port`: Database port
- `database`: Database name
- `username`: Database username
- `password`: Database password

## Troubleshooting

### Connection Refused

**Problem**: Can't connect to the database

**Solutions**:
1. Check security groups allow traffic from EKS nodes
2. Verify database is in same VPC or has proper networking
3. Check RDS instance status: `aws rds describe-db-instances --db-instance-identifier <name>`

## Testing

Run tests using the Makefile:

```bash
# Run all tests (validate + render)
make test

# Validate XRD and composition
make validate

# Render all examples
make render

# Clean up test artifacts
make clean
```

## Advanced Configuration

### Explicit Resource ID

If you know the RDS resource ID, provide it to avoid lookup:

```yaml
spec:
  database:
    identifier: myapp-db
    resourceId: db-ABCDEFGHIJKLMNOP123456  # Get from AWS console or CLI
```

To find resource ID:
```bash
aws rds describe-db-instances --db-instance-identifier myapp-db \
  --query 'DBInstances[0].DbiResourceId' --output text
```

### Custom Provider Config

Use a different Crossplane provider config:

```yaml
spec:
  providerConfigRef:
    name: custom-aws-config
    kind: ProviderConfig
```

## Backup and Restore (CNPG)

CloudNativePG supports two backup methods:
1. **VolumeSnapshot**: Fast, storage-level snapshots for quick recovery
2. **Barman Object Store (S3)**: Continuous WAL archiving for point-in-time recovery

### Barman Backup to S3 (Recommended for Production)

Barman provides continuous WAL archiving to S3, enabling point-in-time recovery (PITR) and protection against data loss from node failures.

#### Enabling Barman Backup

```yaml
apiVersion: dip.io/v1alpha1
kind: Postgres
metadata:
  name: myapp-db
  namespace: myapp
spec:
  crossplane:
    compositionSelector:
      matchLabels:
        provider: cnpg
  parameters:
    identifier: myapp-db
    masterUsername: myapp
    databaseName: myapp
    size: small
    barmanBackup:
      enabled: true
      retentionDays: 7           # How long to keep backups
      compression: gzip          # gzip, bzip2, snappy, or none
      # bucketName: custom-bucket  # Optional: defaults to ${resourcePrefix}-barman
      # bucketPrefix: custom/path  # Optional: defaults to ${namespace}/${identifier}
  writeConnectionSecretToRef:
    name: myapp-db-connection
```

This automatically:
1. Creates an IAM Role with S3 access via IRSA
2. Configures continuous WAL archiving to S3
3. Creates a `ScheduledBackup` for daily base backups at midnight

#### Verifying Barman Backup is Working

```bash
# Check continuous archiving status
kubectl get cluster myapp-db -n myapp -o jsonpath='{.status.conditions}' | jq '.[] | select(.type=="ContinuousArchiving")'

# Check WAL files in S3
aws s3 ls s3://${BUCKET}/${NAMESPACE}/${IDENTIFIER}/${IDENTIFIER}/wals/ --recursive

# Check base backups
aws s3 ls s3://${BUCKET}/${NAMESPACE}/${IDENTIFIER}/${IDENTIFIER}/base/

# List available backups from barman
kubectl exec -n myapp myapp-db-1 -- barman-cloud-backup-list \
  --cloud-provider aws-s3 \
  s3://${BUCKET}/${NAMESPACE}/${IDENTIFIER} \
  ${IDENTIFIER}
```

#### Creating On-Demand Barman Backup

```yaml
apiVersion: postgresql.cnpg.io/v1
kind: Backup
metadata:
  name: myapp-db-barman-manual
  namespace: myapp
spec:
  cluster:
    name: myapp-db
  method: barmanObjectStore
```

### Restoring from Barman Backup (S3)

To restore a database from Barman S3 backup (e.g., after data loss or for disaster recovery):

#### Step 1: Enable Restore Mode

Update your Postgres resource to enable barman restore:

```yaml
apiVersion: dip.io/v1alpha1
kind: Postgres
metadata:
  name: myapp-db
  namespace: myapp
spec:
  crossplane:
    compositionSelector:
      matchLabels:
        provider: cnpg
  parameters:
    identifier: myapp-db
    masterUsername: myapp
    databaseName: myapp
    size: small
    barmanBackup:
      enabled: true
      retentionDays: 7
    restore:
      method: barmanObjectStore
      # targetTime: "2024-01-15T10:30:00Z"  # Optional: point-in-time recovery
  writeConnectionSecretToRef:
    name: myapp-db-connection
```

#### Step 2: Delete and Recreate the Postgres Resource

```bash
# Delete the postgres resource (this will delete the CNPG cluster)
kubectl delete postgres myapp-db -n myapp

# Wait for cleanup
kubectl wait --for=delete cluster/myapp-db -n myapp --timeout=120s

# Apply the postgres resource with restore config
kubectl apply -f myapp-db-restore.yaml
```

#### Step 3: Monitor Recovery Progress

```bash
# Watch recovery pods
kubectl get pods -n myapp -l cnpg.io/cluster=myapp-db -w

# Check recovery logs
kubectl logs -n myapp -l cnpg.io/cluster=myapp-db -c full-recovery --tail=50

# Wait for cluster to be healthy
kubectl wait cluster myapp-db -n myapp \
  --for=jsonpath='{.status.phase}'='Cluster in healthy state' --timeout=600s
```

#### Step 4: Fix Password (if needed)

After recovery, the database password is restored from the backup, which may not match the connection secret. Update the password:

```bash
# Get the password from the connection secret
NEW_PASS=$(kubectl get secret myapp-db-connection -n myapp -o jsonpath='{.data.password}' | base64 -d)

# Update the database password to match
kubectl exec -n myapp myapp-db-1 -- psql -U postgres -c "ALTER USER myapp WITH PASSWORD '${NEW_PASS}';"

# Restart your application to reconnect
kubectl rollout restart deployment/myapp -n myapp
```

#### Step 5: Re-enable WAL Archiving

After recovery, disable restore mode to re-enable continuous WAL archiving:

```yaml
apiVersion: dip.io/v1alpha1
kind: Postgres
metadata:
  name: myapp-db
  namespace: myapp
spec:
  parameters:
    # ... same config as before ...
    barmanBackup:
      enabled: true
      retentionDays: 7
    # Remove or comment out the restore section:
    # restore:
    #   method: barmanObjectStore
```

Apply the updated config:

```bash
kubectl apply -f myapp-db.yaml

# Verify WAL archiving resumed
kubectl get cluster myapp-db -n myapp -o jsonpath='{.status.conditions}' | jq '.[] | select(.type=="ContinuousArchiving")'
```

#### Point-in-Time Recovery (PITR)

To restore to a specific point in time, add the `targetTime` parameter:

```yaml
spec:
  parameters:
    restore:
      method: barmanObjectStore
      targetTime: "2024-01-15T10:30:00Z"  # ISO 8601 format
```

### Barman Backup Configuration Options

| Parameter | Required | Default | Description |
|-----------|----------|---------|-------------|
| `barmanBackup.enabled` | No | `false` | Enable continuous WAL archiving to S3 |
| `barmanBackup.bucketName` | No | `${resourcePrefix}-barman` | S3 bucket name |
| `barmanBackup.bucketPrefix` | No | `${namespace}/${identifier}` | Path prefix in bucket |
| `barmanBackup.retentionDays` | No | `7` | Backup retention period (1-365 days) |
| `barmanBackup.compression` | No | `gzip` | Compression: gzip, bzip2, snappy, none |
| `barmanBackup.scheduleBaseBackup` | No | `0 0 0 * * *` | Cron schedule for base backups |
| `restore.method` | No | `backup` | `barmanObjectStore` for S3 restore |
| `restore.targetTime` | No | - | PITR target time (ISO 8601 format) |

### VolumeSnapshot Backups

CloudNativePG also supports VolumeSnapshot-based backups for fast, storage-level snapshots.

#### Enabling Scheduled Snapshots

Enable daily VolumeSnapshot backups by setting `enableSnapshots: true`:

```yaml
apiVersion: dip.io/v1alpha1
kind: Postgres
metadata:
  name: myapp-db
  namespace: myapp
spec:
  crossplane:
    compositionSelector:
      matchLabels:
        provider: cnpg
  parameters:
    identifier: myapp-db
    masterUsername: myapp
    databaseName: myapp
    engineVersion: "18.1"
    size: small
    enableSnapshots: true           # Enables daily snapshots at midnight
    snapshotClassName: default      # VolumeSnapshotClass to use
  writeConnectionSecretToRef:
    name: myapp-db-connection
```

This creates a `ScheduledBackup` resource that runs daily at midnight using the `volumeSnapshot` method.

#### Creating On-Demand VolumeSnapshot Backups

To create an immediate VolumeSnapshot backup:

```yaml
apiVersion: postgresql.cnpg.io/v1
kind: Backup
metadata:
  name: myapp-db-backup-manual
  namespace: myapp
spec:
  cluster:
    name: myapp-db              # The CNPG cluster name (same as identifier)
  method: volumeSnapshot
```

Monitor backup progress:

```bash
# Check backup status
kubectl get backup myapp-db-backup-manual -n myapp

# Wait for completion
kubectl wait backup myapp-db-backup-manual -n myapp \
  --for=jsonpath='{.status.phase}'=completed --timeout=120s

# Verify VolumeSnapshot was created
kubectl get volumesnapshots -n myapp
```

#### Restoring from a VolumeSnapshot

To restore a database from a VolumeSnapshot, create a new Postgres resource with the `restore` configuration:

```yaml
apiVersion: dip.io/v1alpha1
kind: Postgres
metadata:
  name: myapp-db-restored
  namespace: myapp
spec:
  crossplane:
    compositionSelector:
      matchLabels:
        provider: cnpg
  parameters:
    identifier: myapp-db-restored
    masterUsername: myapp
    databaseName: myapp
    engineVersion: "18.1"
    size: small
    snapshotClassName: default
    restore:
      sourceIdentifier: myapp-db-backup-manual   # Name of the VolumeSnapshot
      method: volumeSnapshot
  writeConnectionSecretToRef:
    name: myapp-db-restored-connection
```

The new cluster will bootstrap by:
1. Creating a PVC from the VolumeSnapshot
2. Starting PostgreSQL recovery from the snapshot
3. Replaying WAL to reach consistency
4. Bringing up replica instances

#### VolumeSnapshot Restore Options

| Parameter | Required | Default | Description |
|-----------|----------|---------|-------------|
| `restore.sourceIdentifier` | Yes | - | Name of the VolumeSnapshot or Backup object |
| `restore.method` | No | `backup` | Restore method: `volumeSnapshot` or `backup` |

#### Listing Available Snapshots

```bash
# List all VolumeSnapshots in a namespace
kubectl get volumesnapshots -n myapp

# List CNPG backups
kubectl get backups.postgresql.cnpg.io -n myapp
```

## Security Best Practices

### For All Databases

1. **Least Privilege**: Create database users with only necessary permissions
2. **Network Security**: Use security groups to restrict database access
3. **SSL/TLS**: Always use `sslmode=require` in connections
4. **Audit Logging**: Enable RDS audit logging for compliance
5. **Resource Tags**: Use tags for cost tracking and access control

### For Created Databases (using `identifier`)

8. **Strong Master Passwords**: Use long, random passwords stored securely
9. **Secret Management**: Never commit secrets to Git; use secret managers
10. **Private Subnets**: Place databases in private subnets (not publicly accessible)
11. **Multi-AZ**: Enable Multi-AZ for production databases
12. **Backup Retention**: Set appropriate backup retention (30 days for production)
13. **Encryption**: Keep `storageEncrypted: true` (default)
14. **Security Groups**: Restrict ingress to only necessary CIDR blocks
15. **Instance Sizing**: Choose appropriate instance class for workload
