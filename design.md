# Veltrix – Energy Fingerprint Security Engine
## System Design Document

## 1. System Architecture Overview

### 1.1 High-Level Architecture

Veltrix follows a distributed edge-cloud architecture:

```
┌─────────────────────────────────────────────────────────────┐
│                    Customer AWS Account                      │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  EC2 Instance (GPU-enabled)                            │ │
│  │  ┌──────────────────────────────────────────────────┐  │ │
│  │  │  Edge Telemetry Collector (systemd service)      │  │ │
│  │  │  - NVIDIA-SMI poller                              │  │ │
│  │  │  - RAPL reader                                    │  │ │
│  │  │  - Process monitor                                │  │ │
│  │  │  - Local buffer (ring buffer)                     │  │ │
│  │  └──────────────────────────────────────────────────┘  │ │
│  └────────────────────────────────────────────────────────┘ │
│                           │                                  │
│                           │ Kinesis Producer                 │
│                           ▼                                  │
└─────────────────────────────────────────────────────────────┘
                            │
                            │ TLS 1.2+
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                    Veltrix SaaS Platform                     │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  Kinesis Data Streams (telemetry ingestion)          │  │
│  └──────────────────────────────────────────────────────┘  │
│                           │                                  │
│                           ▼                                  │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  Lambda: Time-Series Transformation Engine           │  │
│  │  - FFT computation                                    │  │
│  │  - Spectral density extraction                        │  │
│  │  - Z-score normalization                              │  │
│  └──────────────────────────────────────────────────────┘  │
│                           │                                  │
│                           ▼                                  │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  SageMaker Endpoint: LSTM Autoencoder                │  │
│  │  - Real-time inference                                │  │
│  │  - Anomaly score generation                           │  │
│  └──────────────────────────────────────────────────────┘  │
│                           │                                  │
│                           ▼                                  │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  Lambda: Symbolic Policy Engine                      │  │
│  │  - Threshold evaluation                               │  │
│  │  - Policy lookup (DynamoDB)                           │  │
│  │  - Decision: alert / throttle / terminate             │  │
│  └──────────────────────────────────────────────────────┘  │
│                           │                                  │
│              ┌────────────┴────────────┐                     │
│              ▼                         ▼                     │
│  ┌─────────────────────┐   ┌─────────────────────────────┐ │
│  │  SNS: Alert         │   │  Lambda: Kill Switch        │ │
│  │  Notifications      │   │  Executor                   │ │
│  └─────────────────────┘   │  - SSM Run Command          │ │
│                             │  - Process termination      │ │
│                             └─────────────────────────────┘ │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  Data Layer                                           │  │
│  │  - DynamoDB: Fingerprint registry, policies, events  │  │
│  │  - S3: Raw telemetry, audit logs                     │  │
│  │  - CloudWatch: Metrics, logs                         │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  React Dashboard (Amplify Hosting)                   │  │
│  │  - Real-time anomaly view                            │  │
│  │  - Fingerprint management                            │  │
│  │  - Policy configuration                              │  │
│  └──────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```


### 1.2 Component Responsibilities

| Component | Responsibility | Technology |
|-----------|---------------|------------|
| Edge Telemetry Collector | Capture GPU/CPU power metrics, buffer locally, stream to cloud | Python 3.11+, systemd |
| Time-Series Transformation Engine | Apply FFT, extract spectral features, normalize | AWS Lambda (Python) |
| Neural Fingerprinting | Compute anomaly scores via LSTM Autoencoder | Amazon SageMaker |
| Symbolic Policy Engine | Evaluate policies, make enforcement decisions | AWS Lambda (Python) |
| Kill Switch Executor | Terminate processes via SSM Run Command | AWS Lambda + SSM |
| Fingerprint Registry | Store golden signatures | DynamoDB |
| Telemetry Archive | Store raw telemetry for forensics | S3 |
| Dashboard | Visualize anomalies, manage policies | React + Amplify |

### 1.3 Data Flow

1. **Telemetry Collection** (every 10ms): Edge collector polls NVIDIA-SMI and RAPL
2. **Buffering** (100ms-1s): Local ring buffer accumulates samples for sliding window
3. **Streaming** (every 1s): Batch upload to Kinesis Data Streams
4. **Transformation** (Lambda trigger): FFT → spectral density → normalization
5. **Inference** (SageMaker): LSTM Autoencoder computes reconstruction error
6. **Policy Evaluation** (Lambda): Compare anomaly score to threshold, lookup policy
7. **Enforcement** (Lambda + SSM): Execute alert/throttle/terminate action
8. **Logging** (S3 + DynamoDB): Persist events and audit trail

---

## 2. Edge Telemetry Collector Design

### 2.1 Architecture

The edge collector is a Python daemon running as a systemd service on each monitored EC2 instance.

**Key Design Decisions:**
- **Polling vs Kernel Module**: MVP uses NVIDIA-SMI polling (10ms interval) to avoid kernel module complexity. Phase 2 may introduce eBPF/kernel module for sub-millisecond sampling.
- **Local Buffering**: Ring buffer prevents data loss during network interruptions.
- **Fail-Open**: Collector failures do not impact monitored workloads.


### 2.2 Telemetry Sources

#### 2.2.1 GPU Telemetry (NVIDIA-SMI)
```bash
nvidia-smi --query-gpu=timestamp,power.draw,utilization.gpu,utilization.memory,temperature.gpu,clocks.sm,clocks.mem --format=csv,noheader,nounits
```

**Output Example:**
```
2026/02/15 10:23:45.123, 250.5, 95, 80, 72, 1410, 5001
```

**Parsed Metrics:**
- `power.draw`: GPU power consumption (Watts)
- `utilization.gpu`: GPU compute utilization (%)
- `utilization.memory`: GPU memory utilization (%)
- `temperature.gpu`: GPU temperature (°C)
- `clocks.sm`: SM clock frequency (MHz)
- `clocks.mem`: Memory clock frequency (MHz)

#### 2.2.2 CPU Power (Intel RAPL)
```python
# Read from /sys/class/powercap/intel-rapl/intel-rapl:0/energy_uj
# Compute power: ΔE / Δt
```

**Calculation:**
```python
energy_uj_t1 = read_rapl_energy()
time.sleep(0.01)  # 10ms
energy_uj_t2 = read_rapl_energy()
power_watts = (energy_uj_t2 - energy_uj_t1) / 10000  # Convert µJ to W
```

#### 2.2.3 System Metrics (hwmon)
```python
# /sys/class/hwmon/hwmon*/temp*_input (CPU temperature)
# /sys/class/hwmon/hwmon*/in*_input (voltage)
```

### 2.3 Data Structure

**Telemetry Record (JSON):**
```json
{
  "timestamp": "2026-02-15T10:23:45.123Z",
  "instance_id": "i-0abc123def456",
  "tenant_id": "tenant-xyz",
  "gpu": {
    "power_w": 250.5,
    "utilization_pct": 95,
    "memory_utilization_pct": 80,
    "temperature_c": 72,
    "sm_clock_mhz": 1410,
    "mem_clock_mhz": 5001
  },
  "cpu": {
    "power_w": 85.2,
    "temperature_c": 68,
    "frequency_mhz": 3200
  },
  "processes": [
    {
      "pid": 12345,
      "name": "python3",
      "cmdline": "python3 train.py",
      "gpu_memory_mb": 8192
    }
  ]
}
```


### 2.4 Buffering Strategy

**Ring Buffer Implementation:**
- Fixed-size circular buffer (1000 samples = 10 seconds at 10ms sampling)
- Overwrite oldest samples when full
- Flush to Kinesis every 1 second (100 samples per batch)

**Memory Footprint:**
- 1 sample ≈ 500 bytes (JSON)
- 1000 samples = 500 KB
- Total collector memory: <500 MB (including Python runtime)

### 2.5 Kinesis Producer

**Batching:**
- Batch size: 100 records (1 second of data)
- Max batch size: 500 KB (Kinesis limit: 1 MB)
- Compression: gzip (reduces payload by ~70%)

**Error Handling:**
- Retry with exponential backoff (max 3 retries)
- Drop oldest data if buffer full and Kinesis unavailable
- Emit CloudWatch metric: `telemetry_dropped_records`

### 2.6 Deployment

**Installation:**
```bash
# Install via user-data script or SSM
curl -O https://veltrix-artifacts.s3.amazonaws.com/collector/install.sh
sudo bash install.sh --tenant-id <TENANT_ID> --api-key <API_KEY>
```

**Systemd Service:**
```ini
[Unit]
Description=Veltrix Energy Telemetry Collector
After=network.target

[Service]
Type=simple
User=veltrix
ExecStart=/usr/local/bin/veltrix-collector --config /etc/veltrix/config.yaml
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

---

## 3. Time-Series Transformation Engine

### 3.1 Architecture

Lambda function triggered by Kinesis Data Streams. Processes batches of telemetry records and extracts spectral features.

**Lambda Configuration:**
- Runtime: Python 3.11
- Memory: 1024 MB
- Timeout: 60 seconds
- Concurrency: 100 (auto-scaling)


### 3.2 Signal Processing Pipeline

#### Step 1: Sliding Window Extraction
```python
# Extract 1-second sliding window (100 samples at 10ms)
window_size = 100
power_series = [record['gpu']['power_w'] for record in telemetry_batch]
```

#### Step 2: Fast Fourier Transform (FFT)
```python
import numpy as np
from scipy.fft import fft, fftfreq

# Apply FFT
fft_values = fft(power_series)
fft_freq = fftfreq(len(power_series), d=0.01)  # 10ms sampling

# Extract magnitude spectrum (ignore phase)
magnitude_spectrum = np.abs(fft_values)
```

#### Step 3: Spectral Density Extraction
```python
# Power spectral density (PSD)
psd = (magnitude_spectrum ** 2) / len(power_series)

# Focus on 0-50 Hz range (relevant for workload patterns)
freq_mask = (fft_freq >= 0) & (fft_freq <= 50)
psd_filtered = psd[freq_mask]
```

#### Step 4: Feature Vector Construction
```python
# Compute statistical features
features = {
    'mean_power': np.mean(power_series),
    'std_power': np.std(power_series),
    'peak_freq': fft_freq[np.argmax(psd_filtered)],
    'spectral_centroid': np.sum(fft_freq[freq_mask] * psd_filtered) / np.sum(psd_filtered),
    'spectral_rolloff': compute_rolloff(fft_freq[freq_mask], psd_filtered, 0.85),
    'psd_vector': psd_filtered.tolist()  # 50-dimensional vector
}
```

#### Step 5: Z-Score Normalization
```python
# Normalize per workload type (lookup from DynamoDB)
golden_signature = get_golden_signature(tenant_id, workload_id)
normalized_features = {
    'mean_power': (features['mean_power'] - golden_signature['mean_power_mean']) / golden_signature['mean_power_std'],
    'std_power': (features['std_power'] - golden_signature['std_power_mean']) / golden_signature['std_power_std'],
    # ... normalize all features
}
```

### 3.3 Output

**Transformed Record (sent to SageMaker):**
```json
{
  "timestamp": "2026-02-15T10:23:46.123Z",
  "instance_id": "i-0abc123def456",
  "tenant_id": "tenant-xyz",
  "process_id": 12345,
  "features": {
    "mean_power": 0.85,
    "std_power": 0.12,
    "peak_freq": 12.5,
    "spectral_centroid": 18.3,
    "spectral_rolloff": 35.2,
    "psd_vector": [0.1, 0.15, 0.2, ...]
  }
}
```

---

## 4. Neural Fingerprinting Architecture

### 4.1 LSTM Autoencoder Design

**Model Architecture:**
```
Input Layer: 55 features (5 statistical + 50 PSD bins)
    ↓
LSTM Encoder: 128 units → 64 units → 32 units (latent space)
    ↓
LSTM Decoder: 32 units → 64 units → 128 units
    ↓
Output Layer: 55 features (reconstruction)
```


**Training Objective:**
```python
# Minimize reconstruction error on authorized workloads
loss = mean_squared_error(input_features, reconstructed_features)
```

**Anomaly Score Computation:**
```python
# Reconstruction error for test sample
reconstruction_error = np.linalg.norm(input_features - reconstructed_features)

# Baseline error (from golden signature training)
baseline_error = golden_signature['baseline_reconstruction_error']

# Anomaly score (0-1 scale, higher = more anomalous)
anomaly_score = min(reconstruction_error / (baseline_error * 3), 1.0)
```

### 4.2 Golden Signature Training

**Training Process:**
1. Collect 5 minutes of clean workload execution (3000 samples)
2. Split: 80% training, 20% validation
3. Train LSTM Autoencoder for 50 epochs
4. Compute baseline reconstruction error on validation set
5. Store golden signature in DynamoDB

**Golden Signature Schema:**
```json
{
  "tenant_id": "tenant-xyz",
  "workload_id": "pytorch-training-v1",
  "created_at": "2026-02-15T10:00:00Z",
  "hardware_profile": {
    "gpu_model": "NVIDIA A100",
    "cpu_model": "Intel Xeon Platinum 8375C"
  },
  "statistical_params": {
    "mean_power_mean": 245.3,
    "mean_power_std": 12.5,
    "std_power_mean": 8.2,
    "std_power_std": 1.5
  },
  "baseline_reconstruction_error": 0.08,
  "model_version": "v1.0",
  "s3_model_path": "s3://veltrix-models/tenant-xyz/pytorch-training-v1/model.tar.gz"
}
```

### 4.3 SageMaker Deployment

**Endpoint Configuration:**
- Instance type: ml.g4dn.xlarge (1 GPU, 16 GB RAM)
- Auto-scaling: 1-10 instances based on invocation rate
- Model format: TensorFlow SavedModel or PyTorch TorchScript

**Inference API:**
```python
import boto3

sagemaker_runtime = boto3.client('sagemaker-runtime')

response = sagemaker_runtime.invoke_endpoint(
    EndpointName='veltrix-fingerprint-model',
    ContentType='application/json',
    Body=json.dumps({
        'features': normalized_features,
        'golden_signature_id': 'pytorch-training-v1'
    })
)

result = json.loads(response['Body'].read())
# {'anomaly_score': 0.12, 'reconstruction_error': 0.096}
```

### 4.4 Model Versioning & A/B Testing

**Versioning Strategy:**
- Models tagged with semantic versioning (v1.0, v1.1, etc.)
- Blue-green deployment for zero-downtime updates
- Rollback capability within 5 minutes

**A/B Testing:**
- Route 10% of traffic to new model version
- Compare false positive/negative rates over 24 hours
- Promote to 100% if metrics improve by >5%

---

## 5. Symbolic Policy Engine

### 5.1 Architecture

Lambda function that evaluates anomaly scores against tenant-specific policies and makes enforcement decisions.


### 5.2 Policy Schema

**Policy Record (DynamoDB):**
```json
{
  "tenant_id": "tenant-xyz",
  "policy_id": "default-policy",
  "rules": [
    {
      "condition": "anomaly_score > 0.85",
      "action": "terminate",
      "require_manual_approval": false
    },
    {
      "condition": "anomaly_score > 0.70 AND anomaly_score <= 0.85",
      "action": "alert",
      "notification_channels": ["email", "sns"]
    },
    {
      "condition": "anomaly_score > 0.50 AND anomaly_score <= 0.70",
      "action": "log_only"
    }
  ],
  "whitelisted_processes": [
    {"binary_hash": "sha256:abc123...", "description": "Authorized training script"}
  ],
  "enabled": true
}
```

### 5.3 Decision Logic

```python
def evaluate_policy(anomaly_score, process_info, tenant_id):
    # 1. Check whitelist
    if is_whitelisted(process_info, tenant_id):
        return {'action': 'allow', 'reason': 'whitelisted'}
    
    # 2. Lookup policy
    policy = get_policy(tenant_id)
    
    # 3. Evaluate rules in order
    for rule in policy['rules']:
        if evaluate_condition(rule['condition'], anomaly_score):
            return {
                'action': rule['action'],
                'require_approval': rule.get('require_manual_approval', False),
                'notification_channels': rule.get('notification_channels', [])
            }
    
    # 4. Default: log only
    return {'action': 'log_only'}
```

### 5.4 Action Types

| Action | Description | Implementation |
|--------|-------------|----------------|
| `allow` | No action, process continues | No-op |
| `log_only` | Record event to DynamoDB/S3 | Write to audit log |
| `alert` | Send notification via SNS | Publish to SNS topic |
| `throttle` | Reduce process priority (Phase 2) | `renice` command via SSM |
| `terminate` | Kill process immediately | Invoke Kill Switch Lambda |

---

## 6. Kill Switch Execution Layer

### 6.1 Architecture

Lambda function that executes process termination via AWS Systems Manager (SSM) Run Command.

**Why SSM Run Command?**
- Secure: No SSH keys or open ports required
- Auditable: All commands logged to CloudWatch
- Scalable: Supports 1000+ concurrent executions


### 6.2 Termination Flow

```python
import boto3

ssm = boto3.client('ssm')

def terminate_process(instance_id, process_id, anomaly_score):
    # 1. Send kill command via SSM
    response = ssm.send_command(
        InstanceIds=[instance_id],
        DocumentName='AWS-RunShellScript',
        Parameters={
            'commands': [
                f'sudo kill -9 {process_id}',
                f'ps -p {process_id} || echo "Process terminated"'
            ]
        },
        TimeoutSeconds=10
    )
    
    command_id = response['Command']['CommandId']
    
    # 2. Wait for command completion (max 10 seconds)
    waiter = ssm.get_waiter('command_executed')
    waiter.wait(
        CommandId=command_id,
        InstanceId=instance_id,
        WaiterConfig={'Delay': 1, 'MaxAttempts': 10}
    )
    
    # 3. Verify termination
    output = ssm.get_command_invocation(
        CommandId=command_id,
        InstanceId=instance_id
    )
    
    # 4. Log to audit trail
    log_enforcement_action({
        'timestamp': datetime.utcnow().isoformat(),
        'instance_id': instance_id,
        'process_id': process_id,
        'anomaly_score': anomaly_score,
        'action': 'terminate',
        'command_id': command_id,
        'status': output['Status']
    })
    
    return output['Status'] == 'Success'
```

### 6.3 Retry Logic

```python
def terminate_with_retry(instance_id, process_id, anomaly_score, max_retries=3):
    for attempt in range(max_retries):
        success = terminate_process(instance_id, process_id, anomaly_score)
        if success:
            return True
        time.sleep(2 ** attempt)  # Exponential backoff
    
    # Escalate to manual intervention
    send_alert(f"Failed to terminate process {process_id} after {max_retries} attempts")
    return False
```

### 6.4 Manual Override

**Override Mechanism:**
- Dashboard provides "Pause Kill Switch" button
- Sets `kill_switch_enabled=false` in DynamoDB policy
- All termination requests blocked until re-enabled
- Override logged to audit trail

---

## 7. AWS Cloud Integration

### 7.1 Service Mapping

| Requirement | AWS Service | Configuration |
|-------------|-------------|---------------|
| Telemetry ingestion | Kinesis Data Streams | 1 shard per 1 MB/s, auto-scaling enabled |
| Signal processing | Lambda | Python 3.11, 1024 MB, Kinesis trigger |
| Model inference | SageMaker | ml.g4dn.xlarge, auto-scaling 1-10 instances |
| Policy evaluation | Lambda | Python 3.11, 512 MB, async invocation |
| Process termination | Lambda + SSM | Python 3.11, 256 MB, SSM Run Command |
| Fingerprint storage | DynamoDB | On-demand billing, GSI on workload_id |
| Telemetry archive | S3 | Intelligent-Tiering, 30-day lifecycle |
| Audit logs | S3 | Standard storage, 90-day retention |
| Metrics | CloudWatch | Custom metrics, 15-month retention |
| Alerts | SNS | Email/SMS notifications |
| Dashboard | Amplify | React SPA, Cognito auth |


### 7.2 IAM Roles & Policies

#### Edge Collector Role
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "kinesis:PutRecord",
        "kinesis:PutRecords"
      ],
      "Resource": "arn:aws:kinesis:*:*:stream/veltrix-telemetry-*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "cloudwatch:PutMetricData"
      ],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "cloudwatch:namespace": "Veltrix/Collector"
        }
      }
    }
  ]
}
```

#### Lambda Execution Role (Transformation Engine)
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "kinesis:GetRecords",
        "kinesis:GetShardIterator",
        "kinesis:DescribeStream",
        "kinesis:ListStreams"
      ],
      "Resource": "arn:aws:kinesis:*:*:stream/veltrix-telemetry-*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "sagemaker:InvokeEndpoint"
      ],
      "Resource": "arn:aws:sagemaker:*:*:endpoint/veltrix-fingerprint-*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "dynamodb:GetItem",
        "dynamodb:Query"
      ],
      "Resource": "arn:aws:dynamodb:*:*:table/veltrix-fingerprints"
    }
  ]
}
```

#### Kill Switch Lambda Role
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ssm:SendCommand",
        "ssm:GetCommandInvocation"
      ],
      "Resource": [
        "arn:aws:ec2:*:*:instance/*",
        "arn:aws:ssm:*:*:document/AWS-RunShellScript"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:PutObject"
      ],
      "Resource": "arn:aws:s3:::veltrix-audit-logs-*/*"
    }
  ]
}
```

### 7.3 KMS Encryption

**Key Hierarchy:**
- **Master Key**: AWS-managed KMS key per tenant
- **Data Keys**: Generated per-object for S3, per-table for DynamoDB
- **Rotation**: Automatic annual rotation

**Encryption Scope:**
- DynamoDB tables: Server-side encryption with KMS
- S3 buckets: SSE-KMS with bucket keys
- Kinesis streams: Server-side encryption with KMS
- SageMaker endpoints: Encryption at rest for model artifacts

---

## 8. Multi-Tenant Isolation Design

### 8.1 Tenant Identification

**Tenant ID Propagation:**
1. Edge collector includes `tenant_id` in every telemetry record
2. Kinesis stream partitioned by `tenant_id` (partition key)
3. Lambda functions extract `tenant_id` from event payload
4. DynamoDB queries scoped by `tenant_id` (partition key)


### 8.2 Data Isolation

**DynamoDB Schema:**
```
Table: veltrix-fingerprints
Partition Key: tenant_id (String)
Sort Key: workload_id (String)
GSI: workload_id-index (for cross-tenant lookups by admin)

Table: veltrix-policies
Partition Key: tenant_id (String)
Sort Key: policy_id (String)

Table: veltrix-events
Partition Key: tenant_id (String)
Sort Key: timestamp (Number)
TTL: 90 days
```

**S3 Bucket Structure:**
```
s3://veltrix-telemetry-archive/
  ├── tenant-xyz/
  │   ├── 2026/02/15/
  │   │   └── telemetry-10-23-45.json.gz
  └── tenant-abc/
      └── 2026/02/15/
          └── telemetry-10-23-45.json.gz

s3://veltrix-audit-logs/
  ├── tenant-xyz/
  │   └── 2026/02/15/
  │       └── enforcement-10-23-45.json
  └── tenant-abc/
      └── 2026/02/15/
          └── enforcement-10-23-45.json
```

**IAM Policy Enforcement:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:PutObject"],
      "Resource": "arn:aws:s3:::veltrix-*/${aws:PrincipalTag/tenant_id}/*"
    }
  ]
}
```

### 8.3 Compute Isolation

**Lambda Concurrency:**
- Reserved concurrency per tenant (prevents noisy neighbor)
- Default: 10 concurrent executions per tenant
- Scales with tenant tier (Enterprise: 100 concurrent)

**SageMaker Endpoints:**
- Dedicated endpoint per tenant (Enterprise tier)
- Shared endpoint with request throttling (Standard tier)

---

## 9. Monitoring & Observability

### 9.1 CloudWatch Metrics

**Custom Metrics:**
```python
cloudwatch.put_metric_data(
    Namespace='Veltrix/Detection',
    MetricData=[
        {
            'MetricName': 'AnomalyScore',
            'Value': anomaly_score,
            'Unit': 'None',
            'Dimensions': [
                {'Name': 'TenantId', 'Value': tenant_id},
                {'Name': 'InstanceId', 'Value': instance_id}
            ]
        },
        {
            'MetricName': 'DetectionLatency',
            'Value': latency_ms,
            'Unit': 'Milliseconds',
            'Dimensions': [
                {'Name': 'TenantId', 'Value': tenant_id}
            ]
        },
        {
            'MetricName': 'FalsePositiveRate',
            'Value': fpr,
            'Unit': 'Percent',
            'Dimensions': [
                {'Name': 'TenantId', 'Value': tenant_id}
            ]
        }
    ]
)
```


### 9.2 CloudWatch Alarms

**Critical Alarms:**
```yaml
HighFalsePositiveRate:
  Metric: FalsePositiveRate
  Threshold: 5%
  EvaluationPeriods: 2
  Action: SNS notification to ops team

HighDetectionLatency:
  Metric: DetectionLatency (P95)
  Threshold: 500ms
  EvaluationPeriods: 3
  Action: SNS notification + auto-scale SageMaker

ModelEndpointFailure:
  Metric: SageMaker/ModelInvocationErrors
  Threshold: 10 errors in 5 minutes
  Action: SNS notification + rollback to previous model

CollectorHeartbeatMissing:
  Metric: Veltrix/Collector/Heartbeat
  Threshold: No data for 5 minutes
  Action: SNS notification to customer
```

### 9.3 Distributed Tracing

**AWS X-Ray Integration:**
- Trace telemetry flow: Collector → Kinesis → Lambda → SageMaker → Policy Engine
- Identify bottlenecks in processing pipeline
- Correlate anomaly events with infrastructure issues

**Trace Example:**
```
[Collector] 10ms → [Kinesis] 20ms → [Transform Lambda] 50ms → [SageMaker] 45ms → [Policy Lambda] 15ms
Total: 140ms (within 150ms target)
```

### 9.4 Logging Strategy

**Log Levels:**
- **DEBUG**: Telemetry samples, feature vectors (disabled in production)
- **INFO**: Anomaly detections, policy decisions, enforcement actions
- **WARN**: Fingerprint drift, high false positive rate, retry attempts
- **ERROR**: Lambda failures, SageMaker errors, SSM command failures

**Log Aggregation:**
- CloudWatch Logs Insights for querying
- S3 export for long-term retention (1 year)

---

## 10. Failure Handling Strategy

### 10.1 Failure Modes

| Component | Failure Mode | Impact | Mitigation |
|-----------|--------------|--------|------------|
| Edge Collector | Process crash | No telemetry from instance | Systemd auto-restart, CloudWatch alarm |
| Kinesis Stream | Throttling | Telemetry backlog | Auto-scaling shards, local buffering |
| Lambda (Transform) | Timeout | Delayed detection | Increase memory, optimize FFT code |
| SageMaker Endpoint | Model error | No anomaly scores | Fallback to rule-based detection |
| Lambda (Policy) | DynamoDB unavailable | No policy evaluation | Cache policies in Lambda memory |
| SSM Run Command | Instance unreachable | Termination fails | Retry 3x, escalate to manual |

### 10.2 Graceful Degradation

**Fallback Detection (when SageMaker unavailable):**
```python
def fallback_detection(features):
    # Simple rule-based detection
    if features['mean_power'] > 300:  # Suspiciously high power
        return {'anomaly_score': 0.9, 'method': 'fallback'}
    if features['std_power'] < 1:  # Suspiciously stable (crypto-mining)
        return {'anomaly_score': 0.85, 'method': 'fallback'}
    return {'anomaly_score': 0.1, 'method': 'fallback'}
```


### 10.3 Circuit Breaker Pattern

**Kill Switch Circuit Breaker:**
```python
class KillSwitchCircuitBreaker:
    def __init__(self, failure_threshold=5, timeout=60):
        self.failure_count = 0
        self.failure_threshold = failure_threshold
        self.timeout = timeout
        self.last_failure_time = None
        self.state = 'CLOSED'  # CLOSED, OPEN, HALF_OPEN
    
    def call(self, func, *args, **kwargs):
        if self.state == 'OPEN':
            if time.time() - self.last_failure_time > self.timeout:
                self.state = 'HALF_OPEN'
            else:
                raise CircuitBreakerOpenError("Kill switch circuit breaker is OPEN")
        
        try:
            result = func(*args, **kwargs)
            if self.state == 'HALF_OPEN':
                self.state = 'CLOSED'
                self.failure_count = 0
            return result
        except Exception as e:
            self.failure_count += 1
            self.last_failure_time = time.time()
            if self.failure_count >= self.failure_threshold:
                self.state = 'OPEN'
                send_alert("Kill switch circuit breaker opened due to repeated failures")
            raise
```

---

## 11. Scalability Design

### 11.1 Horizontal Scaling

**Kinesis Shards:**
- Auto-scaling based on incoming data rate
- Target: 1 shard per 1 MB/s
- Max shards: 100 per tenant (supports 100 MB/s = 10,000 instances at 10 KB/s each)

**Lambda Concurrency:**
- Transform Engine: 100 concurrent executions (processes 100 batches/second)
- Policy Engine: 50 concurrent executions
- Kill Switch: 20 concurrent executions

**SageMaker Endpoints:**
- Auto-scaling: 1-10 instances per tenant
- Target invocation rate: 100 requests/second per instance
- Scale-up trigger: >80% utilization for 2 minutes
- Scale-down trigger: <20% utilization for 10 minutes

### 11.2 Vertical Scaling

**Lambda Memory Optimization:**
- Transform Engine: 1024 MB (FFT computation is memory-intensive)
- Policy Engine: 512 MB (lightweight logic)
- Kill Switch: 256 MB (minimal processing)

**SageMaker Instance Types:**
- Standard tier: ml.g4dn.xlarge (1 GPU, 16 GB RAM)
- Enterprise tier: ml.g4dn.2xlarge (1 GPU, 32 GB RAM)

### 11.3 Cost Optimization

**Estimated Monthly Cost (per 1000 GPU-hours):**
```
Kinesis Data Streams: $15 (1 shard, 1 GB/day)
Lambda (Transform): $25 (100M invocations, 1024 MB)
SageMaker (Inference): $150 (ml.g4dn.xlarge, 50% utilization)
DynamoDB: $10 (on-demand, 1M reads/writes)
S3: $5 (100 GB storage, Intelligent-Tiering)
CloudWatch: $10 (custom metrics, logs)
Total: $215/month for 1000 GPU-hours = $0.215 per GPU-hour

Revenue: $500/month (1000 GPU-hours × $0.50)
Gross Margin: 57%
```

---

## 12. Correctness & Safety Properties

### 12.1 Correctness Properties

**Property 1: Detection Completeness**
- **Statement**: Every anomalous workload SHALL be assigned an anomaly score within 250ms (p95).
- **Validation**: Property-based test with synthetic anomalies.

**Property 2: False Positive Bound**
- **Statement**: False positive rate on authorized workloads SHALL be <2%.
- **Validation**: Statistical test on 1000+ authorized workload samples.

**Property 3: Termination Guarantee**
- **Statement**: If policy action is "terminate" and process exists, process SHALL be killed within 200ms (p95) or escalated.
- **Validation**: Integration test with mock processes.


**Property 4: Data Isolation**
- **Statement**: Tenant A SHALL NOT access tenant B's fingerprints, policies, or telemetry data.
- **Validation**: IAM policy analyzer + integration tests.

**Property 5: Audit Immutability**
- **Statement**: Enforcement action logs SHALL be immutable and retained for 90 days.
- **Validation**: S3 Object Lock verification.

### 12.2 Safety Properties

**Safety 1: Fail-Open**
- **Statement**: Edge collector failures SHALL NOT impact monitored workloads.
- **Implementation**: Collector runs as separate process, no kernel hooks.

**Safety 2: Manual Override**
- **Statement**: Administrators SHALL be able to disable kill switch within 5 seconds.
- **Implementation**: Dashboard "Pause Kill Switch" button updates DynamoDB policy.

**Safety 3: Whitelist Precedence**
- **Statement**: Whitelisted processes SHALL NEVER be terminated, regardless of anomaly score.
- **Implementation**: Whitelist check before policy evaluation.

**Safety 4: Graceful Degradation**
- **Statement**: If SageMaker endpoint fails, system SHALL fall back to rule-based detection.
- **Implementation**: Try-catch with fallback logic in Transform Lambda.

### 12.3 Testing Strategy

**Unit Tests:**
- FFT computation correctness
- Anomaly score calculation
- Policy evaluation logic
- Whitelist matching

**Integration Tests:**
- End-to-end telemetry flow (Collector → Kinesis → Lambda → SageMaker)
- Kill switch execution (Lambda → SSM → process termination)
- Multi-tenant isolation (cross-tenant access attempts)

**Property-Based Tests:**
- Anomaly score is always in [0, 1] range
- Detection latency is always <500ms (with timeout)
- False positive rate converges to <2% over 1000 samples

**Load Tests:**
- 10,000 concurrent instances sending telemetry
- 1000 anomaly detections per second
- SageMaker endpoint auto-scaling under load

---

## 13. Deployment Architecture (Terraform/CDK)

### 13.1 Infrastructure as Code

**Technology Choice: AWS CDK (Python)**
- Reason: Type-safe, integrates with Python ecosystem, supports L2 constructs

**Repository Structure:**
```
veltrix-infrastructure/
├── cdk.json
├── app.py
├── stacks/
│   ├── telemetry_stack.py       # Kinesis, Lambda (Transform)
│   ├── ml_stack.py              # SageMaker endpoints
│   ├── policy_stack.py          # Lambda (Policy, Kill Switch)
│   ├── data_stack.py            # DynamoDB, S3
│   ├── monitoring_stack.py      # CloudWatch, SNS
│   └── dashboard_stack.py       # Amplify, Cognito
├── constructs/
│   ├── edge_collector.py        # EC2 user-data script
│   └── multi_tenant_table.py    # DynamoDB with tenant isolation
└── tests/
    └── test_stacks.py
```


### 13.2 CDK Stack Example (Telemetry Stack)

```python
from aws_cdk import (
    Stack,
    aws_kinesis as kinesis,
    aws_lambda as lambda_,
    aws_lambda_event_sources as event_sources,
    aws_iam as iam,
    Duration
)
from constructs import Construct

class TelemetryStack(Stack):
    def __init__(self, scope: Construct, id: str, **kwargs):
        super().__init__(scope, id, **kwargs)
        
        # Kinesis Data Stream
        self.telemetry_stream = kinesis.Stream(
            self, "TelemetryStream",
            stream_name="veltrix-telemetry",
            shard_count=1,
            retention_period=Duration.hours(24),
            encryption=kinesis.StreamEncryption.KMS
        )
        
        # Transform Lambda
        transform_lambda = lambda_.Function(
            self, "TransformFunction",
            runtime=lambda_.Runtime.PYTHON_3_11,
            handler="transform.handler",
            code=lambda_.Code.from_asset("lambda/transform"),
            memory_size=1024,
            timeout=Duration.seconds(60),
            environment={
                "SAGEMAKER_ENDPOINT": "veltrix-fingerprint-model",
                "FINGERPRINT_TABLE": "veltrix-fingerprints"
            }
        )
        
        # Grant permissions
        self.telemetry_stream.grant_read(transform_lambda)
        
        # Kinesis trigger
        transform_lambda.add_event_source(
            event_sources.KinesisEventSource(
                self.telemetry_stream,
                starting_position=lambda_.StartingPosition.LATEST,
                batch_size=100,
                max_batching_window=Duration.seconds(1)
            )
        )
```

### 13.3 Deployment Pipeline

**CI/CD Workflow (GitHub Actions):**
```yaml
name: Deploy Veltrix Infrastructure

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'
      
      - name: Install CDK
        run: npm install -g aws-cdk
      
      - name: Install dependencies
        run: pip install -r requirements.txt
      
      - name: Run tests
        run: pytest tests/
      
      - name: CDK synth
        run: cdk synth
      
      - name: CDK deploy (staging)
        run: cdk deploy --all --require-approval never
        env:
          AWS_REGION: us-east-1
          ENVIRONMENT: staging
      
      - name: Integration tests
        run: pytest tests/integration/
      
      - name: CDK deploy (production)
        run: cdk deploy --all --require-approval never
        env:
          AWS_REGION: us-east-1
          ENVIRONMENT: production
        if: github.ref == 'refs/heads/main'
```

### 13.4 Multi-Region Deployment (Phase 2)

**Active-Passive DR Strategy:**
- Primary region: us-east-1
- DR region: us-west-2
- DynamoDB global tables for fingerprint registry
- S3 cross-region replication for audit logs
- Route 53 health checks with automatic failover

---

## 14. MVP Implementation Plan

### 14.1 MVP Scope

**Included:**
- Edge collector (NVIDIA-SMI polling, 10ms fixed interval)
- Kinesis ingestion
- FFT-based feature extraction
- LSTM Autoencoder (crypto-mining vs idle detection only)
- Alert-only mode (no automated termination)
- Basic React dashboard
- Single-tenant deployment

**Excluded:**
- Multi-tenant isolation
- Kill switch execution
- Fingerprint drift detection
- Advanced anomaly types (model theft)
- Auto-scaling
- Multi-region deployment


### 14.2 MVP Architecture Simplifications

**Simplified Data Flow:**
```
EC2 Instance → Kinesis → Lambda (Transform + Inference) → DynamoDB (Events) → Dashboard
```

**Consolidated Lambda:**
- Single Lambda function combines transformation, inference, and alerting
- Reduces latency and operational complexity
- SageMaker endpoint still separate for model hosting

### 14.3 MVP Implementation Tasks

**Phase 1: Telemetry Collection (Week 1-2)**
1. Implement edge collector in Python
   - NVIDIA-SMI polling (10ms interval)
   - RAPL power reading
   - Process enumeration
   - Kinesis producer
2. Create systemd service
3. Test on EC2 g4dn.xlarge instance

**Phase 2: Signal Processing (Week 3)**
1. Implement FFT transformation in Lambda
2. Extract spectral density features
3. Implement Z-score normalization
4. Unit tests for signal processing

**Phase 3: ML Model (Week 4-5)**
1. Collect training data (crypto-mining vs idle)
2. Train LSTM Autoencoder
3. Deploy to SageMaker endpoint
4. Validate accuracy (>90% TPR, <5% FPR)

**Phase 4: Detection Pipeline (Week 6)**
1. Integrate Lambda with SageMaker
2. Implement anomaly score calculation
3. Store events in DynamoDB
4. Publish CloudWatch metrics

**Phase 5: Dashboard (Week 7-8)**
1. Create React app with Amplify
2. Implement real-time anomaly view (WebSocket)
3. Add golden signature management UI
4. Deploy to Amplify Hosting

**Phase 6: Testing & Validation (Week 9-10)**
1. End-to-end integration tests
2. Load testing (100 instances)
3. False positive/negative rate validation
4. Performance benchmarking (latency, overhead)

### 14.4 MVP Success Criteria

**Functional:**
- ✅ Detect crypto-mining (xmrig) with >90% accuracy
- ✅ False positive rate <5% on PyTorch training workload
- ✅ Dashboard displays alerts within 2 seconds

**Performance:**
- ✅ Detection latency <500ms (p95)
- ✅ Runtime overhead <5% CPU/GPU
- ✅ Collector memory footprint <500 MB

**Operational:**
- ✅ Collector runs continuously for 24 hours without crashes
- ✅ Kinesis ingestion handles 100 instances
- ✅ SageMaker endpoint responds within 100ms (p95)

---

## 15. Production Roadmap (Post-MVP)

### 15.1 Phase 2: Multi-Tenant & Kill Switch (Month 3-4)

**Features:**
- Multi-tenant isolation (KMS keys, IAM policies)
- Kill switch execution via SSM
- Policy engine with configurable thresholds
- Manual override capability
- Enhanced dashboard (per-tenant views)

**Deliverables:**
- Multi-tenant CDK stacks
- Kill switch Lambda + integration tests
- Policy management UI
- SOC 2 compliance documentation

### 15.2 Phase 3: Advanced Detection (Month 5-6)

**Features:**
- Model theft detection
- Unauthorized inference detection
- Fingerprint drift detection
- Automated retraining pipeline
- A/B testing framework

**Deliverables:**
- Enhanced LSTM Autoencoder (multi-class)
- Drift detection algorithm
- Retraining automation (SageMaker Pipelines)
- Model versioning system

### 15.3 Phase 4: Enterprise Features (Month 7-9)

**Features:**
- Multi-region deployment (active-passive DR)
- Nitro Enclave integration
- SIEM integration (Splunk, Datadog)
- Advanced analytics dashboard
- Custom alerting rules

**Deliverables:**
- Multi-region CDK stacks
- Nitro Enclave model hosting
- SIEM connectors
- Advanced React dashboard

### 15.4 Phase 5: AWS Marketplace Launch (Month 10-12)

**Features:**
- AWS Marketplace listing
- Metering API integration
- Free trial (100 GPU-hours)
- Customer onboarding automation
- 24/7 support infrastructure

**Deliverables:**
- Marketplace SaaS listing
- Billing integration
- Onboarding wizard
- Support ticketing system
- Customer documentation

---

## 16. Open Questions & Risks

### 16.1 Technical Risks

**Risk 1: NVIDIA-SMI Latency**
- **Issue**: nvidia-smi polling may have >10ms latency on some GPU models
- **Mitigation**: Benchmark on target GPU types, consider NVML API for lower latency

**Risk 2: FFT Noise Sensitivity**
- **Issue**: Power measurements may be noisy, affecting FFT quality
- **Mitigation**: Apply low-pass filter, increase window size to 2 seconds

**Risk 3: Model Generalization**
- **Issue**: LSTM trained on specific workloads may not generalize to new workload types
- **Mitigation**: Collect diverse training data, implement online learning

### 16.2 Operational Risks

**Risk 1: False Positive Impact**
- **Issue**: High false positive rate may cause alert fatigue
- **Mitigation**: Conservative thresholds in MVP, user feedback loop

**Risk 2: Kill Switch Abuse**
- **Issue**: Automated termination may kill legitimate workloads
- **Mitigation**: Alert-only mode in MVP, manual approval for production

**Risk 3: Cost Overruns**
- **Issue**: SageMaker inference costs may exceed budget at scale
- **Mitigation**: Batch inference, model optimization, reserved instances

### 16.3 Open Questions

1. **Hardware Diversity**: How to handle AMD GPUs, Intel GPUs, ARM CPUs?
   - **Answer**: Phase 2 feature, focus on NVIDIA + Intel x86 for MVP

2. **Container Environments**: How to monitor workloads in Kubernetes pods?
   - **Answer**: DaemonSet deployment, cgroup-aware telemetry

3. **Windows Support**: What's the equivalent of RAPL on Windows?
   - **Answer**: Intel Power Gadget API, Phase 2 feature

4. **Regulatory Compliance**: Does telemetry collection require user consent?
   - **Answer**: Legal review required, add opt-in mechanism

---

## 17. Appendix

### 17.1 Glossary

- **Golden Signature**: Baseline energy fingerprint of an authorized workload
- **Anomaly Score**: Normalized reconstruction error (0-1 scale)
- **Spectral Density**: Frequency-domain representation of power signal
- **Kill Switch**: Automated process termination mechanism
- **Fingerprint Drift**: Gradual change in golden signature over time

### 17.2 References

- NVIDIA-SMI Documentation: https://developer.nvidia.com/nvidia-system-management-interface
- Intel RAPL: https://www.intel.com/content/www/us/en/developer/articles/technical/software-security-guidance/advisory-guidance/running-average-power-limit-energy-reporting.html
- AWS SageMaker: https://docs.aws.amazon.com/sagemaker/
- LSTM Autoencoders for Anomaly Detection: https://arxiv.org/abs/1607.00148

### 17.3 Document History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-02-15 | Veltrix Team | Initial design document |
