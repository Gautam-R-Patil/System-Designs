# Requirements: Adaptive ICU Early Warning Intelligence

## Overview

This document specifies the requirements for an AI-powered early warning system designed to predict hypoxic events in ICU patients. The system leverages AWS cloud infrastructure, machine learning (LSTM models), and real-time data processing to provide clinical decision support. The goal is to predict hypoxic events at least 10 minutes in advance with high accuracy, providing interpretable risk scores to healthcare providers.

**Target Users**: ICU clinical staff (nurses, physicians) and system administrators

**Primary Goal**: Reduce adverse hypoxic events through early detection and actionable alerts

**Technology Stack**: AWS (S3, SageMaker, Lambda, API Gateway, Cognito, CloudWatch), Python, PyTorch, React

## 1. Functional Requirements

### 1.1 Data Ingestion
- **FR-1.1.1**: System must ingest ICU time-series data from Amazon S3 buckets
- **FR-1.1.2**: System must support MIMIC-IV and PhysioNet dataset formats
- **FR-1.1.3**: System must validate data integrity upon ingestion
- **FR-1.1.4**: System must handle missing or incomplete time-series data gracefully

### 1.2 Data Preprocessing
- **FR-1.2.1**: System must preprocess raw ICU data using serverless AWS components (Lambda or Glue)
- **FR-1.2.2**: System must extract relevant vital signs (SpO2, heart rate, respiratory rate, blood pressure)
- **FR-1.2.3**: System must create time-windowed features for model input
- **FR-1.2.4**: System must normalize and scale features appropriately
- **FR-1.2.5**: System must handle temporal alignment of multi-modal signals

### 1.3 Model Training
- **FR-1.3.1**: System must train predictive models using Amazon SageMaker
- **FR-1.3.2**: System must support LSTM-based time-series modeling
- **FR-1.3.3**: System must implement cross-validation during training
- **FR-1.3.4**: System must log training metrics to CloudWatch
- **FR-1.3.5**: System must save trained model artifacts to S3

### 1.4 Model Deployment
- **FR-1.4.1**: System must deploy trained model as a SageMaker Endpoint
- **FR-1.4.2**: System must support real-time inference requests
- **FR-1.4.3**: System must handle endpoint scaling based on load
- **FR-1.4.4**: System must version deployed models

### 1.5 Risk Prediction
- **FR-1.5.1**: System must generate hypoxic risk scores ranging from 0-100
- **FR-1.5.2**: System must predict risk at least 10 minutes before SpO2 threshold breach
- **FR-1.5.3**: System must provide confidence intervals for predictions
- **FR-1.5.4**: System must classify risk levels (Low: 0-33, Moderate: 34-66, High: 67-100)

#### 1.5.1 Alert Trigger Logic (Added in v2)
- **FR-1.5.1.1**: High-risk alert triggered only if risk score >= 67 for 2 consecutive prediction cycles
- **FR-1.5.1.2**: Prediction cycle interval must be 5 minutes
- **FR-1.5.1.3**: Alert cooldown period of 15 minutes must suppress repeat alerts for same patient
- **FR-1.5.1.4**: System must simulate clinician acknowledgment workflow (acknowledge/dismiss)
- **FR-1.5.1.5**: System must track alert response times and acknowledgment rates
- **FR-1.5.1.6**: Alarm fatigue mitigation: Maximum 2 alerts per patient per hour unless risk escalates

### 1.6 Explainability
- **FR-1.6.1**: System must provide feature importance scores for each prediction
- **FR-1.6.2**: System must identify which vital signs contribute most to risk score
- **FR-1.6.3**: System must generate human-readable explanations for clinical staff
- **FR-1.6.4**: System must visualize temporal patterns leading to risk elevation

### 1.7 API Layer
- **FR-1.7.1**: System must expose prediction API via Amazon API Gateway
- **FR-1.7.2**: API must accept patient time-series data as input
- **FR-1.7.3**: API must return risk score, confidence, and explainability data
- **FR-1.7.4**: API must implement rate limiting and throttling
- **FR-1.7.5**: API must support batch prediction requests

### 1.8 User Interface
- **FR-1.8.1**: System must provide a web-based dashboard hosted on AWS Amplify or S3
- **FR-1.8.2**: Dashboard must display real-time risk scores for monitored patients
- **FR-1.8.3**: Dashboard must visualize vital sign trends
- **FR-1.8.4**: Dashboard must show explainability information
- **FR-1.8.5**: Dashboard must support filtering and sorting of patient list
- **FR-1.8.6**: Dashboard must provide alert notifications for high-risk patients

### 1.9 Authentication & Authorization
- **FR-1.9.1**: System must authenticate users via Amazon Cognito
- **FR-1.9.2**: System must support role-based access control (clinician, administrator)
- **FR-1.9.3**: System must enforce secure session management
- **FR-1.9.4**: System must log all authentication attempts

### 1.10 Monitoring & Logging
- **FR-1.10.1**: System must log all inference requests to CloudWatch
- **FR-1.10.2**: System must track model performance metrics over time
- **FR-1.10.3**: System must alert on anomalous prediction patterns
- **FR-1.10.4**: System must monitor API latency and error rates
- **FR-1.10.5**: System must track AWS resource utilization

#### 1.10.1 Operational Monitoring Thresholds (Added in v2)
- **FR-1.10.1.1**: CloudWatch alarm must trigger when API latency exceeds 2 seconds for 3 consecutive periods
- **FR-1.10.1.2**: CloudWatch alarm must trigger when SageMaker endpoint error rate exceeds 5%
- **FR-1.10.1.3**: System must log model input feature distribution statistics for drift detection
- **FR-1.10.1.4**: System must track prediction score distribution (mean, std, percentiles) per hour
- **FR-1.10.1.5**: System must maintain placeholder metric for future drift detection implementation

### 1.11 Failure & Degradation Strategy (Added in v2)
- **FR-1.11.1**: If SageMaker endpoint unavailable, system must fallback to baseline statistical model
- **FR-1.11.2**: Baseline model: Simple threshold-based alert (SpO2 < 92% or declining trend)
- **FR-1.11.3**: System must retry failed SageMaker invocations maximum 3 times with exponential backoff
- **FR-1.11.4**: Backoff strategy: 1 second, 2 seconds, 4 seconds between retries
- **FR-1.11.5**: System must log all failed inference attempts to CloudWatch with error details
- **FR-1.11.6**: Dashboard must display "Degraded Mode" warning banner when using fallback model
- **FR-1.11.7**: System must automatically resume normal operation when SageMaker endpoint recovers

## 2. Non-Functional Requirements

### 2.1 Performance
- **NFR-2.1.1**: Inference latency must be < 2 seconds (95th percentile)
- **NFR-2.1.2**: API response time must be < 3 seconds end-to-end
- **NFR-2.1.3**: Dashboard must load within 5 seconds
- **NFR-2.1.4**: System must support concurrent requests from multiple users

### 2.2 Scalability
- **NFR-2.2.1**: Architecture must scale horizontally using AWS managed services
- **NFR-2.2.2**: System must handle up to 100 concurrent patient monitoring sessions
- **NFR-2.2.3**: Data pipeline must process batch uploads of up to 10GB
- **NFR-2.2.4**: SageMaker endpoint must auto-scale based on traffic

### 2.3 Reliability
- **NFR-2.3.1**: System uptime must be >= 99% during evaluation period
- **NFR-2.3.2**: Model endpoint must implement automatic failover
- **NFR-2.3.3**: Data pipeline must implement retry logic for transient failures
- **NFR-2.3.4**: System must gracefully degrade if explainability service fails

### 2.4 Security
- **NFR-2.4.1**: All data in transit must be encrypted using TLS 1.2+
- **NFR-2.4.2**: All data at rest in S3 must be encrypted using SSE-S3 or SSE-KMS
- **NFR-2.4.3**: API Gateway must enforce authentication on all endpoints
- **NFR-2.4.4**: IAM roles must follow principle of least privilege
- **NFR-2.4.5**: No real patient data may be used (simulated data only)

#### 2.4.1 Security Hardening (Added in v2)
- **NFR-2.4.1.1**: Enable AWS KMS-managed encryption keys (CMK) for S3 buckets containing sensitive data
- **NFR-2.4.1.2**: Enable AWS CloudTrail logging for audit tracking of all API calls
- **NFR-2.4.1.3**: Deploy SageMaker endpoint inside VPC with private subnets
- **NFR-2.4.1.4**: Restrict S3 public access at bucket level using Block Public Access settings
- **NFR-2.4.1.5**: Enable S3 bucket logging for access audit trail
- **NFR-2.4.1.6**: Configure VPC security groups to allow only necessary traffic to SageMaker endpoint

### 2.5 Maintainability
- **NFR-2.5.1**: Infrastructure must be defined as code (CloudFormation or Terraform)
- **NFR-2.5.2**: Code must follow PEP 8 style guidelines for Python
- **NFR-2.5.3**: All components must have comprehensive logging
- **NFR-2.5.4**: Model training pipeline must be reproducible

### 2.6 Usability
- **NFR-2.6.1**: Dashboard must be intuitive for clinical staff without ML expertise
- **NFR-2.6.2**: Risk scores must be clearly visualized with color coding
- **NFR-2.6.3**: Explainability output must be clinically interpretable
- **NFR-2.6.4**: System must provide helpful error messages

### 2.7 Cost Optimization
- **NFR-2.7.1**: Use serverless components where possible to minimize idle costs
- **NFR-2.7.2**: Implement S3 lifecycle policies for data retention
- **NFR-2.7.3**: Use SageMaker spot instances for training when feasible
- **NFR-2.7.4**: Implement API Gateway caching to reduce Lambda invocations
- **NFR-2.7.5**: Target total AWS cost < $150-200/month for active MVP operation

#### 2.7.1 Cost Control Mechanisms (Added in v2)
- **NFR-2.7.1.1**: Auto-shutdown policy for development SageMaker notebooks after 1 hour of inactivity
- **NFR-2.7.1.2**: SageMaker endpoint must scale to minimum 1 instance during low-traffic periods
- **NFR-2.7.1.3**: SageMaker endpoint must scale to zero instances during maintenance windows
- **NFR-2.7.1.4**: AWS Budgets alert must trigger at 80% of monthly cost threshold ($160)
- **NFR-2.7.1.5**: AWS Budgets alert must trigger at 100% of monthly cost threshold ($200)
- **NFR-2.7.1.6**: S3 lifecycle policy must transition raw data to Glacier after 90 days
- **NFR-2.7.1.7**: CloudWatch log retention must be limited to 30 days for non-critical logs

## 3. Data Requirements

### 3.1 Dataset Source
- **DR-3.1.1**: Use MIMIC-IV or PhysioNet ICU datasets exclusively
- **DR-3.1.2**: Dataset must include time-series vital signs data
- **DR-3.1.3**: Dataset must include documented hypoxic events (SpO2 < 90%)
- **DR-3.1.4**: Minimum 1000 patient records for training

### 3.2 Model Labeling & Windowing Strategy (Added in v2)
- **DR-3.2.1**: Sliding window size must be 60 minutes of observation data
- **DR-3.2.2**: Sliding window step interval must be 5 minutes (stride)
- **DR-3.2.3**: Label window as positive (1) if hypoxic event (SpO2 < 90%) occurs within next 10-30 minutes
- **DR-3.2.4**: Label window as negative (0) if no hypoxic event occurs within next 30 minutes
- **DR-3.2.5**: Each window represents a binary classification task (hypoxia risk: yes/no)
- **DR-3.2.6**: Exclude windows with insufficient future data (< 30 minutes remaining)
- **DR-3.2.7**: Handle class imbalance using class-weighted loss function (weight positive class 3-5x)
- **DR-3.2.8**: Alternative: Implement focal loss to focus on hard-to-classify examples
- **DR-3.2.9**: Temporal split strategy: Train on first 70% of timeline, validate on next 15%, test on final 15%
- **DR-3.2.10**: Ensure no patient data leakage between train/validation/test splits

### 3.3 Required Features
- **DR-3.3.1**: SpO2 (oxygen saturation) - primary target variable
- **DR-3.3.2**: Heart rate (HR)
- **DR-3.3.3**: Respiratory rate (RR)
- **DR-3.3.4**: Blood pressure (systolic/diastolic)
- **DR-3.3.5**: Timestamp for temporal alignment

### 3.4 Data Quality
- **DR-3.4.1**: Handle missing values using forward-fill or interpolation
- **DR-3.4.2**: Remove outliers beyond physiologically plausible ranges
- **DR-3.4.3**: Ensure minimum sampling rate of 1 measurement per 5 minutes
- **DR-3.4.4**: Document data preprocessing steps for reproducibility

### 3.5 Data Storage
- **DR-3.5.1**: Raw data stored in S3 with versioning enabled
- **DR-3.5.2**: Preprocessed data stored in separate S3 bucket/prefix
- **DR-3.5.3**: Model artifacts stored in S3 with metadata tags
- **DR-3.5.4**: Implement S3 bucket policies for access control

## 4. Model Performance Requirements

### 4.1 Classification Metrics
- **MPR-4.1.1**: AUC-ROC >= 0.80 on held-out test set
- **MPR-4.1.2**: Precision >= 0.75 for hypoxic event prediction
- **MPR-4.1.3**: Recall >= 0.75 for hypoxic event prediction
- **MPR-4.1.4**: F1 Score >= 0.75
- **MPR-4.1.5**: Brier score <= 0.15 for probability calibration quality (Added in v2)
- **MPR-4.1.6**: Expected Calibration Error (ECE) <= 0.10 (Added in v2)

#### 4.1.1 Evaluation Granularity (Added in v2)
- **MPR-4.1.1.1**: Report metrics at both window-level and patient-level
- **MPR-4.1.1.2**: Window-level: Each 60-minute window is one prediction instance
- **MPR-4.1.1.3**: Patient-level: Aggregate predictions per patient (any high-risk window = patient flagged)
- **MPR-4.1.1.4**: Generate reliability curve (calibration plot) with 10 bins
- **MPR-4.1.1.5**: Define acceptable precision-recall tradeoff: Minimum precision 0.70 when recall >= 0.80

### 4.2 Early Detection
- **MPR-4.2.1**: Predict hypoxic events >= 10 minutes before SpO2 < 90%
- **MPR-4.2.2**: Minimize false positive rate to < 30%
- **MPR-4.2.3**: Achieve positive predictive value >= 0.70

### 4.3 Model Robustness
- **MPR-4.3.1**: Performance must be consistent across patient demographics
- **MPR-4.3.2**: Model must handle varying time-series lengths
- **MPR-4.3.3**: Model must degrade gracefully with missing features

### 4.4 Calibration
- **MPR-4.4.1**: Predicted probabilities must be well-calibrated
- **MPR-4.4.2**: Risk scores must correlate with actual event rates

## 5. AWS Infrastructure Requirements

### 5.1 Data Storage
- **AIR-5.1.1**: Use Amazon S3 for all data storage
- **AIR-5.1.2**: Implement S3 bucket versioning for data lineage
- **AIR-5.1.3**: Configure S3 lifecycle policies for cost optimization

### 5.2 Data Processing
- **AIR-5.2.1**: Use AWS Lambda or AWS Glue for preprocessing pipeline
- **AIR-5.2.2**: Implement serverless architecture for scalability
- **AIR-5.2.3**: Use Step Functions for orchestrating multi-stage pipelines

### 5.3 Model Training
- **AIR-5.3.1**: Use Amazon SageMaker for model training
- **AIR-5.3.2**: Configure training jobs with appropriate instance types
- **AIR-5.3.3**: Enable SageMaker Experiments for tracking

### 5.4 Model Deployment
- **AIR-5.4.1**: Deploy model to SageMaker real-time endpoint
- **AIR-5.4.2**: Configure auto-scaling policies for endpoint
- **AIR-5.4.3**: Implement A/B testing capability for model versions

### 5.5 API Layer
- **AIR-5.5.1**: Use Amazon API Gateway for REST API
- **AIR-5.5.2**: Integrate API Gateway with Lambda for business logic
- **AIR-5.5.3**: Enable API Gateway request/response logging

### 5.6 Frontend Hosting
- **AIR-5.6.1**: Host dashboard on AWS Amplify or S3 + CloudFront
- **AIR-5.6.2**: Configure HTTPS with ACM certificates
- **AIR-5.6.3**: Enable CloudFront caching for static assets

### 5.7 Authentication
- **AIR-5.7.1**: Use Amazon Cognito for user authentication
- **AIR-5.7.2**: Configure user pools with appropriate password policies
- **AIR-5.7.3**: Integrate Cognito with API Gateway authorizers

### 5.8 Monitoring
- **AIR-5.8.1**: Use Amazon CloudWatch for all logging and monitoring
- **AIR-5.8.2**: Create CloudWatch dashboards for system health
- **AIR-5.8.3**: Configure CloudWatch alarms for critical metrics
- **AIR-5.8.4**: Enable AWS X-Ray for distributed tracing

## 6. Security & Compliance Constraints

### 6.1 Data Privacy
- **SCC-6.1.1**: No real patient data may be used in MVP
- **SCC-6.1.2**: All data must be de-identified and anonymized
- **SCC-6.1.3**: System must be designed with HIPAA principles in mind (for future)

### 6.2 Access Control
- **SCC-6.2.1**: Implement IAM roles with least privilege access
- **SCC-6.2.2**: Enable MFA for administrative access
- **SCC-6.2.3**: Audit all access to sensitive resources

### 6.3 Network Security
- **SCC-6.3.1**: Deploy resources in VPC where applicable
- **SCC-6.3.2**: Use security groups to restrict network access
- **SCC-6.3.3**: Enable VPC Flow Logs for network monitoring

### 6.4 Compliance
- **SCC-6.4.1**: Document all data handling procedures
- **SCC-6.4.2**: Maintain audit logs for 90 days minimum
- **SCC-6.4.3**: Implement data retention and deletion policies

## 7. Evaluation Criteria

### 7.1 Model Performance
- **EC-7.1.1**: Evaluate using AUC-ROC, Precision, Recall, F1 Score, Brier Score, ECE
- **EC-7.1.2**: Measure early detection time gain in minutes (median, 25th, 75th percentiles)
- **EC-7.1.3**: Assess calibration using reliability diagrams with 10 bins
- **EC-7.1.4**: Compare against baseline models (logistic regression, random forest, simple threshold)
- **EC-7.1.5**: Report confidence intervals using bootstrapping (1000 iterations)
- **EC-7.1.6**: Analyze performance across patient subgroups (age, gender, admission type)

### 7.2 System Performance
- **EC-7.2.1**: Measure end-to-end inference latency (p50, p95, p99)
- **EC-7.2.2**: Track API response times under load (simulate 10, 50, 100 concurrent users)
- **EC-7.2.3**: Monitor resource utilization and costs (daily breakdown by service)
- **EC-7.2.4**: Evaluate system reliability and uptime (target >= 99%)
- **EC-7.2.5**: Measure alert response time from prediction to dashboard display

### 7.3 Clinical Utility
- **EC-7.3.1**: Assess interpretability of risk scores (clinician survey if possible)
- **EC-7.3.2**: Evaluate usefulness of explainability features (feature importance clarity)
- **EC-7.3.3**: Measure false positive burden on clinical staff (alerts per patient per day)
- **EC-7.3.4**: Gather qualitative feedback on dashboard usability
- **EC-7.3.5**: Calculate potential time savings from early detection

### 7.4 MVP Success Criteria
- **EC-7.4.1**: All functional requirements implemented
- **EC-7.4.2**: Model performance targets met (AUC-ROC >= 0.80, Precision/Recall >= 0.75)
- **EC-7.4.3**: System deployed and operational on AWS
- **EC-7.4.4**: Dashboard accessible and functional
- **EC-7.4.5**: Total AWS cost within budget ($150-200/month)
- **EC-7.4.6**: Comprehensive evaluation report delivered

## 8. MVP Scope Constraints

### 8.1 Clinical Focus
- **Scope**: Early hypoxic event prediction only
- **Out of Scope**: Other deterioration events (sepsis, cardiac arrest, etc.)

### 8.2 Data Constraints
- **Scope**: MIMIC-IV / PhysioNet datasets only
- **Out of Scope**: Real-time hospital integration, proprietary datasets

### 8.3 Deployment Type
- **Scope**: Simulated real-time using historical data replay
- **Out of Scope**: Live patient monitoring, EHR integration

### 8.4 System Type
- **Scope**: Assistive AI for clinical decision support
- **Out of Scope**: Autonomous decision-making, automated interventions

### 8.5 Timeline
- **Scope**: Hackathon-feasible MVP (2-4 weeks)
- **Out of Scope**: Production-grade system, extensive validation studies

## 9. Acceptance Criteria

### 9.1 Documentation Complete
- [ ] requirements.md finalized and approved
- [ ] design.md with AWS architecture created
- [ ] All sections contain sufficient detail for implementation

### 9.2 Infrastructure Deployed
- [ ] S3 buckets created and configured
- [ ] SageMaker training environment set up
- [ ] SageMaker endpoint deployed and accessible
- [ ] API Gateway configured with Lambda integration
- [ ] Frontend hosted and accessible via HTTPS
- [ ] Cognito user pool configured

### 9.3 Model Trained and Validated
- [ ] Model achieves AUC-ROC >= 0.80
- [ ] Model achieves Precision >= 0.75
- [ ] Model achieves Recall >= 0.75
- [ ] Early detection gain >= 10 minutes demonstrated

### 9.4 System Functional
- [ ] Data ingestion pipeline operational
- [ ] Preprocessing pipeline processes data correctly
- [ ] API returns predictions within 2 seconds
- [ ] Dashboard displays risk scores and visualizations
- [ ] Authentication works correctly
- [ ] CloudWatch logging enabled

### 9.5 Evaluation Complete
- [ ] Model performance metrics calculated
- [ ] System performance benchmarks measured
- [ ] Evaluation report documenting results
- [ ] Cost analysis completed

## 10. Future Enhancements (Out of MVP Scope)

### 10.1 Clinical Capabilities
- **FE-10.1.1**: Multi-condition prediction (sepsis, cardiac events, respiratory failure)
- **FE-10.1.2**: Medication interaction analysis and recommendations
- **FE-10.1.3**: Integration with nurse call systems for automated alerts
- **FE-10.1.4**: Predictive alerts via SMS/pager for critical events

### 10.2 Data & Integration
- **FE-10.2.1**: Real-time EHR integration (HL7 FHIR)
- **FE-10.2.2**: Live patient monitoring with streaming data
- **FE-10.2.3**: Multi-hospital federated learning for model improvement
- **FE-10.2.4**: Integration with wearable devices and remote monitoring

### 10.3 Model Improvements
- **FE-10.3.1**: Advanced explainability (SHAP values, attention visualization)
- **FE-10.3.2**: Continuous model retraining pipeline with online learning
- **FE-10.3.3**: Ensemble models combining multiple architectures
- **FE-10.3.4**: Transfer learning from larger medical datasets

### 10.4 User Experience
- **FE-10.4.1**: Mobile application for clinicians (iOS/Android)
- **FE-10.4.2**: Voice-activated alerts and queries
- **FE-10.4.3**: Customizable alert thresholds per patient
- **FE-10.4.4**: Integration with clinical workflow management systems

### 10.5 Infrastructure
- **FE-10.5.1**: Multi-region deployment for disaster recovery
- **FE-10.5.2**: Edge computing for low-latency predictions
- **FE-10.5.3**: HIPAA compliance certification
- **FE-10.5.4**: Production-grade monitoring and observability
