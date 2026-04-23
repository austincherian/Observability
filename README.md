# Observability for SageMaker HyperPod Slurm Clusters

This standalone sample installs a metrics collection and export pipeline on
SageMaker HyperPod Slurm clusters. It deploys per-node metric exporters and an
OpenTelemetry (OTel) collector that ships metrics to
[Amazon Managed Service for Prometheus (AMP)](https://docs.aws.amazon.com/prometheus/latest/userguide/what-is-Amazon-Managed-Service-Prometheus.html),
where they can be visualized with
[Amazon Managed Grafana](https://docs.aws.amazon.com/grafana/latest/userguide/what-is-Amazon-Managed-Service-Grafana.html).

## What gets installed

The stack is node-type aware. The entrypoint script auto-detects whether a
node is a controller, compute, or login node by checking which Slurm daemon
is running.

| Component | Controller | Compute | Login | Port | Description |
| --- | :---: | :---: | :---: | --- | --- |
| Node Exporter | âœ“ | âœ“ | âœ“ | 9100 | OS-level metrics (CPU, memory, disk, network) |
| DCGM Exporter | | âœ“ | | 9400 | NVIDIA GPU metrics with Slurm job-ID mapping |
| EFA Exporter | | âœ“ | | 9109 | Elastic Fabric Adapter network metrics |
| Slurm Exporter | âœ“ | | | 9341 | Slurm scheduler metrics (jobs, nodes, partitions) |
| OTel Collector | âœ“ | âœ“ | âœ“ | 4317/4318 | Scrapes all local exporters and remote-writes to AMP |


## Prerequisites

> **Note:** The Slurm Exporter installation on the controller node requires
> internet access to download Go and clone the exporter repository from
> GitHub. Ensure your VPC configuration allows outbound internet access
> (e.g., via a NAT gateway) or pre-install the exporter in a custom AMI.

### 1. Enable IAM Identity Center

IAM Identity Center is required for Amazon Managed Grafana authentication.
Follow the instructions in
[Enabling IAM Identity Center](https://docs.aws.amazon.com/singlesignon/latest/userguide/get-set-up-for-idc.html)
and create an admin user account.

### 2. Create the observability infrastructure

Deploy the CloudFormation stack that provisions an Amazon Managed Service for
Prometheus (AMP) workspace and an Amazon Managed Grafana workspace:

```bash
# Download the template
wget https://raw.githubusercontent.com/aws-samples/awsome-distributed-training/main/4.validation_and_observability/4.prometheus-grafana/cluster-observability.yaml

# Deploy the stack
aws cloudformation create-stack \
    --stack-name hyperpod-observability \
    --template-body file://cluster-observability.yaml \
    --capabilities CAPABILITY_NAMED_IAM
```

After the stack completes, note the `AMPRemoteWriteURL` from the outputs:

```bash
aws cloudformation describe-stacks \
    --stack-name hyperpod-observability \
    --query "Stacks[0].Outputs[?OutputKey=='AMPRemoteWriteURL'].OutputValue" \
    --output text
```

This URL goes into `config.json` as `prometheus_remote_write_url`.

If you already have existing AMP and Grafana workspaces, you can pass their
IDs as parameters:

```bash
aws cloudformation create-stack \
    --stack-name hyperpod-observability \
    --template-body file://cluster-observability.yaml \
    --capabilities CAPABILITY_NAMED_IAM \
    --parameters \
        ParameterKey=PrometheusExistingWorkspaceId,ParameterValue=ws-abc123 \
        ParameterKey=GrafanaExistingWorkspaceId,ParameterValue=g-xyz789
```

### 3. Add AMP remote write permissions to the cluster execution role

The HyperPod cluster execution role must have permissions to remote-write
metrics to AMP. Attach a policy like:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "aps:RemoteWrite"
            ],
            "Resource": "arn:aws:aps:<region>:<account-id>:workspace/<workspace-id>"
        }
    ]
}
```

### 4. Import Grafana dashboards

After the cluster is running and exporting metrics, open your Grafana
workspace URL and import the following dashboards via
**Dashboards â†’ New â†’ Import**:

| Dashboard | URL |
| --- | --- |
| Slurm Dashboard | `https://grafana.com/grafana/dashboards/4323-slurm-dashboard/` |
| Node Exporter Full | `https://grafana.com/grafana/dashboards/1860-node-exporter-full/` |
| NVIDIA DCGM Exporter | `https://grafana.com/grafana/dashboards/12239-nvidia-dcgm-exporter-dashboard/` |
| EFA Metrics | `https://grafana.com/grafana/dashboards/20579-efa-metrics-dev/` |
| FSx for Lustre Metrics | `https://grafana.com/grafana/dashboards/20906-fsx-lustre/` |

Ensure that the AMP workspace is configured as a Prometheus data source in
Grafana. Navigate to **Apps â†’ AWS Data Sources â†’ Data sources**, select
your region, and choose the AMP workspace.

## Usage

### Option A: OnInitComplete (recommended for new clusters)

Use this when creating a cluster with AMI-based configuration. HyperPod sets
up Slurm and Docker automatically, then runs your extension script.

1. Edit `config.json` with your AMP remote write URL:
   ```json
   {
       "prometheus_remote_write_url": "https://aps-workspaces.us-west-2.amazonaws.com/workspaces/ws-abc123/api/v1/remote_write",
       "advanced_metrics": false,
       "nccl_metrics_enabled": false,
       "nccl_metrics_dump_interval_seconds": 30,
       "nccl_profiler_plugin_path": "/opt/aws/hyperpod/observability/lib/libnccl-profiler-inspector.so"
   }
   ```

2. Upload to S3:
   ```bash
   aws s3 sync ./observability s3://sagemaker-<your-bucket>/observability/
   ```

3. Reference in your `CreateCluster` request. Specify `OnInitComplete` on
   each instance group:
   ```json
   {
       "InstanceGroupName": "worker-group-1",
       "InstanceType": "ml.p4d.24xlarge",
       "InstanceCount": 4,
       "SlurmConfig": {
           "NodeType": "Compute",
           "PartitionNames": ["gpu-training"]
       },
       "LifeCycleConfig": {
           "OnInitComplete": "setup_observability.sh",
           "SourceS3Uri": "s3://sagemaker-<your-bucket>/observability/"
       },
       "ExecutionRole": "arn:aws:iam::<account-id>:role/HyperPodExecutionRole"
   }
   ```

### Option B: Manual execution on a running cluster

SSH into each node and run:
```bash
sudo bash /path/to/setup_observability.sh
```

### Option C: OnCreate (legacy lifecycle scripts)

If you are using full custom lifecycle scripts with `OnCreate`, you can call
`setup_observability.sh` from your `lifecycle_script.py` or `on_create.sh`
after Slurm has been started.

## Configuration

All configuration is in `config.json`:

| Field | Type | Default | Description |
| --- | --- | --- | --- |
| `prometheus_remote_write_url` | string | (required) | AMP workspace remote write endpoint |
| `advanced_metrics` | bool | `false` | Enable extended DCGM GPU metrics and additional Node Exporter collectors |
| `nccl_metrics_enabled` | bool | `false` | Enable NCCL Inspector metrics via Slurm task prolog |
| `nccl_metrics_dump_interval_seconds` | int | `30` | NCCL metrics dump interval |
| `nccl_profiler_plugin_path` | string | `/opt/aws/hyperpod/observability/lib/libnccl-profiler-inspector.so` | Path to the NCCL Inspector `.so` plugin |

### NCCL metrics note

When `nccl_metrics_enabled` is set to `true`, the setup script configures a
Slurm task prolog that injects NCCL Inspector environment variables into every
job. During a multi-GPU NCCL job, the Inspector plugin writes
Prometheus-format metrics to `/var/lib/node_exporter/nccl_inspector/` on
compute nodes, where Node Exporter's textfile collector picks them up and
the OTel Collector ships them to AMP.

NCCL metrics require:
- The NCCL Inspector profiler plugin (`.so`) installed on compute nodes.
  The HyperPod AMI includes the plugin pre-built at
  `/opt/aws/hyperpod/observability/lib/libnccl-profiler-inspector.so`.
  If using a custom AMI, build from NCCL source (post-v2.28.3) at
  `plugins/profiler/inspector/`. See the
  [NCCL Inspector source](https://github.com/NVIDIA/nccl/tree/master/plugins/profiler/inspector).
- A running multi-GPU distributed training job that uses NCCL collectives.
  Metrics are only produced while NCCL operations are active.

There is no pre-built Grafana dashboard for NCCL Inspector metrics. The
metrics appear in AMP with `nccl_` prefixed names and can be queried via
Grafana's Explore view using PromQL, or visualized by building a custom
dashboard.

## Stopping observability

To stop all observability containers and services on a node:
```bash
# Detect node type and stop
sudo python3 stop_observability.py --node-type controller  # or compute, login
```

## Validating the setup

After the cluster reaches `InService` status, connect to a node and verify
the exporters are running.

On the controller node:
```bash
# Check Node Exporter
curl -s http://localhost:9100/metrics | head -5

# Check Slurm Exporter
curl -s http://localhost:9341/metrics | grep slurm | head -5

# Check OTel Collector
docker ps --filter name=otel-collector
```

On a compute node:
```bash
# Check Node Exporter
curl -s http://localhost:9100/metrics | head -5

# Check DCGM Exporter (GPU metrics)
curl -s http://localhost:9400/metrics | grep DCGM | head -5

# Check EFA Exporter
curl -s http://localhost:9109/metrics | grep efa | head -5

# Check OTel Collector
docker ps --filter name=otel-collector
```

To verify metrics are reaching AMP, open your Grafana workspace and run a
PromQL query like `up` in the Explore view. You should see targets from
your cluster nodes.

## File structure

```
observability/
â”œâ”€â”€ README.md                            # This file
â”œâ”€â”€ config.json                          # User configuration
â”œâ”€â”€ setup_observability.sh               # Entrypoint script (OnInitComplete)
â”œâ”€â”€ install_observability.py             # Orchestrator
â”œâ”€â”€ install_node_exporter.sh             # Node Exporter (all nodes)
â”œâ”€â”€ install_dcgm_exporter.sh             # DCGM Exporter (compute nodes)
â”œâ”€â”€ install_efa_exporter.sh              # EFA Exporter (compute nodes)
â”œâ”€â”€ install_slurm_exporter.sh            # Slurm Exporter (controller node)
â”œâ”€â”€ install_otel_collector.sh            # OTel Collector (all nodes)
â”œâ”€â”€ stop_observability.py                # Stop all observability services
â”œâ”€â”€ LICENSE_SLURM_EXPORTER.txt           # License for Slurm Exporter dependency
â”œâ”€â”€ otel_config/
â”‚   â”œâ”€â”€ config-head-template.yaml        # OTel config for controller
â”‚   â”œâ”€â”€ config-compute-template.yaml     # OTel config for compute
â”‚   â””â”€â”€ config-login-template.yaml       # OTel config for login
â””â”€â”€ dcgm_metrics_config/
    â”œâ”€â”€ dcgm-metrics-basic.csv           # Basic DCGM metrics
    â””â”€â”€ dcgm-metrics-advanced.csv        # Advanced DCGM metrics
```

## Related resources

- [SageMaker HyperPod cluster resources monitoring](https://docs.aws.amazon.com/sagemaker/latest/dg/sagemaker-hyperpod-cluster-observability-slurm.html)
- [Slurm Dashboard for Grafana](https://grafana.com/grafana/dashboards/4323-slurm-dashboard/)
- [NVIDIA DCGM Exporter Dashboard for Grafana](https://grafana.com/grafana/dashboards/12239-nvidia-dcgm-exporter-dashboard/)
- [EFA Metrics Dashboard for Grafana](https://grafana.com/grafana/dashboards/20579-efa-metrics-dev/)
