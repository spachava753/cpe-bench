# CPE-Bench Design

A benchmark system for testing agentic coding CLIs using Modal Sandboxes and Go's test framework.

## Overview

CPE-Bench provides a framework to evaluate different agentic coding CLIs (CPE, Claude Code, Codex CLI) across various software development and general computer use tasks. Each task returns a score from 0.0 to 1.0, enabling quantitative comparison of agent capabilities.

## Design Principles

1. **Go test native**: All tasks are Go tests, leveraging standard tooling
2. **Sandbox isolation**: Each test runs in its own Modal Sandbox
3. **No custom CLI**: Use `go test` directly, no additional tooling needed
4. **Flexible evaluation**: Support binary, content matching, and LLM-as-judge scoring
5. **Agent abstraction**: Easy to add new agents via common interface

## Modal Sandbox Integration

### Key Findings from Modal Go SDK

Based on exploration of `/tmp/libmodal/modal-go/examples/`:

1. **Lifecycle**: Create sandbox → Execute commands → Terminate (defer cleanup)
2. **Capabilities**: 
   - Execute commands via `sb.Exec()`
   - Read/write files via `sb.Open()`
   - Mount volumes, set environment, inject secrets
   - PTY support for interactive CLIs
3. **Agent support**: Can install and run agentic CLIs using Docker commands

### Example Sandbox Lifecycle

```go
// Create sandbox with custom image
image := mc.Images.FromRegistry("alpine:3.21", nil).DockerfileCommands([]string{
    "RUN apk add --no-cache bash curl git",
    "RUN curl -fsSL https://install.sh | bash",
}, nil)

sb, err := mc.Sandboxes.Create(ctx, app, image, nil)
defer sb.Terminate(context.Background())

// Execute commands
proc, err := sb.Exec(ctx, []string{"agent", "do-task"}, &modal.SandboxExecParams{
    PTY: true,
    Secrets: []*modal.Secret{secret},
})
```

## Package Structure

```
cpe-bench/
├── go.mod
├── go.sum
├── README.md
├── DESIGN.md
├── tasks/                    # All benchmark tasks as tests
│   ├── software_dev/
│   │   ├── edit_file_test.go
│   │   ├── refactor_code_test.go
│   │   └── add_feature_test.go
│   ├── general_use/
│   │   ├── file_ops_test.go
│   │   └── git_workflow_test.go
│   └── complex/
│       ├── debug_issue_test.go
│       └── multi_file_refactor_test.go
├── pkg/
│   ├── sandbox/              # Sandbox lifecycle management
│   │   └── sandbox.go
│   ├── agent/                # Agent runner interface
│   │   ├── agent.go
│   │   ├── cpe.go
│   │   ├── claude_code.go
│   │   └── codex.go
│   ├── evaluator/            # Score evaluation
│   │   ├── evaluator.go
│   │   ├── binary.go         # Simple pass/fail
│   │   ├── content.go        # File content matching
│   │   └── llm_judge.go      # LLM-as-judge
│   ├── task/                 # Task framework
│   │   └── task.go
│   └── config/               # Configuration
│       └── config.go
└── testdata/                 # Task fixtures
    ├── repos/
    └── expected/
```

## Core Interfaces

### Task Definition

```go
// pkg/task/task.go
package task

import "context"

// Task represents a single benchmark task
type Task struct {
    Name        string
    Description string
    Setup       SetupFunc          // Initialize sandbox state
    Prompt      string             // Task instruction for the agent
    Evaluate    EvaluateFunc       // Compute score after execution
    Timeout     time.Duration      // Maximum execution time
}

type SetupFunc func(ctx context.Context, sb *modal.Sandbox) error
type EvaluateFunc func(ctx context.Context, sb *modal.Sandbox) (float64, error)
```

### Agent Interface

```go
// pkg/agent/agent.go
package agent

import (
    "context"
    "github.com/modal-labs/libmodal/modal-go"
)

// Agent interface for different agentic CLIs
type Agent interface {
    // Name returns the agent identifier (e.g., "cpe", "claude-code")
    Name() string
    
    // InstallCommands returns Dockerfile commands to install the agent
    InstallCommands() []string
    
    // Execute runs the agent with the given prompt in the sandbox
    Execute(ctx context.Context, sb *modal.Sandbox, prompt string) error
    
    // RequiredSecrets returns secret names needed by the agent
    RequiredSecrets() []string
}
```

**Implementations:**
- `CPE`: Runs CPE CLI with configuration
- `ClaudeCode`: Runs Claude Code CLI
- `Codex`: Runs Codex CLI

### Sandbox Manager

```go
// pkg/sandbox/sandbox.go
package sandbox

import (
    "context"
    "github.com/modal-labs/libmodal/modal-go"
)

// Manager handles sandbox lifecycle
type Manager struct {
    client *modal.Client
    app    *modal.App
}

// CreateForAgent creates a sandbox with the agent installed
func (m *Manager) CreateForAgent(ctx context.Context, agent Agent) (*modal.Sandbox, error)

// Cleanup terminates the sandbox
func (m *Manager) Cleanup(ctx context.Context, sb *modal.Sandbox) error
```

### Evaluator Interface

```go
// pkg/evaluator/evaluator.go
package evaluator

import (
    "context"
    "github.com/modal-labs/libmodal/modal-go"
)

// Evaluator produces a score for task completion
type Evaluator interface {
    Evaluate(ctx context.Context, sb *modal.Sandbox) (float64, error)
}
```

**Implementations:**

1. **Binary Evaluator** (`binary.go`): Returns 0.0 or 1.0
   ```go
   type BinaryEvaluator struct {
       Check func(ctx context.Context, sb *modal.Sandbox) (bool, error)
   }
   ```

2. **Content Evaluator** (`content.go`): Checks file content
   ```go
   type ContentEvaluator struct {
       FilePath        string
       ExpectedContent string
       Exact           bool  // exact match vs. contains
   }
   ```

3. **LLM Judge Evaluator** (`llm_judge.go`): Uses LLM to score
   ```go
   type LLMJudgeEvaluator struct {
       Criteria string
       Model    string
   }
   ```

## Test Pattern

Each task is implemented as a Go test that can be run independently:

```go
// tasks/software_dev/edit_file_test.go
package software_dev_test

import (
    "context"
    "testing"
    "github.com/yourusername/cpe-bench/pkg/agent"
    "github.com/yourusername/cpe-bench/pkg/sandbox"
    "github.com/yourusername/cpe-bench/pkg/evaluator"
    "github.com/yourusername/cpe-bench/pkg/task"
)

func Test_EditFile_CPE(t *testing.T) {
    runTaskForAgent(t, &agent.CPE{}, editFileTask)
}

func Test_EditFile_ClaudeCode(t *testing.T) {
    runTaskForAgent(t, &agent.ClaudeCode{}, editFileTask)
}

func Test_EditFile_Codex(t *testing.T) {
    runTaskForAgent(t, &agent.Codex{}, editFileTask)
}

var editFileTask = &task.Task{
    Name:        "edit_file",
    Description: "Edit a Python file to print Hello World",
    Setup: func(ctx context.Context, sb *modal.Sandbox) error {
        // Write initial file
        return writeFile(sb, "/root/test.py", "def hello(): pass")
    },
    Prompt: "Change the function to print 'Hello, World!'",
    Evaluate: &evaluator.ContentEvaluator{
        FilePath:        "/root/test.py",
        ExpectedContent: "print('Hello, World!')",
        Exact:           false,
    },
    Timeout: 5 * time.Minute,
}

func runTaskForAgent(t *testing.T, agent agent.Agent, task *task.Task) {
    ctx := context.Background()
    
    // Create sandbox manager
    mgr, err := sandbox.NewManager()
    if err != nil {
        t.Fatal(err)
    }
    
    // Create sandbox with agent installed
    sb, err := mgr.CreateForAgent(ctx, agent)
    if err != nil {
        t.Fatal(err)
    }
    defer mgr.Cleanup(context.Background(), sb)
    
    // Run task setup
    if err := task.Setup(ctx, sb); err != nil {
        t.Fatalf("Setup failed: %v", err)
    }
    
    // Execute agent with prompt
    if err := agent.Execute(ctx, sb, task.Prompt); err != nil {
        t.Fatalf("Agent execution failed: %v", err)
    }
    
    // Evaluate result
    score, err := task.Evaluate.Evaluate(ctx, sb)
    if err != nil {
        t.Fatalf("Evaluation failed: %v", err)
    }
    
    t.Logf("%s score for %s: %.2f", agent.Name(), task.Name, score)
    
    if score < 1.0 {
        t.Errorf("Task did not achieve full score: got %.2f, want 1.0", score)
    }
}
```

## Configuration

```go
// pkg/config/config.go
package config

type Config struct {
    ModalAppName   string
    ModalAPIKey    string
    AnthropicKey   string   // For CPE and Claude Code
    OpenAIKey      string   // For Codex
    Agents         []string // Filter which agents to test
    TaskFilter     string   // Regex to filter tasks
    Timeout        time.Duration
}

// Load configuration from environment variables or config file
func Load() (*Config, error)
```

### Environment Variables

- `MODAL_TOKEN_ID`, `MODAL_TOKEN_SECRET`: Modal authentication
- `ANTHROPIC_API_KEY`: For CPE and Claude Code agents
- `OPENAI_API_KEY`: For Codex agent
- `CPE_BENCH_AGENTS`: Comma-separated list of agents to test (default: all)
- `CPE_BENCH_TIMEOUT`: Default timeout for tasks

## Usage Examples

### Run all tasks for all agents
```bash
go test ./tasks/... -v
```

### Run specific task category
```bash
go test ./tasks/software_dev/... -v
```

### Run specific task
```bash
go test ./tasks/software_dev -run Test_EditFile -v
```

### Run all tasks for a specific agent
```bash
go test ./tasks/... -run CPE -v
```

### Run with custom timeout
```bash
go test ./tasks/... -timeout 30m -v
```

### Parallel execution
```bash
go test ./tasks/... -v -parallel 4
```

## Task Categories

### 1. Software Development
- **edit_file**: Simple file modification
- **refactor_code**: Refactor function/class structure
- **add_feature**: Add new functionality to existing code
- **fix_bug**: Debug and fix a specific issue
- **write_tests**: Generate test cases
- **code_review**: Improve code quality

### 2. General Use
- **file_ops**: Create, move, delete files
- **git_workflow**: Clone, commit, branch operations
- **search_replace**: Find and replace across codebase
- **text_processing**: Parse and transform data

### 3. Complex Tasks
- **multi_file_refactor**: Refactor across multiple files
- **debug_complex_issue**: Multi-step debugging
- **implement_spec**: Implement from specification
- **integration_task**: Multiple operations in sequence

## Scoring Methodology

### Binary Tasks (0.0 or 1.0)
Simple pass/fail criteria:
- File exists
- Command succeeds
- Output matches exactly

### Content Matching (0.0 to 1.0)
Graduated scoring:
- 0.0: Completely wrong
- 0.5: Partially correct (substring match, some tests pass)
- 1.0: Fully correct

### LLM Judge (0.0 to 1.0)
LLM evaluates based on criteria:
- Code quality
- Correctness
- Completeness
- Following instructions

## Benefits

1. **Go test native**: Leverage standard tooling (`go test`, `-run`, `-parallel`)
2. **Sandbox isolation**: Each test runs independently, no interference
3. **Reproducible**: Modal Sandboxes provide consistent environment
4. **Extensible**: Easy to add new tasks and agents
5. **Parallel execution**: Go test runner handles concurrency
6. **CI/CD friendly**: Standard test output, exit codes
7. **No custom CLI**: Pure `go test` workflow

## Future Enhancements

- Results aggregation and reporting dashboard
- Historical score tracking
- Task difficulty ratings
- Agent configuration variations (temperature, model, etc.)
- Cost tracking per task
- Time-to-completion metrics
- Multi-turn conversation tasks
