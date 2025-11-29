# Migrating Self-Managed Sharded MongoDB Cluster on EC2 to Percona Server for MongoDB on EKS

# Overview
Community-focused, end-to-end guide and IaC blueprint to migrate from a self-managed, sharded MongoDB cluster on EC2 to an operator-driven, Kubernetes-native deployment with Percona Server for MongoDB on Amazon EKS. Includes architecture, deployment workflow, sanitised Terraform/Helm examples, operational runbooks, lessons learned, and references.

# Purpose
- Provide a practical, repeatable path for migrating sharded MongoDB to Percona on EKS.
- Reduce operational toil, improve security and observability, avoid vendor lock-in, and control cost.
- Share learnings so others can avoid common pitfalls and reach production readiness faster.

# Who this helps
- Teams operating legacy or hard-to-upgrade sharded MongoDB clusters.
- Practitioners seeking an open-source, Kubernetes-native approach with strong automation.
- Engineers looking for Terraform + Helm examples and operational runbooks (PBM/PMM, IRSA, gp3).

# Decision overview (options and trade-offs)
- Upgrade self-managed MongoDB on EC2
  - Often not viable when automation is limited; upgrades and scaling are manual, risky, and downtime-prone.
- Percona Server for MongoDB on EKS (recommended for many teams)
  - Open-source, MongoDB-compatible, operator-based automation for backups, scaling, upgrades, and monitoring; avoids lock-in and aligns with Kubernetes operations.
- MongoDB Atlas
  - Operationally strong managed service; typically higher costs at scale and vendor lock-in considerations.
- AWS DocumentDB
  - Compatibility gaps for sharded workloads (for example, composite shard keys, unique indexes across shards, multi-shard operations) and higher running costs for many use cases.
- Replatform away from MongoDB (PostgreSQL, DynamoDB, OpenSearch)
  - Long-term option that requires broader refactoring; more effort and risk in the near term.

# Architecture overview
- Sharded MongoDB topology on EKS:
  - Shards: multiple data-bearing replica sets
  - Config Server: replica set for metadata
  - mongos: stateless query routers
- Kubernetes primitives:
  - StatefulSets for shard replica sets and config servers
  - PersistentVolumes via a gp3 StorageClass (EBS CSI driver)
- Backups and restore:
  - Percona Backup for MongoDB (PBM) to S3 (weekly full + hourly PITR recommended)
- Observability:
  - Percona Monitoring and Management (PMM): sidecars on nodes, PMM Server with dashboards and alerts
- Security:
  - TLS/SSL in transit; AES256 at rest; IRSA for S3; Kubernetes RBAC; secrets management
- Scaling:
  - Vertical: CPU/memory/PVC size
  - Horizontal: replica counts, mongos/config server sizing, and shard count

# Deployment workflow
## Context and goals
- Authenticate to AWS, configure EKS access, install the Percona Operator via Helm, and apply a PerconaServerMongoDB Custom Resource (CR).
- Persistent storage via gp3 EBS volumes.
- Backups scheduled weekly (full) and hourly (PITR).
- Consistent, repeatable, fully automated from a GitHub repository (Terraform + Helm + CR manifests).

## Layering model
- Dedicated EKS deployment (Terraform)
  - Provision the EKS control plane and managed node groups sized for workload.
  - Install the EBS CSI driver; configure a gp3 StorageClass.
  - Set up IAM Roles for Service Accounts (IRSA) so PBM can access S3.
  - Configure networking (VPC, subnets, security groups).
- Percona MongoDB deployment (Helm + CRDs)
  - Install the Percona Operator via Helm (pin to a known version, for example, 1.20.x).
  - Deploy Percona MongoDB via the Helm chart that defines the PerconaServerMongoDB CR (pin the chart).
  - Pin the MongoDB image tag (for example, 7.0.18-11); enable PMM/PBM sidecars; configure backups.
- PMM Server for monitoring
  - Deploy PMM Server as a pod with persistent storage in the same cluster.
  - Point PMM clients (sidecars) to the PMM Server and verify dashboards and alerts.

## Companion deployment workflow write-up
- CI/CD workflow (GitHub Actions or similar), step-by-step:
  - Deployment Workflow Guide: <add-link-to-your-detailed-workflow-doc>

# Sample Terraform: Helm-based Percona MongoDB on EKS (sanitised)
- Uses the public Percona Helm repository, generic variables, IRSA, and avoids environment-specific references.
- For production, enable TLS (requireSSL) and manage secrets via a secrets manager.

```hcl
# Percona Operator (Helm)
resource "helm_release" "percona_operator" {
  name             = "percona-operator"
  namespace        = var.namespace
  repository       = var.percona_helm_repository            # e.g., "https://percona.github.io/percona-helm-charts"
  chart            = "psmdb-operator"
  version          = var.percona_operator_chart_version     # e.g., "1.20.1"
  create_namespace = true
}

# Percona MongoDB (Helm) – Sharded cluster with PBM + PMM sidecars
resource "helm_release" "percona_mongodb" {
  depends_on = [helm_release.percona_operator]

  name       = "percona-mongodb"
  namespace  = var.namespace
  repository = var.percona_helm_repository                  # public Percona charts
  chart      = "psmdb-db"
  version    = var.percona_db_chart_version                 # e.g., "1.20.1"

  values = [
    yamlencode({
      tls = { mode = "disabled" }                           # set to "requireSSL" in production and configure certs

      enableVolumeExpansion = true

      image = {
        repository = "percona/percona-server-mongodb"
        tag        = var.mongodb_image_tag                  # e.g., "7.0.18-11"
        pullPolicy = "IfNotPresent"
      }

      users = [
        {
          name = var.app_user_name
          db   = var.app_database_name
          passwordSecretRef = { name = var.app_user_secret_name, key = "password" }
          roles = [
            { name = "readWrite", db = var.app_database_name },
            { name = "dbAdmin",   db = var.app_database_name }
          ]
        }
      ]

      systemUsers = {
        MONGODB_BACKUP_USER              = "backup"
        MONGODB_BACKUP_PASSWORD          = var.backup_user_password
        MONGODB_DATABASE_ADMIN_USER      = "databaseAdmin"
        MONGODB_DATABASE_ADMIN_PASSWORD  = var.database_admin_password
        MONGODB_CLUSTER_ADMIN_USER       = "clusterAdmin"
        MONGODB_CLUSTER_ADMIN_PASSWORD   = var.cluster_admin_password
        MONGODB_CLUSTER_MONITOR_USER     = "clusterMonitor"
        MONGODB_CLUSTER_MONITOR_PASSWORD = var.cluster_monitor_password
        MONGODB_USER_ADMIN_USER          = "userAdmin"
        MONGODB_USER_ADMIN_PASSWORD      = var.user_admin_password
        PMM_SERVER_USER                  = "admin"
        PMM_SERVER_PASSWORD              = var.pmm_server_password
      }

      pmm = {
        enabled    = true
        serverHost = var.pmm_server_host                    # e.g., "pmm-server.mongodb.svc.cluster.local"
        image = {
          repository = "percona/pmm-client"
          tag        = var.pmm_client_image_tag
          pullPolicy = "IfNotPresent"
        }
      }

      serviceAccount = { create = false, name = var.irsa_service_account_name }

      backup = {
        enabled = true
        image   = { repository = "percona/percona-backup-mongodb", tag = var.pbm_client_image_tag, pullPolicy = "IfNotPresent" }
        pitr    = { enabled = true, oplogSpanMin = 60 }
        storages = {
          s3 = { type = "s3", s3 = { bucket = var.backup_bucket_name, region = var.aws_region } }
        }
        tasks = [
          { name = "weekly-full", enabled = true, schedule = "0 2 * * 0", keep = 4, storageName = "s3", type = "logical" }
        ]
      }

      sharding = {
        enabled  = true
        balancer = { enabled = true }
        mongos   = {
          size   = 4
          expose = {
            enabled     = true
            type        = "LoadBalancer"
            annotations = {
              "service.beta.kubernetes.io/aws-load-balancer-type"     = "nlb"
              "service.beta.kubernetes.io/aws-load-balancer-internal" = "true"
            }
            ports = [ { port = 27017, targetPort = 27017, protocol = "TCP" } ]
          }
          resources = { requests = { cpu = "2", memory = "8Gi" }, limits = { cpu = "2", memory = "8Gi" } }
        }
        configrs = {
          size      = 3
          resources = { requests = { cpu = "2", memory = "16Gi" }, limits = { cpu = "2", memory = "16Gi" } }
          volumeSpec = {
            pvc = {
              accessModes      = ["ReadWriteOnce"]
              storageClassName = var.storage_class_name              # e.g., "gp3"
              resources        = { requests = { storage = "20Gi" } }
            }
          }
        }
      }

      replsets = [
        for rs_name in var.shard_replset_names : {
          name = rs_name
          size = var.mongodb_replica_size
          arbiter = { enabled = var.mongodb_arbiter_enabled, size = 1 } # PSA reduces resilience; use with care
          storage  = { journal = { enabled = var.storage_journal_enabled } }
          volumeSpec = {
            pvc = {
              accessModes      = ["ReadWriteOnce"]
              storageClassName = var.storage_class_name
              resources        = { requests = { storage = var.pvc_size } }
            }
          }
          resources = {
            requests = { cpu = var.mongodb_replica_cpu, memory = var.mongodb_replica_memory }
            limits   = { cpu = var.mongodb_replica_cpu, memory = var.mongodb_replica_memory }
          }
        }
      ]
    })
  ]
}
```

## Variables to define (replace placeholders)
- namespace
- percona_helm_repository (for example, https://percona.github.io/percona-helm-charts)
- percona_operator_chart_version (for example, 1.20.1)
- percona_db_chart_version (for example, 1.20.1)
- mongodb_image_tag (for example, 7.0.18-11)
- pmm_client_image_tag
- pbm_client_image_tag
- irsa_service_account_name
- backup_bucket_name, aws_region
- storage_class_name (for example, gp3)
- shard_replset_names (for example, ["rs0", "rs1", "rs2"])
- mongodb_replica_size (for example, 3)
- mongodb_arbiter_enabled (true/false)
- storage_journal_enabled (true/false)
- pvc_size (for example, "500Gi")
- mongodb_replica_cpu (for example, "4")
- mongodb_replica_memory (for example, "32Gi")
- app_user_name, app_database_name, app_user_secret_name
- database_admin_password, cluster_admin_password, cluster_monitor_password, user_admin_password, backup_user_password
- pmm_server_host, pmm_server_password

# Data migration strategy
- Bulk import (mongodump/mongorestore) per shard; validate shard keys and unique indexes.
- AWS DMS for collection exports and incremental sync; confirm multi-shard operations and index constraints.
- Pre-cutover checks:
  - Integrity verification (checksums, canary queries)
  - mongos routing validation across shards
  - Rehearsed rollback with restore drills

# Upgrades
- Minor versions: bump container image versions; the Operator orchestrates rolling upgrades.
- Major versions: upgrade one major at a time (for example, 6 → 7 → 8) following compatibility guidance.
- Downgrades: remove incompatible features/settings and validate data format compatibility.
- Change sequencing: config servers → mongos → shards. Always test in non-production first.

# Scaling patterns
- Vertical: adjust CPU/memory and PVC sizes via the CR; enable volume expansion where supported.
- Horizontal: modify replica set sizes, mongos/config server counts; add shards as the dataset and throughput grow.
- Integrate with the cluster autoscaler; be conservative with HPA on StatefulSets; prefer planned CR changes.

# Security hardening
- TLS and certificate management (automatic or custom CA).
- AES256 encryption at rest (Kubernetes Secrets or HashiCorp Vault).
- Authentication: SCRAM/x.509; optional LDAP/Kerberos based on requirements.
- IRSA for S3 access; avoid static credentials in pods.
- Kubernetes RBAC, external secrets manager integration, audit logging, and periodic access reviews.

# Monitoring and observability
- PMM dashboards for:
  - Replication health and lag
  - Query performance and slow operations
  - Node resource utilisation
  - Connections, locks, caches
- Alerts for replication lag thresholds, CPU/memory, disk IOPS/latency, and backup job failures.
- Optional profiling for deep query analysis, index reviews and query plan checks.

# Cost considerations (generic)
- Self-managed EC2 clusters often have a higher total cost of ownership due to manual operations and static capacity.
- Percona on EKS typically enables cost control via right-sizing, modern instance families, and autoscaling patterns.
- Managed services can cost more at a similar capacity for sharded workloads.
- Right-size iteratively using PMM insights; consider gp3 IOPS tuning alongside instance sizing.

# Risks and mitigations
- Operational complexity (Kubernetes + MongoDB)
  - Use Terraform/Helm modules, GitOps workflows, and documented runbooks; start in non-production.
- Upgrades and patching
  - Stage-first testing, rolling upgrades, clear sequencing, maintenance windows.
- Resource tuning and efficiency
  - PMM-driven tuning; regular capacity reviews; avoid over-provisioning; tune gp3 IOPS.
- Disaster recovery
  - Define RPO/RTO; regular restore drills; validate PITR; ensure cross-AZ resilience.
- Compliance
  - Document controls (encryption, RBAC, IRSA, audit logs) and gather evidence for reviews.

# Common pitfalls and how to avoid them
- Shard key design and hot-spotting: validate early; choose keys that distribute load evenly; ensure unique index constraints are met.
- Multi-shard operations: minimise scatter-gather in high-throughput paths; test query patterns via mongos.
- PVC expansion: Use a StorageClass that supports expansion; otherwise, plan controlled migrations to larger volumes.
- IRSA scoping: scope S3 permissions tightly; verify bucket policies; test backup/restore end-to-end.
- mongos/config server sizing: right-size based on connection load and metadata operations; monitor router CPU and latency.
- HPA on StatefulSets: be conservative; validate autoscaler behaviour in staging; prefer planned CR-driven scaling.

# Lessons learned: running Percona MongoDB on EKS
- Deployment and setup
  - The Operator plus Helm/Terraform reduces day-2 toil but requires a solid understanding of CRDs and Kubernetes reconciliation.
  - Sharding via CRD manifests works well; validate the shard and replica configuration before cutover.
- Resource optimisation and cost
  - Low CPU/memory utilisation can mask throughput bottlenecks caused by GP3 IOPS defaults.
  - Increasing GP3 IOPS (for example, higher IOPS tiers) can materially improve throughput and enable instance downsizing.
- Graviton support limitation
  - PMM Server support for ARM/Graviton may be limited depending on the version; this can block Graviton-first strategies until support arrives.
- Secret management and backup dependency
  - Accidental deletion of Kubernetes secrets (authentication credentials, encryption keys) can prevent cluster recovery after a data restore; back up all cluster secrets in a secrets manager alongside data backups.
- Config metadata integrity
  - Applying changes whilst the Operator is reconciling can corrupt the state; wait for Ready before further changes; use health gates in CI/CD.
- PSA configuration (Primary, Secondary, Arbiter)
  - PSA mirrors some legacy patterns but reduces resilience; certain Operator versions require unsafe flags to enable. You can choose fully replicated members when possible; enable PSA only with clear risk acceptance and runbooks.

# Production readiness checklist
- EKS cluster and node groups provisioned; EBS CSI driver installed; gp3 StorageClass available.
- IRSA configured; no static credentials in backup pods.
- Percona Operator installed; PerconaServerMongoDB CR applied and status Ready.
- TLS enabled; AES256 at-rest encryption configured (Secrets or Vault).
- Backups: weekly full + hourly PITR to S3; restore tests validated (including secrets).
- PMM dashboards live; alerts for replication lag, slow queries, backup failures, disk/CPU/memory thresholds.
- Resource requests/limits tuned; PodDisruptionBudgets and affinity/anti-affinity set.
- Runbooks documented: backup/restore, upgrades (minor/major), incident response, DR drills.

# Licence
- Code (Terraform, Helm, manifests, scripts): Apache License 2.0 recommended (permissive with patent grant).
- Documentation (README, guides, diagrams): optionally CC BY 4.0 if preferred.
- Alternatively, license the entire repository under Apache-2.0.

# Contributions
- Contributions are welcome. Share tuning tips, deployment examples, and lessons learned to help others shorten their path to production.
- Please sanitise examples and avoid committing secrets or environment-specific details.

# Public references
- Percona Operator for MongoDB on EKS: https://www.percona.com/doc/kubernetes-operator-for-psmongodb/index.html
- Percona Backup for MongoDB (PBM): https://www.percona.com/software/mongo-database/percona-backup-for-mongodb
- Percona Monitoring and Management (PMM): https://www.percona.com/software/database-tools/percona-monitoring-and-management
- MongoDB Transactions and ACID: https://www.mongodb.com/docs/manual/core/transactions/
- Write Concern: https://www.mongodb.com/docs/manual/reference/write-concern/
- Read Concern: https://www.mongodb.com/docs/manual/reference/read-concern/
- Sharding overview: https://www.mongodb.com/docs/manual/sharding/
- Kubernetes storage: https://kubernetes.io/docs/concepts/storage/persistent-volumes/
- AWS IRSA: https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html
