# Barman Plugin Migration Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Migrate postgres-cnpg composition from deprecated native barmanObjectStore to plugin-based ObjectStore CRD

**Architecture:** Create ObjectStore resource, update Cluster to use plugins section, update ScheduledBackup to method: plugin, update externalClusters recovery to use plugin reference

**Tech Stack:** Crossplane, go-templating function, CloudNativePG, barmancloud.cnpg.io/v1 CRD

**Spec:** `docs/superpowers/specs/2026-05-13-barman-plugin-migration-design.md`

---

### Task 1: Add ObjectStore Resource

**Files:**
- Modify: `kustomize/base/postgres/postgres-composition-cnpg.yaml:358-373`

- [ ] **Step 1: Add ObjectStore resource template**

After the VPA resource block (around line 401) and before the IAM Role block (line 403), add the ObjectStore resource:

```yaml
          {{- if and $barmanEnabled (not $skipBarmanBackup) }}
          ---
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
          {{- end }}
```

- [ ] **Step 2: Verify template syntax**

Run:
```bash
cd /Users/andy/DEV/Personal/crossplane-compositions
cat kustomize/base/postgres/postgres-composition-cnpg.yaml | grep -A 20 "kind: ObjectStore"
```

Expected: ObjectStore resource block with correct templating

- [ ] **Step 3: Commit**

```bash
git add kustomize/base/postgres/postgres-composition-cnpg.yaml
git commit -m "feat(postgres-cnpg): add ObjectStore resource for plugin-based barman backup"
```

---

### Task 2: Update Cluster to Use Plugin Reference

**Files:**
- Modify: `kustomize/base/postgres/postgres-composition-cnpg.yaml:247-272` (backup section)
- Modify: `kustomize/base/postgres/postgres-composition-cnpg.yaml:177-186` (add plugins section)

- [ ] **Step 1: Add plugins section to Cluster spec**

After the `serviceAccountTemplate` block (line 186), add the plugins section:

```yaml
            {{- if and $barmanEnabled (not $skipBarmanBackup) }}
            plugins:
              - name: barman-cloud.cloudnative-pg.io
                parameters:
                  barmanObjectName: {{ $targetIdentifier }}-backup
            {{- end }}
```

- [ ] **Step 2: Remove barmanObjectStore from backup section**

Replace the backup section (lines 247-272) with volumeSnapshot-only logic:

```yaml
            {{- if $enableSnapshots }}
            backup:
              volumeSnapshot:
                className: {{ $snapshotClassName }}
                {{- if and $barmanEnabled (not $skipBarmanBackup) }}
                online: true
                onlineConfiguration:
                  immediateCheckpoint: false
                  waitForArchive: true
                {{- end }}
              target: prefer-standby
            {{- end }}
```

Note: The `barmanObjectStore` and `retentionPolicy` fields are removed entirely - they now live in the ObjectStore resource.

- [ ] **Step 3: Verify backup section no longer has barmanObjectStore**

Run:
```bash
grep -n "barmanObjectStore" kustomize/base/postgres/postgres-composition-cnpg.yaml
```

Expected: Only matches in externalClusters section (to be updated in Task 4)

- [ ] **Step 4: Commit**

```bash
git add kustomize/base/postgres/postgres-composition-cnpg.yaml
git commit -m "feat(postgres-cnpg): update Cluster to use plugins section for barman"
```

---

### Task 3: Update ScheduledBackup to Use Plugin Method

**Files:**
- Modify: `kustomize/base/postgres/postgres-composition-cnpg.yaml:358-373`

- [ ] **Step 1: Update barman ScheduledBackup**

Replace the barman ScheduledBackup (lines 358-373):

```yaml
          {{- if and $barmanEnabled (not $skipBarmanBackup) }}
          ---
          apiVersion: postgresql.cnpg.io/v1
          kind: ScheduledBackup
          metadata:
            name: {{ $name }}-barman-backup
            namespace: {{ $namespace }}
            annotations:
              gotemplating.fn.crossplane.io/composition-resource-name: barman-scheduled-backup
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
          {{- end }}
```

- [ ] **Step 2: Verify method changed**

Run:
```bash
grep -A 10 "barman-scheduled-backup" kustomize/base/postgres/postgres-composition-cnpg.yaml | grep "method:"
```

Expected: `method: plugin`

- [ ] **Step 3: Commit**

```bash
git add kustomize/base/postgres/postgres-composition-cnpg.yaml
git commit -m "feat(postgres-cnpg): update ScheduledBackup to use plugin method"
```

---

### Task 4: Update externalClusters Recovery to Use Plugin

**Files:**
- Modify: `kustomize/base/postgres/postgres-composition-cnpg.yaml:217-228`

- [ ] **Step 1: Update externalClusters section**

Replace the externalClusters block (lines 217-228):

```yaml
            {{- if eq $restoreMethod "barmanObjectStore" }}
            # External cluster for barman recovery source
            externalClusters:
            - name: barmanBackupSource
              plugin:
                name: barman-cloud.cloudnative-pg.io
                parameters:
                  barmanObjectName: {{ $targetIdentifier }}-backup
                  serverName: {{ $targetIdentifier }}
            {{- end }}
```

- [ ] **Step 2: Verify no inline barmanObjectStore remains**

Run:
```bash
grep -c "barmanObjectStore:" kustomize/base/postgres/postgres-composition-cnpg.yaml
```

Expected: `0` (no matches)

- [ ] **Step 3: Commit**

```bash
git add kustomize/base/postgres/postgres-composition-cnpg.yaml
git commit -m "feat(postgres-cnpg): update externalClusters recovery to use plugin"
```

---

### Task 5: Bump Revision and Final Verification

**Files:**
- Modify: `kustomize/base/postgres/postgres-composition-cnpg.yaml:7`

- [ ] **Step 1: Bump revision**

Change line 7 from:
```yaml
    revision: "57"
```

To:
```yaml
    revision: "58"
```

- [ ] **Step 2: Validate YAML syntax**

Run:
```bash
cd /Users/andy/DEV/Personal/crossplane-compositions
kubectl kustomize kustomize/base/postgres/ > /dev/null && echo "YAML valid"
```

Expected: `YAML valid`

- [ ] **Step 3: Verify all changes**

Run:
```bash
git diff HEAD~4 --stat
```

Expected: Only `postgres-composition-cnpg.yaml` modified

- [ ] **Step 4: Commit and push**

```bash
git add kustomize/base/postgres/postgres-composition-cnpg.yaml
git commit -m "chore(postgres-cnpg): bump revision to 58 for plugin migration"
git push
```

---

### Task 6: Test in obs-us-east-ct Cluster

**Files:**
- None (verification only)

- [ ] **Step 1: Apply updated composition**

Run:
```bash
export KUBECONFIG=/Users/andy/DEV/Personal/pulumi/k3s-on-ec2/local.yaml
kubectl apply -k /Users/andy/DEV/Personal/crossplane-compositions/kustomize/base/postgres/
```

Expected: `composition.apiextensions.crossplane.io/postgres-cnpg configured`

- [ ] **Step 2: Trigger reconciliation of iam-dex Postgres**

Run:
```bash
kubectl annotate postgres iam-dex -n iam-dex crossplane.io/composition-revision-trigger=$(date +%s) --overwrite
```

- [ ] **Step 3: Verify ObjectStore created**

Run:
```bash
kubectl get objectstore -n iam-dex
```

Expected: `obs-us-east-ct-dex-backup` ObjectStore exists

- [ ] **Step 4: Verify Cluster has plugins section**

Run:
```bash
kubectl get cluster obs-us-east-ct-dex -n iam-dex -o yaml | grep -A 5 "plugins:"
```

Expected: Plugin reference to `barman-cloud.cloudnative-pg.io`

- [ ] **Step 5: Verify ScheduledBackup uses plugin method**

Run:
```bash
kubectl get scheduledbackup -n iam-dex -o yaml | grep -A 5 "method:"
```

Expected: `method: plugin` with pluginConfiguration

- [ ] **Step 6: Trigger manual backup and verify**

Run:
```bash
kubectl create -f - <<EOF
apiVersion: postgresql.cnpg.io/v1
kind: Backup
metadata:
  name: test-plugin-backup
  namespace: iam-dex
spec:
  method: plugin
  pluginConfiguration:
    name: barman-cloud.cloudnative-pg.io
    parameters:
      barmanObjectName: obs-us-east-ct-dex-backup
  cluster:
    name: obs-us-east-ct-dex
EOF
```

Wait for completion:
```bash
kubectl get backup test-plugin-backup -n iam-dex -w
```

Expected: Backup completes successfully

- [ ] **Step 7: Verify S3 upload**

Run:
```bash
aws s3 ls s3://obs-us-east-ct-barman/iam-dex/obs-us-east-ct-dex/ --profile obs-ct --recursive | tail -5
```

Expected: Recent backup files present
