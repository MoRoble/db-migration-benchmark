## Migrating an On-Premises MariaDB Database to Amazon RDS and Amazon Aurora: A Comparative Evaluation of Performance, Cost, and Downtime

Infrastructure as Code (Terraform), benchmarking automation (`sysbench`), and empirical dataset evaluating **Amazon RDS for MySQL (Multi-AZ)** and **Amazon Aurora MySQL** as migration targets for a self-managed **MariaDB** database using **AWS Database Migration Service (DMS)**.

*Conducted by **Mohamed Roble** as part of an MSc in Cloud Computing (Distinction, 90%) and currently being developed into a conference paper.*

![Experimental AWS Architecture](docs/architecture.png)

### Key Empirical Findings (`db.t3.medium`, Default Parameter Groups)

All three environments were benchmarked across a ~5.94 GB working set (25 tables × 1,000,000 rows, sized to exceed the InnoDB buffer pool and exercise storage I/O) under read-only, write-only, and mixed OLTP profiles at 8, 32, and 64 concurrent threads:

* **Full-Load Migration Speed:** Amazon RDS completed the 5.94 GB full load **2.6x faster** than Aurora (**3m 10s** vs. **8m 27s**), driven by Aurora's 6-copy storage quorum write amplification across 3 Availability Zones.
* **Throughput & Price-Performance:** RDS delivered **10% to 79% higher sustained throughput** across every tested workload profile and concurrency level, resulting in **23% to 37% lower cost per million transactions**.
* **High Availability & Failover:** Aurora recovered from manually triggered failover **2.6x faster** than RDS Multi-AZ (**7.1–8.1s** vs. **19.2–21.2s**), as promotion attaches an existing reader to the shared storage volume without crash recovery.
* **Three Undocumented Operational Findings:**
  1. **MariaDB CDC Crash:** MariaDB's default `binlog_annotate_row_events=ON` fatally crashes the AWS DMS Change Data Capture (CDC) binary log parser; full load succeeds, but CDC requires `binlog_annotate_row_events=OFF` and `binlog_checksum=NONE`.
  2. **DMS Replication Convergence:** Under default transactional apply settings, DMS capped apply throughput at **~420 rows/s** and failed to converge under sustained write load (lag exceeding 7,178s). Enabling `BatchApplyEnabled` increased apply throughput **18x to ~7,705 rows/s**, holding steady-state lag under 8 seconds.
  3. **Engine Version Surcharges:** Provisioning RDS on MySQL `8.0.46` incurred an AWS Extended Support surcharge equal to **2.5x the underlying compute instance cost** ($206.12 surcharge vs. $82.01 comparable compute/storage over the 20-day test window).

---

## Repository Structure

| Path | Contents |
| :--- | :--- |
| `terraform/` | Modular Infrastructure as Code (`vpc`, `security`, `compute`, `database`, `dms`) |
| `scripts/` | Automated `sysbench` benchmark sweeps, 1-second failover probes, and CSV parsers |
| `results/` | Summarised CSV metrics and raw `sysbench` output across 81 benchmark runs |
| `dataset/` | Dataset generation instructions and schema validation queries (`dataset/DATASET.md`) |

---

## Experimental Environment & Security Architecture

All resources are provisioned in AWS `eu-west-2` (London) using `t3.medium` / `db.t3.medium` instances for a like-for-like hardware comparison:
* **Source:** Self-managed MariaDB `10.11.14` on Amazon EC2 (`2.00 GB` InnoDB buffer pool, ROW binary logging).
* **Target 1:** Amazon RDS for MySQL `8.0.46` in a Multi-AZ deployment (`2.00 GB` default buffer pool).
* **Target 2:** Amazon Aurora MySQL `3.10.5` (`8.0.mysql_aurora.3.10.5`) cluster with 1 writer and 1 reader (`1.46 GB` default buffer pool).
* **Migration & Load:** AWS DMS replication instance and a co-located EC2 `sysbench` client inside the same Availability Zone to eliminate cross-AZ network jitter.

### Zero-Trust Access Model (`terraform/`)
* **No Public Access or SSH Keys:** EC2 instances have no public IPs and no security group permits inbound port `22`. All administration is performed via **AWS Systems Manager Session Manager**.
* **Security Group Chaining:** Database security groups permit inbound port `3306` exclusively from the security group IDs of the DMS replication instance and the benchmark client.
* **Secrets Management:** Database master credentials are generated and managed automatically via **AWS Secrets Manager** (`manage_master_user_password = true`), keeping plaintext secrets out of Terraform code and state.

### Deploying the Stack
Remote state is configured for HCP Terraform with local execution. To deploy:

```bash
cd terraform
terraform init
terraform plan
terraform apply
