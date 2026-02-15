# Design: Cross-Platform OS Developer Workflow AI Copilot (AWS-Powered)

## 1. System Architecture Overview

### 1.1 Architecture Philosophy
The system follows a serverless, event-driven architecture using AWS managed services with strict layer separation. The design prioritizes safety, auditability, cross-platform compatibility, and modularity with clear separation between planning, schema validation, guardrail enforcement, and execution layers.

**Key Principles**:
- Strict layer separation: Planner → Validator → Guardrail → Executor
- LLM has no direct execution capabilities
- All plans validated before execution
- Cross-platform abstraction for OS interactions
- Explicit state machine for workflow tracking

### 1.2 Core Components
1. **Frontend Dashboard** (AWS Amplify) - User interface for workflow submission and monitoring
2. **API Gateway** - REST API endpoint management and routing
3. **Task Planner Lambda** - Natural language to structured plan conversion (LLM integration)
4. **Plan Validator Lambda** - JSON schema validation
5. **Guardrail Layer Lambda** - Safety checks and command filtering
6. **Execution Engine** - Local agent that executes validated plans (cross-platform)
7. **State Manager** - DynamoDB-based state machine and log storage
8. **LLM Service** - Amazon Bedrock (preferred) or SageMaker for AI capabilities
9. **Platform Abstraction Layer** - Cross-platform command and path handling

### 1.3 Execution Pipeline (Strict Layer Separation)
```
User Input (Natural Language)
    ↓
API Gateway (Authentication)
    ↓
Task Planner Lambda → LLM (Bedrock/SageMaker) → Generate JSON Plan
    ↓
State: RECEIVED → PLANNED
    ↓
Plan Validator Lambda → Validate JSON Schema
    ↓
State: PLANNED → VALIDATED (or FAILED if invalid)
    ↓
Guardrail Layer Lambda → Safety Checks & Command Filtering
    ↓
State: VALIDATED → AWAITING_CONFIRMATION (if high-risk) or EXECUTING
    ↓
User Confirmation (if required)
    ↓
State: AWAITING_CONFIRMATION → EXECUTING
    ↓
Execution Engine (Local Agent) → Platform Abstraction Layer → OS Actions
    ↓
State: EXECUTING → COMPLETED / FAILED / DEGRADED
    ↓
DynamoDB (State Storage) ← Execution Logs
    ↓
CloudWatch (Monitoring) + X-Ray (Tracing)
    ↓
Frontend Dashboard (Real-time Updates)
```

### 1.4 Cross-Platform Support
The system supports macOS, Linux, and Windows through:
- Platform detection and plan generation
- Cross-platform command abstraction
- Normalized file path handling
- Platform-specific guardrails

## 2. High-Level Component Diagram

### 2.1 AWS Cloud Components
```
┌─────────────────────────────────────────────────────────────────────────┐
│                           AWS Cloud                                      │
│                                                                          │
│  ┌──────────────┐      ┌─────────────────┐                             │
│  │   Amplify    │◄─────┤  API Gateway    │                             │
│  │  (Frontend)  │      │   (REST API)    │                             │
│  └──────────────┘      └────────┬────────┘                             │
│                                  │                                       │
│                         ┌────────▼────────┐                             │
│                         │  Lambda         │                             │
│                         │  Task Planner   │                             │
│                         └────────┬────────┘                             │
│                                  │                                       │
│                    ┌─────────────▼──────────────┐                       │
│                    │  Amazon Bedrock/SageMaker  │                       │
│                    │  (LLM Service)             │                       │
│                    └─────────────┬──────────────┘                       │
│                                  │                                       │
│                         ┌────────▼────────┐                             │
│                         │  Lambda         │                             │
│                         │  Plan Validator │                             │
│                         └────────┬────────┘                             │
│                                  │                                       │
│                         ┌────────▼────────┐                             │
│                         │  Lambda         │                             │
│                         │  Guardrail      │                             │
│                         │  Layer          │                             │
│                         └────────┬────────┘                             │
│                                  │                                       │
│  ┌──────────────┐      ┌────────▼────────┐                             │
│  │  DynamoDB    │◄─────┤  State Manager  │                             │
│  │  (State)     │      │  (Workflows,    │                             │
│  │              │      │   Logs, States) │                             │
│  └──────────────┘      └─────────────────┘                             │
│                                                                          │
│  ┌──────────────┐      ┌─────────────────┐                             │
│  │ CloudWatch   │      │    X-Ray        │                             │
│  │ (Logs)       │      │   (Tracing)     │                             │
│  └──────────────┘      └─────────────────┘                             │
│                                                                          │
│  ┌──────────────┐      ┌─────────────────┐                             │
│  │   Cognito    │      │  AWS Budgets    │                             │
│  │   (Auth)     │      │  (Cost Alerts)  │                             │
│  └──────────────┘      └─────────────────┘                             │
└─────────────────────────────────────────────────────────────────────────┘
                           │
                           │ Validated Plan + State Updates
                           ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                    Local Environment (User Machine)                      │
│                                                                          │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │           Execution Engine (Local Agent)                         │   │
│  │                                                                  │   │
│  │  ┌────────────────────────────────────────────────────────────┐ │   │
│  │  │         Platform Abstraction Layer                         │ │   │
│  │  │  - Command Translator (bash/zsh/powershell/cmd)           │ │   │
│  │  │  - Path Normalizer (/ vs \)                               │ │   │
│  │  │  - Platform Detector (macOS/Linux/Windows)                │ │   │
│  │  └────────────────────────────────────────────────────────────┘ │   │
│  │                                                                  │   │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────────────┐              │   │
│  │  │ Terminal │  │ VS Code  │  │ Browser/GitHub   │              │   │
│  │  │ Executor │  │ Executor │  │ Executor         │              │   │
│  │  │ (Cross-  │  │ (Cross-  │  │ (Cross-platform) │              │   │
│  │  │ platform)│  │ platform)│  │                  │              │   │
│  │  └──────────┘  └──────────┘  └──────────────────┘              │   │
│  └──────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │                    Operating System                              │   │
│  │  macOS (bash/zsh) | Linux (bash) | Windows (PowerShell/cmd)     │   │
│  └──────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────┘
```

## 3. Task Plan JSON Schema

### 3.1 Complete Schema Definition
All task plans generated by the LLM must conform to this exact JSON schema:

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "required": ["task_id", "user_id", "platform", "description", "working_directory", "created_at", "estimated_duration_seconds", "risk_level", "steps"],
  "properties": {
    "task_id": {
      "type": "string",
      "format": "uuid",
      "description": "Unique identifier for the workflow"
    },
    "user_id": {
      "type": "string",
      "description": "User who created the workflow"
    },
    "platform": {
      "type": "string",
      "enum": ["macOS", "Linux", "Windows"],
      "description": "Target operating system"
    },
    "description": {
      "type": "string",
      "minLength": 1,
      "maxLength": 500,
      "description": "Human-readable workflow description"
    },
    "working_directory": {
      "type": "string",
      "pattern": "^(/|[A-Z]:\\\\).*",
      "description": "Absolute path to project directory"
    },
    "created_at": {
      "type": "string",
      "format": "date-time",
      "description": "ISO 8601 timestamp"
    },
    "estimated_duration_seconds": {
      "type": "integer",
      "minimum": 1,
      "maximum": 3600,
      "description": "Estimated total execution time"
    },
    "risk_level": {
      "type": "string",
      "enum": ["low", "medium", "high"],
      "description": "Overall workflow risk assessment"
    },
    "steps": {
      "type": "array",
      "minItems": 1,
      "maxItems": 50,
      "items": {
        "type": "object",
        "required": ["step_id", "action_type", "description", "requires_confirmation", "timeout_seconds", "retry_count"],
        "properties": {
          "step_id": {
            "type": "integer",
            "minimum": 1,
            "description": "Sequential step identifier"
          },
          "action_type": {
            "type": "string",
            "enum": ["terminal_command", "open_app", "write_file", "browser_action"],
            "description": "Type of action to execute"
          },
          "description": {
            "type": "string",
            "minLength": 1,
            "maxLength": 200,
            "description": "Human-readable step description"
          },
          "command": {
            "type": "string",
            "description": "Command to execute (required for terminal_command)"
          },
          "shell": {
            "type": "string",
            "enum": ["bash", "zsh", "powershell", "cmd"],
            "description": "Shell to use (required for terminal_command)"
          },
          "working_directory": {
            "type": "string",
            "description": "Override working directory for this step"
          },
          "file_path": {
            "type": "string",
            "description": "File path (required for write_file)"
          },
          "content": {
            "type": "string",
            "description": "File content (required for write_file)"
          },
          "app_name": {
            "type": "string",
            "description": "Application name (required for open_app)"
          },
          "app_args": {
            "type": "array",
            "items": {"type": "string"},
            "description": "Application arguments (optional for open_app)"
          },
          "url": {
            "type": "string",
            "format": "uri",
            "description": "URL (required for browser_action)"
          },
          "browser_action_type": {
            "type": "string",
            "enum": ["navigate", "click", "fill_form"],
            "description": "Browser action type (required for browser_action)"
          },
          "requires_confirmation": {
            "type": "boolean",
            "description": "Whether user confirmation is required"
          },
          "timeout_seconds": {
            "type": "integer",
            "minimum": 1,
            "maximum": 300,
            "description": "Maximum execution time for this step"
          },
          "retry_count": {
            "type": "integer",
            "minimum": 0,
            "maximum": 3,
            "description": "Number of retry attempts on failure"
          },
          "dependencies": {
            "type": "array",
            "items": {"type": "integer"},
            "description": "Array of step_ids that must complete first"
          },
          "rollback_action": {
            "type": "object",
            "properties": {
              "action_type": {"type": "string"},
              "command": {"type": "string"},
              "description": {"type": "string"}
            },
            "description": "Optional rollback action if step fails"
          }
        },
        "allOf": [
          {
            "if": {"properties": {"action_type": {"const": "terminal_command"}}},
            "then": {"required": ["command", "shell"]}
          },
          {
            "if": {"properties": {"action_type": {"const": "write_file"}}},
            "then": {"required": ["file_path", "content"]}
          },
          {
            "if": {"properties": {"action_type": {"const": "open_app"}}},
            "then": {"required": ["app_name"]}
          },
          {
            "if": {"properties": {"action_type": {"const": "browser_action"}}},
            "then": {"required": ["url", "browser_action_type"]}
          }
        ]
      }
    }
  }
}
```

### 3.2 Example Task Plan
```json
{
  "task_id": "550e8400-e29b-41d4-a716-446655440000",
  "user_id": "user123",
  "platform": "macOS",
  "description": "Create new React project with Tailwind CSS",
  "working_directory": "/Users/developer/projects",
  "created_at": "2026-02-15T10:30:00Z",
  "estimated_duration_seconds": 120,
  "risk_level": "low",
  "steps": [
    {
      "step_id": 1,
      "action_type": "terminal_command",
      "description": "Create React app using Vite",
      "command": "npm create vite@latest my-app -- --template react",
      "shell": "zsh",
      "requires_confirmation": false,
      "timeout_seconds": 60,
      "retry_count": 2,
      "dependencies": []
    },
    {
      "step_id": 2,
      "action_type": "terminal_command",
      "description": "Navigate to project directory",
      "command": "cd my-app",
      "shell": "zsh",
      "working_directory": "/Users/developer/projects/my-app",
      "requires_confirmation": false,
      "timeout_seconds": 5,
      "retry_count": 0,
      "dependencies": [1]
    },
    {
      "step_id": 3,
      "action_type": "terminal_command",
      "description": "Install Tailwind CSS",
      "command": "npm install -D tailwindcss postcss autoprefixer",
      "shell": "zsh",
      "requires_confirmation": false,
      "timeout_seconds": 60,
      "retry_count": 2,
      "dependencies": [2]
    },
    {
      "step_id": 4,
      "action_type": "terminal_command",
      "description": "Initialize Tailwind config",
      "command": "npx tailwindcss init -p",
      "shell": "zsh",
      "requires_confirmation": false,
      "timeout_seconds": 30,
      "retry_count": 1,
      "dependencies": [3]
    },
    {
      "step_id": 5,
      "action_type": "write_file",
      "description": "Update Tailwind config",
      "file_path": "/Users/developer/projects/my-app/tailwind.config.js",
      "content": "/** @type {import('tailwindcss').Config} */\nexport default {\n  content: ['./index.html', './src/**/*.{js,ts,jsx,tsx}'],\n  theme: { extend: {} },\n  plugins: [],\n}",
      "requires_confirmation": false,
      "timeout_seconds": 5,
      "retry_count": 0,
      "dependencies": [4],
      "rollback_action": {
        "action_type": "restore_file",
        "description": "Restore original tailwind.config.js"
      }
    }
  ]
}
```

### 3.3 Schema Validation Rules
1. **Plan Validator Lambda** must validate all plans against this schema
2. Plans with missing required fields must be rejected
3. Plans with invalid enum values must be rejected
4. Plans with circular dependencies must be rejected
5. Plans with timeout_seconds > 300 must be rejected
6. Plans with retry_count > 3 must be rejected
7. Platform must match user's detected OS
8. Working directory must be an absolute path
9. Step dependencies must reference valid step_ids
10. Each action_type must have its required fields

## 4. Workflow State Machine

### 4.1 State Diagram
```
                    ┌──────────────┐
                    │   RECEIVED   │ (User submits request)
                    └──────┬───────┘
                           │
                           ▼
                    ┌──────────────┐
                    │   PLANNED    │ (LLM generates plan)
                    └──────┬───────┘
                           │
                ┌──────────┴──────────┐
                │                     │
                ▼                     ▼
         ┌──────────────┐      ┌──────────┐
         │  VALIDATED   │      │  FAILED  │ (Schema invalid)
         └──────┬───────┘      └──────────┘
                │
    ┌───────────┴───────────┐
    │                       │
    ▼                       ▼
┌─────────────────┐   ┌──────────────┐
│ AWAITING_       │   │  EXECUTING   │ (No confirmation needed)
│ CONFIRMATION    │   └──────┬───────┘
└────────┬────────┘          │
         │              ┌────┴────┬──────────┐
         │              │         │          │
         ▼              ▼         ▼          ▼
    ┌──────────┐  ┌──────────┐ ┌────────┐ ┌──────────┐
    │ FAILED   │  │COMPLETED │ │ FAILED │ │ DEGRADED │
    └──────────┘  └──────────┘ └────────┘ └──────────┘
    (User rejects  (All steps   (Critical  (Partial
     or timeout)    succeed)     step fails) success)
```

### 4.2 State Definitions

#### RECEIVED
- **Entry**: User submits natural language request via API
- **Actions**: Store request in DynamoDB, invoke Task Planner Lambda
- **Exit**: Task Planner generates JSON plan
- **Next States**: PLANNED

#### PLANNED
- **Entry**: LLM generates structured JSON plan
- **Actions**: Store plan in DynamoDB, invoke Plan Validator Lambda
- **Exit**: Plan Validator completes validation
- **Next States**: VALIDATED (if valid), FAILED (if invalid)

#### VALIDATED
- **Entry**: Plan Validator approves schema
- **Actions**: Invoke Guardrail Layer Lambda, check for high-risk actions
- **Exit**: Guardrail Layer completes safety checks
- **Next States**: AWAITING_CONFIRMATION (if high-risk), EXECUTING (if safe)

#### AWAITING_CONFIRMATION
- **Entry**: High-risk actions detected, user confirmation required
- **Actions**: Send confirmation request to frontend, start 60-second timeout
- **Exit**: User responds or timeout expires
- **Next States**: EXECUTING (if approved), FAILED (if rejected/timeout)

#### EXECUTING
- **Entry**: Execution Engine begins running steps
- **Actions**: Execute steps sequentially, log progress, update state after each step
- **Exit**: All steps complete or critical failure occurs
- **Next States**: COMPLETED (all succeed), FAILED (critical failure), DEGRADED (partial success)

#### COMPLETED
- **Entry**: All steps executed successfully
- **Actions**: Log completion, notify user, clean up resources
- **Exit**: Terminal state
- **Next States**: None (terminal)

#### FAILED
- **Entry**: Workflow failed due to validation error, user rejection, or critical step failure
- **Actions**: Log failure reason, attempt rollback if applicable, notify user
- **Exit**: Terminal state
- **Next States**: None (terminal)

#### DEGRADED
- **Entry**: Workflow partially completed with non-critical step failures
- **Actions**: Log partial completion, notify user of failed steps, preserve successful work
- **Exit**: Terminal state
- **Next States**: None (terminal)

### 4.3 State Transition Table

| Current State         | Event                          | Next State            | Actions                                    |
|-----------------------|--------------------------------|-----------------------|--------------------------------------------|
| RECEIVED              | Plan generated                 | PLANNED               | Store plan, invoke validator               |
| PLANNED               | Schema valid                   | VALIDATED             | Invoke guardrail layer                     |
| PLANNED               | Schema invalid                 | FAILED                | Log error, notify user                     |
| VALIDATED             | High-risk detected             | AWAITING_CONFIRMATION | Send confirmation request                  |
| VALIDATED             | No high-risk                   | EXECUTING             | Start execution engine                     |
| AWAITING_CONFIRMATION | User approves                  | EXECUTING             | Start execution engine                     |
| AWAITING_CONFIRMATION | User rejects                   | FAILED                | Log rejection, notify user                 |
| AWAITING_CONFIRMATION | Timeout (60s)                  | FAILED                | Log timeout, notify user                   |
| EXECUTING             | All steps succeed              | COMPLETED             | Log success, clean up                      |
| EXECUTING             | Critical step fails            | FAILED                | Attempt rollback, log failure              |
| EXECUTING             | Non-critical steps fail        | DEGRADED              | Log partial success                        |

### 4.4 State Persistence Schema (DynamoDB)

```json
{
  "workflow_id": "UUID",
  "current_state": "EXECUTING",
  "state_history": [
    {
      "state": "RECEIVED",
      "timestamp": "2026-02-15T10:30:00Z",
      "reason": "User submitted request"
    },
    {
      "state": "PLANNED",
      "timestamp": "2026-02-15T10:30:02Z",
      "reason": "LLM generated plan"
    },
    {
      "state": "VALIDATED",
      "timestamp": "2026-02-15T10:30:03Z",
      "reason": "Schema validation passed"
    },
    {
      "state": "EXECUTING",
      "timestamp": "2026-02-15T10:30:04Z",
      "reason": "No confirmation required, starting execution"
    }
  ],
  "current_step": 3,
  "total_steps": 5,
  "updated_at": "2026-02-15T10:30:15Z"
}
```

### 4.5 State Transition Implementation

```python
class WorkflowStateMachine:
    VALID_TRANSITIONS = {
        "RECEIVED": ["PLANNED"],
        "PLANNED": ["VALIDATED", "FAILED"],
        "VALIDATED": ["AWAITING_CONFIRMATION", "EXECUTING"],
        "AWAITING_CONFIRMATION": ["EXECUTING", "FAILED"],
        "EXECUTING": ["COMPLETED", "FAILED", "DEGRADED"],
        "COMPLETED": [],
        "FAILED": [],
        "DEGRADED": []
    }
    
    def transition(self, workflow_id: str, from_state: str, to_state: str, reason: str):
        # Validate transition
        if to_state not in self.VALID_TRANSITIONS.get(from_state, []):
            raise InvalidStateTransitionError(
                f"Cannot transition from {from_state} to {to_state}"
            )
        
        # Persist state change
        self.db.update_workflow_state(
            workflow_id=workflow_id,
            new_state=to_state,
            timestamp=datetime.utcnow().isoformat(),
            reason=reason
        )
        
        # Log to CloudWatch
        logger.info(f"Workflow {workflow_id}: {from_state} → {to_state} ({reason})")
        
        # Notify frontend
        self.notify_state_change(workflow_id, to_state)
```

## 5. Task Planning Architecture

### 5.1 Task Planner Lambda Design
**Input**: Natural language workflow description + user context (platform, working directory)
**Output**: Structured JSON execution plan conforming to schema (Section 3)

**Critical Constraint**: LLM has NO direct execution capabilities. It only generates plans.

**Processing Steps**:
1. Receive user request via API Gateway
2. Detect user's platform (macOS, Linux, Windows)
3. Construct platform-aware LLM prompt with workflow context
4. Invoke Bedrock/SageMaker with prompt
5. Parse LLM response into structured JSON
6. Perform basic JSON parsing validation
7. Return plan to Plan Validator Lambda (NOT to execution)

### 5.2 LLM Prompt Engineering

**System Prompt**:
```
You are a developer workflow automation assistant. Generate structured execution plans for OS-level tasks.

CRITICAL RULES:
- You CANNOT execute commands directly
- You can ONLY generate JSON plans
- Break workflows into atomic steps
- Use only allowed action types: terminal_command, open_app, write_file, browser_action
- Mark destructive actions with requires_confirmation: true
- Estimate realistic timeouts (max 300 seconds per step)
- Specify step dependencies for correct ordering
- NEVER include sudo/admin commands
- Keep file operations within project directory
- Generate platform-specific commands based on target OS

Platform-Specific Guidelines:
- macOS/Linux: Use bash/zsh, forward slashes (/), standard Unix commands
- Windows: Use PowerShell or cmd, backslashes (\), Windows-compatible commands

Output ONLY valid JSON conforming to the task plan schema.
```

**User Prompt Template**:
```
Task: {user_description}
Platform: {detected_platform}
Working Directory: {cwd}
Available Tools: Terminal, VS Code, Browser
Shell: {default_shell}

Generate a JSON execution plan following the schema. Ensure all commands are compatible with {detected_platform}.
```

### 5.3 Platform Detection
```python
def detect_platform() -> str:
    import platform
    system = platform.system()
    if system == "Darwin":
        return "macOS"
    elif system == "Linux":
        return "Linux"
    elif system == "Windows":
        return "Windows"
    else:
        raise UnsupportedPlatformError(f"Platform {system} not supported")
```

### 5.4 Response Caching Strategy
To reduce costs and latency:
- Cache LLM responses for common workflow patterns
- Use workflow description hash as cache key
- Cache TTL: 24 hours
- Invalidate cache on model version change
- Store in DynamoDB or ElastiCache

```python
def get_cached_plan(workflow_hash: str) -> Optional[dict]:
    cache_key = f"plan:{workflow_hash}"
    cached = redis_client.get(cache_key)
    if cached:
        logger.info(f"Cache hit for workflow {workflow_hash}")
        return json.loads(cached)
    return None

def cache_plan(workflow_hash: str, plan: dict):
    cache_key = f"plan:{workflow_hash}"
    redis_client.setex(cache_key, 86400, json.dumps(plan))  # 24 hour TTL
```

## 6. Plan Validator Lambda

### 6.1 Purpose
Validate JSON schema compliance BEFORE guardrail checks. This is a separate layer from guardrails.

**Input**: JSON plan from Task Planner
**Output**: Validation result (pass/fail) + error details

### 6.2 Validation Steps
1. Validate JSON structure against schema (Section 3.1)
2. Check all required fields are present
3. Validate enum values (platform, action_type, shell, etc.)
4. Validate data types and ranges
5. Check for circular dependencies in steps
6. Validate timeout_seconds <= 300
7. Validate retry_count <= 3
8. Validate working_directory is absolute path
9. Validate platform matches user's OS
10. Return validation result

### 6.3 Implementation
```python
import jsonschema

class PlanValidator:
    def __init__(self, schema: dict):
        self.schema = schema
        self.validator = jsonschema.Draft7Validator(schema)
    
    def validate(self, plan: dict) -> ValidationResult:
        errors = []
        
        # Schema validation
        for error in self.validator.iter_errors(plan):
            errors.append(f"Schema error: {error.message} at {'.'.join(error.path)}")
        
        # Dependency cycle check
        if self.has_circular_dependencies(plan["steps"]):
            errors.append("Circular dependencies detected in steps")
        
        # Platform validation
        if plan["platform"] != self.detected_platform:
            errors.append(f"Platform mismatch: plan is for {plan['platform']}, user is on {self.detected_platform}")
        
        return ValidationResult(
            valid=len(errors) == 0,
            errors=errors
        )
    
    def has_circular_dependencies(self, steps: list) -> bool:
        # Build dependency graph
        graph = {step["step_id"]: step.get("dependencies", []) for step in steps}
        
        # DFS cycle detection
        visited = set()
        rec_stack = set()
        
        def has_cycle(node):
            visited.add(node)
            rec_stack.add(node)
            
            for neighbor in graph.get(node, []):
                if neighbor not in visited:
                    if has_cycle(neighbor):
                        return True
                elif neighbor in rec_stack:
                    return True
            
            rec_stack.remove(node)
            return False
        
        for node in graph:
            if node not in visited:
                if has_cycle(node):
                    return True
        
        return False
```

## 7. Guardrail Layer Lambda

### 7.1 Purpose
Enforce safety constraints AFTER schema validation. This is a separate layer from validation.

**Input**: Schema-valid JSON plan from Plan Validator
**Output**: Guardrail result (approved/rejected) + risk assessment

### 7.2 Guardrail Checks
1. **Command Safety**: Check all terminal commands against whitelist/blacklist
2. **Path Validation**: Ensure all file paths are within project directory
3. **Confirmation Requirements**: Flag destructive actions
4. **Resource Limits**: Validate timeouts and step counts
5. **Platform-Specific Safety**: Apply platform-specific restrictions

### 7.3 Command Whitelist/Blacklist

#### Cross-Platform Allowed Patterns
```python
ALLOWED_COMMANDS = [
    r"^npm\s+(install|run|create|init|start|build|test)",
    r"^yarn\s+(install|add|run|start|build|test)",
    r"^pnpm\s+(install|add|run|start|build|test)",
    r"^git\s+(init|add|commit|status|log|diff|clone)",
    r"^node\s+",
    r"^python\s+",
    r"^python3\s+",
    r"^java\s+",
    r"^mkdir\s+",
    r"^touch\s+",
    r"^cp\s+",
    r"^mv\s+",
    r"^code\s+",
    r"^curl\s+",
    r"^wget\s+",
]
```

#### Unix-like (macOS, Linux) Blocked Patterns
```python
UNIX_BLOCKED_COMMANDS = [
    r"sudo",
    r"su\s",
    r"rm\s+-rf\s+/",
    r"chmod\s+777",
    r"chown",
    r"apt-get",
    r"yum",
    r"systemctl",
    r"service\s",
]

UNIX_BLOCKED_PATHS = [
    "/etc", "/usr", "/bin", "/sbin", "/var", "/sys", "/proc", "/boot"
]
```

#### Windows Blocked Patterns
```python
WINDOWS_BLOCKED_COMMANDS = [
    r"Set-ExecutionPolicy",
    r"reg\s+(add|delete)",
    r"regedit",
    r"net\s+user",
    r"net\s+localgroup",
    r"format\s",
    r"diskpart",
    r"del\s+/f\s+/s\s+/q",
    r"rmdir\s+/s\s+/q\s+C:\\",
]

WINDOWS_BLOCKED_PATHS = [
    "C:\\Windows", "C:\\Program Files", "C:\\Program Files (x86)", "C:\\System32"
]
```

### 7.4 Guardrail Implementation
```python
class GuardrailLayer:
    def __init__(self, platform: str, project_dir: str):
        self.platform = platform
        self.project_dir = os.path.abspath(project_dir)
        self.sandbox = FileSystemSandbox(project_dir)
    
    def validate_plan(self, plan: dict) -> GuardrailResult:
        errors = []
        warnings = []
        high_risk_steps = []
        
        for step in plan["steps"]:
            # Check command safety
            if step["action_type"] == "terminal_command":
                if not self.is_safe_command(step["command"]):
                    errors.append(f"Step {step['step_id']}: Unsafe command blocked: {step['command']}")
                
                if self.is_high_risk_command(step["command"]):
                    high_risk_steps.append(step["step_id"])
                    if not step.get("requires_confirmation"):
                        warnings.append(f"Step {step['step_id']}: High-risk command should require confirmation")
            
            # Check file paths
            if step["action_type"] == "write_file":
                if not self.sandbox.validate_path(step["file_path"]):
                    errors.append(f"Step {step['step_id']}: File path outside project: {step['file_path']}")
            
            # Check for destructive actions
            if self.is_destructive(step):
                if not step.get("requires_confirmation"):
                    errors.append(f"Step {step['step_id']}: Destructive action must require confirmation")
        
        return GuardrailResult(
            approved=len(errors) == 0,
            errors=errors,
            warnings=warnings,
            risk_level=self.assess_risk(plan, high_risk_steps),
            requires_confirmation=len(high_risk_steps) > 0
        )
    
    def is_safe_command(self, command: str) -> bool:
        # Check against blocked patterns
        blocked = WINDOWS_BLOCKED_COMMANDS if self.platform == "Windows" else UNIX_BLOCKED_COMMANDS
        for pattern in blocked:
            if re.search(pattern, command, re.IGNORECASE):
                return False
        
        # Check against allowed patterns
        for pattern in ALLOWED_COMMANDS:
            if re.match(pattern, command):
                return True
        
        # Default deny
        return False
    
    def is_high_risk_command(self, command: str) -> bool:
        high_risk_patterns = [
            r"git\s+push",
            r"rm\s+",
            r"del\s+",
            r"curl.*\|\s*bash",
            r"wget.*\|\s*sh",
        ]
        return any(re.search(pattern, command, re.IGNORECASE) for pattern in high_risk_patterns)
    
    def is_destructive(self, step: dict) -> bool:
        if step["action_type"] == "terminal_command":
            destructive_patterns = [r"rm\s+", r"del\s+", r"git\s+push"]
            return any(re.search(pattern, step["command"], re.IGNORECASE) for pattern in destructive_patterns)
        return False
    
    def assess_risk(self, plan: dict, high_risk_steps: list) -> str:
        if len(high_risk_steps) > 0:
            return "high"
        elif any(step["action_type"] == "write_file" for step in plan["steps"]):
            return "medium"
        else:
            return "low"
```

## 8. Platform Abstraction Layer

### 8.1 Purpose
Provide cross-platform compatibility for command execution, file operations, and path handling across macOS, Linux, and Windows.

### 8.2 Command Translator
```python
class CommandTranslator:
    def __init__(self, platform: str):
        self.platform = platform
    
    def get_default_shell(self) -> str:
        if self.platform == "Windows":
            return "powershell"
        elif self.platform == "macOS":
            return "zsh"
        else:  # Linux
            return "bash"
    
    def translate_path(self, path: str) -> str:
        """Normalize path for target platform"""
        if self.platform == "Windows":
            return path.replace("/", "\\")
        else:
            return path.replace("\\", "/")
    
    def get_path_separator(self) -> str:
        return "\\" if self.platform == "Windows" else "/"
    
    def normalize_command(self, command: str, shell: str) -> str:
        """Adapt command for target shell"""
        if shell == "powershell" and self.platform == "Windows":
            # PowerShell-specific adaptations
            return command
        elif shell == "cmd" and self.platform == "Windows":
            # CMD-specific adaptations
            return command
        else:
            # Unix shell (bash/zsh)
            return command
```

### 8.3 Path Normalizer
```python
class PathNormalizer:
    def __init__(self, platform: str, project_dir: str):
        self.platform = platform
        self.project_dir = os.path.abspath(project_dir)
    
    def normalize(self, path: str) -> str:
        """Convert path to absolute, normalized form"""
        # Handle relative paths
        if not os.path.isabs(path):
            path = os.path.join(self.project_dir, path)
        
        # Normalize separators and resolve ../ 
        path = os.path.normpath(path)
        
        # Convert to absolute
        path = os.path.abspath(path)
        
        return path
    
    def is_within_project(self, path: str) -> bool:
        """Check if path is within project directory"""
        normalized = self.normalize(path)
        return normalized.startswith(self.project_dir)
    
    def is_system_directory(self, path: str) -> bool:
        """Check if path is a system directory"""
        normalized = self.normalize(path)
        
        if self.platform == "Windows":
            system_dirs = ["C:\\Windows", "C:\\Program Files", "C:\\Program Files (x86)"]
            return any(normalized.startswith(d) for d in system_dirs)
        else:
            system_dirs = ["/etc", "/usr", "/bin", "/sbin", "/var", "/sys", "/proc", "/boot"]
            return any(normalized.startswith(d) for d in system_dirs)
```

### 8.4 Platform Detector
```python
class PlatformDetector:
    @staticmethod
    def detect() -> str:
        import platform
        system = platform.system()
        if system == "Darwin":
            return "macOS"
        elif system == "Linux":
            return "Linux"
        elif system == "Windows":
            return "Windows"
        else:
            raise UnsupportedPlatformError(f"Platform {system} not supported")
    
    @staticmethod
    def get_shell_info() -> dict:
        platform_name = PlatformDetector.detect()
        if platform_name == "Windows":
            return {
                "platform": "Windows",
                "default_shell": "powershell",
                "available_shells": ["powershell", "cmd"]
            }
        elif platform_name == "macOS":
            return {
                "platform": "macOS",
                "default_shell": "zsh",
                "available_shells": ["zsh", "bash"]
            }
        else:  # Linux
            return {
                "platform": "Linux",
                "default_shell": "bash",
                "available_shells": ["bash", "sh"]
            }
```

## 9. Execution Engine Design

### 9.1 Local Agent Architecture
The execution engine runs as a local agent on the user's machine, receiving validated plans from AWS and executing them safely with cross-platform support.

**Components**:
- **Plan Receiver**: Polls API for new validated plans
- **Platform Abstraction Layer**: Handles cross-platform compatibility
- **Step Executor**: Executes individual steps based on action type
- **State Reporter**: Reports progress back to AWS
- **Error Handler**: Manages failures and retries
- **Rollback Manager**: Handles rollback operations

### 9.2 Action Executors

#### 9.2.1 Terminal Command Executor (Cross-Platform)
```python
class TerminalExecutor:
    def __init__(self, platform: str, project_dir: str):
        self.platform = platform
        self.project_dir = project_dir
        self.translator = CommandTranslator(platform)
    
    def execute(self, step: dict) -> ExecutionResult:
        command = step["command"]
        shell = step.get("shell", self.translator.get_default_shell())
        timeout = step["timeout_seconds"]
        working_dir = step.get("working_directory", self.project_dir)
        
        # Validate command safety (should already be validated, but double-check)
        if not self.is_safe_command(command):
            return ExecutionResult(success=False, error="Command blocked by local safety check")
        
        # Prepare shell-specific execution
        if self.platform == "Windows":
            if shell == "powershell":
                shell_cmd = ["powershell", "-Command", command]
            else:  # cmd
                shell_cmd = ["cmd", "/c", command]
        else:  # Unix-like
            shell_cmd = [shell, "-c", command]
        
        try:
            # Execute with timeout
            result = subprocess.run(
                shell_cmd,
                timeout=timeout,
                capture_output=True,
                cwd=working_dir,
                text=True
            )
            
            return ExecutionResult(
                success=result.returncode == 0,
                stdout=result.stdout,
                stderr=result.stderr,
                exit_code=result.returncode
            )
        except subprocess.TimeoutExpired:
            return ExecutionResult(
                success=False,
                error=f"Command timed out after {timeout} seconds"
            )
        except Exception as e:
            return ExecutionResult(
                success=False,
                error=f"Execution error: {str(e)}"
            )
```

#### 9.2.2 File Write Executor (Cross-Platform)
```python
class FileWriteExecutor:
    def __init__(self, platform: str, project_dir: str):
        self.platform = platform
        self.normalizer = PathNormalizer(platform, project_dir)
        self.backup_dir = os.path.join(project_dir, ".workflow_backups")
    
    def execute(self, step: dict) -> ExecutionResult:
        file_path = step["file_path"]
        content = step["content"]
        
        # Normalize and validate path
        normalized_path = self.normalizer.normalize(file_path)
        
        if not self.normalizer.is_within_project(normalized_path):
            return ExecutionResult(success=False, error="Path outside project directory")
        
        if self.normalizer.is_system_directory(normalized_path):
            return ExecutionResult(success=False, error="Cannot write to system directory")
        
        try:
            # Create backup if file exists
            if os.path.exists(normalized_path):
                self.create_backup(normalized_path)
            
            # Ensure directory exists
            os.makedirs(os.path.dirname(normalized_path), exist_ok=True)
            
            # Write file
            with open(normalized_path, 'w', encoding='utf-8') as f:
                f.write(content)
            
            return ExecutionResult(
                success=True,
                message=f"File written: {normalized_path}"
            )
        except Exception as e:
            return ExecutionResult(
                success=False,
                error=f"File write error: {str(e)}"
            )
    
    def create_backup(self, file_path: str):
        """Create backup for rollback"""
        os.makedirs(self.backup_dir, exist_ok=True)
        backup_name = f"{os.path.basename(file_path)}.{int(time.time())}.bak"
        backup_path = os.path.join(self.backup_dir, backup_name)
        shutil.copy2(file_path, backup_path)
        return backup_path
```

#### 9.2.3 VS Code Executor (Cross-Platform)
```python
class VSCodeExecutor:
    def __init__(self, platform: str):
        self.platform = platform
        self.code_cmd = self.get_code_command()
    
    def get_code_command(self) -> str:
        """Get VS Code CLI command for platform"""
        if self.platform == "Windows":
            return "code.cmd"
        else:
            return "code"
    
    def execute(self, step: dict) -> ExecutionResult:
        app_name = step["app_name"]
        app_args = step.get("app_args", [])
        
        if app_name.lower() != "code":
            return ExecutionResult(success=False, error=f"Unsupported app: {app_name}")
        
        try:
            # Check if VS Code is installed
            result = subprocess.run(
                [self.code_cmd, "--version"],
                capture_output=True,
                timeout=5
            )
            
            if result.returncode != 0:
                return ExecutionResult(
                    success=False,
                    error="VS Code not installed or not in PATH"
                )
            
            # Execute VS Code command
            subprocess.run([self.code_cmd] + app_args, timeout=10)
            
            return ExecutionResult(success=True, message="VS Code opened")
        except Exception as e:
            return ExecutionResult(
                success=False,
                error=f"VS Code execution error: {str(e)}"
            )
```

### 9.3 Step Execution Orchestration
```python
class StepOrchestrator:
    def __init__(self, platform: str, project_dir: str):
        self.platform = platform
        self.project_dir = project_dir
        self.executors = {
            "terminal_command": TerminalExecutor(platform, project_dir),
            "write_file": FileWriteExecutor(platform, project_dir),
            "open_app": VSCodeExecutor(platform),
            "browser_action": BrowserExecutor(platform)
        }
        self.state_reporter = StateReporter()
        self.rollback_manager = RollbackManager(platform, project_dir)
    
    def execute_workflow(self, plan: dict):
        workflow_id = plan["task_id"]
        steps = plan["steps"]
        
        # Sort steps by dependencies (topological sort)
        sorted_steps = self.topological_sort(steps)
        
        completed_steps = []
        
        for step in sorted_steps:
            step_id = step["step_id"]
            
            # Check if user confirmation required
            if step.get("requires_confirmation"):
                if not self.request_confirmation(workflow_id, step):
                    self.state_reporter.update_state(workflow_id, "FAILED", "User rejected confirmation")
                    return
            
            # Execute step with retry
            result = self.execute_step_with_retry(step)
            
            # Log result
            self.state_reporter.log_step_result(workflow_id, step_id, result)
            
            if result.success:
                completed_steps.append(step)
            else:
                # Handle failure
                if self.is_critical_step(step):
                    # Attempt rollback
                    self.rollback_manager.rollback(completed_steps)
                    self.state_reporter.update_state(workflow_id, "FAILED", f"Critical step {step_id} failed")
                    return
                else:
                    # Continue with degraded state
                    self.state_reporter.update_state(workflow_id, "DEGRADED", f"Non-critical step {step_id} failed")
        
        # All steps completed
        self.state_reporter.update_state(workflow_id, "COMPLETED", "All steps succeeded")
    
    def execute_step_with_retry(self, step: dict) -> ExecutionResult:
        retry_count = step.get("retry_count", 0)
        action_type = step["action_type"]
        executor = self.executors[action_type]
        
        for attempt in range(retry_count + 1):
            result = executor.execute(step)
            
            if result.success:
                return result
            
            if attempt < retry_count:
                # Exponential backoff
                time.sleep(2 ** attempt)
        
        return result
    
    def topological_sort(self, steps: list) -> list:
        """Sort steps by dependencies"""
        # Build adjacency list
        graph = {step["step_id"]: step.get("dependencies", []) for step in steps}
        step_map = {step["step_id"]: step for step in steps}
        
        # Kahn's algorithm
        in_degree = {step_id: 0 for step_id in graph}
        for deps in graph.values():
            for dep in deps:
                in_degree[dep] = in_degree.get(dep, 0) + 1
        
        queue = [step_id for step_id, degree in in_degree.items() if degree == 0]
        sorted_steps = []
        
        while queue:
            step_id = queue.pop(0)
            sorted_steps.append(step_map[step_id])
            
            for neighbor in graph[step_id]:
                in_degree[neighbor] -= 1
                if in_degree[neighbor] == 0:
                    queue.append(neighbor)
        
        return sorted_steps
```

## 10. Rollback & Recovery Strategy

### 10.1 Rollback Scope and Limitations
- **Scope**: Limited to project workspace only
- **File Operations**: Snapshot-based rollback supported
- **Git Operations**: Local git operations can be rolled back
- **External Services**: NO rollback for GitHub API, npm registry, etc.
- **User Communication**: Clearly communicate limitations

### 10.2 Rollback Manager Implementation
```python
class RollbackManager:
    def __init__(self, platform: str, project_dir: str):
        self.platform = platform
        self.project_dir = project_dir
        self.backup_dir = os.path.join(project_dir, ".workflow_backups")
        self.rollback_log = []
    
    def rollback(self, completed_steps: list):
        """Rollback completed steps in reverse order"""
        logger.info(f"Starting rollback for {len(completed_steps)} steps")
        
        for step in reversed(completed_steps):
            if "rollback_action" in step:
                self.execute_rollback_action(step)
            else:
                self.auto_rollback(step)
        
        logger.info(f"Rollback completed. Log: {self.rollback_log}")
    
    def auto_rollback(self, step: dict):
        """Automatic rollback based on action type"""
        action_type = step["action_type"]
        
        if action_type == "write_file":
            self.rollback_file_write(step)
        elif action_type == "terminal_command":
            self.rollback_terminal_command(step)
        else:
            self.rollback_log.append(f"No auto-rollback for {action_type}")
    
    def rollback_file_write(self, step: dict):
        """Rollback file write operation"""
        file_path = step["file_path"]
        
        # Find backup
        backup_files = glob.glob(os.path.join(self.backup_dir, f"{os.path.basename(file_path)}.*.bak"))
        
        if backup_files:
            # Restore most recent backup
            latest_backup = max(backup_files, key=os.path.getctime)
            shutil.copy2(latest_backup, file_path)
            self.rollback_log.append(f"Restored {file_path} from backup")
        else:
            # File was created, delete it
            if os.path.exists(file_path):
                os.remove(file_path)
                self.rollback_log.append(f"Deleted created file {file_path}")
    
    def rollback_terminal_command(self, step: dict):
        """Rollback terminal command if possible"""
        command = step["command"]
        
        # Git operations
        if "git commit" in command:
            subprocess.run(["git", "reset", "--soft", "HEAD~1"], cwd=self.project_dir)
            self.rollback_log.append("Rolled back git commit")
        elif "git add" in command:
            subprocess.run(["git", "reset", "HEAD"], cwd=self.project_dir)
            self.rollback_log.append("Rolled back git add")
        else:
            self.rollback_log.append(f"No rollback available for command: {command}")
    
    def execute_rollback_action(self, step: dict):
        """Execute explicit rollback action"""
        rollback = step["rollback_action"]
        action_type = rollback.get("action_type")
        
        if action_type == "restore_file":
            self.rollback_file_write(step)
        elif action_type == "terminal_command":
            command = rollback.get("command")
            subprocess.run(command, shell=True, cwd=self.project_dir)
            self.rollback_log.append(f"Executed rollback command: {command}")
```

### 10.3 Recovery Strategy
```python
class RecoveryManager:
    def __init__(self):
        self.max_retries = 3
    
    def retry_with_backoff(self, func, *args, **kwargs):
        """Retry function with exponential backoff"""
        for attempt in range(self.max_retries):
            try:
                result = func(*args, **kwargs)
                if result.success:
                    return result
            except Exception as e:
                logger.error(f"Attempt {attempt + 1} failed: {str(e)}")
            
            if attempt < self.max_retries - 1:
                time.sleep(2 ** attempt)
        
        return ExecutionResult(success=False, error="Max retries exceeded")
    
    def handle_partial_completion(self, workflow_id: str, completed_steps: list, failed_step: dict):
        """Handle workflows that partially completed"""
        # Log partial completion
        logger.info(f"Workflow {workflow_id} partially completed: {len(completed_steps)} steps succeeded")
        
        # Offer user options:
        # 1. Retry from failed step
        # 2. Rollback all changes
        # 3. Accept partial completion
        
        return {
            "status": "DEGRADED",
            "completed_steps": len(completed_steps),
            "failed_step": failed_step["step_id"],
            "options": ["retry", "rollback", "accept"]
        }
```

### 4.1 Local Agent Architecture
The execution engine runs as a local agent on the user's machine, receiving validated plans from AWS and executing them safely.

**Components**:
- **Plan Receiver**: Polls API for new validated plans
- **Step Executor**: Executes individual steps based on action type
- **State Reporter**: Reports progress back to AWS
- **Error Handler**: Manages failures and retries

### 4.2 Action Executors

#### 4.2.1 Terminal Command Executor
```python
class TerminalExecutor:
    def execute(self, command: str, timeout: int) -> ExecutionResult:
        # Validate command against whitelist
        if not self.is_safe_command(command):
            return ExecutionResult(success=False, error="Command blocked")
        
        # Execute with timeout
        result = subprocess.run(
            command,
            shell=True,
            timeout=timeout,
            capture_output=True,
            cwd=self.project_dir
        )
        
        return ExecutionResult(
            success=result.returncode == 0,
            stdout=result.stdout,
            stderr=result.stderr
        )
```

#### 4.2.2 File Writer Executor
```python
class FileWriteExecutor:
    def execute(self, file_path: str, content: str) -> ExecutionResult:
        # Validate path is within project directory
        abs_path = os.path.abspath(file_path)
        if not abs_path.startswith(self.project_dir):
            return ExecutionResult(success=False, error="Path outside project")
        
        # Write file
        with open(abs_path, 'w') as f:
            f.write(content)
        
        return ExecutionResult(success=True)
```

#### 4.2.3 VS Code Executor
```python
class VSCodeExecutor:
    def execute(self, action: str, args: dict) -> ExecutionResult:
        # Use VS Code CLI
        if action == "open_file":
            subprocess.run(["code", args["file_path"]])
        elif action == "open_folder":
            subprocess.run(["code", args["folder_path"]])
        
        return ExecutionResult(success=True)
```

#### 4.2.4 Browser/GitHub Executor
```python
class BrowserExecutor:
    def execute(self, action: str, args: dict) -> ExecutionResult:
        # Use Selenium or Playwright for browser automation
        driver = self.get_driver()
        
        if action == "create_repo":
            driver.get("https://github.com/new")
            # Fill form and submit
        
        return ExecutionResult(success=True)
```

### 4.3 Execution Flow
1. Receive validated plan from API
2. For each step in order:
   - Check dependencies are complete
   - If requires_confirmation, prompt user
   - Execute step with appropriate executor
   - Log result to DynamoDB
   - Report progress to frontend
   - If failure, retry up to retry_count
3. Mark workflow as complete or failed

## 11. AWS Integration Architecture

### 11.1 Cost Management Strategy

**Target Costs**:
- MVP operational cost: < $150/month during active usage
- Cost per workflow execution: < $0.20

**Cost Breakdown Estimate**:
- Amazon Bedrock (Claude 3 Sonnet): ~$80/month (400 workflows @ $0.20 each)
- Lambda (3 functions): ~$20/month
- API Gateway: ~$10/month
- DynamoDB: ~$15/month (on-demand)
- CloudWatch Logs: ~$10/month
- Amplify Hosting: ~$5/month
- Cognito: ~$5/month (< 50 users)
- X-Ray: ~$5/month

**Cost Controls**:
1. **Bedrock Response Caching**: Cache common workflow patterns for 24 hours
2. **Lambda Memory Optimization**: Tune memory allocation for cost/performance
3. **DynamoDB TTL**: Auto-delete logs after 30 days
4. **CloudWatch Log Retention**: 7 days for most logs
5. **AWS Budgets**: Alerts at 80% ($120) and 100% ($150) thresholds

```python
# Bedrock caching implementation
class BedrockClient:
    def __init__(self):
        self.client = boto3.client('bedrock-runtime')
        self.cache = {}
        self.cache_ttl = 86400  # 24 hours
    
    def invoke_with_cache(self, prompt: str) -> str:
        cache_key = hashlib.sha256(prompt.encode()).hexdigest()
        
        # Check cache
        if cache_key in self.cache:
            cached_data = self.cache[cache_key]
            if time.time() - cached_data['timestamp'] < self.cache_ttl:
                logger.info(f"Cache hit for prompt hash {cache_key[:8]}")
                return cached_data['response']
        
        # Invoke Bedrock
        response = self.client.invoke_model(
            modelId='anthropic.claude-3-sonnet-20240229-v1:0',
            body=json.dumps({
                "anthropic_version": "bedrock-2023-05-31",
                "max_tokens": 4096,
                "messages": [{"role": "user", "content": prompt}]
            })
        )
        
        result = json.loads(response['body'].read())
        response_text = result['content'][0]['text']
        
        # Cache response
        self.cache[cache_key] = {
            'response': response_text,
            'timestamp': time.time()
        }
        
        return response_text
```

### 11.2 AWS Budgets Configuration
```python
import boto3

def setup_budget_alerts():
    budgets_client = boto3.client('budgets')
    
    budgets_client.create_budget(
        AccountId='123456789012',
        Budget={
            'BudgetName': 'WorkflowCopilotMVP',
            'BudgetLimit': {
                'Amount': '150',
                'Unit': 'USD'
            },
            'TimeUnit': 'MONTHLY',
            'BudgetType': 'COST'
        },
        NotificationsWithSubscribers=[
            {
                'Notification': {
                    'NotificationType': 'ACTUAL',
                    'ComparisonOperator': 'GREATER_THAN',
                    'Threshold': 80,
                    'ThresholdType': 'PERCENTAGE'
                },
                'Subscribers': [
                    {
                        'SubscriptionType': 'EMAIL',
                        'Address': '[email]'
                    }
                ]
            },
            {
                'Notification': {
                    'NotificationType': 'ACTUAL',
                    'ComparisonOperator': 'GREATER_THAN',
                    'Threshold': 100,
                    'ThresholdType': 'PERCENTAGE'
                },
                'Subscribers': [
                    {
                        'SubscriptionType': 'EMAIL',
                        'Address': '[email]'
                    }
                ]
            }
        ]
    )
```

### 5.1 Command Whitelist
Allowed command patterns:
- `npm install`, `npm run *`, `npm create *`
- `git init`, `git add`, `git commit`, `git push` (with confirmation)
- `mkdir`, `touch`, `cp`, `mv` (within project dir)
- `code` (VS Code CLI)
- `curl`, `wget` (for package downloads)

Blocked patterns:
- `sudo *`
- `rm -rf /`
- `chmod 777`
- System directory access (`/etc`, `/usr`, `/bin`)

### 5.2 File System Sandbox
```python
class FileSystemSandbox:
    def __init__(self, project_dir: str):
        self.project_dir = os.path.abspath(project_dir)
        self.allowed_dirs = [self.project_dir]
    
    def validate_path(self, path: str) -> bool:
        abs_path = os.path.abspath(path)
        return any(abs_path.startswith(d) for d in self.allowed_dirs)
    
    def safe_delete(self, path: str) -> bool:
        if not self.validate_path(path):
            raise SecurityError("Path outside sandbox")
        os.remove(path)
        return True
```

### 5.3 Process Management
- All subprocesses run with timeout
- Resource limits (CPU, memory) enforced
- Orphaned processes cleaned up
- Signal handling for graceful shutdown

## 6. Safety & Guardrail Layer

### 6.1 Guardrail Validator Lambda
**Input**: Structured execution plan
**Output**: Validation result with risk assessment

**Validation Checks**:
1. **Command Safety**: Check all terminal commands against whitelist
2. **Path Validation**: Ensure all file paths are within project directory
3. **Confirmation Requirements**: Flag destructive actions
4. **Resource Limits**: Validate timeouts and step counts
5. **Dependency Validation**: Check for circular dependencies

```python
class GuardrailValidator:
    def validate_plan(self, plan: dict) -> ValidationResult:
        errors = []
        warnings = []
        
        for step in plan["steps"]:
            # Check command safety
            if step["action_type"] == "terminal_command":
                if not self.is_safe_command(step["command"]):
                    errors.append(f"Unsafe command: {step['command']}")
            
            # Check file paths
            if step["action_type"] == "write_file":
                if not self.is_safe_path(step["file_path"]):
                    errors.append(f"Unsafe path: {step['file_path']}")
            
            # Check for destructive actions
            if self.is_destructive(step):
                if not step.get("requires_confirmation"):
                    warnings.append(f"Step {step['step_id']} should require confirmation")
        
        return ValidationResult(
            valid=len(errors) == 0,
            errors=errors,
            warnings=warnings,
            risk_level=self.assess_risk(plan)
        )
```

### 6.2 Risk Assessment
**Low Risk**: Read-only operations, safe commands
**Medium Risk**: File writes, package installations
**High Risk**: Git push, file deletion, network operations

### 6.3 User Confirmation Flow
For high-risk actions:
1. Execution engine pauses before step
2. Display action details to user
3. Request explicit confirmation (yes/no)
4. Log user decision
5. Proceed or skip based on response

## 7. State & Memory Management

### 7.1 DynamoDB Schema

#### 7.1.1 Workflows Table
```
Partition Key: workflow_id (String)
Sort Key: timestamp (Number)

Attributes:
- user_id: String
- description: String
- status: String (pending, in_progress, completed, failed)
- plan: Map (JSON plan)
- created_at: Number
- updated_at: Number
- completed_at: Number
- error_message: String
```

#### 7.1.2 Execution Logs Table
```
Partition Key: workflow_id (String)
Sort Key: step_id (String)

Attributes:
- action_type: String
- command: String
- status: String (pending, running, completed, failed)
- stdout: String
- stderr: String
- started_at: Number
- completed_at: Number
- retry_count: Number
```

#### 7.1.3 User Preferences Table
```
Partition Key: user_id (String)

Attributes:
- default_project_dir: String
- confirmation_preferences: Map
- workflow_history: List
- created_at: Number
```

### 7.2 State Transitions
```
Workflow States:
pending → in_progress → completed
                     → failed

Step States:
pending → running → completed
                 → failed → retrying → running
```

### 7.3 TTL Configuration
- Execution logs: 30 days
- Completed workflows: 90 days
- Failed workflows: 180 days (for debugging)

## 8. AWS Integration Architecture

### 8.1 API Gateway Configuration
**Endpoints**:
- `POST /workflows` - Submit new workflow
- `GET /workflows/{id}` - Get workflow status
- `GET /workflows/{id}/logs` - Get execution logs
- `POST /workflows/{id}/confirm` - Confirm pending action
- `DELETE /workflows/{id}` - Cancel workflow

**Authentication**: Cognito User Pool authorizer
**Rate Limiting**: 100 requests per minute per user
**CORS**: Enabled for Amplify frontend domain

### 8.2 Lambda Functions

#### 8.2.1 Task Planner Lambda
- **Runtime**: Python 3.11
- **Memory**: 512 MB
- **Timeout**: 30 seconds
- **Environment Variables**:
  - `BEDROCK_MODEL_ID` or `SAGEMAKER_ENDPOINT`
  - `DYNAMODB_TABLE_NAME`
  - `LOG_LEVEL`

#### 8.2.2 Guardrail Validator Lambda
- **Runtime**: Python 3.11
- **Memory**: 256 MB
- **Timeout**: 10 seconds
- **Layers**: Command whitelist configuration

### 8.3 Bedrock/SageMaker Integration
**Option 1: Amazon Bedrock**
- Model: Claude 3 Sonnet or GPT-4 equivalent
- API: `bedrock-runtime.invoke_model()`
- Benefits: Fully managed, no infrastructure

**Option 2: SageMaker Endpoint**
- Model: Custom fine-tuned model
- API: `sagemaker-runtime.invoke_endpoint()`
- Benefits: More control, potential cost savings

### 8.4 CloudWatch Integration
**Log Groups**:
- `/aws/lambda/task-planner`
- `/aws/lambda/guardrail-validator`
- `/aws/apigateway/workflow-api`

**Metrics**:
- Workflow success rate
- Average execution time
- Error rate by type
- API latency

**Alarms**:
- Error rate > 10%
- API latency > 3 seconds
- Lambda throttling

### 8.5 X-Ray Tracing
Enable tracing for:
- API Gateway requests
- Lambda invocations
- DynamoDB operations
- Bedrock/SageMaker calls

## 9. Failure Handling Strategy

### 9.1 Failure Categories
1. **Planning Failures**: LLM errors, invalid JSON
2. **Validation Failures**: Unsafe commands, policy violations
3. **Execution Failures**: Command errors, timeouts
4. **Infrastructure Failures**: AWS service outages

### 9.2 Retry Logic
```python
class RetryHandler:
    def execute_with_retry(self, step: dict, max_retries: int = 2):
        for attempt in range(max_retries + 1):
            try:
                result = self.execute_step(step)
                if result.success:
                    return result
            except Exception as e:
                if attempt == max_retries:
                    return ExecutionResult(
                        success=False,
                        error=f"Failed after {max_retries} retries: {str(e)}"
                    )
                time.sleep(2 ** attempt)  # Exponential backoff
```

### 9.3 Rollback Mechanism
For critical failures:
1. Identify completed steps
2. Generate reverse operations (if possible)
3. Execute rollback steps
4. Log rollback results

Example: If `git push` fails, rollback might include `git reset --hard HEAD~1`

### 9.4 Error Reporting
```json
{
  "workflow_id": "uuid",
  "status": "failed",
  "failed_step": {
    "step_id": "3",
    "action_type": "terminal_command",
    "command": "npm install",
    "error": "Network timeout"
  },
  "completed_steps": ["1", "2"],
  "rollback_performed": false,
  "user_action_required": "Check network connection and retry"
}
```

## 10. Scalability Considerations

### 10.1 Horizontal Scaling
- **API Gateway**: Auto-scales with traffic
- **Lambda**: Concurrent execution limit (default 1000)
- **DynamoDB**: On-demand capacity mode for MVP
- **Execution Engine**: One agent per user machine

### 10.2 Performance Optimization
- **Caching**: Cache LLM responses for common workflows
- **Connection Pooling**: Reuse DynamoDB connections
- **Async Processing**: Use SQS for long-running workflows
- **CDN**: CloudFront for static frontend assets

### 10.3 Cost Optimization
- **Lambda**: Use ARM architecture (Graviton2)
- **DynamoDB**: Use on-demand pricing for MVP
- **Bedrock**: Batch requests when possible
- **CloudWatch**: Set log retention to 7 days

### 10.4 Monitoring & Alerts
**Key Metrics**:
- Workflow throughput (workflows/hour)
- Success rate (%)
- P50, P95, P99 latency
- Error rate by category
- Cost per workflow

**Dashboards**:
- Real-time workflow status
- System health overview
- Cost tracking
- User activity

## 11. Correctness Properties

### 11.1 Safety Properties
**Property 11.1.1**: No sudo command shall ever be executed
- **Test Strategy**: Generate random commands including sudo variants, verify all are blocked

**Property 11.1.2**: All file operations must be within project directory
- **Test Strategy**: Generate random file paths including parent directory traversal, verify all outside paths are rejected

**Property 11.1.3**: Destructive actions must require confirmation
- **Test Strategy**: Generate plans with destructive actions, verify requires_confirmation flag is set

### 11.2 Correctness Properties
**Property 11.2.1**: All generated plans must conform to JSON schema
- **Test Strategy**: Generate plans from various inputs, validate against schema

**Property 11.2.2**: Step dependencies must form a valid DAG (no cycles)
- **Test Strategy**: Generate plans with various dependency structures, verify no cycles exist

**Property 11.2.3**: Execution order must respect dependencies
- **Test Strategy**: Generate plans with dependencies, verify steps execute in valid topological order

### 11.3 Liveness Properties
**Property 11.3.1**: All workflows must eventually complete or fail (no infinite loops)
- **Test Strategy**: Execute workflows with various configurations, verify all terminate within timeout

**Property 11.3.2**: Failed steps with retry_count > 0 must be retried
- **Test Strategy**: Inject failures, verify retry logic executes correct number of times

### 11.4 Idempotency Properties
**Property 11.4.1**: Executing the same plan twice should produce consistent results
- **Test Strategy**: Execute same plan multiple times, verify outcomes are equivalent

## 12. Testing Framework

### 12.1 Property-Based Testing Setup
**Framework**: Hypothesis (Python)

**Test Structure**:
```python
from hypothesis import given, strategies as st
import pytest

@given(st.text())
def test_no_sudo_commands_allowed(command):
    """Property 11.1.1: No sudo command shall ever be executed"""
    validator = GuardrailValidator()
    if "sudo" in command.lower():
        assert not validator.is_safe_command(command)

@given(st.text())
def test_file_paths_within_project(file_path):
    """Property 11.1.2: All file operations must be within project directory"""
    sandbox = FileSystemSandbox("/project")
    if not file_path.startswith("/project"):
        assert not sandbox.validate_path(file_path)
```

### 12.2 Unit Testing
- Test each executor independently
- Mock AWS services (moto library)
- Test guardrail validation logic
- Test state transitions

### 12.3 Integration Testing
- End-to-end workflow execution
- AWS service integration
- Frontend-backend communication
- Authentication flow

### 12.4 Load Testing
- Simulate 20 concurrent users
- Measure API latency under load
- Verify no resource exhaustion
- Test auto-scaling behavior

## 13. Deployment Strategy

### 13.1 Infrastructure as Code
Use AWS CDK or Terraform for:
- Lambda functions
- API Gateway
- DynamoDB tables
- Cognito user pool
- CloudWatch dashboards
- IAM roles and policies

### 13.2 CI/CD Pipeline
1. Code commit triggers build
2. Run unit tests
3. Run integration tests
4. Deploy to staging environment
5. Run smoke tests
6. Manual approval
7. Deploy to production

### 13.3 Monitoring & Rollback
- Monitor error rates post-deployment
- Automatic rollback if error rate > 15%
- Canary deployment for high-risk changes

## 14. Security Considerations

### 14.1 Authentication & Authorization
- Cognito for user management
- JWT tokens for API authentication
- Role-based access control (RBAC)
- API key rotation every 90 days

### 14.2 Data Protection
- Encrypt DynamoDB tables at rest
- Use HTTPS for all API communication
- Sanitize logs to remove sensitive data
- Implement data retention policies

### 14.3 Audit & Compliance
- Log all user actions
- Track command execution history
- Implement audit trail for compliance
- Regular security reviews

## 15. MVP Implementation Phases

### Phase 1: Core Infrastructure (Week 1-2)
- Set up AWS account and services
- Deploy Lambda functions
- Configure API Gateway
- Set up DynamoDB tables
- Implement authentication

### Phase 2: Task Planning (Week 3-4)
- Integrate Bedrock/SageMaker
- Implement prompt engineering
- Build guardrail validator
- Test plan generation

### Phase 3: Execution Engine (Week 5-6)
- Build local agent
- Implement executors
- Add safety checks
- Test workflow execution

### Phase 4: Frontend & Integration (Week 7-8)
- Build Amplify frontend
- Implement real-time updates
- End-to-end testing
- Performance optimization

### Phase 5: Testing & Launch (Week 9-10)
- Property-based testing
- Load testing
- Security audit
- MVP launch


## 16. Design Summary

### 16.1 Key Architectural Decisions

1. **Strict Layer Separation**: Planner → Validator → Guardrail → Executor ensures LLM cannot directly execute commands
2. **Explicit State Machine**: 8 well-defined states (RECEIVED, PLANNED, VALIDATED, AWAITING_CONFIRMATION, EXECUTING, COMPLETED, FAILED, DEGRADED) with clear transitions
3. **Cross-Platform Support**: Platform Abstraction Layer handles macOS, Linux, and Windows differences
4. **Explicit JSON Schema**: Complete schema definition with validation rules ensures plan consistency
5. **Realistic Rollback**: Limited to project workspace, file operations, and local git operations only
6. **Cost Management**: Target $150/month with caching, TTL, and budget alerts

### 16.2 Safety Guarantees

- LLM has NO direct execution capabilities
- All plans validated against JSON schema before guardrail checks
- Guardrail layer independently validates safety constraints
- Command whitelist/blacklist enforced at multiple layers
- File operations sandboxed to project directory
- Platform-specific restrictions (Windows registry, Unix sudo, etc.)
- User confirmation required for high-risk actions

### 16.3 Cross-Platform Compatibility

- Platform detection at request time
- Platform-aware plan generation
- Command translation (bash/zsh/powershell/cmd)
- Path normalization (/ vs \)
- Platform-specific guardrails
- Tested on macOS, Linux, and Windows

### 16.4 Implementation Priorities

1. **Phase 1**: AWS infrastructure and authentication
2. **Phase 2**: Task Planner, Plan Validator, and Guardrail Layer Lambdas
3. **Phase 3**: Local Execution Engine with Platform Abstraction Layer
4. **Phase 4**: Frontend dashboard with real-time state updates
5. **Phase 5**: Testing, optimization, and launch

### 16.5 Success Criteria

- Task completion success rate >= 85%
- Execution accuracy >= 90%
- Plan generation accuracy >= 95%
- Cost per workflow < $0.20
- Monthly cost < $150
- System uptime >= 99%
- Cross-platform compatibility verified
