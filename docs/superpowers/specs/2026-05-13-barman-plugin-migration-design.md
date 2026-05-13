# Crossplane Composition Migration: Native Barman to Plugin-Based ObjectStore

## Goal

Migrate the `postgres-cnpg` Crossplane composition from the deprecated native `barmanObjectStore` configuration to the plugin-based `ObjectStore` CRD (barmancloud.cnpg.io/v1), ensuring compatibility with CNPG 1.30+.

## Background

CloudNativePG is deprecating the native barman support in version 1.30. The new architecture uses:
- `ObjectStore` CRD to define backup storage configuration
- Plugin references in Cluster and ScheduledBackup resources
- Same IRSA authentication pattern

## Architecture

The composition creates an `ObjectStore` resource that encapsulates S3 backup configuration. The Cluster references this via a `plugins` section instead of inline `backup.barmanObjectStore`. ScheduledBackup uses `method: plugin` with `pluginConfiguration`. IAM resources for IRSA remain unchanged.

## Component Changes

### 1. New ObjectStore Resource

Created when `barmanEnabled` is true:

```yaml
apiVersion: barmancloud.cnpg.io/v1
kind: ObjectStore
metadata:
  name: {{ $targetIdentifier }}-backup
  namespace: {{ $namespace }}
  annotations:
    gotemplating.fn.crossplane.io/composition-resource-name: barman-objectstore
spec:
  configuration:
    destinationPath: {{ $barmanDestinationPath }}
    s3Credentials:
      inheritFromIAMRole: true
    wal:
      compression: {{ $barmanCompression }}
      maxParallel: 4
    data:
      compression: {{ $barmanCompression }}
  retentionPolicy: "{{ $barmanRetentionDays }}d"
```

### 2. Cluster Changes

**Remove:**
- `backup.barmanObjectStore` section
- `backup.retentionPolicy` (moves to ObjectStore)

**Add** (when barmanEnabled and not skipBarmanBackup):
```yaml
plugins:
  - name: barman-cloud.cloudnative-pg.io
    parameters:
      barmanObjectName: {{ $targetIdentifier }}-backup
```

**Note:** The `backup` section may still exist for volumeSnapshot configuration.

### 3. ScheduledBackup Changes

**Before:**
```yaml
apiVersion: postgresql.cnpg.io/v1
kind: ScheduledBackup
metadata:
  name: {{ $name }}-barman-backup
spec:
  schedule: {{ $barmanSchedule | quote }}
  method: barmanObjectStore
  backupOwnerReference: self
  cluster:
    name: {{ $targetIdentifier }}
```

**After:**
```yaml
apiVersion: postgresql.cnpg.io/v1
kind: ScheduledBackup
metadata:
  name: {{ $name }}-barman-backup
spec:
  schedule: {{ $barmanSchedule | quote }}
  method: plugin
  pluginConfiguration:
    name: barman-cloud.cloudnative-pg.io
    parameters:
      barmanObjectName: {{ $targetIdentifier }}-backup
  backupOwnerReference: self
  cluster:
    name: {{ $targetIdentifier }}
```

### 4. Recovery (externalClusters) Changes

**Before:**
```yaml
externalClusters:
- name: barmanBackupSource
  barmanObjectStore:
    serverName: {{ $targetIdentifier }}
    destinationPath: {{ $barmanDestinationPath }}
    s3Credentials:
      inheritFromIAMRole: true
    wal:
      maxParallel: 4
```

**After:**
```yaml
externalClusters:
- name: barmanBackupSource
  plugin:
    name: barman-cloud.cloudnative-pg.io
    parameters:
      barmanObjectName: {{ $targetIdentifier }}-backup
      serverName: {{ $targetIdentifier }}
```

### 5. Unchanged Components

- IAM Role, Policy, RolePolicyAttachment (IRSA pattern unchanged)
- ServiceAccountTemplate annotation
- S3 destination path format (`s3://{bucket}/{namespace}/{identifier}`)
- VolumeSnapshot backup configuration (if enabled)
- All other Cluster configuration (storage, resources, bootstrap initdb, etc.)
- Password and connection secrets

## Composition Metadata

- Bump revision label from `57` to `58`

## Testing

1. Deploy updated composition to test cluster
2. Create new Postgres CR with barman enabled
3. Verify ObjectStore created with correct configuration
4. Verify Cluster has plugins section (not backup.barmanObjectStore)
5. Verify ScheduledBackup uses method: plugin
6. Trigger manual backup and verify S3 upload
7. Test recovery by creating new Postgres CR with restore.method: barmanObjectStore

## Rollback

If issues occur, revert composition to revision 57. Existing databases continue operating; only new creations or updates would use old config.
