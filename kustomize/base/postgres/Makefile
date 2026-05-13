.PHONY: test validate render render-check test-restore-from-volumesnapshot test-barman-backup clean help

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

test: validate render render-check test-restore-from-volumesnapshot test-barman-backup ## Run all tests (validate + render + snapshot assertions)
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

test-barman-backup: ## Assert rendered output of cnpg-barman-backup contains barman config, IAM resources, and serviceAccountTemplate
	@echo "🔍 Checking rendered output for Barman backup configuration..."
	@OUTPUT=$$(crossplane render \
		$(BASE_DIR)examples/cnpg-barman-backup.yaml \
		$(BASE_DIR)postgres-composition-cnpg.yaml \
		$(BASE_DIR)functions-only.yaml \
		--xrd=$(BASE_DIR)postgres-xrd.yaml \
		--required-resources=$(BASE_DIR)test-environment-config.yaml 2>&1) || { echo "❌ Render failed"; echo "$$OUTPUT"; exit 1; }; \
	echo "$$OUTPUT" | grep -q "barmanObjectStore:" \
		|| { echo "❌ Expected 'barmanObjectStore:' not found in rendered output"; echo "$$OUTPUT"; exit 1; }; \
	echo "$$OUTPUT" | grep -q "destinationPath: s3://test-cluster-barman/test-namespace/myapp-with-barman" \
		|| { echo "❌ Expected S3 destination path not found in rendered output"; echo "$$OUTPUT"; exit 1; }; \
	echo "$$OUTPUT" | grep -q "inheritFromIAMRole: true" \
		|| { echo "❌ Expected 'inheritFromIAMRole: true' not found in rendered output"; echo "$$OUTPUT"; exit 1; }; \
	echo "$$OUTPUT" | grep -q "retentionPolicy:" \
		|| { echo "❌ Expected 'retentionPolicy:' not found in rendered output"; echo "$$OUTPUT"; exit 1; }; \
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
	echo "$$OUTPUT" | grep -q "method: barmanObjectStore" \
		|| { echo "❌ Expected 'method: barmanObjectStore' not found in rendered output"; echo "$$OUTPUT"; exit 1; }; \
	echo "$$OUTPUT" | grep -q "compression: gzip" \
		|| { echo "❌ Expected 'compression: gzip' not found in rendered output"; echo "$$OUTPUT"; exit 1; }; \
	echo "✅ Barman backup render-check passed!"

clean: ## Clean up any generated files
	@echo "🧹 Cleaning up postgres test artifacts..."
	@rm -f $(BASE_DIR).crossplane-render-*
	@echo "✅ Clean complete!"
