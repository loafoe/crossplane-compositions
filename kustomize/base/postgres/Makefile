.PHONY: test validate render render-check render-check-no-backups test-restore-from-volumesnapshot test-barman-backup test-barman-restore test-pgvector test-engine-version clean help

# Default target
.DEFAULT_GOAL := help

# Base directory for postgres
BASE_DIR := $(dir $(abspath $(lastword $(MAKEFILE_LIST))))

# Test examples
EXAMPLES := $(wildcard $(BASE_DIR)examples/*.yaml)

help: ## Show this help message
	@echo 'Usage: make [target]'
	@echo ''
	@echo 'Available targets:'
	@grep -E '^[a-zA-Z_-]+:.*?## .*$$' $(MAKEFILE_LIST) | sort | awk 'BEGIN {FS = ":.*?## "}; {printf "  %-15s %s\n", $$1, $$2}'

test: validate render render-check render-check-no-backups test-restore-from-volumesnapshot test-barman-backup test-barman-s3compatible test-barman-restore test-pgvector test-engine-version ## Run all tests (validate + render + snapshot assertions)
	@echo "✅ All postgres tests passed!"

validate: ## Validate the composition and XRD using crossplane beta validate
	@echo "🔍 Validating Postgres XRD and Composition..."
	@crossplane beta validate \
		$(BASE_DIR)postgres-xrd.yaml,$(BASE_DIR)functions-only.yaml \
		$(BASE_DIR)postgres-composition-aws.yaml,$(BASE_DIR)postgres-composition-cnpg.yaml
	@echo "✅ Validation passed!"

render: ## Render compositions with example Postgres resources
	@echo "🎨 Rendering Postgres compositions with examples..."
	@for example in $(EXAMPLES); do \
		echo ""; \
		echo "Testing $$(basename $$example)..."; \
		if grep -q "provider: cnpg" $$example; then \
			COMP=$(BASE_DIR)postgres-composition-cnpg.yaml; \
		else \
			COMP=$(BASE_DIR)postgres-composition-aws.yaml; \
		fi; \
		crossplane render \
			$$example \
			$$COMP \
			$(BASE_DIR)functions-only.yaml \
			--xrd=$(BASE_DIR)postgres-xrd.yaml \
			--required-resources=$(BASE_DIR)test-environment-config.yaml \
			|| exit 1; \
		echo "✅ $$(basename $$example) rendered successfully with $$(basename $$COMP)"; \
	done
	@echo ""
	@echo "✅ All examples rendered successfully!"

render-check: ## Assert rendered output of cnpg-snapshots example contains storageClass and backup.volumeSnapshot
	@echo "🔍 Checking rendered output for snapshot fields..."
	@OUTPUT=$$(crossplane render \
		$(BASE_DIR)examples/cnpg-snapshots.yaml \
		$(BASE_DIR)postgres-composition-cnpg.yaml \
		$(BASE_DIR)functions-only.yaml \
		--xrd=$(BASE_DIR)postgres-xrd.yaml \
		--required-resources=$(BASE_DIR)test-environment-config.yaml 2>&1) || { echo "❌ Render failed"; echo "$$OUTPUT"; exit 1; }; \
	echo "$$OUTPUT" | grep -q "storageClass:" \
		|| { echo "❌ Expected 'storageClass:' not found in rendered output"; echo "$$OUTPUT"; exit 1; }; \
	echo "$$OUTPUT" | grep -q "volumeSnapshot:" \
		|| { echo "❌ Expected 'volumeSnapshot:' not found in rendered output"; echo "$$OUTPUT"; exit 1; }; \
	echo "$$OUTPUT" | grep -q "className: csi-snapclass" \
		|| { echo "❌ Expected 'className: csi-snapclass' not found in rendered output"; echo "$$OUTPUT"; exit 1; }; \
	echo "✅ Snapshot render-check passed!"

render-check-no-backups: ## Regression: cnpg-minimal (no snapshots/barman) must render WITHOUT the patch-and-transform "no base template" fatal
	@echo "🔍 Checking cnpg-minimal renders with no backup resources (regression for unconditional p&t entries)..."
	@OUTPUT=$$(crossplane render \
		$(BASE_DIR)examples/cnpg-minimal.yaml \
		$(BASE_DIR)postgres-composition-cnpg.yaml \
		$(BASE_DIR)functions-only.yaml \
		--xrd=$(BASE_DIR)postgres-xrd.yaml \
		--required-resources=$(BASE_DIR)test-environment-config.yaml 2>&1) || { echo "❌ Render failed (regression: disabled backups must not error)"; echo "$$OUTPUT"; exit 1; }; \
	echo "$$OUTPUT" | grep -q "no base template" \
		&& { echo "❌ Regression: 'no base template' fatal returned — backup resources must not be listed unconditionally in patch-and-transform"; echo "$$OUTPUT"; exit 1; }; \
	echo "$$OUTPUT" | grep -q "kind: Cluster" \
		|| { echo "❌ Expected 'kind: Cluster' not found — composition did not render the CNPG cluster"; echo "$$OUTPUT"; exit 1; }; \
	echo "$$OUTPUT" | grep -q "kind: ScheduledBackup" \
		&& { echo "❌ Unexpected 'kind: ScheduledBackup' — minimal example must not render backups"; echo "$$OUTPUT"; exit 1; }; \
	echo "$$OUTPUT" | grep -q "kind: ObjectStore" \
		&& { echo "❌ Unexpected 'kind: ObjectStore' — minimal example must not render barman"; echo "$$OUTPUT"; exit 1; }; \
	echo "$$OUTPUT" | grep -q "serviceAccountTemplate:" \
		&& { echo "❌ Regression: 'serviceAccountTemplate:' rendered with backups disabled - the IRSA annotation is an AWS-barman-only artefact"; echo "$$OUTPUT"; exit 1; }; \
	echo "$$OUTPUT" | grep -q "eks.amazonaws.com/role-arn" \
		&& { echo "❌ Regression: IRSA annotation rendered with backups disabled"; echo "$$OUTPUT"; exit 1; }; \
	echo "✅ No-backups render-check passed!"

test-restore-from-volumesnapshot: ## Assert rendered output of cnpg-restore-from-volumesnapshot uses volumeSnapshots bootstrap
	@echo "🔍 Checking rendered output for VolumeSnapshot restore bootstrap..."
	@OUTPUT=$$(crossplane render \
		$(BASE_DIR)examples/cnpg-restore-from-volumesnapshot.yaml \
		$(BASE_DIR)postgres-composition-cnpg.yaml \
		$(BASE_DIR)functions-only.yaml \
		--xrd=$(BASE_DIR)postgres-xrd.yaml \
		--required-resources=$(BASE_DIR)test-environment-config.yaml 2>&1) || { echo "❌ Render failed"; echo "$$OUTPUT"; exit 1; }; \
	echo "$$OUTPUT" | grep -q "volumeSnapshots:" \
		|| { echo "❌ Expected 'volumeSnapshots:' not found in rendered output"; echo "$$OUTPUT"; exit 1; }; \
	echo "$$OUTPUT" | grep -q "apiGroup: snapshot.storage.k8s.io" \
		|| { echo "❌ Expected 'apiGroup: snapshot.storage.k8s.io' not found in rendered output"; echo "$$OUTPUT"; exit 1; }; \
	echo "$$OUTPUT" | grep -q "myapp-local-snapshots-20240101" \
		|| { echo "❌ Expected snapshot name not found in rendered output"; echo "$$OUTPUT"; exit 1; }; \
	echo "✅ VolumeSnapshot restore render-check passed!"

test-barman-backup: ## Assert rendered output of cnpg-barman-backup contains plugin-based ObjectStore, IAM resources, and serviceAccountTemplate
	@echo "🔍 Checking rendered output for Barman backup configuration (plugin-based)..."
	@OUTPUT=$$(crossplane render \
		$(BASE_DIR)examples/cnpg-barman-backup.yaml \
		$(BASE_DIR)postgres-composition-cnpg.yaml \
		$(BASE_DIR)functions-only.yaml \
		--xrd=$(BASE_DIR)postgres-xrd.yaml \
		--required-resources=$(BASE_DIR)test-environment-config.yaml 2>&1) || { echo "❌ Render failed"; echo "$$OUTPUT"; exit 1; }; \
	echo "$$OUTPUT" | grep -q "kind: ObjectStore" \
		|| { echo "❌ Expected 'kind: ObjectStore' not found in rendered output"; echo "$$OUTPUT"; exit 1; }; \
	echo "$$OUTPUT" | grep -q "apiVersion: barmancloud.cnpg.io/v1" \
		|| { echo "❌ Expected 'apiVersion: barmancloud.cnpg.io/v1' not found in rendered output"; echo "$$OUTPUT"; exit 1; }; \
	echo "$$OUTPUT" | grep -q "destinationPath: s3://test-cluster-barman/test-namespace/myapp-with-barman" \
		|| { echo "❌ Expected S3 destination path not found in rendered output"; echo "$$OUTPUT"; exit 1; }; \
	echo "$$OUTPUT" | grep -q "inheritFromIAMRole: true" \
		|| { echo "❌ Expected 'inheritFromIAMRole: true' not found in rendered output"; echo "$$OUTPUT"; exit 1; }; \
	echo "$$OUTPUT" | grep -q "retentionPolicy:" \
		|| { echo "❌ Expected 'retentionPolicy:' not found in rendered output"; echo "$$OUTPUT"; exit 1; }; \
	echo "$$OUTPUT" | grep -q "plugins:" \
		|| { echo "❌ Expected 'plugins:' section not found in Cluster spec"; echo "$$OUTPUT"; exit 1; }; \
	echo "$$OUTPUT" | grep -q "name: barman-cloud.cloudnative-pg.io" \
		|| { echo "❌ Expected plugin name 'barman-cloud.cloudnative-pg.io' not found in rendered output"; echo "$$OUTPUT"; exit 1; }; \
	echo "$$OUTPUT" | grep -q "isWALArchiver: true" \
		|| { echo "❌ Expected 'isWALArchiver: true' not found - without it archive_mode stays forced on (waitForArchive needs it) but nothing archives, so WAL never recycles and silently fills the PV"; echo "$$OUTPUT"; exit 1; }; \
	echo "$$OUTPUT" | grep -q "barmanObjectName: myapp-with-barman-backup" \
		|| { echo "❌ Expected 'barmanObjectName: myapp-with-barman-backup' not found in rendered output"; echo "$$OUTPUT"; exit 1; }; \
	echo "$$OUTPUT" | grep -q "serviceAccountTemplate:" \
		|| { echo "❌ Expected 'serviceAccountTemplate:' not found in rendered output"; echo "$$OUTPUT"; exit 1; }; \
	echo "$$OUTPUT" | grep -q "eks.amazonaws.com/role-arn:" \
		|| { echo "❌ Expected IRSA annotation not found in rendered output"; echo "$$OUTPUT"; exit 1; }; \
	echo "$$OUTPUT" | grep -q "arn:aws:iam::123456789012:role/test-namespace-myapp-with-barman-cnpg-backup" \
		|| { echo "❌ Expected IAM role ARN not found in rendered output"; echo "$$OUTPUT"; exit 1; }; \
	echo "$$OUTPUT" | grep -q "kind: Role" \
		|| { echo "❌ Expected IAM Role resource not found in rendered output"; echo "$$OUTPUT"; exit 1; }; \
	echo "$$OUTPUT" | grep -q "kind: Policy" \
		|| { echo "❌ Expected IAM Policy resource not found in rendered output"; echo "$$OUTPUT"; exit 1; }; \
	echo "$$OUTPUT" | grep -q "kind: RolePolicyAttachment" \
		|| { echo "❌ Expected IAM RolePolicyAttachment resource not found in rendered output"; echo "$$OUTPUT"; exit 1; }; \
	echo "$$OUTPUT" | grep -q "kind: ScheduledBackup" \
		|| { echo "❌ Expected ScheduledBackup resource not found in rendered output"; echo "$$OUTPUT"; exit 1; }; \
	echo "$$OUTPUT" | grep -q "method: plugin" \
		|| { echo "❌ Expected 'method: plugin' not found in ScheduledBackup"; echo "$$OUTPUT"; exit 1; }; \
	echo "$$OUTPUT" | grep -q "pluginConfiguration:" \
		|| { echo "❌ Expected 'pluginConfiguration:' not found in ScheduledBackup"; echo "$$OUTPUT"; exit 1; }; \
	echo "$$OUTPUT" | grep -q "compression: gzip" \
		|| { echo "❌ Expected 'compression: gzip' not found in rendered output"; echo "$$OUTPUT"; exit 1; }; \
	echo "$$OUTPUT" | grep -q "serverName: myapp-with-barman-wal" \
		|| { echo "❌ Expected the default WAL archiver serverName ('<identifier>-wal') not found in rendered output"; echo "$$OUTPUT"; exit 1; }; \
	echo "$$OUTPUT" | grep -q "endpointURL" \
		&& { echo "❌ Regression: 'endpointURL' present - the AWS path must target AWS S3, not a custom endpoint"; echo "$$OUTPUT"; exit 1; }; \
	echo "$$OUTPUT" | grep -q "maxParallel: 4" \
		|| { echo "❌ Expected the default 'maxParallel: 4' not found in rendered output"; echo "$$OUTPUT"; exit 1; }; \
	echo "✅ Barman backup render-check passed!"

test-barman-s3compatible: ## Assert rendered output for cnpg-barman-s3compatible uses static creds + custom endpointURL and renders NO AWS IAM resources
	@echo "🔍 Checking rendered output for S3-compatible (non-AWS) Barman configuration..."
	@OUTPUT=$$(crossplane render \
		$(BASE_DIR)examples/cnpg-barman-s3compatible.yaml \
		$(BASE_DIR)postgres-composition-cnpg.yaml \
		$(BASE_DIR)functions-only.yaml \
		--xrd=$(BASE_DIR)postgres-xrd.yaml \
		--required-resources=$(BASE_DIR)test-environment-config.yaml 2>&1) || { echo "❌ Render failed"; echo "$$OUTPUT"; exit 1; }; \
	echo "$$OUTPUT" | grep -q "kind: ObjectStore" \
		|| { echo "❌ Expected 'kind: ObjectStore' not found in rendered output"; echo "$$OUTPUT"; exit 1; }; \
	echo "$$OUTPUT" | grep -q "apiVersion: barmancloud.cnpg.io/v1" \
		|| { echo "❌ Expected 'apiVersion: barmancloud.cnpg.io/v1' not found in rendered output"; echo "$$OUTPUT"; exit 1; }; \
	echo "$$OUTPUT" | grep -q "endpointURL: http://192.0.2.10:9000" \
		|| { echo "❌ Expected the custom S3 endpointURL not found in rendered output - without it barman targets AWS S3"; echo "$$OUTPUT"; exit 1; }; \
	echo "$$OUTPUT" | grep -q "accessKeyId:" \
		|| { echo "❌ Expected 'accessKeyId:' not found - static credentials must be used for a non-AWS store"; echo "$$OUTPUT"; exit 1; }; \
	echo "$$OUTPUT" | grep -q "name: cnpg-s3-creds" \
		|| { echo "❌ Expected the referenced credentials Secret name not found in rendered output"; echo "$$OUTPUT"; exit 1; }; \
	echo "$$OUTPUT" | grep -q "key: ACCESS_KEY_ID" \
		|| { echo "❌ Expected 'key: ACCESS_KEY_ID' not found in rendered output"; echo "$$OUTPUT"; exit 1; }; \
	echo "$$OUTPUT" | grep -q "secretAccessKey:" \
		|| { echo "❌ Expected 'secretAccessKey:' not found in rendered output"; echo "$$OUTPUT"; exit 1; }; \
	echo "$$OUTPUT" | grep -q "key: ACCESS_SECRET_KEY" \
		|| { echo "❌ Expected 'key: ACCESS_SECRET_KEY' not found in rendered output"; echo "$$OUTPUT"; exit 1; }; \
	echo "$$OUTPUT" | grep -q "inheritFromIAMRole" \
		&& { echo "❌ Regression: 'inheritFromIAMRole' present - static-credential mode must not also ask for an IAM role"; echo "$$OUTPUT"; exit 1; }; \
	echo "$$OUTPUT" | grep -qE "^kind: Role$$" \
		&& { echo "❌ Regression: IAM Role rendered - a cluster with no AWS provider cannot compose iam.aws.m.upbound.io resources"; echo "$$OUTPUT"; exit 1; }; \
	echo "$$OUTPUT" | grep -qE "^kind: Policy$$" \
		&& { echo "❌ Regression: IAM Policy rendered - must be omitted in static-credential mode"; echo "$$OUTPUT"; exit 1; }; \
	echo "$$OUTPUT" | grep -qE "^kind: RolePolicyAttachment$$" \
		&& { echo "❌ Regression: IAM RolePolicyAttachment rendered - must be omitted in static-credential mode"; echo "$$OUTPUT"; exit 1; }; \
	echo "$$OUTPUT" | grep -q "eks.amazonaws.com/role-arn" \
		&& { echo "❌ Regression: IRSA annotation rendered - there is no IAM role to assume in static-credential mode"; echo "$$OUTPUT"; exit 1; }; \
	echo "$$OUTPUT" | grep -q "serviceAccountTemplate:" \
		&& { echo "❌ Regression: 'serviceAccountTemplate:' rendered - must be omitted in static-credential mode"; echo "$$OUTPUT"; exit 1; }; \
	echo "$$OUTPUT" | grep -q "serverName: myapp-with-barman-s3-wal" \
		|| { echo "❌ Expected the WAL archiver serverName not found in rendered output"; echo "$$OUTPUT"; exit 1; }; \
	echo "$$OUTPUT" | grep -q "isWALArchiver: true" \
		|| { echo "❌ Expected 'isWALArchiver: true' not found - without it nothing archives the WAL"; echo "$$OUTPUT"; exit 1; }; \
	echo "$$OUTPUT" | grep -q "maxParallel: 2" \
		|| { echo "❌ Expected 'maxParallel: 2' not found - the requested parallelism was not honoured"; echo "$$OUTPUT"; exit 1; }; \
	echo "$$OUTPUT" | grep -q "retentionPolicy: 14d" \
		|| { echo "❌ Expected 'retentionPolicy: 14d' not found in rendered output"; echo "$$OUTPUT"; exit 1; }; \
	echo "$$OUTPUT" | grep -q "method: plugin" \
		|| { echo "❌ Expected 'method: plugin' not found in ScheduledBackup"; echo "$$OUTPUT"; exit 1; }; \
	echo "✅ S3-compatible Barman render-check passed!"

test-barman-restore: ## Assert rendered output of cnpg-barman-restore contains plugin-based externalClusters and recovery bootstrap
	@echo "🔍 Checking rendered output for Barman restore configuration (plugin-based)..."
	@OUTPUT=$$(crossplane render \
		$(BASE_DIR)examples/cnpg-barman-restore.yaml \
		$(BASE_DIR)postgres-composition-cnpg.yaml \
		$(BASE_DIR)functions-only.yaml \
		--xrd=$(BASE_DIR)postgres-xrd.yaml \
		--required-resources=$(BASE_DIR)test-environment-config.yaml 2>&1) || { echo "❌ Render failed"; echo "$$OUTPUT"; exit 1; }; \
	echo "$$OUTPUT" | grep -q "externalClusters:" \
		|| { echo "❌ Expected 'externalClusters:' not found in rendered output"; echo "$$OUTPUT"; exit 1; }; \
	echo "$$OUTPUT" | grep -q "name: barmanBackupSource" \
		|| { echo "❌ Expected 'name: barmanBackupSource' not found in rendered output"; echo "$$OUTPUT"; exit 1; }; \
	echo "$$OUTPUT" | grep -q "source: barmanBackupSource" \
		|| { echo "❌ Expected 'source: barmanBackupSource' not found in rendered output"; echo "$$OUTPUT"; exit 1; }; \
	echo "$$OUTPUT" | grep -q "plugin:" \
		|| { echo "❌ Expected 'plugin:' section not found in externalClusters"; echo "$$OUTPUT"; exit 1; }; \
	echo "$$OUTPUT" | grep -q "name: barman-cloud.cloudnative-pg.io" \
		|| { echo "❌ Expected plugin name 'barman-cloud.cloudnative-pg.io' not found in externalClusters"; echo "$$OUTPUT"; exit 1; }; \
	echo "$$OUTPUT" | grep -q "barmanObjectName: restored-from-barman-backup" \
		|| { echo "❌ Expected 'barmanObjectName: restored-from-barman-backup' not found in rendered output"; echo "$$OUTPUT"; exit 1; }; \
	echo "$$OUTPUT" | grep -q "serverName: restored-from-barman" \
		|| { echo "❌ Expected 'serverName: restored-from-barman' not found in rendered output"; echo "$$OUTPUT"; exit 1; }; \
	echo "$$OUTPUT" | grep -q "isWALArchiver" \
		&& { echo "❌ Regression: a live WAL archiver was rendered during a barman restore - the archive it would write to is the one being recovered from"; echo "$$OUTPUT"; exit 1; }; \
	echo "$$OUTPUT" | grep -q "kind: ObjectStore" \
		&& { echo "❌ Regression: an ObjectStore was rendered during a barman restore - the destination is the recovery source"; echo "$$OUTPUT"; exit 1; }; \
	echo "✅ Barman restore render-check passed!"

test-pgvector: ## Assert rendered output of cnpg-pgvector uses postInitApplicationSQL (not postInitSQL) to CREATE EXTENSION
	@echo "🔍 Checking rendered output for extensions bootstrap..."
	@OUTPUT=$$(crossplane render \
		$(BASE_DIR)examples/cnpg-pgvector.yaml \
		$(BASE_DIR)postgres-composition-cnpg.yaml \
		$(BASE_DIR)functions-only.yaml \
		--xrd=$(BASE_DIR)postgres-xrd.yaml \
		--required-resources=$(BASE_DIR)test-environment-config.yaml 2>&1) || { echo "❌ Render failed"; echo "$$OUTPUT"; exit 1; }; \
	echo "$$OUTPUT" | grep -q "postInitApplicationSQL:" \
		|| { echo "❌ Expected 'postInitApplicationSQL:' not found in rendered output"; echo "$$OUTPUT"; exit 1; }; \
	echo "$$OUTPUT" | grep -q "CREATE EXTENSION IF NOT EXISTS vector" \
		|| { echo "❌ Expected 'CREATE EXTENSION IF NOT EXISTS vector' not found in rendered output"; echo "$$OUTPUT"; exit 1; }; \
	echo "$$OUTPUT" | grep -q "CREATE EXTENSION IF NOT EXISTS pg_trgm" \
		|| { echo "❌ Expected 'CREATE EXTENSION IF NOT EXISTS pg_trgm' not found in rendered output"; echo "$$OUTPUT"; exit 1; }; \
	echo "$$OUTPUT" | grep -q "postInitSQL:" \
		&& { echo "❌ Regression: 'postInitSQL' used instead of 'postInitApplicationSQL' - extensions would land in the postgres db, not the app db"; echo "$$OUTPUT"; exit 1; }; \
	echo "✅ pgvector render-check passed!"

test-engine-version: ## Assert engineVersion maps to the CNPG Cluster imageName, defaulting to 18.6 when omitted
	@echo "🔍 Checking engineVersion -> spec.imageName mapping..."
	@OUTPUT=$$(crossplane render \
		$(BASE_DIR)examples/cnpg-minimal.yaml \
		$(BASE_DIR)postgres-composition-cnpg.yaml \
		$(BASE_DIR)functions-only.yaml \
		--xrd=$(BASE_DIR)postgres-xrd.yaml \
		--required-resources=$(BASE_DIR)test-environment-config.yaml 2>&1) || { echo "❌ Render failed"; echo "$$OUTPUT"; exit 1; }; \
	echo "$$OUTPUT" | grep -q "imageName: ghcr.io/cloudnative-pg/postgresql:18.6-system-trixie" \
		|| { echo "❌ Expected default 'imageName: ghcr.io/cloudnative-pg/postgresql:18.6-system-trixie' when engineVersion is omitted"; echo "$$OUTPUT"; exit 1; }; \
	OUTPUT=$$(crossplane render \
		$(BASE_DIR)examples/cnpg-explicit-version.yaml \
		$(BASE_DIR)postgres-composition-cnpg.yaml \
		$(BASE_DIR)functions-only.yaml \
		--xrd=$(BASE_DIR)postgres-xrd.yaml \
		--required-resources=$(BASE_DIR)test-environment-config.yaml 2>&1) || { echo "❌ Render failed"; echo "$$OUTPUT"; exit 1; }; \
	echo "$$OUTPUT" | grep -q "imageName: ghcr.io/cloudnative-pg/postgresql:18.6-system-trixie" \
		|| { echo "❌ Explicit engineVersion '18.6' did not render the expected imageName"; echo "$$OUTPUT"; exit 1; }; \
	TMP=$$(mktemp); \
	sed 's/engineVersion: "18.6"/engineVersion: "17.8"/' $(BASE_DIR)examples/cnpg-explicit-version.yaml > $$TMP; \
	OUTPUT=$$(crossplane render \
		$$TMP \
		$(BASE_DIR)postgres-composition-cnpg.yaml \
		$(BASE_DIR)functions-only.yaml \
		--xrd=$(BASE_DIR)postgres-xrd.yaml \
		--required-resources=$(BASE_DIR)test-environment-config.yaml 2>&1); RC=$$?; rm -f $$TMP; \
	[ $$RC -eq 0 ] || { echo "❌ Render failed for engineVersion override"; echo "$$OUTPUT"; exit 1; }; \
	echo "$$OUTPUT" | grep -q "imageName: ghcr.io/cloudnative-pg/postgresql:17.8-system-trixie" \
		|| { echo "❌ engineVersion override '17.8' did not render the expected imageName"; echo "$$OUTPUT"; exit 1; }; \
	OUTPUT=$$(crossplane render \
		$(BASE_DIR)examples/new-rds-instance.yaml \
		$(BASE_DIR)postgres-composition-aws.yaml \
		$(BASE_DIR)functions-only.yaml \
		--xrd=$(BASE_DIR)postgres-xrd.yaml \
		--required-resources=$(BASE_DIR)test-environment-config.yaml 2>&1) || { echo "❌ Render failed"; echo "$$OUTPUT"; exit 1; }; \
	echo "$$OUTPUT" | grep -q "engineVersion: \"18.6\"" \
		|| { echo "❌ Expected the AWS example's explicit engineVersion '18.6' in the rendered RDS instance"; echo "$$OUTPUT"; exit 1; }; \
	echo "✅ engineVersion render-check passed!"

clean: ## Clean up any generated files
	@echo "🧹 Cleaning up postgres test artifacts..."
	@rm -f $(BASE_DIR).crossplane-render-*
	@echo "✅ Clean complete!"