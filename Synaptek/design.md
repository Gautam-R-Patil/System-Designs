# Design: Adaptive ICU Early Warning Intelligence

## 1. Problem Definition

### 1.1 Clinical Challenge
Hypoxic events (low blood oxygen saturation) in ICU patients can lead to severe complications including organ damage, cardiac arrest, and death. Early detection is critical but challenging due to:
- Subtle physiological changes preceding overt hypoxia
- High alarm fatigue from existing monitoring systems
- Cognitive overload on clinical staff monitoring multiple patients
- Delayed recognition of deterioration patterns

### 1.2 Solution Approach
Build an assistive AI system that:
- Continuously analyzes time-series vital signs data
- Predicts hypoxic events 10+ minutes before SpO2 drops below 90%
- Provides interpretable risk scores to support clinical decision-making
- Reduces false alarms through intelligent pattern recognition
- Operates as a decision support tool, not autonomous intervention

### 1.3 Success Metrics
- AUC-ROC >= 0.80 for hypoxic event prediction
- Early detection gain >= 10 minutes before threshold breach
- Precision and Recall >= 0.75
- Inference latency < 2 seconds
- Clinically interpretable explanations

## 2. Clinical Scope (Hypoxia Only)

### 2.1 Target Condition
**Hypoxic Event Definition**: SpO2 (oxygen saturation) falling below 90% for sustained period (>5 minutes)

### 2.2 Prediction Window
- **Lookback**: 60 minutes of historical vital signs
- **Prediction Horizon**: 10-30 minutes ahead
- **Update Frequency**: Every 5 minutes (or on new data arrival)

### 2.3 Input Features
Primary vital signs from ICU monitoring:
- SpO2 (oxygen saturation) - target variable
- Heart Rate (HR)
- Respiratory Rate (RR)
- Systolic Blood Pressure (SBP)
- Diastolic Blood Pressure (DBP)

### 2.4 Out of Scope
- Other deterioration events (sepsis, cardiac arrest, hemorrhage)
- Medication recommendations
- Automated interventions
- Real-time EHR integration (MVP uses historical replay)


## 3. AWS-Based High-Level Architecture

### 3.1 Architecture Overview
```
┌─────────────────────────────────────────────────────────────────────┐
│                         AWS Cloud Infrastructure                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  ┌──────────────┐         ┌──────────────┐        ┌──────────────┐ │
│  │   Amazon S3  │────────▶│  AWS Lambda  │───────▶│   Amazon S3  │ │
│  │  (Raw Data)  │         │  or AWS Glue │        │ (Processed)  │ │
│  └──────────────┘         │(Preprocessing)│        └──────────────┘ │
│                            └──────────────┘                │         │
│                                                             │         │
│                            ┌──────────────┐                │         │
│                            │   Amazon     │◀───────────────┘         │
│                            │  SageMaker   │                          │
│                            │  (Training)  │                          │
│                            └──────────────┘                          │
│                                    │                                 │
│                                    ▼                                 │
│                            ┌──────────────┐                          │
│                            │  SageMaker   │                          │
│                            │   Endpoint   │                          │
│                            │ (Inference)  │                          │
│                            └──────────────┘                          │
│                                    ▲                                 │
│                                    │                                 │
│  ┌──────────────┐         ┌──────────────┐        ┌──────────────┐ │
│  │   Amazon     │────────▶│  AWS Lambda  │───────▶│   Amazon     │ │
│  │ API Gateway  │         │  (Business   │        │  CloudWatch  │ │
│  │              │         │    Logic)    │        │  (Logging)   │ │
│  └──────────────┘         └──────────────┘        └──────────────┘ │
│         ▲                                                            │
│         │                                                            │
│  ┌──────────────┐         ┌──────────────┐                          │
│  │   Amazon     │         │     AWS      │                          │
│  │   Cognito    │         │   Amplify    │                          │
│  │    (Auth)    │         │  (Frontend)  │                          │
│  └──────────────┘         └──────────────┘                          │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### 3.2 Component Responsibilities

**Data Layer**:
- Amazon S3: Store raw datasets, preprocessed data, model artifacts
- S3 Lifecycle Policies: Automatic data retention management

**Processing Layer**:
- AWS Lambda: Serverless preprocessing for small batches
- AWS Glue: ETL jobs for large-scale data transformation
- AWS Step Functions: Orchestrate multi-stage pipelines

**ML Layer**:
- Amazon SageMaker Training: Train LSTM models on historical data
- SageMaker Experiments: Track training runs and hyperparameters
- SageMaker Model Registry: Version and manage models

**Inference Layer**:
- SageMaker Endpoint: Real-time prediction service
- Auto Scaling: Handle variable load
- Model Monitor: Track prediction quality over time

**API Layer**:
- Amazon API Gateway: RESTful API for frontend
- AWS Lambda: Business logic, request validation, response formatting
- API Gateway Authorizer: Integrate with Cognito for auth

**Frontend Layer**:
- AWS Amplify: Host React dashboard with CI/CD
- Amazon CloudFront: CDN for global distribution
- S3 Static Hosting: Alternative lightweight hosting

**Security Layer**:
- Amazon Cognito: User authentication and authorization
- IAM Roles: Service-to-service permissions
- AWS KMS: Encryption key management

**Monitoring Layer**:
- Amazon CloudWatch: Logs, metrics, alarms
- CloudWatch Dashboards: System health visualization
- AWS X-Ray: Distributed tracing for debugging


## 4. Data Flow Diagram

### 4.1 Training Pipeline Flow
```
1. Data Upload
   └─▶ Public ICU Dataset (MIMIC-IV) uploaded to S3 bucket
       └─▶ s3://icu-ews-data/raw/mimic-iv/

2. Preprocessing
   └─▶ AWS Lambda/Glue triggered on S3 event
       ├─▶ Extract vital signs time-series
       ├─▶ Handle missing values (forward-fill, interpolation)
       ├─▶ Normalize features (z-score normalization)
       ├─▶ Create sliding windows (60-min lookback)
       ├─▶ Label hypoxic events (SpO2 < 90%)
       └─▶ Save to s3://icu-ews-data/processed/

3. Model Training
   └─▶ SageMaker Training Job
       ├─▶ Load processed data from S3
       ├─▶ Train LSTM model with hyperparameter tuning
       ├─▶ Validate on held-out test set
       ├─▶ Log metrics to CloudWatch & SageMaker Experiments
       └─▶ Save model artifacts to s3://icu-ews-models/

4. Model Deployment
   └─▶ SageMaker Endpoint Creation
       ├─▶ Load model from S3
       ├─▶ Configure auto-scaling (min: 1, max: 5 instances)
       └─▶ Endpoint ready for inference
```

### 4.2 Inference Pipeline Flow
```
1. User Authentication
   └─▶ User logs into dashboard
       └─▶ Cognito authenticates and issues JWT token

2. Data Submission
   └─▶ Dashboard sends patient vital signs to API
       └─▶ POST /predict with 60-min time-series data

3. API Gateway Processing
   └─▶ API Gateway receives request
       ├─▶ Validates JWT token with Cognito Authorizer
       ├─▶ Validates request schema
       └─▶ Invokes Lambda function

4. Lambda Business Logic
   └─▶ Lambda function processes request
       ├─▶ Preprocesses input data (normalization)
       ├─▶ Calls SageMaker Endpoint for prediction
       ├─▶ Generates explainability (feature importance)
       ├─▶ Calculates risk score (0-100)
       ├─▶ Logs request to CloudWatch
       └─▶ Returns formatted response

5. SageMaker Inference
   └─▶ SageMaker Endpoint
       ├─▶ Runs LSTM model forward pass
       ├─▶ Returns probability of hypoxic event
       └─▶ Latency: ~500ms

6. Response to Dashboard
   └─▶ API Gateway returns JSON response
       ├─▶ risk_score: 0-100
       ├─▶ confidence: 0.0-1.0
       ├─▶ risk_level: "Low" | "Moderate" | "High"
       ├─▶ feature_importance: {...}
       └─▶ explanation: "High risk due to declining SpO2..."

7. Dashboard Visualization
   └─▶ React dashboard updates UI
       ├─▶ Display risk score with color coding
       ├─▶ Show vital signs trends
       ├─▶ Render explainability charts
       └─▶ Trigger alert if risk_level == "High"
```

### 4.3 Monitoring Flow
```
CloudWatch Logs ◀─── All AWS services send logs
     │
     ├─▶ Lambda execution logs
     ├─▶ API Gateway access logs
     ├─▶ SageMaker endpoint logs
     └─▶ Custom application metrics

CloudWatch Alarms
     ├─▶ API latency > 3 seconds
     ├─▶ SageMaker endpoint errors > 5%
     ├─▶ Lambda throttling detected
     └─▶ Estimated AWS cost > $100/month
```


## 5. Module Breakdown

### 5.1 Data Preprocessing Module
**Location**: `src/preprocessing/`
**AWS Service**: AWS Lambda or AWS Glue

**Components**:
- `data_loader.py`: Load raw MIMIC-IV data from S3
- `feature_extractor.py`: Extract vital signs time-series
- `data_cleaner.py`: Handle missing values, outliers
- `feature_engineer.py`: Create sliding windows, normalize
- `label_generator.py`: Identify hypoxic events for training

**Key Functions**:
```python
def preprocess_patient_data(patient_id: str, s3_bucket: str) -> pd.DataFrame:
    """Load, clean, and engineer features for a patient."""
    
def create_sliding_windows(df: pd.DataFrame, window_size: int = 60) -> np.ndarray:
    """Create 60-minute sliding windows for LSTM input."""
    
def label_hypoxic_events(df: pd.DataFrame, threshold: int = 90, 
                         prediction_horizon: int = 10) -> pd.Series:
    """Label samples based on future hypoxic events."""
```

**AWS Integration**:
- Triggered by S3 event notification on raw data upload
- Reads from `s3://icu-ews-data/raw/`
- Writes to `s3://icu-ews-data/processed/`
- Logs progress to CloudWatch

### 5.2 Model Training Module
**Location**: `src/training/`
**AWS Service**: Amazon SageMaker

**Components**:
- `train.py`: SageMaker training script entry point
- `model.py`: LSTM model architecture definition
- `dataset.py`: PyTorch Dataset for time-series data
- `metrics.py`: Calculate AUC-ROC, precision, recall
- `hyperparameters.py`: Hyperparameter configuration

**Model Architecture**:
```python
class HypoxiaLSTM(nn.Module):
    def __init__(self, input_size=5, hidden_size=64, num_layers=2):
        super().__init__()
        self.lstm = nn.LSTM(input_size, hidden_size, num_layers, 
                            batch_first=True, dropout=0.2)
        self.attention = nn.MultiheadAttention(hidden_size, num_heads=4)
        self.fc = nn.Sequential(
            nn.Linear(hidden_size, 32),
            nn.ReLU(),
            nn.Dropout(0.3),
            nn.Linear(32, 1),
            nn.Sigmoid()
        )
    
    def forward(self, x):
        # x shape: (batch, sequence_length, features)
        lstm_out, _ = self.lstm(x)
        attn_out, attn_weights = self.attention(lstm_out, lstm_out, lstm_out)
        prediction = self.fc(attn_out[:, -1, :])
        return prediction, attn_weights
```

**Training Configuration**:
- Instance Type: ml.p3.2xlarge (GPU for faster training)
- Framework: PyTorch 2.0
- Hyperparameters: Learning rate, batch size, epochs
- Cross-validation: 5-fold stratified

**AWS Integration**:
- SageMaker Training Job with custom container
- Reads from `s3://icu-ews-data/processed/`
- Saves model to `s3://icu-ews-models/`
- Logs metrics to SageMaker Experiments and CloudWatch

### 5.3 Model Deployment Module
**Location**: `src/deployment/`
**AWS Service**: SageMaker Endpoint

**Components**:
- `inference.py`: SageMaker inference handler
- `model_loader.py`: Load trained model from S3
- `predictor.py`: Generate predictions and confidence scores

**Inference Handler**:
```python
def model_fn(model_dir):
    """Load model from SageMaker model directory."""
    model = HypoxiaLSTM()
    model.load_state_dict(torch.load(f"{model_dir}/model.pth"))
    return model

def input_fn(request_body, content_type):
    """Parse input data from API request."""
    if content_type == 'application/json':
        data = json.loads(request_body)
        return preprocess_input(data)

def predict_fn(input_data, model):
    """Run inference and return prediction."""
    with torch.no_grad():
        prediction, attention = model(input_data)
    return prediction, attention

def output_fn(prediction, content_type):
    """Format prediction for API response."""
    risk_score = float(prediction[0] * 100)
    return json.dumps({
        'risk_score': risk_score,
        'confidence': float(prediction[0]),
        'risk_level': classify_risk(risk_score)
    })
```

**AWS Integration**:
- SageMaker Endpoint with auto-scaling
- Instance Type: ml.t2.medium (cost-effective for MVP)
- Auto-scaling: 1-5 instances based on invocations
- Model Monitor: Track data drift and prediction quality

### 5.4 API Layer Module
**Location**: `src/api/`
**AWS Service**: API Gateway + Lambda

**Components**:
- `handler.py`: Lambda function entry point
- `validators.py`: Request/response validation
- `explainability.py`: Generate feature importance
- `risk_calculator.py`: Convert probability to risk score

**API Endpoints**:
```
POST /predict
  - Input: Patient vital signs (60-min window)
  - Output: Risk score, confidence, explainability
  - Auth: Cognito JWT required

GET /health
  - Output: System health status
  - Auth: None (public)

GET /model/info
  - Output: Model version, metrics, last updated
  - Auth: Cognito JWT required
```

**Lambda Handler**:
```python
def lambda_handler(event, context):
    """Main Lambda function for prediction API."""
    try:
        # Extract and validate input
        body = json.loads(event['body'])
        vital_signs = validate_input(body)
        
        # Preprocess
        processed_data = preprocess_for_inference(vital_signs)
        
        # Call SageMaker endpoint
        response = sagemaker_runtime.invoke_endpoint(
            EndpointName='hypoxia-predictor',
            ContentType='application/json',
            Body=json.dumps(processed_data)
        )
        prediction = json.loads(response['Body'].read())
        
        # Generate explainability
        explanation = generate_explanation(vital_signs, prediction)
        
        # Log to CloudWatch
        logger.info(f"Prediction: {prediction['risk_score']}")
        
        return {
            'statusCode': 200,
            'body': json.dumps({
                **prediction,
                'explanation': explanation
            })
        }
    except Exception as e:
        logger.error(f"Error: {str(e)}")
        return {'statusCode': 500, 'body': json.dumps({'error': str(e)})}
```

**AWS Integration**:
- API Gateway REST API with Cognito Authorizer
- Lambda function with IAM role for SageMaker access
- CloudWatch Logs for all requests
- API Gateway caching (TTL: 60 seconds)


### 5.5 Frontend Dashboard Module
**Location**: `src/frontend/`
**AWS Service**: AWS Amplify or S3 + CloudFront

**Components**:
- `App.tsx`: Main React application
- `components/PatientList.tsx`: List of monitored patients
- `components/RiskScoreCard.tsx`: Display risk score with visualization
- `components/VitalSignsChart.tsx`: Time-series charts for vital signs
- `components/ExplainabilityPanel.tsx`: Feature importance visualization
- `components/AlertNotification.tsx`: High-risk alerts
- `services/api.ts`: API client for backend communication
- `auth/cognito.ts`: Cognito authentication integration

**Key Features**:
- Real-time risk score updates (simulated with historical replay)
- Color-coded risk levels (Green: Low, Yellow: Moderate, Red: High)
- Interactive vital signs charts (using Chart.js or Recharts)
- Explainability visualizations (bar charts for feature importance)
- Alert system for high-risk patients
- Responsive design for desktop and tablet

**Technology Stack**:
- React 18 with TypeScript
- Tailwind CSS for styling
- AWS Amplify SDK for auth and API
- Chart.js or Recharts for visualizations
- React Query for data fetching

**AWS Integration**:
- Hosted on AWS Amplify with CI/CD from GitHub
- Cognito SDK for authentication
- API calls to API Gateway
- CloudFront for CDN (if using S3 hosting)

### 5.6 Authentication Module
**Location**: `src/auth/`
**AWS Service**: Amazon Cognito

**Components**:
- User Pool: Manage user accounts
- App Client: Frontend application integration
- Identity Pool: Federated identities (optional)
- User Groups: Role-based access (clinician, admin)

**User Roles**:
- **Clinician**: View predictions, access dashboard
- **Administrator**: Manage users, view system metrics

**AWS Integration**:
- Cognito User Pool with email/password authentication
- MFA optional for enhanced security
- JWT tokens for API authorization
- API Gateway Cognito Authorizer

### 5.7 Monitoring Module
**Location**: `src/monitoring/`
**AWS Service**: Amazon CloudWatch

**Metrics Tracked**:
- API request count and latency
- SageMaker endpoint invocations and errors
- Lambda function duration and errors
- Model prediction distribution
- System cost metrics

**CloudWatch Dashboards**:
- System Health: API latency, error rates, uptime
- Model Performance: Prediction distribution, confidence scores
- Cost Tracking: Daily AWS spend by service

**Alarms**:
- API latency > 3 seconds (3 consecutive periods)
- SageMaker endpoint error rate > 5%
- Lambda function errors > 10 in 5 minutes
- Estimated monthly cost > $100


## 6. Model Strategy

### 6.1 Model Architecture: LSTM with Attention

**Rationale**:
- LSTMs excel at capturing temporal dependencies in time-series data
- Attention mechanism highlights which time steps are most important
- Proven effectiveness in clinical deterioration prediction

**Architecture Details**:
```
Input Layer: (batch_size, 60, 5)
  └─▶ 60 time steps (minutes)
  └─▶ 5 features (SpO2, HR, RR, SBP, DBP)

LSTM Layers: 2 layers, 64 hidden units each
  └─▶ Captures temporal patterns
  └─▶ Dropout 0.2 for regularization

Attention Layer: Multi-head attention (4 heads)
  └─▶ Identifies critical time windows
  └─▶ Provides interpretability

Fully Connected Layers:
  └─▶ FC1: 64 → 32 (ReLU, Dropout 0.3)
  └─▶ FC2: 32 → 1 (Sigmoid)

Output: Probability of hypoxic event in next 10-30 minutes
```

### 6.2 Training Strategy

**Data Split**:
- Training: 70% of patients
- Validation: 15% of patients
- Test: 15% of patients
- Stratified by hypoxic event occurrence

**Loss Function**: Binary Cross-Entropy with class weighting
- Weight positive class higher due to class imbalance
- Typical ratio: 1:5 (hypoxic events are rare)

**Optimization**:
- Optimizer: Adam with learning rate 0.001
- Learning rate scheduler: ReduceLROnPlateau
- Early stopping: Patience of 10 epochs on validation AUC-ROC
- Batch size: 32
- Max epochs: 100

**Regularization**:
- Dropout in LSTM (0.2) and FC layers (0.3)
- L2 weight decay: 1e-5
- Gradient clipping: Max norm 1.0

**Hyperparameter Tuning**:
- Use SageMaker Hyperparameter Tuning Jobs
- Parameters to tune:
  - Learning rate: [1e-4, 1e-2]
  - Hidden size: [32, 64, 128]
  - Number of LSTM layers: [1, 2, 3]
  - Dropout rate: [0.1, 0.3, 0.5]
- Objective: Maximize validation AUC-ROC
- Max trials: 20

### 6.3 Baseline Models

Compare LSTM against simpler baselines:
- **Logistic Regression**: On hand-crafted features (mean, std, trend)
- **Random Forest**: On same hand-crafted features
- **Simple Threshold**: Alert if SpO2 < 92% (current practice)

### 6.4 Model Evaluation

**Metrics**:
- AUC-ROC: Overall discrimination ability
- Precision-Recall Curve: Handle class imbalance
- Calibration Plot: Assess probability calibration
- Early Detection Gain: Minutes of advance warning
- False Positive Rate: Minimize alarm fatigue

**Evaluation Protocol**:
1. Train on training set with cross-validation
2. Select best model based on validation AUC-ROC
3. Evaluate on held-out test set (never seen during training)
4. Report confidence intervals using bootstrapping
5. Analyze performance across patient subgroups

### 6.5 Model Versioning

**Strategy**:
- Use SageMaker Model Registry
- Tag models with version, date, metrics
- A/B testing capability for comparing versions
- Rollback mechanism if new model underperforms

**Version Naming**: `hypoxia-lstm-v{major}.{minor}.{patch}`
- Major: Architecture changes
- Minor: Retraining with new data
- Patch: Hyperparameter adjustments


## 7. Risk Scoring Logic

### 7.1 Probability to Risk Score Conversion

**Formula**:
```
risk_score = probability × 100
```

Where `probability` is the model's output (0.0 to 1.0) representing the likelihood of a hypoxic event in the next 10-30 minutes.

### 7.2 Risk Level Classification

**Thresholds**:
- **Low Risk**: 0-33 (Green)
  - Routine monitoring
  - No immediate action required
  
- **Moderate Risk**: 34-66 (Yellow)
  - Increased vigilance recommended
  - Consider checking patient
  - Review recent vital signs trends
  
- **High Risk**: 67-100 (Red)
  - Immediate clinical attention
  - Alert nursing staff
  - Prepare for potential intervention

### 7.3 Confidence Scoring

**Confidence Calculation**:
```python
def calculate_confidence(prediction, attention_weights, data_quality):
    """
    Calculate confidence in the prediction.
    
    Factors:
    - Model uncertainty (prediction close to 0.5 = low confidence)
    - Attention weight distribution (focused = high confidence)
    - Data quality (missing values = low confidence)
    """
    # Model certainty: distance from 0.5
    certainty = abs(prediction - 0.5) * 2
    
    # Attention focus: entropy of attention weights
    attention_entropy = -sum(w * log(w) for w in attention_weights)
    attention_focus = 1 - (attention_entropy / max_entropy)
    
    # Data quality: percentage of non-missing values
    data_completeness = 1 - (missing_count / total_values)
    
    # Weighted combination
    confidence = (0.5 * certainty + 
                  0.3 * attention_focus + 
                  0.2 * data_completeness)
    
    return confidence
```

### 7.4 Temporal Smoothing

To reduce prediction jitter:
```python
def smooth_risk_score(current_score, previous_scores, alpha=0.3):
    """
    Exponential moving average for risk score smoothing.
    
    alpha: Smoothing factor (0 = all history, 1 = only current)
    """
    if not previous_scores:
        return current_score
    
    smoothed = alpha * current_score + (1 - alpha) * previous_scores[-1]
    return smoothed
```

### 7.5 Alert Logic

**Alert Triggering**:
```python
def should_trigger_alert(risk_score, risk_history, confidence):
    """
    Determine if an alert should be sent to clinical staff.
    
    Criteria:
    - Risk score >= 67 (High)
    - Sustained high risk (2+ consecutive high readings)
    - Confidence >= 0.6
    - No alert sent in last 15 minutes (prevent spam)
    """
    is_high_risk = risk_score >= 67
    is_sustained = len([r for r in risk_history[-3:] if r >= 67]) >= 2
    is_confident = confidence >= 0.6
    
    return is_high_risk and is_sustained and is_confident
```

**Alert Content**:
- Patient ID
- Current risk score
- Risk level (High)
- Contributing factors (from explainability)
- Recommended actions
- Timestamp


## 8. Explainability Strategy

### 8.1 Feature Importance

**Method**: Attention Weights + Gradient-Based Attribution

**Attention-Based Importance**:
```python
def get_attention_importance(attention_weights, feature_names):
    """
    Extract feature importance from attention mechanism.
    
    Returns: Dictionary of feature → importance score
    """
    # Average attention across time steps for each feature
    time_importance = attention_weights.mean(dim=0)  # (60,)
    
    # Weight recent time steps more heavily
    time_decay = torch.exp(-torch.arange(60) / 20)
    weighted_importance = time_importance * time_decay
    
    # Map to features (all features get same time importance)
    feature_importance = {
        feature: weighted_importance.sum().item()
        for feature in feature_names
    }
    
    return feature_importance
```

**Gradient-Based Attribution**:
```python
def get_gradient_importance(model, input_data, feature_names):
    """
    Calculate feature importance using input gradients.
    
    Returns: Dictionary of feature → importance score
    """
    input_data.requires_grad = True
    output = model(input_data)
    output.backward()
    
    # Gradient magnitude indicates feature importance
    gradients = input_data.grad.abs().mean(dim=1)  # Average over time
    
    feature_importance = {
        feature_names[i]: gradients[0, i].item()
        for i in range(len(feature_names))
    }
    
    return feature_importance
```

### 8.2 Temporal Patterns

**Identify Critical Time Windows**:
```python
def identify_critical_windows(attention_weights, window_size=5):
    """
    Find time windows with highest attention.
    
    Returns: List of (start_time, end_time, importance) tuples
    """
    # Smooth attention weights
    smoothed = moving_average(attention_weights, window_size)
    
    # Find peaks
    peaks = find_peaks(smoothed, threshold=0.8 * smoothed.max())
    
    critical_windows = [
        (max(0, peak - window_size), peak + window_size, smoothed[peak])
        for peak in peaks
    ]
    
    return critical_windows
```

### 8.3 Natural Language Explanations

**Generate Human-Readable Explanations**:
```python
def generate_explanation(risk_score, feature_importance, vital_signs):
    """
    Create natural language explanation for clinicians.
    
    Returns: String explanation
    """
    risk_level = classify_risk(risk_score)
    top_features = sorted(feature_importance.items(), 
                          key=lambda x: x[1], reverse=True)[:3]
    
    explanation = f"{risk_level} risk of hypoxia detected. "
    
    # Describe top contributing factors
    for feature, importance in top_features:
        trend = analyze_trend(vital_signs[feature])
        current_value = vital_signs[feature][-1]
        
        if feature == "SpO2":
            explanation += f"SpO2 is {trend} (current: {current_value}%). "
        elif feature == "HR":
            explanation += f"Heart rate is {trend} (current: {current_value} bpm). "
        elif feature == "RR":
            explanation += f"Respiratory rate is {trend} (current: {current_value}/min). "
    
    # Add recommendation
    if risk_level == "High":
        explanation += "Recommend immediate assessment and consider oxygen therapy."
    elif risk_level == "Moderate":
        explanation += "Recommend increased monitoring frequency."
    
    return explanation

def analyze_trend(time_series):
    """Determine if vital sign is increasing, decreasing, or stable."""
    recent = time_series[-10:]
    slope = np.polyfit(range(len(recent)), recent, 1)[0]
    
    if slope < -0.5:
        return "declining"
    elif slope > 0.5:
        return "increasing"
    else:
        return "stable"
```

### 8.4 Visualization Components

**Dashboard Explainability Visualizations**:

1. **Feature Importance Bar Chart**:
   - Horizontal bars showing relative importance of each vital sign
   - Color-coded by feature type
   - Tooltip with exact importance values

2. **Temporal Attention Heatmap**:
   - X-axis: Time (60 minutes)
   - Y-axis: Features
   - Color intensity: Attention weight
   - Highlights critical time windows

3. **Vital Signs Trend Lines**:
   - Line charts for each vital sign over 60-minute window
   - Overlay attention weights as background shading
   - Mark predicted hypoxic event time

4. **Counterfactual Explanation** (Optional):
   - "If SpO2 had been 2% higher, risk would decrease to 45%"
   - Shows actionable insights for intervention

### 8.5 Explainability Validation

**Sanity Checks**:
- Feature importance should align with clinical knowledge
  - SpO2 should be most important for hypoxia prediction
  - Respiratory rate should be highly relevant
- Attention should focus on recent time steps (recency bias)
- Explanations should be consistent across similar cases

**Clinical Review**:
- Present explanations to clinicians for feedback
- Validate that explanations match clinical reasoning
- Iterate on explanation generation based on feedback


## 9. Evaluation Metrics

### 9.1 Model Performance Metrics

**Primary Metric: AUC-ROC**
- Target: >= 0.80
- Measures overall discrimination ability
- Robust to class imbalance
- Interpretation: Probability that model ranks a positive case higher than a negative case

**Precision and Recall**
- Target: Both >= 0.75
- Precision: Of predicted hypoxic events, how many are true positives?
  - High precision = Low false alarm rate
- Recall: Of actual hypoxic events, how many are detected?
  - High recall = Few missed events

**F1 Score**
- Target: >= 0.75
- Harmonic mean of precision and recall
- Balances false positives and false negatives

**Precision-Recall AUC**
- Better than ROC for imbalanced datasets
- Shows trade-off between precision and recall at different thresholds

**Calibration Metrics**
- Brier Score: Mean squared error of probability predictions
- Expected Calibration Error (ECE): Difference between predicted and observed probabilities
- Calibration plot: Visual assessment of calibration

### 9.2 Clinical Utility Metrics

**Early Detection Gain**
- Target: >= 10 minutes before SpO2 < 90%
- Calculation: Time between prediction and actual event
- Distribution: Report median, 25th, and 75th percentiles
- Clinical Impact: More time for intervention

**Alert Rate**
- Alerts per patient per day
- Target: < 2 alerts/patient/day to minimize alarm fatigue
- Balance sensitivity with specificity

**Positive Predictive Value (PPV)**
- Target: >= 0.70
- Of all alerts, how many are true positives?
- Critical for clinical acceptance

**Negative Predictive Value (NPV)**
- Should be very high (>0.95)
- When model says low risk, patient should be safe

**Number Needed to Alert (NNA)**
- How many alerts to prevent one hypoxic event?
- Lower is better
- Analogous to Number Needed to Treat in clinical trials

### 9.3 System Performance Metrics

**Latency**
- End-to-end API response time
- Target: < 2 seconds (95th percentile)
- Breakdown: Preprocessing, inference, explainability

**Throughput**
- Predictions per second
- Target: >= 10 predictions/second
- Test under load with concurrent requests

**Availability**
- System uptime percentage
- Target: >= 99% during evaluation period
- Track downtime incidents

**Error Rate**
- API errors / total requests
- Target: < 1%
- Categorize by error type (4xx vs 5xx)

### 9.4 Cost Metrics

**AWS Service Costs**
- S3 storage: Data and model artifacts
- SageMaker: Training and endpoint hosting
- Lambda: API function invocations
- API Gateway: API requests
- CloudWatch: Logging and monitoring
- Cognito: User authentication

**Total Monthly Cost**
- Target: < $100/month for MVP
- Track daily spend to avoid surprises
- Identify cost optimization opportunities

**Cost per Prediction**
- Total cost / number of predictions
- Benchmark for scalability assessment

### 9.5 Evaluation Protocol

**Test Set Evaluation**:
1. Load held-out test set (15% of patients, never seen during training)
2. Generate predictions for all test samples
3. Calculate all metrics with 95% confidence intervals (bootstrapping)
4. Stratify analysis by:
   - Patient demographics (age, gender)
   - ICU type (medical, surgical, cardiac)
   - Time of day (day shift vs night shift)
5. Compare against baseline models

**Temporal Validation**:
- Train on earlier time period, test on later period
- Assess model degradation over time
- Simulate real-world deployment scenario

**Subgroup Analysis**:
- Ensure model performs equitably across patient groups
- Identify any performance disparities
- Document limitations

**Error Analysis**:
- Manually review false positives and false negatives
- Identify common failure modes
- Generate insights for model improvement

### 9.6 Reporting

**Evaluation Report Contents**:
1. Executive Summary
2. Dataset Description
3. Model Architecture and Training
4. Performance Metrics (with confidence intervals)
5. Comparison to Baselines
6. Subgroup Analysis
7. Error Analysis
8. System Performance and Cost
9. Limitations and Future Work
10. Conclusions and Recommendations

**Visualizations**:
- ROC curve and Precision-Recall curve
- Calibration plot
- Confusion matrix
- Early detection gain distribution
- Feature importance across test set
- Example predictions with explanations


## 10. MVP Constraints

### 10.1 Scope Limitations

**In Scope**:
- Hypoxic event prediction only (SpO2 < 90%)
- MIMIC-IV / PhysioNet datasets
- Simulated real-time using historical data replay
- Web-based dashboard for visualization
- AWS-based infrastructure
- Basic explainability features

**Out of Scope**:
- Real-time hospital EHR integration
- Other deterioration events (sepsis, cardiac arrest, etc.)
- Mobile applications
- Multi-hospital deployment
- Regulatory approval (FDA, CE marking)
- Prospective clinical validation
- Automated interventions
- Integration with nurse call systems

### 10.2 Technical Constraints

**Data Constraints**:
- Limited to publicly available ICU datasets
- No real patient data (privacy/compliance)
- Historical data only (no live streaming)
- Fixed feature set (5 vital signs)

**Model Constraints**:
- Single model architecture (LSTM)
- Limited hyperparameter tuning (20 trials max)
- No ensemble methods
- No online learning / model updates

**Infrastructure Constraints**:
- AWS services only (no multi-cloud)
- Minimal instance types for cost control
- Single region deployment (us-east-1)
- No high availability / disaster recovery

**Development Constraints**:
- 2-4 week timeline (hackathon pace)
- Solo or small team development
- Minimal documentation
- No extensive testing (unit tests only)

### 10.3 Performance Constraints

**Acceptable Trade-offs**:
- Inference latency: 2 seconds acceptable (not real-time)
- Throughput: 10 predictions/second sufficient for MVP
- Availability: 99% acceptable (not 99.9%)
- Scalability: 100 concurrent users max

**Quality Thresholds**:
- Model performance: AUC-ROC >= 0.80 (not state-of-the-art)
- False positive rate: < 30% acceptable for MVP
- Explainability: Basic feature importance (not SHAP/LIME)

### 10.4 Cost Constraints

**Budget**: $100/month maximum

**Cost Optimization Strategies**:
- Use smallest viable instance types
- Leverage serverless (Lambda) where possible
- Implement S3 lifecycle policies
- Use SageMaker spot instances for training
- Enable API Gateway caching
- Shut down resources when not in use

**Cost Monitoring**:
- Daily cost tracking via CloudWatch
- Alerts if approaching budget limit
- Regular cost optimization reviews

### 10.5 Timeline Constraints

**Phase 1: Documentation (Days 1-2)**
- Requirements and design documents
- AWS architecture planning

**Phase 2: Data Pipeline (Days 3-5)**
- Download MIMIC-IV dataset
- Implement preprocessing pipeline
- Upload to S3

**Phase 3: Model Development (Days 6-10)**
- Implement LSTM model
- Train and tune hyperparameters
- Evaluate on test set

**Phase 4: Deployment (Days 11-13)**
- Deploy SageMaker endpoint
- Implement API with Lambda
- Set up API Gateway

**Phase 5: Frontend (Days 14-16)**
- Build React dashboard
- Integrate with API
- Deploy on AWS Amplify

**Phase 6: Testing & Evaluation (Days 17-18)**
- End-to-end testing
- Performance benchmarking
- Generate evaluation report

**Phase 7: Documentation & Presentation (Days 19-20)**
- Final documentation
- Demo preparation
- Presentation materials

### 10.6 Risk Mitigation

**Technical Risks**:
- Model doesn't achieve target performance
  - Mitigation: Start with baseline models, iterate quickly
- AWS costs exceed budget
  - Mitigation: Daily cost monitoring, auto-shutdown policies
- Integration issues between components
  - Mitigation: Test each component independently first

**Timeline Risks**:
- Data preprocessing takes longer than expected
  - Mitigation: Use pre-processed MIMIC-IV if available
- Model training is slow
  - Mitigation: Use smaller dataset for initial experiments
- Frontend development delays
  - Mitigation: Use simple UI, focus on functionality over aesthetics

**Scope Risks**:
- Feature creep
  - Mitigation: Strict adherence to MVP scope, defer enhancements
- Over-engineering
  - Mitigation: Simplest solution that meets requirements


## 11. AWS Cost Optimization Strategy

### 11.1 Cost Breakdown Estimate

**Monthly Cost Projection**:

| Service | Usage | Estimated Cost |
|---------|-------|----------------|
| S3 Storage | 50 GB data + models | $1.15 |
| S3 Requests | 100K GET, 10K PUT | $0.50 |
| Lambda | 100K invocations, 512MB, 1s avg | $2.00 |
| API Gateway | 100K requests | $0.35 |
| SageMaker Endpoint | ml.t2.medium, 24/7 | $35.00 |
| SageMaker Training | ml.p3.2xlarge, 10 hours | $30.00 |
| CloudWatch | Logs (5 GB), Metrics, Alarms | $5.00 |
| Cognito | 1000 MAU | $0.00 (free tier) |
| Amplify | Static hosting, 10 GB transfer | $1.00 |
| **Total** | | **~$75/month** |

### 11.2 Cost Optimization Techniques

**S3 Optimization**:
- Use S3 Intelligent-Tiering for infrequently accessed data
- Implement lifecycle policies:
  - Move raw data to Glacier after 30 days
  - Delete temporary preprocessing files after 7 days
- Enable S3 compression for large datasets
- Use S3 Select to query data without downloading

**Lambda Optimization**:
- Right-size memory allocation (test 256MB, 512MB, 1024MB)
- Minimize cold starts with provisioned concurrency (if needed)
- Reuse connections and resources across invocations
- Use Lambda layers for shared dependencies

**SageMaker Optimization**:
- Use spot instances for training (up to 70% savings)
- Shut down endpoint when not in use (dev/test)
- Use smallest instance type that meets latency requirements
- Consider SageMaker Serverless Inference for low traffic
- Enable auto-scaling with conservative thresholds

**API Gateway Optimization**:
- Enable caching (TTL: 60 seconds) to reduce Lambda invocations
- Use API Gateway HTTP API instead of REST API (cheaper)
- Implement request throttling to prevent abuse

**CloudWatch Optimization**:
- Set log retention to 7 days (not indefinite)
- Use log filtering to reduce ingestion volume
- Aggregate metrics before sending to CloudWatch
- Use CloudWatch Logs Insights instead of exporting logs

**General Strategies**:
- Use AWS Free Tier where applicable
- Tag all resources for cost tracking
- Set up billing alerts at $50, $75, $100
- Review AWS Cost Explorer weekly
- Shut down non-production resources outside business hours

### 11.3 Cost Monitoring

**CloudWatch Billing Alarms**:
```python
# Create billing alarm at $75
alarm = cloudwatch.Alarm(
    alarm_name="ICU-EWS-Cost-Alert",
    comparison_operator="GreaterThanThreshold",
    evaluation_periods=1,
    metric_name="EstimatedCharges",
    namespace="AWS/Billing",
    period=21600,  # 6 hours
    statistic="Maximum",
    threshold=75.0,
    actions_enabled=True,
    alarm_actions=[sns_topic_arn]
)
```

**Daily Cost Tracking**:
- Automated script to fetch daily costs via AWS Cost Explorer API
- Log to CloudWatch Metrics for visualization
- Alert if daily cost > $5

**Cost Attribution**:
- Tag resources by component (preprocessing, training, inference, frontend)
- Generate cost breakdown by component
- Identify optimization opportunities

### 11.4 Scaling Cost Projections

**If scaling to 1000 concurrent users**:
- SageMaker Endpoint: Scale to 5 instances → $175/month
- Lambda: 1M invocations → $20/month
- API Gateway: 1M requests → $3.50/month
- S3 Transfer: Increase to 100 GB → $9/month
- **Total**: ~$250/month

**Cost per User**:
- MVP (100 users): $0.75/user/month
- Scaled (1000 users): $0.25/user/month
- Economies of scale with fixed costs (training, storage)


## 12. Future Scalability using AWS

### 12.1 Horizontal Scaling

**Current MVP Architecture**:
- Single SageMaker endpoint
- Single region (us-east-1)
- Synchronous API calls

**Scaled Architecture**:
- Multiple SageMaker endpoints with load balancing
- Multi-region deployment (us-east-1, us-west-2, eu-west-1)
- Asynchronous processing with SQS for batch predictions
- ElastiCache for caching frequent predictions

**Scaling Triggers**:
- Auto-scale SageMaker endpoint based on invocations/minute
- Scale Lambda concurrency based on API Gateway requests
- Add CloudFront for global CDN distribution

### 12.2 Data Pipeline Scaling

**Current MVP**:
- Lambda for small-scale preprocessing
- Manual data uploads to S3

**Scaled Pipeline**:
- AWS Glue for large-scale ETL (millions of records)
- AWS Step Functions for complex workflow orchestration
- Amazon Kinesis for real-time data streaming
- AWS Data Pipeline for scheduled batch processing
- Amazon Athena for ad-hoc data analysis

**Real-Time Integration**:
- Kinesis Data Streams: Ingest live vital signs from hospital systems
- Kinesis Data Firehose: Deliver to S3 for archival
- Lambda: Real-time preprocessing and inference
- DynamoDB: Store patient state and prediction history

### 12.3 Model Management at Scale

**Model Training**:
- SageMaker Pipelines: Automated retraining workflows
- SageMaker Feature Store: Centralized feature management
- SageMaker Experiments: Track hundreds of training runs
- Distributed training: Multi-GPU, multi-node for large datasets

**Model Deployment**:
- SageMaker Multi-Model Endpoints: Host multiple models on single endpoint
- SageMaker A/B Testing: Gradual rollout of new models
- SageMaker Model Monitor: Detect data drift and model degradation
- Automated rollback on performance degradation

**Model Versioning**:
- SageMaker Model Registry: Centralized model catalog
- CI/CD pipeline: Automated testing and deployment
- Blue-green deployment: Zero-downtime updates

### 12.4 Enhanced Monitoring and Observability

**Current MVP**:
- Basic CloudWatch logs and metrics
- Manual dashboard review

**Scaled Monitoring**:
- CloudWatch ServiceLens: End-to-end service map
- AWS X-Ray: Distributed tracing across all components
- CloudWatch Insights: Advanced log analytics
- CloudWatch Anomaly Detection: Automatic anomaly alerts
- Amazon QuickSight: Business intelligence dashboards

**Alerting**:
- SNS: Multi-channel notifications (email, SMS, Slack)
- PagerDuty integration: On-call escalation
- EventBridge: Event-driven automation

### 12.5 Security and Compliance at Scale

**Enhanced Security**:
- AWS WAF: Protect API Gateway from attacks
- AWS Shield: DDoS protection
- AWS Secrets Manager: Secure credential management
- VPC: Isolate resources in private network
- PrivateLink: Secure service-to-service communication

**Compliance**:
- AWS Config: Track configuration changes
- AWS CloudTrail: Audit all API calls
- AWS Artifact: Access compliance reports
- HIPAA-eligible services: For production healthcare use
- Encryption: KMS for all data at rest and in transit

**Access Control**:
- AWS SSO: Centralized user management
- IAM Identity Center: Fine-grained permissions
- AWS Organizations: Multi-account strategy
- Service Control Policies: Guardrails across accounts

### 12.6 Multi-Condition Prediction

**Expand Beyond Hypoxia**:
- Train separate models for:
  - Sepsis prediction
  - Cardiac arrest prediction
  - Acute kidney injury
  - Hemodynamic instability
- Multi-task learning: Single model predicting multiple outcomes
- Ensemble methods: Combine predictions from multiple models

**Unified Risk Dashboard**:
- Composite risk score across all conditions
- Prioritization algorithm for clinical attention
- Condition-specific explainability

### 12.7 Advanced Features

**Personalization**:
- Patient-specific models using transfer learning
- Incorporate patient history and comorbidities
- Adaptive thresholds based on patient baseline

**Federated Learning**:
- Train models across multiple hospitals without sharing data
- SageMaker + AWS PrivateLink for secure federation
- Aggregate model updates while preserving privacy

**Continuous Learning**:
- Online learning: Update model with new data
- Active learning: Prioritize labeling of uncertain cases
- Automated retraining pipeline triggered by performance degradation

**Advanced Explainability**:
- SHAP values for global and local interpretability
- Counterfactual explanations: "What if" scenarios
- Attention visualization: Highlight critical time windows
- Natural language generation: Detailed clinical narratives

### 12.8 Integration Ecosystem

**EHR Integration**:
- HL7 FHIR API: Standard healthcare data exchange
- AWS HealthLake: Store and query FHIR data
- Real-time bidirectional sync with hospital systems

**Clinical Workflow Integration**:
- Nurse call system integration
- Automated paging for high-risk alerts
- Integration with clinical decision support systems
- Mobile app for clinicians (iOS/Android)

**Research Platform**:
- Data lake for retrospective studies
- Jupyter notebooks on SageMaker for researchers
- Secure data sharing with research partners
- Synthetic data generation for privacy-preserving research

### 12.9 Cost at Scale

**Projected Costs for 10,000 Patients**:
- SageMaker Endpoints: $500/month (10 instances)
- Lambda: $100/month (10M invocations)
- API Gateway: $35/month (10M requests)
- S3: $50/month (500 GB)
- CloudWatch: $50/month
- Other services: $100/month
- **Total**: ~$835/month (~$0.08/patient/month)

**Cost Optimization at Scale**:
- Reserved instances for SageMaker (up to 75% savings)
- Savings Plans for Lambda and other compute
- S3 Intelligent-Tiering for automatic cost optimization
- Spot instances for non-critical workloads

### 12.10 Migration Path from MVP to Production

**Phase 1: MVP** (Current)
- Single model, single region
- Historical data replay
- Basic monitoring

**Phase 2: Pilot** (3-6 months)
- Multi-model support
- Enhanced monitoring and alerting
- Prospective validation study
- Regulatory preparation

**Phase 3: Production** (6-12 months)
- Real-time EHR integration
- Multi-region deployment
- HIPAA compliance
- Clinical workflow integration
- Regulatory approval (FDA 510(k) or De Novo)

**Phase 4: Scale** (12+ months)
- Multi-hospital deployment
- Federated learning
- Advanced features (personalization, multi-condition)
- Research platform


## 13. Implementation Roadmap

### 13.1 Prerequisites

**AWS Account Setup**:
- Create AWS account (or use existing)
- Set up IAM user with appropriate permissions
- Configure AWS CLI with credentials
- Enable billing alerts

**Development Environment**:
- Python 3.9+ with virtual environment
- AWS SDK (boto3)
- PyTorch 2.0+
- Node.js 18+ for frontend
- Git for version control

**Dataset Access**:
- Register for PhysioNet account
- Complete CITI training for MIMIC-IV access
- Download MIMIC-IV dataset (or subset)

### 13.2 Phase-by-Phase Implementation

**Phase 1: Infrastructure Setup (Days 1-2)**
- [ ] Create S3 buckets (raw, processed, models)
- [ ] Set up IAM roles and policies
- [ ] Configure CloudWatch logging
- [ ] Create Cognito user pool
- [ ] Set up development environment

**Phase 2: Data Pipeline (Days 3-5)**
- [ ] Download MIMIC-IV dataset
- [ ] Implement data loading scripts
- [ ] Develop preprocessing Lambda/Glue job
- [ ] Create feature engineering pipeline
- [ ] Generate training/validation/test splits
- [ ] Upload processed data to S3

**Phase 3: Model Development (Days 6-10)**
- [ ] Implement LSTM model architecture
- [ ] Create PyTorch Dataset and DataLoader
- [ ] Implement training script for SageMaker
- [ ] Configure hyperparameter tuning
- [ ] Train baseline models (logistic regression, random forest)
- [ ] Train LSTM model
- [ ] Evaluate on test set
- [ ] Generate evaluation metrics

**Phase 4: Model Deployment (Days 11-12)**
- [ ] Package model for SageMaker deployment
- [ ] Create SageMaker endpoint
- [ ] Test endpoint with sample data
- [ ] Configure auto-scaling
- [ ] Set up Model Monitor

**Phase 5: API Development (Day 13)**
- [ ] Implement Lambda function for API
- [ ] Create API Gateway REST API
- [ ] Configure Cognito authorizer
- [ ] Implement explainability logic
- [ ] Test API endpoints
- [ ] Enable API Gateway caching

**Phase 6: Frontend Development (Days 14-16)**
- [ ] Set up React project with TypeScript
- [ ] Implement authentication with Cognito
- [ ] Create patient list component
- [ ] Build risk score visualization
- [ ] Implement vital signs charts
- [ ] Add explainability panel
- [ ] Implement alert notifications
- [ ] Deploy to AWS Amplify

**Phase 7: Integration Testing (Day 17)**
- [ ] End-to-end testing of full pipeline
- [ ] Load testing with concurrent requests
- [ ] Verify monitoring and logging
- [ ] Test alert triggering logic
- [ ] Validate explainability output

**Phase 8: Evaluation & Documentation (Days 18-20)**
- [ ] Generate comprehensive evaluation report
- [ ] Document system architecture
- [ ] Create user guide for dashboard
- [ ] Prepare demo scenarios
- [ ] Create presentation materials
- [ ] Final cost analysis

### 13.3 Key Deliverables

**Documentation**:
- ✓ requirements.md (this document)
- ✓ design.md (this document)
- README.md with setup instructions
- API documentation
- User guide for dashboard
- Evaluation report

**Code Repositories**:
- `icu-ews-preprocessing/`: Data pipeline code
- `icu-ews-model/`: Model training and inference code
- `icu-ews-api/`: Lambda functions and API code
- `icu-ews-dashboard/`: React frontend code
- `icu-ews-infrastructure/`: CloudFormation/Terraform templates

**AWS Resources**:
- S3 buckets with data and models
- SageMaker trained model and endpoint
- Lambda functions deployed
- API Gateway configured
- Cognito user pool
- CloudWatch dashboards and alarms
- Amplify hosted dashboard

**Evaluation Artifacts**:
- Model performance metrics (AUC-ROC, precision, recall, F1)
- Evaluation report with visualizations
- Cost analysis spreadsheet
- Demo video or presentation

### 13.4 Success Criteria

**Technical Success**:
- [ ] Model achieves AUC-ROC >= 0.80
- [ ] Model achieves Precision >= 0.75
- [ ] Model achieves Recall >= 0.75
- [ ] Early detection gain >= 10 minutes
- [ ] API latency < 2 seconds (95th percentile)
- [ ] System uptime >= 99%
- [ ] Total AWS cost < $100/month

**Functional Success**:
- [ ] All functional requirements implemented
- [ ] Dashboard is usable and intuitive
- [ ] Explainability features provide clinical insights
- [ ] Authentication and authorization work correctly
- [ ] Monitoring and logging are comprehensive

**Documentation Success**:
- [ ] All deliverables completed
- [ ] Code is well-documented
- [ ] System is reproducible from documentation
- [ ] Evaluation report is comprehensive

### 13.5 Risk Mitigation Strategies

**If model performance is insufficient**:
- Try different architectures (GRU, Transformer)
- Increase model capacity (more layers, hidden units)
- Collect more training data
- Improve feature engineering
- Adjust class weights or use focal loss

**If AWS costs exceed budget**:
- Shut down endpoint when not in use
- Use smaller instance types
- Reduce logging verbosity
- Implement more aggressive caching
- Use spot instances for training

**If timeline is at risk**:
- Reduce scope (e.g., simpler dashboard)
- Use pre-built components (e.g., AWS Amplify UI)
- Parallelize independent tasks
- Defer non-critical features

**If integration issues arise**:
- Test each component independently
- Use mock services for testing
- Implement comprehensive error handling
- Add detailed logging for debugging

## 14. Conclusion

This design document provides a comprehensive blueprint for building an AWS-powered ICU Early Warning System for hypoxia prediction. The system leverages modern machine learning techniques (LSTM with attention) and AWS managed services to create a scalable, cost-effective, and clinically useful assistive AI tool.

Key highlights:
- **Clinical Focus**: Early hypoxia detection with 10+ minute advance warning
- **AWS Architecture**: Fully serverless and managed services for scalability
- **Explainability**: Interpretable predictions for clinical decision support
- **Cost-Effective**: Target $75/month for MVP operation
- **Scalable**: Clear path from MVP to production deployment

The next step is to proceed with implementation following the roadmap outlined in Section 13, starting with infrastructure setup and data pipeline development.
