# Boardgame Jenkins CI/CD Pipeline

A production-style Jenkins CI/CD pipeline for a Java/Maven application, deployed to a self-managed Kubernetes cluster with enforced quality gates, security scanning, RBAC-scoped deployment access, and full observability.

Built end-to-end on self-provisioned AWS infrastructure — no managed CI/CD or managed Kubernetes control plane. Base application forked and adapted from a public reference project; infrastructure, pipeline, RBAC, hardening decisions, and monitoring stack are original work.

## Architecture

```
GitHub (private repo, Jenkinsfile as code)
        │
        ▼
   Jenkins (EC2) ──► Trivy (fs + image scan)
        │           ► SonarQube + Postgres (quality gate, enforced)
        │           ► Nexus (Maven artifact repository)
        │           ► Docker Hub (image registry)
        │
        ▼
Kubernetes cluster (kubeadm, 1 master + 2 workers)
   RBAC-scoped ServiceAccount → namespaced Role → deployment
        │
        ▼
Prometheus + Grafana + Blackbox Exporter (dedicated monitoring server)
```

7 dedicated EC2 servers — Jenkins, Nexus, SonarQube, Kubernetes master, 2 Kubernetes workers, monitoring — mirroring how larger orgs isolate these tools rather than running them on one box.

## Pipeline Stages

1. Git Checkout
2. Compile (`mvn compile`)
3. Test (`mvn test`)
4. File System Scan — Trivy, dependency/source vulnerability scan
5. SonarQube Analysis — static code analysis
6. **Quality Gate** — enforced (`abortPipeline: true`); a failing gate blocks the pipeline
7. Build (`mvn package`)
8. Publish to Nexus (`mvn deploy`)
9. Build & Tag Docker Image — tagged with `BUILD_NUMBER` for traceability
10. Docker Image Scan — Trivy, container image scan
11. Push Docker Image
12. Deploy to Kubernetes — via RBAC-scoped ServiceAccount token
13. Verify Deployment

Post-pipeline: HTML email notification (pass/fail banner, attached scan reports) on every run, regardless of outcome.

## Key Engineering Decisions

- **kubeadm cluster, not managed EKS** — built the control plane, CNI (Calico), and CRI setup manually to understand what a managed Kubernetes service abstracts away, rather than only knowing the managed-service version.
- **External Postgres for SonarQube**, not the bundled H2 database — SonarSource documents H2 as unsupported for production use.
- **Namespace-scoped RBAC for Jenkins**, not cluster-admin — a dedicated ServiceAccount bound to a `Role` (not `ClusterRole`) with only the permissions needed to manage resources in one namespace.
- **Quality Gate enforced, not report-only** — `abortPipeline: true` means a failing SonarQube gate stops the deploy, not just logs a warning. Validated live: a run failed on a 79% code-duplication finding, traced to the pipeline's own Trivy scan reports being swept into the analysis scope — fixed via `sonar.exclusions`, not by lowering the gate's standard.
- **Docker images tagged by build number**, not `:latest` — every deployed image is traceable back to the exact Jenkins run that produced it.
- **Pipeline defined as code** — Jenkinsfile lives in this repo (`Pipeline script from SCM`), not configured only in Jenkins' UI, for version history and portability.

## Security & Observability

- **Trivy** — filesystem and container image vulnerability scanning, both gating the pipeline
- **SonarQube** — static analysis + enforced quality gate
- **kubeaudit / kube-bench** — cluster configuration audited against the CIS Kubernetes Benchmark; findings triaged to distinguish genuine gaps from expected infrastructure behavior (e.g. Calico and kube-proxy require privileged/host-network access by design)
- **Prometheus + Grafana** — cluster and service metrics via Node Exporter and the Jenkins Prometheus plugin
- **Blackbox Exporter** — external HTTP uptime probing of Jenkins, SonarQube, and the deployed application

## Stack

Jenkins · Kubernetes (kubeadm) · Docker · Maven · SonarQube · PostgreSQL · Nexus · Trivy · Prometheus · Grafana · Blackbox Exporter · AWS EC2
