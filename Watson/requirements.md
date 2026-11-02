# Watson – Deterministic Forensic Reconstruction Engine
## Requirements Specification

---

## 1. Overview

Watson is a deterministic, neuro-symbolic forensic reconstruction platform designed to produce court-admissible, reproducible analysis of crime scene evidence. The system separates neural perception (object detection) from symbolic reasoning (physics-based reconstruction) to ensure deterministic outputs with cryptographic verification.

**Document Version:** 1.0  
**Target Deployment:** AWS Cloud Infrastructure  
**Scope:** MVP - Ballistic Trajectory Reconstruction

---

## 2. Problem Statement

Current digital forensic reconstruction systems exhibit the following limitations:

- Non-deterministic outputs from generative AI systems preclude court admissibility
- Lack of reproducibility across analysis runs undermines expert testimony
- Absence of cryptographic audit trails compromises evidence integrity
- Heavy reliance on subjective expert interpretation introduces variability
- Probabilistic analysis methods fail to meet judicial standards for physical evidence

Watson addresses these gaps through deterministic physics-based reconstruction with verifiable outputs.

---

## 3. System Scope

### 3.1 MVP Scope (Hackathon Implementation)

- Single scenario type: ballistic trajectory reconstruction
- Single video file ingestion per analysis session
- Manual coordinate input capability for bullet impact points
- 2D to 3D scene reconstruction from monocular video
- Muzzle origin estimation using inverse ballistics
- Basic blood spatter trajectory analysis
- Deterministic physics simulation engine
- Cryptographic hash generation for all outputs
- Immutable audit logging
- Basic 3D visualization dashboard

### 3.2 Out of Scope (Future Enhancements)

- Real-time multi-camera sensor fusion
- Advanced fluid dynamics for blood spatter
- Automated coordinate extraction from images
- Multi-scenario analysis (vehicle collision, explosion reconstruction)
- Mobile edge deployment
- International compliance frameworks beyond US jurisdiction

---

## 4. Functional Requirements

### FR-1: Video Ingestion and Processing
**FR-1.1** The system shall accept video files in MP4, AVI, and MOV formats up to 2GB in size.  
**FR-1.2** The system shall extract frames at configurable intervals (default: 1 frame per second).  
**FR-1.3** The system shall store extracted frames in S3 with versioning enabled.  
**FR-1.4** The system shall generate SHA-256 hash of the original video file upon ingestion.

### FR-2: Scene Reconstruction
**FR-2.1** The system shall perform 2D to 3D scene mapping using monocular depth estimation.  
**FR-2.2** The system shall identify reference objects for scale calibration (minimum 1 object required).  
**FR-2.3** The system shall generate a 3D point cloud representation of the scene.  
**FR-2.4** The system shall allow manual adjustment of coordinate system origin.

### FR-3: Object Detection
**FR-3.1** The system shall detect bullet impact points using AWS Rekognition Custom Labels or SageMaker endpoint.  
**FR-3.2** The system shall detect blood spatter patterns in video frames.  
**FR-3.3** The system shall log all neural network outputs with confidence scores.  
**FR-3.4** The system shall not use neural network outputs directly in final reconstruction decisions.

### FR-4: Ballistic Trajectory Reconstruction
**FR-4.1** The system shall calculate bullet trajectories using projectile motion equations.  
**FR-4.2** The system shall estimate muzzle origin coordinates given minimum 2 impact points.  
**FR-4.3** The system shall compute trajectory angles (azimuth and elevation).  
**FR-4.4** The system shall calculate error margins for trajectory estimates.

### FR-5: Blood Spatter Trajectory Analysis
**FR-5.1** The system shall calculate impact angles from blood spatter ellipse geometry.  
**FR-5.2** The system shall estimate point of origin using minimum 3 spatter samples.  
**FR-5.3** The system shall apply basic fluid dynamics constraints (gravity, air resistance).

### FR-6: Physics Simulation Engine
**FR-6.1** The system shall implement deterministic rigid body physics simulation.  
**FR-6.2** The system shall use fixed random seeds for any stochastic components.  
**FR-6.3** The system shall log all simulation parameters (initial conditions, constants, timestep).  
**FR-6.4** The system shall support replay of simulations with identical outputs.

### FR-7: Risk Classification
**FR-7.1** The system shall classify reconstruction confidence as High, Medium, or Low based on deterministic thresholds.  
**FR-7.2** Classification thresholds shall be configurable and logged with each analysis.  
**FR-7.3** The system shall not use machine learning models for final classification decisions.

### FR-8: Cryptographic Verification
**FR-8.1** The system shall generate SHA-256 hashes for all input evidence files.  
**FR-8.2** The system shall generate SHA-256 hashes for all simulation outputs.  
**FR-8.3** The system shall store hash values in immutable audit log.  
**FR-8.4** The system shall provide hash verification API endpoint.

### FR-9: Audit Logging
**FR-9.1** The system shall log all analysis operations to AWS QLDB.  
**FR-9.2** Audit logs shall include timestamps, user identity, input hashes, and output hashes.  
**FR-9.3** The system shall provide cryptographic proof of log integrity.  
**FR-9.4** Audit logs shall be queryable by case ID and timestamp range.

### FR-10: Visualization and Reporting
**FR-10.1** The system shall generate 3D visualization of reconstructed trajectories.  
**FR-10.2** The system shall export analysis reports in PDF format.  
**FR-10.3** Reports shall include all input parameters, simulation results, and hash values.  
**FR-10.4** The system shall provide web-based dashboard for result viewing.

---

## 5. Neuro-Symbolic Requirements

### NSR-1: Architectural Separation
**NSR-1.1** Neural components shall be restricted to perception tasks only (object detection, segmentation).  
**NSR-1.2** Symbolic components shall perform all reasoning and decision-making tasks.  
**NSR-1.3** No neural network outputs shall directly influence final reconstruction results without symbolic validation.

### NSR-2: Neural Layer Constraints
**NSR-2.1** Neural models shall output only bounding boxes, masks, and confidence scores.  
**NSR-2.2** Neural outputs shall be logged but treated as suggestions, not ground truth.  
**NSR-2.3** The system shall allow manual override of neural detections.

### NSR-3: Symbolic Layer Requirements
**NSR-3.1** All physics calculations shall use explicit mathematical equations.  
**NSR-3.2** All decision thresholds shall be deterministic and configurable.  
**NSR-3.3** The symbolic engine shall produce identical outputs for identical inputs.

---

## 6. Determinism Requirements

### DR-1: Reproducibility
**DR-1.1** Given identical input evidence and parameters, the system shall produce byte-identical outputs across all runs.  
**DR-1.2** The system shall fix all random seeds at initialization.  
**DR-1.3** The system shall use deterministic algorithms for all computations.

### DR-2: Parameter Logging
**DR-2.1** The system shall log all simulation parameters before execution.  
**DR-2.2** Logged parameters shall include: timestep size, gravity constant, air resistance coefficient, initial velocities.  
**DR-2.3** Parameter logs shall be stored with immutable timestamps.

### DR-3: Replay Capability
**DR-3.1** The system shall provide replay functionality using stored parameters.  
**DR-3.2** Replayed simulations shall produce outputs matching original run hashes.  
**DR-3.3** Replay verification shall be automated and logged.

---

## 7. Physics Simulation Requirements

### PSR-1: Projectile Motion
**PSR-1.1** The system shall implement standard projectile motion equations: x(t) = x₀ + v₀ₓt, y(t) = y₀ + v₀ᵧt - ½gt².  
**PSR-1.2** The system shall account for air resistance using drag coefficient.  
**PSR-1.3** Gravity constant shall be configurable (default: 9.81 m/s²).

### PSR-2: Inverse Trajectory Calculation
**PSR-2.1** The system shall solve inverse kinematics to estimate muzzle origin from impact points.  
**PSR-2.2** The system shall compute least-squares fit for overdetermined systems (>2 impact points).  
**PSR-2.3** The system shall report residual error for trajectory fits.

### PSR-3: Collision Physics
**PSR-3.1** The system shall implement basic rigid body collision detection.  
**PSR-3.2** The system shall calculate post-collision velocities using conservation of momentum.  
**PSR-3.3** Coefficient of restitution shall be configurable per material type.

### PSR-4: Error Estimation
**PSR-4.1** The system shall calculate 95% confidence intervals for trajectory estimates.  
**PSR-4.2** Error margins shall account for measurement uncertainty in input coordinates.  
**PSR-4.3** The system shall propagate uncertainty through physics calculations.

---

## 8. Evidence Integrity & Chain of Custody

### EI-1: Evidence Ingestion
**EI-1.1** All evidence files shall be hashed immediately upon ingestion.  
**EI-1.2** Original evidence files shall be stored immutably in S3 Glacier.  
**EI-1.3** Evidence metadata shall include: case ID, timestamp, submitting officer, file hash.

### EI-2: Chain of Custody Logging
**EI-2.1** All access to evidence files shall be logged with user identity and timestamp.  
**EI-2.2** Chain of custody logs shall be stored in AWS QLDB.  
**EI-2.3** The system shall detect and alert on hash mismatches.

### EI-3: Tamper Detection
**EI-3.1** The system shall verify file hashes before processing.  
**EI-3.2** Any hash mismatch shall halt processing and generate alert.  
**EI-3.3** Tamper detection events shall be logged with high priority.

---

## 9. Data Requirements

### DR-1: Input Data
**DR-1.1** Video resolution: minimum 1280x720, maximum 3840x2160.  
**DR-1.2** Frame rate: minimum 24 fps, maximum 60 fps.  
**DR-1.3** Coordinate precision: millimeter-level accuracy required.  
**DR-1.4** Reference object dimensions: user-provided with unit specification.

### DR-2: Output Data
**DR-2.1** Trajectory coordinates: 3D vectors with millimeter precision.  
**DR-2.2** Confidence intervals: percentage values with 2 decimal places.  
**DR-2.3** Simulation logs: JSON format with ISO 8601 timestamps.  
**DR-2.4** Visualization exports: PNG or WebGL format.

### DR-3: Storage Requirements
**DR-3.1** Evidence files: S3 Standard-IA with Glacier transition after 90 days.  
**DR-3.2** Audit logs: QLDB with indefinite retention.  
**DR-3.3** Simulation outputs: S3 Standard with versioning enabled.  
**DR-3.4** Temporary processing data: S3 with 7-day lifecycle policy.

---

## 10. Model Performance Requirements

### MPR-1: Object Detection Accuracy
**MPR-1.1** Bullet impact detection: minimum 90% recall on controlled test dataset.  
**MPR-1.2** Blood spatter detection: minimum 85% recall on controlled test dataset.  
**MPR-1.3** False positive rate: maximum 10% on test dataset.

### MPR-2: Trajectory Estimation Accuracy
**MPR-2.1** Muzzle origin estimation error: maximum 5% deviation from ground truth in controlled scenarios.  
**MPR-2.2** Trajectory angle error: maximum 3 degrees deviation from ground truth.  
**MPR-2.3** Impact point prediction error: maximum 10 cm deviation for 10-meter trajectories.

### MPR-3: Simulation Performance
**MPR-3.1** Simulation execution time: maximum 10 seconds for single trajectory analysis.  
**MPR-3.2** Scene reconstruction time: maximum 30 seconds for 60-second video.  
**MPR-3.3** End-to-end analysis time: maximum 2 minutes from ingestion to report generation.

---

## 11. Non-Functional Requirements

### NFR-1: Reliability
**NFR-1.1** System uptime: 99.5% availability during business hours.  
**NFR-1.2** Data durability: 99.999999999% (11 nines) for evidence files.  
**NFR-1.3** Audit log integrity: zero data loss tolerance.

### NFR-2: Scalability
**NFR-2.1** The system shall support concurrent analysis of up to 10 cases.  
**NFR-2.2** The system shall scale horizontally for compute-intensive simulations.  
**NFR-2.3** Storage shall scale to accommodate 1000 cases per year.

### NFR-3: Usability
**NFR-3.1** Dashboard shall be accessible via modern web browsers (Chrome, Firefox, Safari).  
**NFR-3.2** API response time: maximum 200ms for non-compute endpoints.  
**NFR-3.3** Documentation shall include API reference and user guide.

### NFR-4: Maintainability
**NFR-4.1** Code shall follow PEP 8 style guidelines for Python.  
**NFR-4.2** All functions shall include docstrings with parameter descriptions.  
**NFR-4.3** Infrastructure shall be defined as code using AWS CDK or Terraform.

---

## 12. AWS Infrastructure Requirements

### AIR-1: Compute Services
**AIR-1.1** Physics engine: AWS Lambda (Python 3.11 runtime) or ECS Fargate.  
**AIR-1.2** Heavy simulations: AWS Batch with spot instances.  
**AIR-1.3** API orchestration: API Gateway + Lambda integration.

### AIR-2: Storage Services
**AIR-2.1** Evidence storage: S3 with versioning and object lock enabled.  
**AIR-2.2** Audit logs: AWS QLDB with cryptographic verification.  
**AIR-2.3** Long-term archival: S3 Glacier Deep Archive.

### AIR-3: ML Services
**AIR-3.1** Object detection: AWS Rekognition Custom Labels or SageMaker endpoint.  
**AIR-3.2** Model versioning: SageMaker Model Registry.  
**AIR-3.3** Inference logging: CloudWatch Logs with structured JSON.

### AIR-4: Workflow Orchestration
**AIR-4.1** Simulation workflow: AWS Step Functions with error handling.  
**AIR-4.2** Video processing pipeline: Kinesis Video Streams or S3 event triggers.  
**AIR-4.3** Batch processing: EventBridge scheduled rules.

### AIR-5: Networking
**AIR-5.1** VPC: Private subnets for compute, public subnets for load balancers.  
**AIR-5.2** API access: API Gateway with IAM authentication.  
**AIR-5.3** Data transfer: VPC endpoints for S3 and QLDB access.

---

## 13. Security & Compliance Requirements

### SCR-1: Authentication & Authorization
**SCR-1.1** User authentication: AWS Cognito with MFA enabled.  
**SCR-1.2** API authorization: IAM roles with least-privilege policies.  
**SCR-1.3** Evidence access: Role-based access control (RBAC) with audit logging.

### SCR-2: Data Encryption
**SCR-2.1** Data at rest: AES-256 encryption for all S3 objects.  
**SCR-2.2** Data in transit: TLS 1.3 for all API communications.  
**SCR-2.3** Key management: AWS KMS with automatic key rotation.

### SCR-3: Compliance
**SCR-3.1** The system shall maintain audit logs compliant with Federal Rules of Evidence.  
**SCR-3.2** Evidence handling shall follow NIST SP 800-86 guidelines.  
**SCR-3.3** Data retention policies shall be configurable per jurisdiction.

### SCR-4: Monitoring & Alerting
**SCR-4.1** Security events: CloudWatch alarms for unauthorized access attempts.  
**SCR-4.2** Integrity violations: SNS notifications for hash mismatches.  
**SCR-4.3** System health: CloudWatch dashboards with key metrics.

---

## 14. Acceptance Criteria

### AC-1: Determinism Verification
**Given:** Identical video file and manual coordinate inputs  
**When:** Analysis is executed twice  
**Then:** Output hashes are identical across both runs

### AC-2: Trajectory Accuracy
**Given:** Controlled test scenario with known muzzle origin  
**When:** System analyzes bullet impact points  
**Then:** Estimated muzzle origin is within 5% of ground truth

### AC-3: Hash Integrity
**Given:** Evidence file with known SHA-256 hash  
**When:** File is ingested and processed  
**Then:** Stored hash matches original and is logged in QLDB

### AC-4: Replay Consistency
**Given:** Completed analysis with stored parameters  
**When:** Replay is executed using stored parameters  
**Then:** Replay output hash matches original output hash

### AC-5: Simulation Performance
**Given:** Single ballistic trajectory analysis  
**When:** Simulation is executed  
**Then:** Execution time is less than 10 seconds

### AC-6: Audit Trail Completeness
**Given:** Analysis workflow from ingestion to report  
**When:** Audit log is queried  
**Then:** All operations are logged with timestamps and user identity

### AC-7: Neural-Symbolic Separation
**Given:** Object detection results from neural model  
**When:** Final reconstruction is computed  
**Then:** Symbolic physics engine produces output independent of neural confidence scores

---

## 15. Evaluation Metrics

### EM-1: Correctness Metrics
- Trajectory estimation error (percentage deviation from ground truth)
- Impact point prediction error (centimeters)
- Angle estimation error (degrees)
- Hash verification success rate (percentage)

### EM-2: Performance Metrics
- Simulation execution time (seconds)
- End-to-end analysis time (minutes)
- API response time (milliseconds)
- Concurrent case processing capacity

### EM-3: Reliability Metrics
- System uptime (percentage)
- Failed analysis rate (percentage)
- Data loss incidents (count)
- Hash mismatch incidents (count)

### EM-4: Determinism Metrics
- Replay consistency rate (percentage of identical outputs)
- Parameter logging completeness (percentage)
- Random seed fixation verification (boolean)

---

## 16. MVP Constraints

### MC-1: Scope Limitations
- Single scenario type: ballistic reconstruction only
- Single video file per analysis session
- Manual coordinate input required (no automated extraction)
- Basic 3D visualization (no advanced rendering)
- Single AWS region deployment

### MC-2: Performance Constraints
- Maximum video file size: 2GB
- Maximum concurrent analyses: 10
- Maximum simulation complexity: 100 timesteps
- Maximum trajectory points: 50

### MC-3: Integration Constraints
- AWS services only (no third-party APIs)
- Python 3.11 for physics engine
- React for frontend dashboard
- REST API only (no GraphQL)

### MC-4: Testing Constraints
- Controlled test dataset with ground truth required
- Manual validation of trajectory accuracy
- Property-based testing for determinism verification
- No real-world case data in MVP

---

**Document Status:** Draft for Review  
**Next Steps:** Design document creation, architecture specification, implementation planning
