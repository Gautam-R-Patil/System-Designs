# Axon – Neuro-Symbolic Structural Integrity Engine
## Design Document

## 1. System Architecture Overview

Axon implements a four-layer neuro-symbolic architecture that separates neural perception from deterministic reasoning:

```
┌─────────────────────────────────────────────────────────────┐
│                    Edge Layer (Drone)                        │
│  NVIDIA Jetson + AWS IoT Greengrass                         │
│  - RGB Camera (4K) + LiDAR Sensor                           │
│  - Edge preprocessing & compression                          │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│                   Ingestion Layer (AWS)                      │
│  Kinesis Video Streams + S3 + IoT Core                      │
│  - Stream buffering & storage                                │
│  - Metadata extraction                                       │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│              Neural Perception Layer (SageMaker)             │
│  Vision Transformer + PointNet++                            │
│  - Crack detection (perception only)                         │
│  - Component segmentation                                    │
│  - Confidence scoring                                        │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│           Symbolic Reasoning Layer (Lambda + Batch)          │
│  Rule Engine + Physics Simulator                            │
│  - IS 456 compliance checking                                │
│  - Risk classification (FINAL DECISION)                      │
│  - Explainable reasoning                                     │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│          Digital Twin Layer (IoT TwinMaker + Amplify)        │
│  3D Visualization + Dashboard                                │
│  - Real-time risk overlay                                    │
│  - Historical tracking                                       │
└─────────────────────────────────────────────────────────────┘
```

### Design Principles
1. **Separation of Concerns**: Neural models perceive, symbolic rules decide
2. **Explainability First**: All safety decisions must be traceable
3. **Physics-Grounded**: Compliance with engineering standards (IS 456)
4. **Scalable by Design**: Multi-tenant SaaS architecture
5. **Security-Hardened**: Nitro Enclaves for IP protection

## 2. Component Design

### 2.1 Edge Layer Design

**Hardware Configuration**:
- NVIDIA Jetson Xavier NX (16GB RAM, 384 CUDA cores)
- RGB Camera: 4K @ 30fps (Sony IMX577 or equivalent)
- LiDAR: Velodyne Puck LITE (16 channels, 100m range)
- GPS/IMU: RTK-GPS with ±2cm accuracy

**Software Stack**:
- AWS IoT Greengrass v2 for orchestration
- Edge preprocessing pipeline:
  - Video compression (H.264, CRF 23)
  - LiDAR downsampling (adaptive voxel grid)
  - Timestamp synchronization (PTP protocol)
  - Local buffering for network resilience

**Data Flow**:
```
Camera → Frame Buffer → H.264 Encoder → Kinesis Video Streams
LiDAR → Point Cloud → Compression → S3 (via IoT Core)
GPS/IMU → Telemetry → IoT Core MQTT
```

### 2.2 Ingestion Layer Design

**AWS Kinesis Video Streams**:
- Stream retention: 7 days
- Fragment duration: 2 seconds
- HLS endpoint for playback
- GetMedia API for processing

**S3 Storage Architecture**:
```
s3://axon-data-{tenant-id}/
├── raw/
│   ├── video/{asset-id}/{timestamp}.mp4
│   ├── lidar/{asset-id}/{timestamp}.pcd
│   └── metadata/{asset-id}/{timestamp}.json
├── processed/
│   ├── point-clouds/{asset-id}/{timestamp}.ply
│   ├── detections/{asset-id}/{timestamp}.json
│   └── reports/{asset-id}/{timestamp}.pdf
└── models/
    └── digital-twins/{asset-id}/latest.gltf
```

**Lifecycle Policies**:
- Raw data: Transition to Glacier after 90 days
- Processed data: Retain in S3 Standard for 1 year
- Reports: Retain indefinitely in S3 Intelligent-Tiering

### 2.3 Neural Perception Layer Design

#### 2.3.1 Crack Detection Model (Vision Transformer)

**Architecture**:
- Base: ViT-B/16 (86M parameters)
- Input: 224×224 RGB patches
- Output: Crack segmentation mask + confidence score
- Training: Transfer learning from ImageNet

**Training Pipeline**:
```python
# Pseudocode for training workflow
dataset = SyntheticCrackDataset(100_000_samples)
model = VisionTransformer(
    patch_size=16,
    num_classes=2,  # crack vs background
    pretrained="imagenet21k"
)
optimizer = AdamW(lr=1e-4, weight_decay=0.01)
loss_fn = FocalLoss(alpha=0.25, gamma=2.0)  # Handle class imbalance
```

**Inference**:
- SageMaker endpoint with GPU instance (ml.g4dn.xlarge)
- Batch size: 32 frames
- Latency target: < 100ms per frame

#### 2.3.2 Structural Segmentation Model (PointNet++)

**Architecture**:
- PointNet++ with multi-scale grouping
- Input: N×6 point cloud (XYZ + RGB)
- Output: Per-point semantic labels (beam, column, slab, etc.)
- Training: Supervised learning on annotated bridge models

**Point Cloud Processing**:
```python
# Pseudocode for point cloud segmentation
def segment_structure(point_cloud):
    # Downsample to 50K points
    points = voxel_downsample(point_cloud, voxel_size=0.05)
    
    # Extract features
    features = pointnet_encoder(points)
    
    # Classify each point
    labels = pointnet_decoder(features)
    
    return labels, confidence_scores
```

#### 2.3.3 Depth Fusion Model

**Purpose**: Combine multi-angle LiDAR measurements for accurate crack depth

**Algorithm**:
1. Register point clouds from multiple viewpoints (ICP alignment)
2. Project crack pixels onto 3D surface
3. Measure depth using local surface normal
4. Aggregate measurements with uncertainty quantification

**Output**: Crack geometry (width, depth, length, orientation)

### 2.4 Symbolic Reasoning Layer Design

#### 2.4.1 Rule Engine Architecture

**Technology**: AWS Lambda (Python 3.11, 3GB memory)

**Rule Structure**:
```python
# Pseudocode for IS 456 rule engine
class IS456RuleEngine:
    def evaluate_crack(self, crack_geometry, component_type, load_zone):
        """
        Deterministic rule-based risk classification
        """
        risk_level = "Low"
        reasoning = []
        
        # Rule 1: Crack width threshold
        if crack_geometry.width > 0.3:  # mm
            if load_zone == "tensile":
                risk_level = "Critical"
                reasoning.append("IS 456 Clause 35.3.2: Crack width > 0.3mm in tensile zone")
        
        # Rule 2: Crack depth threshold
        if crack_geometry.depth > 5.0:  # mm
            risk_level = max(risk_level, "Moderate")
            reasoning.append("Crack depth exceeds 5mm threshold")
        
        # Rule 3: Structural component criticality
        if component_type == "column" and crack_geometry.orientation == "horizontal":
            risk_level = "Critical"
            reasoning.append("Horizontal crack in load-bearing column")
        
        return {
            "risk_level": risk_level,
            "reasoning": reasoning,
            "confidence": 1.0,  # Deterministic rules have 100% confidence
            "standard": "IS 456:2000"
        }
```

**Rule Categories**:
1. Crack geometry thresholds (width, depth, length)
2. Location-based rules (tensile vs compressive zones)
3. Component-specific rules (beam, column, slab)
4. Material-specific rules (concrete grade, reinforcement)
5. Environmental factors (corrosion, spalling)

#### 2.4.2 Physics Simulation Engine

**Technology**: AWS Batch (Docker containers with FEniCS/OpenFOAM)

**Simulation Workflow**:
```
1. Load digital twin geometry (GLTF → mesh)
2. Apply detected crack as boundary condition
3. Run finite element analysis (stress redistribution)
4. Calculate residual load capacity
5. Predict crack propagation trajectory
6. Output: Updated risk assessment + time-to-failure estimate
```

**Batch Job Configuration**:
- Compute environment: EC2 Spot instances (c5.4xlarge)
- Job queue: Priority-based (critical findings first)
- Timeout: 30 minutes per simulation
- Parallelization: 10 concurrent jobs

### 2.5 Digital Twin Layer Design

#### 2.5.1 AWS IoT TwinMaker Integration

**Entity Model**:
```json
{
  "entityId": "bridge-001",
  "entityName": "Main Street Bridge",
  "components": {
    "geometry": {
      "componentTypeId": "com.axon.bridge.geometry",
      "properties": {
        "model": "s3://axon-models/bridge-001/latest.gltf",
        "location": {"lat": 28.6139, "lon": 77.2090}
      }
    },
    "inspection": {
      "componentTypeId": "com.axon.bridge.inspection",
      "properties": {
        "lastInspection": "2026-02-15T10:30:00Z",
        "riskLevel": "Moderate",
        "crackCount": 12
      }
    }
  }
}
```

**Scene Composition**:
- Base layer: 3D bridge model (GLTF format)
- Overlay layer: Crack annotations (color-coded by risk)
- Metadata layer: Inspection history timeline
- Camera controls: Orbit, pan, zoom, first-person

#### 2.5.2 Dashboard Design (React + Amplify)

**Key Views**:
1. **Live Inspection View**: Real-time drone feed + digital twin sync
2. **Asset Overview**: Grid of all monitored bridges with status
3. **Risk Heatmap**: Geographic view with color-coded risk zones
4. **Inspection History**: Timeline with playback and comparison
5. **Compliance Reports**: Downloadable PDF/JSON reports

**GraphQL API Schema**:
```graphql
type Bridge {
  id: ID!
  name: String!
  location: Location!
  riskLevel: RiskLevel!
  inspections: [Inspection!]!
  cracks: [Crack!]!
}

type Crack {
  id: ID!
  geometry: CrackGeometry!
  riskLevel: RiskLevel!
  reasoning: [String!]!
  detectedAt: DateTime!
  component: StructuralComponent!
}

type Query {
  getBridge(id: ID!): Bridge
  listBridges(filter: BridgeFilter): [Bridge!]!
  getInspectionHistory(bridgeId: ID!, startDate: DateTime, endDate: DateTime): [Inspection!]!
}
```

## 3. Data Flow Design

### 3.1 Real-Time Inspection Flow

```
1. Drone captures RGB frame + LiDAR scan
2. Edge preprocessing (compression, sync)
3. Upload to Kinesis Video Streams + S3
4. Lambda trigger on new S3 object
5. Invoke SageMaker endpoint (crack detection)
6. Invoke SageMaker endpoint (segmentation)
7. Depth fusion processing
8. Lambda invokes rule engine with neural outputs
9. Rule engine produces risk classification + reasoning
10. Update IoT TwinMaker entity
11. Publish alert to dashboard (WebSocket)
12. Log to DynamoDB for history
```

**Latency Budget**:
- Steps 1-3: 200ms (ingestion)
- Steps 4-6: 2s (neural inference)
- Steps 7-9: 1s (fusion + reasoning)
- Steps 10-12: 500ms (update + alert)
- **Total: ~3.7s end-to-end**

### 3.2 Batch Processing Flow

```
1. Upload video + LiDAR files to S3
2. Step Functions orchestrates batch workflow
3. Process all frames in parallel (SageMaker Batch Transform)
4. Aggregate detections per structural component
5. Submit physics simulation jobs (AWS Batch)
6. Generate compliance report (Lambda + PDF generator)
7. Update digital twin with batch results
8. Send email notification to user
```

## 4. Security Design

### 4.1 Multi-Tenant Isolation

**Strategy**: Separate S3 buckets per tenant with KMS encryption

```
Tenant A → KMS Key A → S3 Bucket A
Tenant B → KMS Key B → S3 Bucket B
```

**IAM Policy**:
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": ["s3:GetObject", "s3:PutObject"],
    "Resource": "arn:aws:s3:::axon-data-${aws:PrincipalTag/TenantId}/*",
    "Condition": {
      "StringEquals": {
        "s3:x-amz-server-side-encryption": "aws:kms"
      }
    }
  }]
}
```

### 4.2 Model Protection (Nitro Enclaves)

**Protected Assets**:
- Neural model weights (ViT, PointNet++)
- Proprietary rule engine logic
- Physics simulation parameters

**Enclave Architecture**:
```
┌─────────────────────────────────┐
│      EC2 Instance (Parent)      │
│  - API Gateway                   │
│  - Request validation            │
│  - Logging                       │
└─────────────────────────────────┘
              ↓ (vsock)
┌─────────────────────────────────┐
│    Nitro Enclave (Isolated)     │
│  - Model inference               │
│  - Rule engine execution         │
│  - No network access             │
│  - Attestation required          │
└─────────────────────────────────┘
```

## 5. Testing Strategy

### 5.1 Unit Tests
- Neural model accuracy on synthetic dataset
- Rule engine logic validation
- API endpoint response validation

### 5.2 Integration Tests
- End-to-end data flow (drone → dashboard)
- Multi-tenant isolation verification
- Failover and retry mechanisms

### 5.3 Property-Based Tests

**Property 1: Crack Detection Monotonicity**
```
∀ crack width w1, w2 where w1 < w2:
  risk_level(w1) ≤ risk_level(w2)
```

**Property 2: Rule Engine Determinism**
```
∀ crack geometry g:
  rule_engine(g) = rule_engine(g)  (idempotent)
```

**Property 3: Neural-Symbolic Consistency**
```
∀ crack c where neural_confidence(c) < 0.7:
  final_decision(c) = symbolic_override(c)
```

**Property 4: Digital Twin Synchronization**
```
∀ inspection i:
  twin_state(i.timestamp + Δt) reflects i.findings
  where Δt < 10 seconds
```

**Property 5: Multi-Tenant Isolation**
```
∀ tenants A, B:
  data_access(A) ∩ data_access(B) = ∅
```

### 5.4 Performance Tests
- Load testing: 10 concurrent drone streams
- Stress testing: 1000 bridges in dashboard
- Latency testing: P95 < 5s for risk evaluation

## 6. Deployment Design

### 6.1 Infrastructure as Code (Terraform)

**Module Structure**:
```
terraform/
├── modules/
│   ├── ingestion/      # Kinesis, S3, IoT Core
│   ├── compute/        # SageMaker, Lambda, Batch
│   ├── storage/        # DynamoDB, RDS
│   ├── visualization/  # TwinMaker, Amplify
│   └── security/       # KMS, IAM, Nitro Enclaves
├── environments/
│   ├── dev/
│   ├── staging/
│   └── prod/
└── main.tf
```

### 6.2 CI/CD Pipeline (GitHub Actions)

```yaml
# .github/workflows/deploy.yml
name: Deploy Axon
on:
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - name: Run unit tests
      - name: Run property-based tests
      - name: Run integration tests
  
  deploy:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to staging
      - name: Run smoke tests
      - name: Deploy to production (manual approval)
```

### 6.3 Monitoring & Observability

**CloudWatch Metrics**:
- Ingestion rate (frames/sec)
- Inference latency (P50, P95, P99)
- Rule engine execution time
- Digital twin update lag
- API error rates

**CloudWatch Alarms**:
- Crack detection accuracy < 95%
- End-to-end latency > 10s
- S3 storage > 80% quota
- Lambda error rate > 1%

**X-Ray Tracing**:
- Distributed trace for each inspection
- Identify bottlenecks in data flow
- Track neural vs symbolic processing time

## 7. Correctness Properties

### 7.1 Safety Properties

**P1: No False Negatives for Critical Cracks**
```
∀ crack c where ground_truth(c) = "Critical":
  system_output(c) ∈ {"Critical", "Moderate"}
  (never classify as "Low")
```

**P2: Explainability Requirement**
```
∀ risk classification r:
  ∃ reasoning chain R such that:
    R traces r to specific IS 456 clauses
```

**P3: Neural Override Safety**
```
∀ crack c where neural_confidence(c) < threshold:
  final_decision(c) = symbolic_fallback(c)
```

### 7.2 Liveness Properties

**P4: Inspection Completeness**
```
∀ inspection i:
  eventually(digital_twin reflects i.findings)
  within 30 seconds
```

**P5: Alert Delivery**
```
∀ critical finding f:
  eventually(dashboard displays alert(f))
  within 10 seconds
```

### 7.3 Consistency Properties

**P6: Temporal Consistency**
```
∀ bridge b, timestamps t1 < t2:
  crack_count(b, t2) ≥ crack_count(b, t1)
  (cracks don't disappear without intervention)
```

**P7: Physics Consistency**
```
∀ crack c:
  load_capacity(bridge_with_crack(c)) ≤ load_capacity(bridge_without_crack)
```

## 8. MVP Implementation Plan

### Phase 1: Data Pipeline (Week 1-2)
- Set up S3 buckets and Kinesis streams
- Implement edge preprocessing on Jetson
- Build point cloud generation pipeline

### Phase 2: Neural Models (Week 3-4)
- Train ViT crack detection model
- Train PointNet++ segmentation model
- Deploy SageMaker endpoints

### Phase 3: Symbolic Engine (Week 5)
- Implement IS 456 rule engine
- Build explainability module
- Integrate with neural outputs

### Phase 4: Digital Twin (Week 6)
- Set up IoT TwinMaker workspace
- Build React dashboard
- Implement GraphQL API

### Phase 5: Testing & Demo (Week 7-8)
- Run property-based tests
- Conduct end-to-end validation
- Prepare demo with sample bridge

## 9. Future Enhancements (Post-MVP)

- ACI 318 compliance support
- Autonomous drone navigation
- Real-time physics simulation
- Mobile app for field inspectors
- Predictive maintenance ML models
- Multi-asset type support (tunnels, dams)
