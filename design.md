# Watson – Deterministic Forensic Reconstruction Engine
## Design Specification

---

## 1. System Architecture Overview

Watson implements a neuro-symbolic architecture that strictly separates perception (neural) from reasoning (symbolic). The system is built on AWS cloud infrastructure with deterministic physics simulation at its core.

**Architecture Principles:**
- Deterministic execution: identical inputs produce identical outputs
- Immutable evidence storage with cryptographic verification
- Neuro-symbolic separation: neural models detect, symbolic engine decides
- Cloud-native design using AWS managed services
- Stateless compute with persistent audit logging

**Key Design Decisions:**
- Python 3.11 for physics engine (numerical stability, scientific libraries)
- AWS Lambda for lightweight orchestration, ECS Fargate for heavy simulation
- QLDB for immutable audit trail (cryptographic verification built-in)
- Step Functions for workflow orchestration (visual debugging, error handling)
- S3 with object lock for evidence integrity

---

## 2. High-Level Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                         INGESTION LAYER                              │
│  ┌──────────────┐      ┌──────────────┐      ┌──────────────┐     │
│  │   S3 Upload  │─────▶│  Lambda      │─────▶│  S3 Evidence │     │
│  │   (Video)    │      │  Hash + Meta │      │  Bucket      │     │
│  └──────────────┘      └──────────────┘      └──────────────┘     │
│                              │                                       │
│                              ▼                                       │
│                        ┌──────────────┐                             │
│                        │  QLDB        │                             │
│                        │  Audit Log   │                             │
│                        └──────────────┘                             │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      NEURAL PERCEPTION LAYER                         │
│  ┌──────────────┐      ┌──────────────┐      ┌──────────────┐     │
│  │  Frame       │─────▶│  Rekognition │─────▶│  Detection   │     │
│  │  Extraction  │      │  Custom      │      │  Results     │     │
│  │  (Lambda)    │      │  Labels      │      │  (S3)        │     │
│  └──────────────┘      └──────────────┘      └──────────────┘     │
│                                                      │               │
│                                                      ▼               │
│                        ┌──────────────┐                             │
│                        │  Manual      │                             │
│                        │  Coordinate  │                             │
│                        │  Input (API) │                             │
│                        └──────────────┘                             │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                   SYMBOLIC REASONING LAYER                           │
│  ┌──────────────┐      ┌──────────────┐      ┌──────────────┐     │
│  │  Step        │─────▶│  Physics     │─────▶│  Trajectory  │     │
│  │  Functions   │      │  Engine      │      │  Results     │     │
│  │  Workflow    │      │  (ECS/Lambda)│      │  (S3)        │     │
│  └──────────────┘      └──────────────┘      └──────────────┘     │
│                              │                       │               │
│                              ▼                       ▼               │
│                        ┌──────────────┐      ┌──────────────┐     │
│                        │  Hash        │      │  QLDB        │     │
│                        │  Generator   │      │  Audit Log   │     │
│                        └──────────────┘      └──────────────┘     │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      PRESENTATION LAYER                              │
│  ┌──────────────┐      ┌──────────────┐      ┌──────────────┐     │
│  │  API Gateway │─────▶│  React       │      │  PDF Report  │     │
│  │  + Lambda    │      │  Dashboard   │      │  Generator   │     │
│  │              │      │  (Amplify)   │      │  (Lambda)    │     │
│  └──────────────┘      └──────────────┘      └──────────────┘     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 3. Component Design

### 3.1 Evidence Ingestion Service

**Purpose:** Securely ingest video evidence with cryptographic verification

**Implementation:**
- S3 bucket with event notification trigger
- Lambda function for hash generation and metadata extraction
- QLDB for chain of custody logging

**Pseudocode:**
```python
def ingest_evidence(video_file, case_id, officer_id):
    # Generate hash
    file_hash = sha256(video_file)
    
    # Store original in S3 with object lock
    s3_key = f"evidence/{case_id}/{file_hash}.mp4"
    s3.put_object(
        Bucket="watson-evidence",
        Key=s3_key,
        Body=video_file,
        ObjectLockMode="GOVERNANCE",
        Metadata={
            "case_id": case_id,
            "officer_id": officer_id,
            "timestamp": iso8601_now(),
            "hash": file_hash
        }
    )
    
    # Log to QLDB
    qldb.execute(
        "INSERT INTO evidence_log VALUES (?, ?, ?, ?)",
        case_id, file_hash, officer_id, iso8601_now()
    )
    
    return {"evidence_id": file_hash, "status": "ingested"}
```


**Determinism Guarantees:**
- Hash function is deterministic (SHA-256)
- Timestamps are immutable once logged
- S3 object lock prevents modification

---

### 3.2 Frame Extraction Service

**Purpose:** Extract frames from video at configurable intervals

**Implementation:**
- Lambda function triggered by S3 event
- FFmpeg layer for video processing
- Frame storage in S3 with versioning

**Pseudocode:**
```python
def extract_frames(video_s3_key, fps=1):
    # Download video from S3
    video_path = download_from_s3(video_s3_key)
    
    # Extract frames using FFmpeg (deterministic)
    frames = ffmpeg.extract_frames(
        input=video_path,
        fps=fps,
        format="png"
    )
    
    # Upload frames to S3
    frame_keys = []
    for idx, frame in enumerate(frames):
        frame_hash = sha256(frame)
        frame_key = f"frames/{video_s3_key}/frame_{idx:06d}_{frame_hash}.png"
        s3.put_object(Bucket="watson-frames", Key=frame_key, Body=frame)
        frame_keys.append(frame_key)
    
    return {"frame_count": len(frames), "frame_keys": frame_keys}
```

**Determinism Guarantees:**
- Fixed FPS parameter
- Frame extraction order is deterministic
- Frame hashes verify identical extraction

---

### 3.3 Object Detection Service

**Purpose:** Detect bullet impacts and blood spatter using neural models

**Implementation:**
- AWS Rekognition Custom Labels or SageMaker endpoint
- Detection results logged but not trusted blindly
- Manual override capability

**Pseudocode:**
```python
def detect_objects(frame_s3_key, model_arn):
    # Call Rekognition
    response = rekognition.detect_custom_labels(
        Image={"S3Object": {"Bucket": "watson-frames", "Name": frame_s3_key}},
        ProjectVersionArn=model_arn
    )
    
    # Extract detections
    detections = []
    for label in response["CustomLabels"]:
        detections.append({
            "type": label["Name"],
            "confidence": label["Confidence"],
            "bounding_box": label["Geometry"]["BoundingBox"],
            "timestamp": iso8601_now()
        })
    
    # Log detections (not used directly in physics)
    log_neural_output(frame_s3_key, detections)
    
    return {"detections": detections, "requires_manual_validation": True}
```

**Neural-Symbolic Boundary:**
- Detections are suggestions only
- Symbolic engine requires manual coordinate confirmation
- Neural confidence scores are logged but not used in final calculations

---

### 3.4 Scene Reconstruction Service

**Purpose:** Convert 2D video frames to 3D scene representation

**Implementation:**
- Monocular depth estimation (MiDaS model on SageMaker)
- Scale calibration using reference objects
- 3D point cloud generation

**Pseudocode:**
```python
def reconstruct_scene(frame_keys, reference_object_size_meters):
    # Estimate depth for each frame
    depth_maps = []
    for frame_key in frame_keys:
        depth_map = sagemaker_invoke(
            endpoint="midas-depth-estimation",
            payload={"image_s3_key": frame_key}
        )
        depth_maps.append(depth_map)
    
    # Calibrate scale using reference object
    scale_factor = calibrate_scale(depth_maps[0], reference_object_size_meters)
    
    # Generate 3D point cloud
    point_cloud = []
    for depth_map in depth_maps:
        points_3d = depth_to_3d(depth_map, scale_factor)
        point_cloud.extend(points_3d)
    
    # Store point cloud
    point_cloud_hash = sha256(json.dumps(point_cloud))
    s3.put_object(
        Bucket="watson-reconstructions",
        Key=f"scenes/{point_cloud_hash}.json",
        Body=json.dumps(point_cloud)
    )
    
    return {"point_cloud_hash": point_cloud_hash, "point_count": len(point_cloud)}
```

**Determinism Guarantees:**
- Depth estimation model is frozen (no training during inference)
- Scale calibration uses fixed reference
- Point cloud generation is deterministic given depth maps

---

### 3.5 Physics Simulation Engine

**Purpose:** Deterministic ballistic trajectory reconstruction

**Implementation:**
- Python physics engine with NumPy
- Fixed random seeds for any stochastic components
- All parameters logged before execution

**Core Physics Equations:**
```python
# Projectile motion with air resistance
def simulate_trajectory(initial_position, initial_velocity, timestep=0.01):
    """
    Deterministic projectile motion simulation
    
    Equations:
    - x(t) = x0 + vx*t
    - y(t) = y0 + vy*t - 0.5*g*t^2
    - vx(t) = vx0 * exp(-k*t)  # air resistance
    - vy(t) = vy0 * exp(-k*t) - g*t
    
    Where:
    - g = 9.81 m/s^2 (gravity)
    - k = 0.01 (air resistance coefficient)
    """
    # Fix random seed for reproducibility
    np.random.seed(42)
    
    # Constants (logged)
    g = 9.81  # m/s^2
    k = 0.01  # air resistance
    
    # Log parameters
    params = {
        "initial_position": initial_position,
        "initial_velocity": initial_velocity,
        "timestep": timestep,
        "gravity": g,
        "air_resistance": k,
        "random_seed": 42
    }
    log_simulation_params(params)
    
    # Simulate
    trajectory = []
    pos = np.array(initial_position)
    vel = np.array(initial_velocity)
    t = 0
    
    while pos[2] >= 0:  # z-coordinate (height) >= 0
        trajectory.append(pos.copy())
        
        # Update velocity
        vel[0] *= np.exp(-k * timestep)
        vel[1] *= np.exp(-k * timestep)
        vel[2] = vel[2] * np.exp(-k * timestep) - g * timestep
        
        # Update position
        pos += vel * timestep
        t += timestep
    
    return trajectory
```


**Inverse Trajectory Calculation:**
```python
def estimate_muzzle_origin(impact_points):
    """
    Estimate muzzle origin from bullet impact points
    
    Uses least-squares optimization to find origin that minimizes
    trajectory error across all impact points.
    """
    # Fix random seed
    np.random.seed(42)
    
    # Initial guess (centroid of impact points)
    initial_guess = np.mean(impact_points, axis=0)
    
    # Objective function: minimize trajectory error
    def objective(origin):
        total_error = 0
        for impact in impact_points:
            # Calculate expected trajectory
            trajectory = simulate_trajectory(origin, compute_velocity(origin, impact))
            # Measure error at impact point
            error = np.linalg.norm(trajectory[-1] - impact)
            total_error += error ** 2
        return total_error
    
    # Optimize (deterministic optimizer)
    result = scipy.optimize.minimize(
        objective,
        initial_guess,
        method="L-BFGS-B",
        options={"maxiter": 1000}
    )
    
    # Calculate confidence interval
    confidence_interval = calculate_confidence(result, impact_points)
    
    return {
        "muzzle_origin": result.x.tolist(),
        "error": result.fun,
        "confidence_95": confidence_interval
    }
```

**Determinism Guarantees:**
- Fixed random seed (42)
- Deterministic optimizer (L-BFGS-B with fixed parameters)
- All floating-point operations use NumPy (consistent across platforms)
- Parameters logged before execution

---

### 3.6 Blood Spatter Analysis Service

**Purpose:** Estimate point of origin from blood spatter patterns

**Implementation:**
- Ellipse geometry analysis for impact angles
- Triangulation from multiple spatter samples
- Basic fluid dynamics constraints

**Pseudocode:**
```python
def analyze_blood_spatter(spatter_samples):
    """
    Estimate point of origin from blood spatter ellipses
    
    Each sample contains:
    - major_axis: length of ellipse major axis (mm)
    - minor_axis: length of ellipse minor axis (mm)
    - position: 3D coordinates of spatter (mm)
    - direction: surface normal vector
    """
    # Calculate impact angles
    impact_angles = []
    for sample in spatter_samples:
        # Impact angle from ellipse geometry
        angle = np.arcsin(sample["minor_axis"] / sample["major_axis"])
        impact_angles.append(angle)
    
    # Triangulate origin
    origin_candidates = []
    for i, sample in enumerate(spatter_samples):
        # Ray from spatter position along impact angle
        ray_direction = compute_ray_direction(sample["direction"], impact_angles[i])
        origin_candidates.append({
            "position": sample["position"],
            "direction": ray_direction
        })
    
    # Find point that minimizes distance to all rays
    origin = triangulate_rays(origin_candidates)
    
    # Calculate error
    error = calculate_triangulation_error(origin, origin_candidates)
    
    return {
        "origin": origin.tolist(),
        "error_mm": error,
        "sample_count": len(spatter_samples)
    }
```

**Determinism Guarantees:**
- Trigonometric functions are deterministic
- Triangulation uses deterministic linear algebra
- No stochastic components

---

### 3.7 Cryptographic Verification Service

**Purpose:** Generate and verify hashes for all inputs and outputs

**Implementation:**
- SHA-256 hashing for all files and data structures
- Hash storage in QLDB
- Verification API endpoint

**Pseudocode:**
```python
def generate_output_hash(simulation_result):
    """
    Generate deterministic hash of simulation output
    """
    # Serialize to canonical JSON (sorted keys)
    canonical_json = json.dumps(simulation_result, sort_keys=True, separators=(',', ':'))
    
    # Generate SHA-256 hash
    output_hash = hashlib.sha256(canonical_json.encode('utf-8')).hexdigest()
    
    # Store in QLDB
    qldb.execute(
        "INSERT INTO output_hashes VALUES (?, ?, ?)",
        simulation_result["case_id"],
        output_hash,
        iso8601_now()
    )
    
    return output_hash

def verify_replay(original_hash, replay_result):
    """
    Verify that replay produces identical output
    """
    replay_hash = generate_output_hash(replay_result)
    
    if replay_hash == original_hash:
        return {"status": "verified", "match": True}
    else:
        return {"status": "mismatch", "match": False, "original": original_hash, "replay": replay_hash}
```

**Determinism Guarantees:**
- SHA-256 is cryptographically deterministic
- Canonical JSON ensures consistent serialization
- QLDB provides immutable storage

---

### 3.8 Audit Logging Service

**Purpose:** Immutable logging of all operations

**Implementation:**
- AWS QLDB for cryptographic verification
- Structured logging with ISO 8601 timestamps
- Query API for chain of custody

**Schema:**
```sql
-- QLDB table schemas
CREATE TABLE evidence_log (
    case_id STRING,
    evidence_hash STRING,
    officer_id STRING,
    timestamp STRING,
    operation STRING
)

CREATE TABLE simulation_log (
    case_id STRING,
    simulation_id STRING,
    parameters JSON,
    output_hash STRING,
    timestamp STRING,
    user_id STRING
)

CREATE TABLE access_log (
    case_id STRING,
    user_id STRING,
    operation STRING,
    timestamp STRING,
    ip_address STRING
)
```

**Pseudocode:**
```python
def log_simulation(case_id, params, output_hash, user_id):
    """
    Log simulation execution to QLDB
    """
    simulation_id = generate_uuid()
    
    qldb.execute(
        """
        INSERT INTO simulation_log 
        VALUES (?, ?, ?, ?, ?, ?)
        """,
        case_id,
        simulation_id,
        json.dumps(params),
        output_hash,
        iso8601_now(),
        user_id
    )
    
    return simulation_id

def query_chain_of_custody(case_id):
    """
    Retrieve complete audit trail for a case
    """
    result = qldb.execute(
        """
        SELECT * FROM evidence_log 
        WHERE case_id = ? 
        ORDER BY timestamp ASC
        """,
        case_id
    )
    
    return list(result)
```

**Determinism Guarantees:**
- QLDB provides cryptographic proof of log integrity
- Timestamps are immutable once written
- Query results are deterministic

---

## 4. Data Flow Design

### 4.1 End-to-End Analysis Workflow

```
1. Evidence Ingestion
   ├─ User uploads video via web UI
   ├─ S3 triggers Lambda for hash generation
   ├─ Evidence stored in S3 with object lock
   └─ Chain of custody logged in QLDB

2. Frame Extraction
   ├─ Lambda extracts frames at 1 fps
   ├─ Frames stored in S3
   └─ Frame hashes logged

3. Object Detection (Neural Layer)
   ├─ Rekognition detects bullet impacts
   ├─ Detection results stored in S3
   └─ Results logged (not trusted blindly)

4. Manual Coordinate Input
   ├─ User reviews neural detections
   ├─ User confirms or corrects coordinates
   └─ Confirmed coordinates logged

5. Scene Reconstruction
   ├─ Depth estimation from frames
   ├─ Scale calibration
   └─ 3D point cloud generated

6. Physics Simulation (Symbolic Layer)
   ├─ Step Functions orchestrates workflow
   ├─ Physics engine calculates trajectories
   ├─ Muzzle origin estimated
   └─ Simulation parameters logged

7. Output Generation
   ├─ Trajectory results stored in S3
   ├─ Output hash generated
   ├─ Hash stored in QLDB
   └─ PDF report generated

8. Verification
   ├─ User can replay simulation
   ├─ Replay hash compared to original
   └─ Verification result logged
```

### 4.2 Data Flow Diagram

```
┌─────────┐
│ Video   │
│ Upload  │
└────┬────┘
     │
     ▼
┌─────────────────┐      ┌──────────────┐
│ S3 Evidence     │─────▶│ Lambda       │
│ Bucket          │      │ Hash Gen     │
└─────────────────┘      └──────┬───────┘
                                │
                                ▼
                         ┌──────────────┐
                         │ QLDB         │
                         │ Evidence Log │
                         └──────────────┘
                                │
                                ▼
                         ┌──────────────┐
                         │ Lambda       │
                         │ Frame Extract│
                         └──────┬───────┘
                                │
                                ▼
                         ┌──────────────┐
                         │ S3 Frames    │
                         └──────┬───────┘
                                │
                                ▼
                         ┌──────────────┐
                         │ Rekognition  │
                         │ Detection    │
                         └──────┬───────┘
                                │
                                ▼
                         ┌──────────────┐
                         │ Manual       │
                         │ Validation   │
                         └──────┬───────┘
                                │
                                ▼
                         ┌──────────────┐
                         │ Step         │
                         │ Functions    │
                         └──────┬───────┘
                                │
                                ▼
                         ┌──────────────┐
                         │ ECS Fargate  │
                         │ Physics Sim  │
                         └──────┬───────┘
                                │
                                ▼
                         ┌──────────────┐
                         │ S3 Results   │
                         └──────┬───────┘
                                │
                                ▼
                         ┌──────────────┐
                         │ Lambda       │
                         │ Hash + Report│
                         └──────┬───────┘
                                │
                                ▼
                         ┌──────────────┐
                         │ QLDB         │
                         │ Output Log   │
                         └──────────────┘
```

---

## 5. Neuro-Symbolic Separation Architecture

### 5.1 Design Principle

The neuro-symbolic architecture enforces a strict boundary between perception and reasoning:

**Neural Layer (Perception):**
- Detects objects in images (bullet holes, blood spatter)
- Outputs bounding boxes and confidence scores
- No decision-making authority
- Outputs are suggestions only

**Symbolic Layer (Reasoning):**
- Applies deterministic physics equations
- Makes all final decisions
- Uses manually validated coordinates
- Produces court-admissible outputs

### 5.2 Boundary Enforcement

```python
class NeuroSymbolicBoundary:
    """
    Enforces separation between neural and symbolic layers
    """
    
    def __init__(self):
        self.neural_outputs = []
        self.validated_inputs = []
    
    def receive_neural_output(self, detection):
        """
        Log neural output but do not trust it
        """
        self.neural_outputs.append({
            "detection": detection,
            "timestamp": iso8601_now(),
            "status": "unvalidated"
        })
        
        # Neural outputs are NOT passed to symbolic layer
        return {"status": "logged", "requires_validation": True}
    
    def validate_coordinate(self, coordinate, user_id):
        """
        User manually validates coordinate
        """
        validated = {
            "coordinate": coordinate,
            "validated_by": user_id,
            "timestamp": iso8601_now(),
            "status": "validated"
        }
        self.validated_inputs.append(validated)
        
        # Only validated inputs go to symbolic layer
        return validated
    
    def symbolic_processing(self, validated_inputs):
        """
        Symbolic layer processes only validated inputs
        """
        # No neural outputs are used here
        results = physics_engine.simulate(validated_inputs)
        return results
```

### 5.3 Validation Workflow

```
Neural Detection → Logging → Manual Review → Validation → Symbolic Processing
                     ↓                           ↓
                  Audit Log              Validated Inputs
                                                ↓
                                         Physics Engine
                                                ↓
                                         Final Results
```

**Key Properties:**
- Neural outputs never directly influence final results
- All inputs to symbolic layer are manually validated
- Validation events are logged in QLDB
- Symbolic layer is deterministic and reproducible

---

## 6. Deterministic Physics Engine Design

### 6.1 Determinism Requirements

**Fixed Random Seeds:**
```python
# Global seed fixation
np.random.seed(42)
random.seed(42)

# Per-simulation seed logging
def run_simulation(params):
    seed = params.get("random_seed", 42)
    np.random.seed(seed)
    log_parameter("random_seed", seed)
    # ... simulation code
```

**Floating-Point Consistency:**
```python
# Use NumPy for consistent floating-point operations
import numpy as np

# Set floating-point precision
np.set_printoptions(precision=15)

# Use deterministic algorithms
np.linalg.lstsq(A, b, rcond=None)  # Deterministic least squares
```

**Parameter Logging:**
```python
def log_all_parameters(params):
    """
    Log every parameter before simulation
    """
    logged_params = {
        "gravity": params["gravity"],
        "air_resistance": params["air_resistance"],
        "timestep": params["timestep"],
        "initial_position": params["initial_position"],
        "initial_velocity": params["initial_velocity"],
        "random_seed": params["random_seed"],
        "timestamp": iso8601_now()
    }
    
    # Store in QLDB
    qldb.execute(
        "INSERT INTO simulation_params VALUES (?)",
        json.dumps(logged_params)
    )
    
    return logged_params
```


### 6.2 Replay Mechanism

```python
def replay_simulation(simulation_id):
    """
    Replay simulation using stored parameters
    """
    # Retrieve parameters from QLDB
    params = qldb.execute(
        "SELECT * FROM simulation_params WHERE simulation_id = ?",
        simulation_id
    ).fetchone()
    
    # Retrieve original output hash
    original_hash = qldb.execute(
        "SELECT output_hash FROM simulation_log WHERE simulation_id = ?",
        simulation_id
    ).fetchone()["output_hash"]
    
    # Re-run simulation with exact same parameters
    replay_result = run_simulation(params)
    
    # Generate hash of replay output
    replay_hash = generate_output_hash(replay_result)
    
    # Verify match
    if replay_hash == original_hash:
        log_verification_success(simulation_id)
        return {"status": "verified", "match": True}
    else:
        log_verification_failure(simulation_id, original_hash, replay_hash)
        return {"status": "mismatch", "match": False}
```

**Replay Guarantees:**
- Identical parameters produce identical outputs
- Hash comparison verifies byte-level equality
- Verification results are logged immutably

---

## 7. Evidence Hashing & Audit Layer

### 7.1 Hash Generation Strategy

**Input Evidence Hashing:**
```python
def hash_evidence_file(file_path):
    """
    Generate SHA-256 hash of evidence file
    """
    sha256_hash = hashlib.sha256()
    
    with open(file_path, "rb") as f:
        # Read in chunks for large files
        for byte_block in iter(lambda: f.read(4096), b""):
            sha256_hash.update(byte_block)
    
    return sha256_hash.hexdigest()
```

**Output Data Hashing:**
```python
def hash_simulation_output(output_data):
    """
    Generate deterministic hash of simulation output
    """
    # Canonical JSON serialization
    canonical_json = json.dumps(
        output_data,
        sort_keys=True,
        separators=(',', ':'),
        ensure_ascii=True
    )
    
    # SHA-256 hash
    return hashlib.sha256(canonical_json.encode('utf-8')).hexdigest()
```

### 7.2 QLDB Audit Schema

```sql
-- Evidence integrity table
CREATE TABLE evidence_integrity (
    evidence_id STRING,
    case_id STRING,
    file_hash STRING,
    file_size_bytes INTEGER,
    upload_timestamp STRING,
    uploaded_by STRING,
    s3_location STRING
)

-- Simulation execution table
CREATE TABLE simulation_execution (
    simulation_id STRING,
    case_id STRING,
    parameters JSON,
    output_hash STRING,
    execution_timestamp STRING,
    executed_by STRING,
    execution_time_seconds FLOAT
)

-- Verification table
CREATE TABLE verification_log (
    verification_id STRING,
    simulation_id STRING,
    original_hash STRING,
    replay_hash STRING,
    match BOOLEAN,
    verification_timestamp STRING,
    verified_by STRING
)

-- Access control table
CREATE TABLE access_log (
    access_id STRING,
    case_id STRING,
    user_id STRING,
    operation STRING,
    timestamp STRING,
    ip_address STRING,
    user_agent STRING
)
```

### 7.3 Chain of Custody Query

```python
def get_chain_of_custody(case_id):
    """
    Retrieve complete audit trail with cryptographic proof
    """
    # Query all events for case
    events = qldb.execute(
        """
        SELECT * FROM evidence_integrity 
        WHERE case_id = ?
        UNION ALL
        SELECT * FROM simulation_execution 
        WHERE case_id = ?
        UNION ALL
        SELECT * FROM access_log 
        WHERE case_id = ?
        ORDER BY timestamp ASC
        """,
        case_id, case_id, case_id
    )
    
    # Get QLDB cryptographic proof
    proof = qldb.get_revision_proof(case_id)
    
    return {
        "case_id": case_id,
        "events": list(events),
        "cryptographic_proof": proof,
        "verified": verify_qldb_proof(proof)
    }
```

---

## 8. AWS Service Mapping

### 8.1 Service Selection Rationale

| Component | AWS Service | Justification |
|-----------|-------------|---------------|
| Evidence Storage | S3 + Glacier | 99.999999999% durability, object lock for immutability |
| Audit Logging | QLDB | Cryptographic verification, immutable ledger |
| Object Detection | Rekognition Custom Labels | Managed service, no infrastructure management |
| Physics Engine | ECS Fargate | Flexible compute, supports long-running simulations |
| Orchestration | Step Functions | Visual workflow, error handling, state management |
| API Layer | API Gateway + Lambda | Serverless, auto-scaling, IAM integration |
| Frontend Hosting | Amplify | CI/CD, HTTPS, CloudFront CDN |
| Monitoring | CloudWatch | Native AWS integration, custom metrics |
| Authentication | Cognito | User pools, MFA, IAM integration |
| Key Management | KMS | Automatic key rotation, audit logging |

### 8.2 Compute Architecture

**Lambda Functions:**
- Evidence ingestion (hash generation)
- Frame extraction (FFmpeg layer)
- API endpoints (CRUD operations)
- Report generation (PDF creation)

**ECS Fargate Tasks:**
- Physics simulation (CPU-intensive)
- Scene reconstruction (memory-intensive)
- Batch processing (multiple cases)

**AWS Batch (Optional):**
- Heavy simulations requiring GPU
- Parallel trajectory analysis
- Large-scale testing

### 8.3 Storage Architecture

**S3 Buckets:**
```
watson-evidence/
├── raw/                    # Original video files
│   └── {case_id}/
│       └── {hash}.mp4
├── frames/                 # Extracted frames
│   └── {case_id}/
│       └── frame_{idx}.png
└── results/                # Simulation outputs
    └── {case_id}/
        └── {simulation_id}.json

watson-archives/            # Glacier Deep Archive
└── {case_id}/
    └── evidence_{hash}.mp4
```

**S3 Configuration:**
```python
# Evidence bucket with object lock
s3.create_bucket(
    Bucket="watson-evidence",
    ObjectLockEnabledForBucket=True
)

# Lifecycle policy for archival
s3.put_bucket_lifecycle_configuration(
    Bucket="watson-evidence",
    LifecycleConfiguration={
        "Rules": [
            {
                "Id": "archive-old-evidence",
                "Status": "Enabled",
                "Transitions": [
                    {
                        "Days": 90,
                        "StorageClass": "GLACIER"
                    },
                    {
                        "Days": 365,
                        "StorageClass": "DEEP_ARCHIVE"
                    }
                ]
            }
        ]
    }
)
```

### 8.4 Network Architecture

**VPC Configuration:**
```
VPC: 10.0.0.0/16
├── Public Subnets (10.0.1.0/24, 10.0.2.0/24)
│   └── Application Load Balancer
├── Private Subnets (10.0.10.0/24, 10.0.11.0/24)
│   ├── ECS Fargate Tasks
│   └── Lambda Functions (VPC-enabled)
└── VPC Endpoints
    ├── S3 Gateway Endpoint
    ├── QLDB Interface Endpoint
    └── SageMaker Interface Endpoint
```

**Security Groups:**
```python
# ECS task security group
ecs_sg = ec2.create_security_group(
    GroupName="watson-ecs-sg",
    Description="Security group for physics engine tasks",
    VpcId=vpc_id
)

# Allow outbound to S3 and QLDB only
ec2.authorize_security_group_egress(
    GroupId=ecs_sg["GroupId"],
    IpPermissions=[
        {
            "IpProtocol": "tcp",
            "FromPort": 443,
            "ToPort": 443,
            "UserIdGroupPairs": [{"GroupId": vpc_endpoint_sg}]
        }
    ]
)
```

---

## 9. Security Architecture

### 9.1 Authentication & Authorization

**Cognito User Pool Configuration:**
```python
cognito.create_user_pool(
    PoolName="watson-users",
    Policies={
        "PasswordPolicy": {
            "MinimumLength": 12,
            "RequireUppercase": True,
            "RequireLowercase": True,
            "RequireNumbers": True,
            "RequireSymbols": True
        }
    },
    MfaConfiguration="REQUIRED",
    UserAttributeUpdateSettings={
        "AttributesRequireVerificationBeforeUpdate": ["email"]
    }
)
```

**IAM Role-Based Access Control:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject"
      ],
      "Resource": "arn:aws:s3:::watson-evidence/${cognito-identity.amazonaws.com:sub}/*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "qldb:SendCommand"
      ],
      "Resource": "arn:aws:qldb:us-east-1:*:ledger/watson-audit",
      "Condition": {
        "StringEquals": {
          "qldb:operation": ["SELECT"]
        }
      }
    }
  ]
}
```

### 9.2 Data Encryption

**Encryption at Rest:**
- S3: AES-256 with KMS customer-managed keys
- QLDB: Encrypted by default with AWS-managed keys
- ECS: Encrypted EBS volumes for task storage

**Encryption in Transit:**
- TLS 1.3 for all API communications
- VPC endpoints for internal AWS service communication
- Certificate management via ACM

**KMS Key Policy:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "Enable IAM User Permissions",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::ACCOUNT_ID:root"
      },
      "Action": "kms:*",
      "Resource": "*"
    },
    {
      "Sid": "Allow Watson Services",
      "Effect": "Allow",
      "Principal": {
        "Service": [
          "s3.amazonaws.com",
          "lambda.amazonaws.com",
          "ecs-tasks.amazonaws.com"
        ]
      },
      "Action": [
        "kms:Decrypt",
        "kms:GenerateDataKey"
      ],
      "Resource": "*"
    }
  ]
}
```

### 9.3 Compliance Controls

**Evidence Integrity Controls:**
- S3 Object Lock in Governance mode
- MFA Delete enabled on evidence bucket
- Versioning enabled with lifecycle policies
- CloudTrail logging for all S3 API calls

**Access Controls:**
- Least-privilege IAM policies
- Service Control Policies (SCPs) for organization-wide restrictions
- Resource-based policies on S3 and QLDB
- VPC isolation for compute resources

**Audit Controls:**
- CloudTrail enabled for all API calls
- QLDB for immutable audit logs
- CloudWatch Logs for application logging
- AWS Config for resource compliance monitoring

---

## 10. Testing Strategy

### 10.1 Unit Testing

**Physics Engine Tests:**
```python
import pytest
import numpy as np

def test_projectile_motion_determinism():
    """Test that identical inputs produce identical outputs"""
    initial_pos = [0, 0, 0]
    initial_vel = [100, 0, 50]
    
    # Run simulation twice
    trajectory1 = simulate_trajectory(initial_pos, initial_vel)
    trajectory2 = simulate_trajectory(initial_pos, initial_vel)
    
    # Assert byte-level equality
    assert np.array_equal(trajectory1, trajectory2)
    assert hash_trajectory(trajectory1) == hash_trajectory(trajectory2)

def test_muzzle_origin_estimation_accuracy():
    """Test trajectory estimation against ground truth"""
    # Known ground truth
    true_origin = [0, 0, 1.5]  # 1.5m height
    true_velocity = [200, 0, 10]  # m/s
    
    # Generate impact points from known trajectory
    trajectory = simulate_trajectory(true_origin, true_velocity)
    impact_points = [trajectory[10], trajectory[20], trajectory[30]]
    
    # Estimate origin
    estimated = estimate_muzzle_origin(impact_points)
    
    # Assert accuracy within 5%
    error = np.linalg.norm(estimated["muzzle_origin"] - true_origin)
    assert error / np.linalg.norm(true_origin) < 0.05

def test_hash_consistency():
    """Test that hash generation is deterministic"""
    data = {"trajectory": [[0, 0, 0], [1, 1, 1]], "timestamp": "2024-01-01T00:00:00Z"}
    
    hash1 = hash_simulation_output(data)
    hash2 = hash_simulation_output(data)
    
    assert hash1 == hash2
```


### 10.2 Property-Based Testing

**Testing Framework:** Hypothesis (Python)

**Property Tests:**
```python
from hypothesis import given, strategies as st, settings
import hypothesis.strategies as st

@given(
    initial_pos=st.lists(st.floats(min_value=-100, max_value=100), min_size=3, max_size=3),
    initial_vel=st.lists(st.floats(min_value=-500, max_value=500), min_size=3, max_size=3)
)
@settings(max_examples=1000)
def test_simulation_determinism_property(initial_pos, initial_vel):
    """
    Property: Simulation is deterministic
    For all valid inputs, running simulation twice produces identical outputs
    """
    result1 = simulate_trajectory(initial_pos, initial_vel)
    result2 = simulate_trajectory(initial_pos, initial_vel)
    
    assert hash_trajectory(result1) == hash_trajectory(result2)

@given(
    impact_points=st.lists(
        st.lists(st.floats(min_value=-100, max_value=100), min_size=3, max_size=3),
        min_size=2,
        max_size=10
    )
)
def test_muzzle_estimation_idempotence(impact_points):
    """
    Property: Muzzle origin estimation is idempotent
    Estimating twice with same inputs produces identical results
    """
    estimate1 = estimate_muzzle_origin(impact_points)
    estimate2 = estimate_muzzle_origin(impact_points)
    
    assert estimate1 == estimate2

@given(
    data=st.dictionaries(
        keys=st.text(min_size=1),
        values=st.one_of(st.floats(), st.integers(), st.text())
    )
)
def test_hash_determinism_property(data):
    """
    Property: Hash generation is deterministic
    For all JSON-serializable data, hash is consistent
    """
    hash1 = hash_simulation_output(data)
    hash2 = hash_simulation_output(data)
    
    assert hash1 == hash2
    assert len(hash1) == 64  # SHA-256 produces 64 hex characters

@given(
    params=st.fixed_dictionaries({
        "gravity": st.floats(min_value=9.0, max_value=10.0),
        "air_resistance": st.floats(min_value=0.0, max_value=0.1),
        "timestep": st.floats(min_value=0.001, max_value=0.1)
    })
)
def test_replay_consistency_property(params):
    """
    Property: Replay produces identical results
    For all parameter sets, replay matches original execution
    """
    # Original execution
    original_result = run_simulation(params)
    original_hash = hash_simulation_output(original_result)
    
    # Replay execution
    replay_result = run_simulation(params)
    replay_hash = hash_simulation_output(replay_result)
    
    assert original_hash == replay_hash
```

### 10.3 Integration Testing

**End-to-End Test:**
```python
def test_full_analysis_workflow():
    """
    Integration test: Complete analysis from ingestion to report
    """
    # 1. Ingest evidence
    video_path = "test_data/ballistic_test_01.mp4"
    evidence_id = ingest_evidence(video_path, "TEST-001", "officer-123")
    
    # 2. Extract frames
    frames = extract_frames(evidence_id)
    assert len(frames["frame_keys"]) > 0
    
    # 3. Detect objects (neural layer)
    detections = detect_objects(frames["frame_keys"][0], model_arn)
    assert len(detections["detections"]) > 0
    
    # 4. Manual validation
    validated_coords = [
        {"x": 1.5, "y": 2.0, "z": 1.2},
        {"x": 3.0, "y": 2.5, "z": 1.0}
    ]
    
    # 5. Run physics simulation
    result = estimate_muzzle_origin(validated_coords)
    assert "muzzle_origin" in result
    assert "error" in result
    
    # 6. Generate hash
    output_hash = generate_output_hash(result)
    assert len(output_hash) == 64
    
    # 7. Verify replay
    replay_result = replay_simulation(result["simulation_id"])
    assert replay_result["match"] is True
    
    # 8. Query audit trail
    audit_trail = get_chain_of_custody("TEST-001")
    assert len(audit_trail["events"]) >= 5

def test_neural_symbolic_separation():
    """
    Integration test: Verify neural outputs don't affect symbolic results
    """
    # Neural detection with low confidence
    neural_output = {
        "detections": [{"confidence": 0.3, "bbox": [100, 100, 50, 50]}]
    }
    
    # Manual validation with different coordinates
    validated_coords = [{"x": 5.0, "y": 5.0, "z": 1.5}]
    
    # Run simulation
    result = run_physics_simulation(validated_coords)
    
    # Assert result depends only on validated coords, not neural output
    assert result["input_source"] == "manual_validation"
    assert "neural_confidence" not in result
```

### 10.4 Stress Testing

**Concurrent Analysis Test:**
```python
import concurrent.futures

def test_concurrent_simulations():
    """
    Stress test: Multiple simulations running concurrently
    """
    test_cases = [
        {"case_id": f"STRESS-{i:03d}", "coords": generate_random_coords()}
        for i in range(10)
    ]
    
    with concurrent.futures.ThreadPoolExecutor(max_workers=10) as executor:
        futures = [
            executor.submit(run_full_analysis, case)
            for case in test_cases
        ]
        
        results = [f.result() for f in concurrent.futures.as_completed(futures)]
    
    # Verify all completed successfully
    assert len(results) == 10
    assert all(r["status"] == "completed" for r in results)
    
    # Verify determinism: re-run and compare hashes
    for case in test_cases:
        replay = replay_simulation(case["case_id"])
        assert replay["match"] is True
```

---

## 11. Correctness Properties

### P1: Determinism Property
**Statement:** For all valid inputs I and simulation parameters P, executing the simulation twice produces identical outputs.

**Formal Definition:**
```
∀ I, P: simulate(I, P) = simulate(I, P)
```

**Verification Method:**
- Property-based testing with Hypothesis
- Run simulation 1000 times with random inputs
- Assert hash equality across all runs

**Test Implementation:**
```python
@given(inputs=valid_simulation_inputs())
def test_determinism_property(inputs):
    result1 = simulate(inputs)
    result2 = simulate(inputs)
    assert hash(result1) == hash(result2)
```

**Validates:** Requirements DR-1.1, DR-1.2, DR-1.3

---

### P2: Idempotent Simulation Property
**Statement:** Running a simulation multiple times with the same parameters does not change the system state or produce different results.

**Formal Definition:**
```
∀ I, P, n ∈ ℕ: simulate^n(I, P) = simulate(I, P)
```

**Verification Method:**
- Execute simulation n times sequentially
- Verify all outputs are identical
- Check that no state is accumulated

**Test Implementation:**
```python
def test_idempotence_property():
    inputs = generate_test_inputs()
    results = [simulate(inputs) for _ in range(10)]
    assert all(hash(r) == hash(results[0]) for r in results)
```

**Validates:** Requirements DR-1.1, PSR-2.1

---

### P3: Replay Consistency Property
**Statement:** Replaying a simulation using stored parameters produces output matching the original execution hash.

**Formal Definition:**
```
∀ S: hash(replay(S.params)) = S.original_hash
```

**Verification Method:**
- Store simulation parameters and output hash
- Replay simulation with stored parameters
- Compare replay hash to original hash

**Test Implementation:**
```python
def test_replay_consistency():
    original = run_simulation(params)
    original_hash = hash(original)
    store_params(params, original_hash)
    
    replay = replay_simulation(params)
    replay_hash = hash(replay)
    
    assert replay_hash == original_hash
```

**Validates:** Requirements DR-3.1, DR-3.2, DR-3.3

---

### P4: Chain-of-Custody Integrity Property
**Statement:** All evidence operations are logged immutably, and tampering is detectable through hash verification.

**Formal Definition:**
```
∀ E: verify_hash(E) = True ⟹ E is unmodified
∀ E: verify_hash(E) = False ⟹ E is tampered
```

**Verification Method:**
- Hash evidence at ingestion
- Store hash in QLDB
- Verify hash before processing
- Detect any modifications

**Test Implementation:**
```python
def test_chain_of_custody_integrity():
    evidence = ingest_evidence(video_file)
    original_hash = evidence["hash"]
    
    # Verify hash matches
    assert verify_evidence_hash(evidence["id"]) == original_hash
    
    # Simulate tampering
    tamper_evidence(evidence["id"])
    
    # Verify tampering is detected
    with pytest.raises(TamperDetectedException):
        verify_evidence_hash(evidence["id"])
```

**Validates:** Requirements EI-3.1, EI-3.2, EI-3.3

---

### P5: No Generative Reasoning Property
**Statement:** Final reconstruction decisions are derived solely from deterministic physics equations, not from neural network outputs.

**Formal Definition:**
```
∀ R: R.final_output = f(R.validated_inputs, R.physics_params)
where f is a deterministic function and R.neural_outputs ∉ f
```

**Verification Method:**
- Trace data flow from input to output
- Verify neural outputs are logged but not used in calculations
- Assert symbolic layer receives only validated inputs

**Test Implementation:**
```python
def test_no_generative_reasoning():
    # Neural detection with arbitrary confidence
    neural_output = detect_objects(frame)
    
    # Manual validation (different from neural output)
    validated_input = manual_validate(frame)
    
    # Run simulation
    result = simulate(validated_input)
    
    # Assert result depends only on validated input
    assert result["input_source"] == "manual_validation"
    assert "neural_output" not in result["dependencies"]
    
    # Verify changing neural output doesn't change result
    neural_output_modified = modify_confidence(neural_output)
    result_modified = simulate(validated_input)
    
    assert hash(result) == hash(result_modified)
```

**Validates:** Requirements NSR-1.3, NSR-2.2, NSR-3.3

---

### P6: Physics Equation Correctness Property
**Statement:** Trajectory calculations conform to standard projectile motion equations within numerical precision limits.

**Formal Definition:**
```
∀ t, v₀, θ: |calculated_position(t) - theoretical_position(t, v₀, θ)| < ε
where ε is numerical precision threshold
```

**Verification Method:**
- Compare simulation output to analytical solutions
- Use known test cases with closed-form solutions
- Verify error is within acceptable bounds

**Test Implementation:**
```python
def test_physics_equation_correctness():
    # Test case: projectile with no air resistance
    v0 = 100  # m/s
    theta = 45  # degrees
    g = 9.81  # m/s^2
    
    # Analytical solution for range
    expected_range = (v0**2 * np.sin(2 * np.radians(theta))) / g
    
    # Simulation
    trajectory = simulate_trajectory(
        initial_velocity=[v0 * np.cos(np.radians(theta)), 0, v0 * np.sin(np.radians(theta))],
        air_resistance=0
    )
    simulated_range = trajectory[-1][0]
    
    # Verify within 1% error
    error = abs(simulated_range - expected_range) / expected_range
    assert error < 0.01
```

**Validates:** Requirements PSR-1.1, PSR-1.2, MPR-2.1

---

### P7: Error Propagation Property
**Statement:** Uncertainty in input measurements propagates correctly through physics calculations to output confidence intervals.

**Formal Definition:**
```
∀ I with uncertainty σᵢ: output_confidence_interval = f(σᵢ, physics_equations)
```

**Verification Method:**
- Monte Carlo simulation with input perturbations
- Verify confidence intervals contain true values
- Check coverage probability matches specified confidence level

**Test Implementation:**
```python
def test_error_propagation():
    # True values
    true_coords = [[1.0, 2.0, 1.5], [3.0, 2.5, 1.0]]
    
    # Add measurement uncertainty
    noisy_coords = [
        add_gaussian_noise(coord, sigma=0.01)
        for coord in true_coords
    ]
    
    # Estimate with confidence interval
    result = estimate_muzzle_origin(noisy_coords)
    
    # Verify confidence interval is reasonable
    assert result["confidence_95"]["lower"] < result["muzzle_origin"]
    assert result["muzzle_origin"] < result["confidence_95"]["upper"]
    
    # Monte Carlo verification
    estimates = [
        estimate_muzzle_origin(add_gaussian_noise(true_coords, 0.01))
        for _ in range(1000)
    ]
    
    # Check 95% of estimates fall within confidence interval
    within_ci = sum(
        result["confidence_95"]["lower"] < e["muzzle_origin"] < result["confidence_95"]["upper"]
        for e in estimates
    )
    assert within_ci / 1000 > 0.90  # Allow some tolerance
```

**Validates:** Requirements PSR-4.1, PSR-4.2, PSR-4.3

---

## 12. Deployment Architecture

### 12.1 Infrastructure as Code

**AWS CDK Stack Definition:**
```python
from aws_cdk import (
    Stack,
    aws_s3 as s3,
    aws_lambda as lambda_,
    aws_ecs as ecs,
    aws_qldb as qldb,
    aws_stepfunctions as sfn,
    aws_apigateway as apigw,
    Duration
)

class WatsonStack(Stack):
    def __init__(self, scope, id, **kwargs):
        super().__init__(scope, id, **kwargs)
        
        # Evidence storage bucket
        evidence_bucket = s3.Bucket(
            self, "EvidenceBucket",
            versioned=True,
            object_lock_enabled=True,
            encryption=s3.BucketEncryption.KMS,
            lifecycle_rules=[
                s3.LifecycleRule(
                    transitions=[
                        s3.Transition(
                            storage_class=s3.StorageClass.GLACIER,
                            transition_after=Duration.days(90)
                        )
                    ]
                )
            ]
        )
        
        # QLDB ledger for audit logs
        audit_ledger = qldb.CfnLedger(
            self, "AuditLedger",
            permissions_mode="STANDARD",
            deletion_protection=True
        )
        
        # Physics engine ECS cluster
        cluster = ecs.Cluster(self, "PhysicsCluster")
        
        task_definition = ecs.FargateTaskDefinition(
            self, "PhysicsTask",
            memory_limit_mib=4096,
            cpu=2048
        )
        
        container = task_definition.add_container(
            "PhysicsEngine",
            image=ecs.ContainerImage.from_asset("./physics-engine"),
            logging=ecs.LogDrivers.aws_logs(stream_prefix="watson")
        )
        
        # Step Functions workflow
        simulation_workflow = sfn.StateMachine(
            self, "SimulationWorkflow",
            definition=self.create_workflow_definition()
        )
        
        # API Gateway
        api = apigw.RestApi(
            self, "WatsonAPI",
            rest_api_name="Watson Forensic API"
        )
```

### 12.2 Deployment Pipeline

**CI/CD Workflow:**
```yaml
# .github/workflows/deploy.yml
name: Deploy Watson

on:
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - name: Run unit tests
        run: pytest tests/unit/
      - name: Run property tests
        run: pytest tests/property/ --hypothesis-profile=ci
      - name: Run integration tests
        run: pytest tests/integration/

  deploy:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v1
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: us-east-1
      - name: Deploy CDK stack
        run: |
          npm install -g aws-cdk
          cdk deploy --require-approval never
```

### 12.3 Environment Configuration

**Development Environment:**
- Single AWS region (us-east-1)
- Reduced compute resources
- Short retention periods
- Test data only

**Production Environment:**
- Multi-AZ deployment
- Full compute resources
- Long-term retention
- Real case data

**Configuration Management:**
```python
# config/production.py
ENVIRONMENT = "production"

S3_CONFIG = {
    "evidence_bucket": "watson-evidence-prod",
    "retention_days": 2555,  # 7 years
    "replication_enabled": True
}

ECS_CONFIG = {
    "task_cpu": 4096,
    "task_memory": 8192,
    "desired_count": 3
}

QLDB_CONFIG = {
    "ledger_name": "watson-audit-prod",
    "deletion_protection": True
}

MONITORING_CONFIG = {
    "alarm_email": "forensics-team@example.com",
    "log_retention_days": 365
}
```

---

## 13. Monitoring & Observability

### 13.1 CloudWatch Metrics

**Custom Metrics:**
```python
def publish_simulation_metrics(simulation_result):
    """
    Publish custom metrics to CloudWatch
    """
    cloudwatch.put_metric_data(
        Namespace="Watson/Physics",
        MetricData=[
            {
                "MetricName": "SimulationExecutionTime",
                "Value": simulation_result["execution_time_seconds"],
                "Unit": "Seconds",
                "Dimensions": [
                    {"Name": "CaseType", "Value": "Ballistic"}
                ]
            },
            {
                "MetricName": "TrajectoryError",
                "Value": simulation_result["error"],
                "Unit": "Percent",
                "Dimensions": [
                    {"Name": "CaseType", "Value": "Ballistic"}
                ]
            },
            {
                "MetricName": "ReplayConsistency",
                "Value": 1 if simulation_result["replay_match"] else 0,
                "Unit": "Count"
            }
        ]
    )
```

**Key Metrics to Monitor:**
- Simulation execution time (target: < 10 seconds)
- Trajectory estimation error (target: < 5%)
- Replay consistency rate (target: 100%)
- Hash verification success rate (target: 100%)
- API response time (target: < 200ms)
- ECS task failure rate (target: < 1%)

### 13.2 CloudWatch Alarms

```python
# Alarm for high simulation error
cloudwatch.put_metric_alarm(
    AlarmName="HighTrajectoryError",
    MetricName="TrajectoryError",
    Namespace="Watson/Physics",
    Statistic="Average",
    Period=300,
    EvaluationPeriods=2,
    Threshold=5.0,
    ComparisonOperator="GreaterThanThreshold",
    AlarmActions=[sns_topic_arn]
)

# Alarm for replay inconsistency
cloudwatch.put_metric_alarm(
    AlarmName="ReplayInconsistency",
    MetricName="ReplayConsistency",
    Namespace="Watson/Physics",
    Statistic="Sum",
    Period=300,
    EvaluationPeriods=1,
    Threshold=1,
    ComparisonOperator="LessThanThreshold",
    AlarmActions=[sns_topic_arn]
)

# Alarm for hash verification failure
cloudwatch.put_metric_alarm(
    AlarmName="HashVerificationFailure",
    MetricName="HashMismatch",
    Namespace="Watson/Evidence",
    Statistic="Sum",
    Period=60,
    EvaluationPeriods=1,
    Threshold=0,
    ComparisonOperator="GreaterThanThreshold",
    AlarmActions=[sns_topic_arn, pagerduty_arn]
)
```

### 13.3 Distributed Tracing

**X-Ray Integration:**
```python
from aws_xray_sdk.core import xray_recorder

@xray_recorder.capture("simulate_trajectory")
def simulate_trajectory(initial_position, initial_velocity):
    # Add metadata
    xray_recorder.put_metadata("initial_position", initial_position)
    xray_recorder.put_metadata("initial_velocity", initial_velocity)
    
    # Simulation code
    trajectory = run_physics_simulation(initial_position, initial_velocity)
    
    # Add annotation for filtering
    xray_recorder.put_annotation("trajectory_length", len(trajectory))
    
    return trajectory
```

### 13.4 Logging Strategy

**Structured Logging:**
```python
import json
import logging

logger = logging.getLogger("watson")

def log_simulation_event(event_type, case_id, details):
    """
    Structured logging for all simulation events
    """
    log_entry = {
        "timestamp": iso8601_now(),
        "event_type": event_type,
        "case_id": case_id,
        "details": details,
        "environment": os.getenv("ENVIRONMENT"),
        "version": os.getenv("APP_VERSION")
    }
    
    logger.info(json.dumps(log_entry))
```

**Log Levels:**
- DEBUG: Detailed simulation parameters
- INFO: Normal operations (ingestion, simulation start/end)
- WARNING: Non-critical issues (low confidence detections)
- ERROR: Failures (hash mismatch, simulation errors)
- CRITICAL: Security events (unauthorized access, tampering)

---

## 14. Failure Handling Strategy

### 14.1 Failure Modes

**Evidence Ingestion Failures:**
- Cause: Corrupted video file, unsupported format
- Detection: Hash generation failure, FFmpeg error
- Response: Reject upload, notify user, log error
- Recovery: User re-uploads valid file

**Object Detection Failures:**
- Cause: Rekognition service unavailable, low-quality frames
- Detection: API timeout, low confidence scores
- Response: Fall back to manual coordinate input
- Recovery: Retry detection after service recovery

**Physics Simulation Failures:**
- Cause: Invalid input coordinates, numerical instability
- Detection: Exception in simulation code, NaN values
- Response: Log error, notify user, request coordinate validation
- Recovery: User corrects coordinates, simulation retries

**Hash Verification Failures:**
- Cause: Evidence tampering, storage corruption
- Detection: Hash mismatch
- Response: Halt processing, alert security team, log incident
- Recovery: Investigate tampering, restore from backup if available

### 14.2 Error Handling Patterns

**Retry with Exponential Backoff:**
```python
import time
from functools import wraps

def retry_with_backoff(max_retries=3, base_delay=1):
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            for attempt in range(max_retries):
                try:
                    return func(*args, **kwargs)
                except Exception as e:
                    if attempt == max_retries - 1:
                        raise
                    delay = base_delay * (2 ** attempt)
                    logger.warning(f"Attempt {attempt + 1} failed: {e}. Retrying in {delay}s")
                    time.sleep(delay)
        return wrapper
    return decorator

@retry_with_backoff(max_retries=3)
def call_rekognition(image_s3_key):
    return rekognition.detect_custom_labels(
        Image={"S3Object": {"Bucket": "watson-frames", "Name": image_s3_key}}
    )
```

**Circuit Breaker Pattern:**
```python
class CircuitBreaker:
    def __init__(self, failure_threshold=5, timeout=60):
        self.failure_count = 0
        self.failure_threshold = failure_threshold
        self.timeout = timeout
        self.last_failure_time = None
        self.state = "CLOSED"  # CLOSED, OPEN, HALF_OPEN
    
    def call(self, func, *args, **kwargs):
        if self.state == "OPEN":
            if time.time() - self.last_failure_time > self.timeout:
                self.state = "HALF_OPEN"
            else:
                raise Exception("Circuit breaker is OPEN")
        
        try:
            result = func(*args, **kwargs)
            if self.state == "HALF_OPEN":
                self.state = "CLOSED"
                self.failure_count = 0
            return result
        except Exception as e:
            self.failure_count += 1
            self.last_failure_time = time.time()
            if self.failure_count >= self.failure_threshold:
                self.state = "OPEN"
            raise

# Usage
rekognition_breaker = CircuitBreaker()

def detect_with_circuit_breaker(image_key):
    return rekognition_breaker.call(call_rekognition, image_key)
```

### 14.3 Step Functions Error Handling

**Workflow with Retry and Catch:**
```json
{
  "Comment": "Ballistic reconstruction workflow with error handling",
  "StartAt": "ExtractFrames",
  "States": {
    "ExtractFrames": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:ACCOUNT:function:extract-frames",
      "Retry": [
        {
          "ErrorEquals": ["States.TaskFailed"],
          "IntervalSeconds": 2,
          "MaxAttempts": 3,
          "BackoffRate": 2.0
        }
      ],
      "Catch": [
        {
          "ErrorEquals": ["States.ALL"],
          "Next": "FrameExtractionFailed"
        }
      ],
      "Next": "DetectObjects"
    },
    "DetectObjects": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:ACCOUNT:function:detect-objects",
      "Retry": [
        {
          "ErrorEquals": ["RekognitionThrottledException"],
          "IntervalSeconds": 5,
          "MaxAttempts": 5,
          "BackoffRate": 2.0
        }
      ],
      "Catch": [
        {
          "ErrorEquals": ["States.ALL"],
          "Next": "FallbackToManualInput"
        }
      ],
      "Next": "RunSimulation"
    },
    "RunSimulation": {
      "Type": "Task",
      "Resource": "arn:aws:ecs:us-east-1:ACCOUNT:cluster/physics-cluster",
      "Retry": [
        {
          "ErrorEquals": ["NumericalInstabilityError"],
          "MaxAttempts": 0
        }
      ],
      "Catch": [
        {
          "ErrorEquals": ["States.ALL"],
          "Next": "SimulationFailed"
        }
      ],
      "Next": "GenerateReport"
    },
    "FallbackToManualInput": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:ACCOUNT:function:request-manual-input",
      "Next": "RunSimulation"
    },
    "FrameExtractionFailed": {
      "Type": "Fail",
      "Error": "FrameExtractionError",
      "Cause": "Failed to extract frames after retries"
    },
    "SimulationFailed": {
      "Type": "Fail",
      "Error": "SimulationError",
      "Cause": "Physics simulation failed"
    },
    "GenerateReport": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:ACCOUNT:function:generate-report",
      "End": true
    }
  }
}
```

### 14.4 Data Consistency Guarantees

**Transactional Operations:**
```python
def ingest_evidence_transactional(video_file, case_id):
    """
    Atomic evidence ingestion with rollback on failure
    """
    s3_key = None
    qldb_record_id = None
    
    try:
        # Step 1: Upload to S3
        file_hash = sha256(video_file)
        s3_key = f"evidence/{case_id}/{file_hash}.mp4"
        s3.put_object(Bucket="watson-evidence", Key=s3_key, Body=video_file)
        
        # Step 2: Log to QLDB
        qldb_record_id = qldb.execute(
            "INSERT INTO evidence_log VALUES (?, ?, ?)",
            case_id, file_hash, iso8601_now()
        )
        
        return {"status": "success", "evidence_id": file_hash}
        
    except Exception as e:
        # Rollback: Delete S3 object if QLDB insert failed
        if s3_key and not qldb_record_id:
            s3.delete_object(Bucket="watson-evidence", Key=s3_key)
        
        logger.error(f"Evidence ingestion failed: {e}")
        raise
```

---

## 15. Scalability Considerations

### 15.1 Horizontal Scaling

**ECS Auto Scaling:**
```python
# Auto scaling for physics engine tasks
autoscaling = ecs.ScalableTarget(
    self, "PhysicsScaling",
    service_namespace=autoscaling.ServiceNamespace.ECS,
    resource_id=f"service/{cluster.cluster_name}/{service.service_name}",
    scalable_dimension="ecs:service:DesiredCount",
    min_capacity=1,
    max_capacity=10
)

# Scale based on CPU utilization
autoscaling.scale_on_cpu_utilization(
    "CpuScaling",
    target_utilization_percent=70,
    scale_in_cooldown=Duration.seconds(60),
    scale_out_cooldown=Duration.seconds(60)
)

# Scale based on custom metric (queue depth)
autoscaling.scale_on_metric(
    "QueueDepthScaling",
    metric=cloudwatch.Metric(
        namespace="Watson/Queue",
        metric_name="PendingSimulations"
    ),
    scaling_steps=[
        {"upper": 10, "change": 0},
        {"lower": 10, "change": +1},
        {"lower": 50, "change": +3}
    ]
)
```

**Lambda Concurrency:**
```python
# Reserve concurrency for critical functions
lambda_function.add_reserved_concurrent_executions(10)

# Provisioned concurrency for predictable load
lambda_function.add_alias(
    "prod",
    provisioned_concurrent_executions=5
)
```

### 15.2 Data Partitioning

**S3 Prefix Strategy:**
```
watson-evidence/
├── year=2024/
│   ├── month=01/
│   │   ├── day=01/
│   │   │   └── case-001.mp4
│   │   └── day=02/
│   └── month=02/
└── year=2025/
```

**QLDB Table Partitioning:**
```sql
-- Partition by case_id for efficient queries
CREATE INDEX idx_case_id ON evidence_log (case_id)
CREATE INDEX idx_timestamp ON evidence_log (timestamp)
```

### 15.3 Caching Strategy

**API Gateway Caching:**
```python
api.add_method(
    "GET",
    integration,
    method_responses=[{
        "statusCode": "200",
        "responseParameters": {
            "method.response.header.Cache-Control": True
        }
    }],
    request_parameters={
        "method.request.querystring.case_id": True
    }
)

# Enable caching
stage = api.deployment_stage
stage.method_settings = {
    "GET/cases/*": {
        "caching_enabled": True,
        "cache_ttl": Duration.minutes(5),
        "cache_data_encrypted": True
    }
}
```

**CloudFront Distribution:**
```python
distribution = cloudfront.Distribution(
    self, "WatsonCDN",
    default_behavior=cloudfront.BehaviorOptions(
        origin=origins.S3Origin(results_bucket),
        viewer_protocol_policy=cloudfront.ViewerProtocolPolicy.REDIRECT_TO_HTTPS,
        cache_policy=cloudfront.CachePolicy.CACHING_OPTIMIZED
    )
)
```

### 15.4 Performance Optimization

**Batch Processing:**
```python
def process_cases_batch(case_ids):
    """
    Process multiple cases in parallel using AWS Batch
    """
    batch_jobs = []
    
    for case_id in case_ids:
        job = batch.submit_job(
            jobName=f"simulation-{case_id}",
            jobQueue="watson-simulation-queue",
            jobDefinition="physics-engine-job",
            parameters={"case_id": case_id}
        )
        batch_jobs.append(job)
    
    # Wait for all jobs to complete
    results = []
    for job in batch_jobs:
        result = wait_for_job_completion(job["jobId"])
        results.append(result)
    
    return results
```

**Parallel Frame Processing:**
```python
from concurrent.futures import ThreadPoolExecutor

def process_frames_parallel(frame_keys, max_workers=10):
    """
    Process frames in parallel for faster analysis
    """
    with ThreadPoolExecutor(max_workers=max_workers) as executor:
        futures = [
            executor.submit(detect_objects, frame_key)
            for frame_key in frame_keys
        ]
        
        results = [f.result() for f in futures]
    
    return results
```

---

**Document Status:** Complete  
**Version:** 1.0  
**Last Updated:** 2024-01-15  
**Next Steps:** Implementation planning, task breakdown, development sprint planning
