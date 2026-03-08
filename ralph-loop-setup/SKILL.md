---
name: ralph-loop-setup
description: Set up the Ralph Loop autonomous agent iteration system for new projects. Use when users ask to set up Ralph Loop, create an autonomous agent workflow, or want to break down a project into stories that AI coding agents can complete iteratively. Supports Docker sandbox isolation and four AI CLIs (amp, copilot, opencode, claude).
---

# Ralph Loop Setup Skill

Sets up the Ralph Loop autonomous agent iteration system for a new project.
Based on [snarktank/ralph](https://github.com/snarktank/ralph) and battle-tested with Docker sandbox support.

**Prompts for project requirements** before creating files — either as a file path or direct description.

## What is Ralph Loop?

An autonomous agent iteration system that orchestrates AI coding CLIs (Amp, GitHub Copilot CLI, OpenCode, or Claude Code) to complete project stories iteratively. Each iteration spawns a **fresh AI instance** with clean context. Memory persists across iterations via three channels:

1. **`prd.json`** — state machine (which stories are done)
2. **`progress.txt`** — institutional memory (patterns, learnings)
3. **Git history** — actual code changes

Supports running directly on host or inside a Docker sandbox for full isolation.

## Key Components

All scaffolding files live inside `.ralph/` at the project root:

| File | Purpose |
|------|---------|
| `.ralph/ralph-claude.sh` | Orchestrator script (4-way CLI detection, error handling, archiving) |
| `.ralph/prd.json` | Stories with acceptance criteria, priority, branch name (`passes: true/false`) |
| `.ralph/prompt.md` | Project requirements and agent instructions |
| `.ralph/progress.txt` | Iteration log with **Codebase Patterns** section at top |
| `.ralph/AGENTS.md` | Knowledge base: patterns, gotchas, solutions (auto-read by AI tools) |

**Docker sandbox files** (optional, at project root):

| File | Purpose |
|------|---------|
| `Dockerfile.ralph` | Container image with toolchain + AI CLI |
| `docker-compose.ralph.yml` | Service config with volume mounts |
| `run-sandbox.sh` | One-command launcher with pre-flight checks |
| `.dockerignore` | Speed up Docker builds |

## User Input Collection

**IMPORTANT: Before creating any files, gather TWO things from the user.**

### 1. Project Requirements

Ask: "How would you like to provide your project requirements?"

Options:
1. **"I have a prompt file"** → ask for file path, read contents
2. **"I'll provide a description now"** → user types/pastes requirements

### 2. Execution Mode

Ask: "How do you want to run the Ralph Loop?"

Options:
1. **"Docker sandbox (Recommended)"** → Creates Dockerfile.ralph, docker-compose.ralph.yml, run-sandbox.sh, .dockerignore. Fully isolated, reproducible.
2. **"Direct on host"** → Just creates .ralph/ files. Requires CLI tools installed locally.
3. **"Both"** → Creates all files. Can run either way.

## Setup Instructions

### 1. Create .ralph/ralph-claude.sh

The orchestrator supports four AI CLIs with automatic detection and failover.

**CLI priority order:** amp > copilot > opencode > claude

**Key features:**
- Four-way CLI detection and cycling on rate limits
- `--tool` flag to force a specific CLI (e.g., `--tool claude`)
- Auth error detection (fatal, stops immediately)
- Generic `Error:` output detection (catches CLIs that exit 0 on failure)
- `<promise>COMPLETE</promise>` sentinel for deterministic completion detection
- Automatic archiving when `branchName` changes in prd.json
- `|| true` on AI command (crashed iteration ≠ loop death)

```bash
#!/bin/bash

# Ralph Loop - Autonomous Agent Iteration System
# Orchestrates Amp, GitHub Copilot CLI, OpenCode, or Claude Code agents
# Based on https://github.com/snarktank/ralph

set -e

# Get script directory and change to it
SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
cd "$SCRIPT_DIR"

# Also change to project root (parent directory) for git operations
PROJECT_ROOT="$(cd "$SCRIPT_DIR/.." && pwd)"

# Configuration
PRD_FILE="$SCRIPT_DIR/prd.json"
PROMPT_FILE="$SCRIPT_DIR/prompt.md"
PROGRESS_FILE="$SCRIPT_DIR/progress.txt"
AGENTS_FILE="$SCRIPT_DIR/AGENTS.md"
LAST_BRANCH_FILE="$SCRIPT_DIR/.last-branch"
ARCHIVE_DIR="$SCRIPT_DIR/archive"
MAX_ITERATIONS=20
MAX_RETRIES=5
RETRY_DELAY=60

# AI CLI tools (auto-detect: amp > copilot > opencode > claude)
AI_CLI=""
AI_CLI_PRIMARY=""
AI_CLI_SECONDARY=""
AI_CLI_TERTIARY=""
AI_CLI_QUATERNARY=""
CURRENT_CLI=""

# Colors for output
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
BLUE='\033[0;34m'
NC='\033[0m' # No Color

# Print colored message
print_status() {
    local color=$1
    local message=$2
    echo -e "${color}${message}${NC}"
}

# Parse command-line flags
FORCE_TOOL=""
parse_args() {
    while [[ $# -gt 0 ]]; do
        case $1 in
            --tool)
                FORCE_TOOL="$2"
                shift 2
                ;;
            *)
                # Positional arg = max iterations
                if [[ "$1" =~ ^[0-9]+$ ]]; then
                    MAX_ITERATIONS="$1"
                fi
                shift
                ;;
        esac
    done
}

# Detect and set AI CLI tools (priority: amp > copilot > opencode > claude)
detect_ai_cli() {
    # If --tool flag was passed, use that exclusively
    if [[ -n "$FORCE_TOOL" ]]; then
        if ! command -v "$FORCE_TOOL" &> /dev/null; then
            print_status "$RED" "Error: Forced tool '$FORCE_TOOL' is not installed"
            exit 1
        fi
        if [[ "$FORCE_TOOL" != "amp" && "$FORCE_TOOL" != "copilot" && "$FORCE_TOOL" != "opencode" && "$FORCE_TOOL" != "claude" ]]; then
            print_status "$RED" "Error: Invalid tool '$FORCE_TOOL'. Must be 'amp', 'copilot', 'opencode', or 'claude'."
            exit 1
        fi
        AI_CLI_PRIMARY="$FORCE_TOOL"
        CURRENT_CLI="$FORCE_TOOL"
        AI_CLI="$CURRENT_CLI"
        print_status "$GREEN" "Using forced tool: $FORCE_TOOL"
        return
    fi

    local available=()

    if command -v amp &> /dev/null; then
        available+=("amp")
    fi

    if command -v copilot &> /dev/null; then
        available+=("copilot")
    fi

    if command -v opencode &> /dev/null; then
        available+=("opencode")
    fi

    if command -v claude &> /dev/null; then
        available+=("claude")
    fi

    if [[ ${#available[@]} -eq 0 ]]; then
        print_status "$RED" "Error: No AI CLI found (amp, copilot, opencode, or claude)"
        print_status "$YELLOW" "Install one of:"
        print_status "$YELLOW" "  Amp: https://ampcode.com"
        print_status "$YELLOW" "  GitHub Copilot CLI: curl -fsSL https://gh.io/copilot-install | bash"
        print_status "$YELLOW" "  OpenCode: https://opencode.ai"
        print_status "$YELLOW" "  Claude Code: https://claude.ai/code"
        exit 1
    fi

    # Set primary / secondary / tertiary / quaternary in priority order
    AI_CLI_PRIMARY="${available[0]}"
    AI_CLI_SECONDARY="${available[1]:-}"
    AI_CLI_TERTIARY="${available[2]:-}"
    AI_CLI_QUATERNARY="${available[3]:-}"

    if [[ ${#available[@]} -ge 2 ]]; then
        print_status "$GREEN" "Detected CLIs: ${available[*]} (auto-switch enabled)"
    else
        print_status "$GREEN" "Using ${available[0]} CLI (single mode)"
    fi

    CURRENT_CLI="$AI_CLI_PRIMARY"
    AI_CLI="$CURRENT_CLI"
    print_status "$BLUE" "Starting with: $CURRENT_CLI"
}

# Switch to the next available CLI tool
switch_cli() {
    local all_clis=("$AI_CLI_PRIMARY" "$AI_CLI_SECONDARY" "$AI_CLI_TERTIARY" "$AI_CLI_QUATERNARY")
    local found_current=false

    for cli in "${all_clis[@]}"; do
        [[ -z "$cli" ]] && continue
        if [[ "$found_current" == true ]]; then
            CURRENT_CLI="$cli"
            AI_CLI="$CURRENT_CLI"
            print_status "$GREEN" "Switched to: $CURRENT_CLI"
            return 0
        fi
        if [[ "$cli" == "$CURRENT_CLI" ]]; then
            found_current=true
        fi
    done

    # Wrap around to primary
    if [[ "$CURRENT_CLI" != "$AI_CLI_PRIMARY" ]]; then
        CURRENT_CLI="$AI_CLI_PRIMARY"
        AI_CLI="$CURRENT_CLI"
        print_status "$GREEN" "Switched to: $CURRENT_CLI (wrapped around)"
        return 0
    fi

    print_status "$YELLOW" "No alternative CLI available to switch to"
    return 1
}

# Check if output indicates rate limit or usage limit
is_limit_error() {
    local output="$1"
    local exit_code="$2"

    if echo "$output" | grep -qiE "rate.?limit|usage.?limit|limit.?reached|quota.?exceeded|too.?many.?requests|capacity|try.?again.?later|overloaded|max.?usage|conversation.?limit|token.?limit|context.?limit|exceeded.?limit"; then
        return 0
    fi

    if [[ "$exit_code" -eq 75 ]]; then
        return 0
    fi

    return 1
}

# Check if output indicates an authentication error (fatal — not retryable)
is_auth_error() {
    local output="$1"
    if echo "$output" | grep -qiE "no.?authentication|not.?authenticated|auth.?fail|unauthorized|login.?required|token.?expired|COPILOT_GITHUB_TOKEN|GH_TOKEN"; then
        return 0
    fi
    return 1
}

# Archive previous run if branch changed
archive_previous_run() {
    local current_branch=$(jq -r '.branchName // empty' "$PRD_FILE" 2>/dev/null)
    [[ -z "$current_branch" ]] && return

    if [[ -f "$LAST_BRANCH_FILE" ]]; then
        local last_branch=$(cat "$LAST_BRANCH_FILE")
        if [[ "$last_branch" != "$current_branch" && -n "$last_branch" ]]; then
            local archive_name="$(date +%Y-%m-%d)-${last_branch##*/}"
            local archive_path="$ARCHIVE_DIR/$archive_name"
            mkdir -p "$archive_path"
            cp "$PRD_FILE" "$archive_path/prd.json" 2>/dev/null || true
            cp "$PROGRESS_FILE" "$archive_path/progress.txt" 2>/dev/null || true
            print_status "$YELLOW" "Archived previous run ($last_branch) to archive/$archive_name/"

            # Reset progress for new branch
            cat > "$PROGRESS_FILE" << 'PROGRESS_HEADER'
# Progress Log
# Each iteration appends learnings below

## Codebase Patterns
<!-- Consolidated patterns go here — agents update this section -->

---
PROGRESS_HEADER
        fi
    fi

    echo "$current_branch" > "$LAST_BRANCH_FILE"
}

# Check prerequisites
check_prerequisites() {
    print_status "$BLUE" "Checking prerequisites..."

    detect_ai_cli

    if ! command -v jq &> /dev/null; then
        print_status "$RED" "Error: jq is not installed"
        print_status "$YELLOW" "Install it with: brew install jq (macOS) or apt install jq (Linux)"
        exit 1
    fi

    if [[ ! -f "$PRD_FILE" ]]; then
        print_status "$RED" "Error: $PRD_FILE not found"
        exit 1
    fi

    if [[ ! -f "$PROMPT_FILE" ]]; then
        print_status "$RED" "Error: $PROMPT_FILE not found"
        exit 1
    fi

    if ! git rev-parse --is-inside-work-tree &> /dev/null; then
        print_status "$RED" "Error: Not a git repository"
        print_status "$YELLOW" "Initialize with: git init"
        exit 1
    fi

    # Initialize progress file if missing
    if [[ ! -f "$PROGRESS_FILE" ]]; then
        cat > "$PROGRESS_FILE" << 'PROGRESS_HEADER'
# Progress Log
# Each iteration appends learnings below

## Codebase Patterns
<!-- Consolidated patterns go here — agents update this section -->

---
PROGRESS_HEADER
        print_status "$BLUE" "Created progress.txt"
    fi

    # Archive if branch changed
    archive_previous_run

    print_status "$GREEN" "All prerequisites met!"
}

# Count incomplete stories
count_incomplete_stories() {
    jq '[.stories[] | select(.passes == false)] | length' "$PRD_FILE"
}

# Get next incomplete story (by priority field, then array order)
get_next_story() {
    jq -r '[.stories[] | select(.passes == false)] | sort_by(.priority // 999) | .[0].id' "$PRD_FILE"
}

# Get story details
get_story_details() {
    local story_id=$1
    jq -r ".stories[] | select(.id == \"$story_id\")" "$PRD_FILE"
}

# Run agent for a story
# Returns: 0 = success, 1 = general failure, 2 = limit reached (should switch CLI)
run_agent() {
    local story_id=$1
    local story_details=$(get_story_details "$story_id")
    local story_title=$(echo "$story_details" | jq -r '.title')
    local story_description=$(echo "$story_details" | jq -r '.description')
    local acceptance_criteria=$(echo "$story_details" | jq -r '.acceptance_criteria | join("\n- ")')
    local branch_name=$(jq -r '.branchName // empty' "$PRD_FILE")

    print_status "$BLUE" "Working on: $story_id - $story_title"
    print_status "$BLUE" "Using CLI: $CURRENT_CLI"

    local prompt="You are an autonomous agent working on completing a project story.

CRITICAL: Before starting, read these files to understand context:
1. Read $PROMPT_FILE - Project requirements and instructions
2. Read $PROGRESS_FILE - Learnings from previous iterations (Codebase Patterns section at top is most important)
3. Read $AGENTS_FILE - Patterns, gotchas, and reusable solutions

WORKING DIRECTORY: $PROJECT_ROOT
$([ -n "$branch_name" ] && echo "GIT BRANCH: $branch_name (create/switch to this branch if not already on it)")

CURRENT STORY TO COMPLETE:
ID: $story_id
Title: $story_title
Description: $story_description

ACCEPTANCE CRITERIA (ALL must be met):
- $acceptance_criteria

YOUR WORKFLOW:
1. READ PHASE: Read prompt.md, progress.txt, and AGENTS.md first
2. BRANCH PHASE: Create/switch to the correct git branch if specified in prd.json
3. IMPLEMENT PHASE: Create/modify code, configurations, scripts as needed
4. QUALITY CHECK: ALL commits must pass typecheck/lint/test. Do NOT commit broken code.
5. VERIFY PHASE: Test thoroughly and verify ALL acceptance criteria are met
6. UPDATE STATE PHASE:
   - Update $PRD_FILE: Set passes=true for this story. Add implementation notes to the story's notes field.
   - Consolidate learnings into the Codebase Patterns section at the TOP of $PROGRESS_FILE
   - Append iteration-specific details to the bottom of $PROGRESS_FILE
   - Update $AGENTS_FILE with any NEW reusable patterns (not story-specific details)
7. GIT COMMIT PHASE: Commit with message format: feat: [$story_id] - $story_title

AFTER COMMITTING: Check if ALL stories in prd.json have passes=true.
If yes, output exactly: <promise>COMPLETE</promise>
This signals the loop to stop.

IMPORTANT RULES:
- Work on ONE story per iteration only
- Do NOT mark a story as complete unless ALL acceptance criteria are verified
- Do NOT commit broken code — quality checks must pass first
- Document any issues, workarounds, or learnings
- If you encounter errors, debug and fix them before proceeding"

    cd "$PROJECT_ROOT"

    # Capture output and exit code
    local output_file=$(mktemp)
    local exit_code=0

    # Use correct CLI syntax based on which tool we're using
    if [[ "$CURRENT_CLI" == "amp" ]]; then
        # Amp: pipe prompt to stdin, --dangerously-allow-all for autonomous mode
        echo "$prompt" | amp --dangerously-allow-all 2>&1 | tee "$output_file" || exit_code=$?
    elif [[ "$CURRENT_CLI" == "copilot" ]]; then
        # GitHub Copilot CLI: -p for non-interactive, --allow-all for autonomous
        copilot -p "$prompt" --allow-all 2>&1 | tee "$output_file" || exit_code=$?
    elif [[ "$CURRENT_CLI" == "opencode" ]]; then
        # OpenCode: OPENCODE_PERMISSION='{"*":"allow"}' for autonomous
        OPENCODE_PERMISSION='{"*":"allow"}' opencode run -m "github-copilot/claude-sonnet-4" "$prompt" 2>&1 | tee "$output_file" || exit_code=$?
    else
        # Claude Code: --dangerously-skip-permissions for autonomous, --print for non-interactive output
        claude --dangerously-skip-permissions -p "$prompt" 2>&1 | tee "$output_file" || exit_code=$?
    fi

    local output=$(cat "$output_file")
    rm -f "$output_file"

    cd "$SCRIPT_DIR"

    # Check for COMPLETE sentinel — all stories done
    if echo "$output" | grep -q "<promise>COMPLETE</promise>"; then
        print_status "$GREEN" "Agent signaled COMPLETE — all stories done"
        return 0
    fi

    # Check for authentication errors first (fatal — abort immediately)
    if is_auth_error "$output"; then
        print_status "$RED" "Authentication error on $CURRENT_CLI — cannot continue"
        print_status "$YELLOW" "Fix auth and re-run. See error output above."
        return 1
    fi

    # Check if this was a limit error
    if is_limit_error "$output" "$exit_code"; then
        print_status "$YELLOW" "Limit reached on $CURRENT_CLI"
        return 2
    fi

    # Some CLIs exit 0 even on errors — detect "Error:" in output as failure
    if [[ "$exit_code" -eq 0 ]] && echo "$output" | grep -q '^Error:'; then
        print_status "$RED" "CLI reported error (exit code was 0 but output contains Error)"
        return 1
    fi

    return $exit_code
}

# Main loop
main() {
    print_status "$BLUE" "=========================================="
    print_status "$BLUE" "   Ralph Loop - Autonomous Agent System   "
    print_status "$BLUE" "=========================================="

    parse_args "$@"
    check_prerequisites

    local iteration=1
    while [[ $iteration -le $MAX_ITERATIONS ]]; do
        print_status "$YELLOW" "\n=== Iteration $iteration of $MAX_ITERATIONS ==="

        local incomplete=$(count_incomplete_stories)

        if [[ $incomplete -eq 0 ]]; then
            print_status "$GREEN" "\n=========================================="
            print_status "$GREEN" "   ALL STORIES COMPLETE! PROJECT DONE!   "
            print_status "$GREEN" "=========================================="
            exit 0
        fi

        print_status "$BLUE" "Stories remaining: $incomplete"

        local next_story=$(get_next_story)

        if [[ -z "$next_story" || "$next_story" == "null" ]]; then
            print_status "$RED" "Error: Could not determine next story"
            exit 1
        fi

        local retry=1
        while [[ $retry -le $MAX_RETRIES ]]; do
            print_status "$BLUE" "Attempt $retry of $MAX_RETRIES for $next_story (using $CURRENT_CLI)"

            run_agent "$next_story"
            local agent_result=$?

            if [[ $agent_result -eq 0 ]]; then
                # Check if the COMPLETE sentinel was found (all stories done)
                if [[ $(count_incomplete_stories) -eq 0 ]]; then
                    print_status "$GREEN" "\n=========================================="
                    print_status "$GREEN" "   ALL STORIES COMPLETE! PROJECT DONE!   "
                    print_status "$GREEN" "=========================================="
                    exit 0
                fi
                print_status "$GREEN" "Agent completed story successfully"
                break
            elif [[ $agent_result -eq 2 ]]; then
                # Limit reached - try switching CLI
                print_status "$YELLOW" "Limit detected on $CURRENT_CLI"

                if switch_cli; then
                    print_status "$GREEN" "Switched to $CURRENT_CLI - retrying immediately"
                    continue
                else
                    print_status "$YELLOW" "No alternative CLI available, waiting ${RETRY_DELAY}s before retry..."
                    sleep $RETRY_DELAY
                    ((retry++))
                fi
            else
                print_status "$YELLOW" "Agent failed (exit code: $agent_result), waiting ${RETRY_DELAY}s before retry..."
                sleep $RETRY_DELAY
                ((retry++))
            fi
        done

        if [[ $retry -gt $MAX_RETRIES ]]; then
            print_status "$RED" "Max retries exceeded for $next_story"
            exit 1
        fi

        # Brief pause between iterations for clean separation
        sleep 2

        ((iteration++))
    done

    print_status "$YELLOW" "Max iterations reached. Some stories may be incomplete."
    exit 1
}

main "$@"
```

### 2. Create .ralph/prd.json template

```json
{
  "project": "PROJECT_NAME",
  "branchName": "ralph/feature-name",
  "description": "PROJECT_DESCRIPTION",
  "stories": [
    {
      "id": "Story-1",
      "title": "First Story Title",
      "description": "Detailed description of what needs to be done",
      "acceptance_criteria": [
        "Criterion 1 that must be met",
        "Criterion 2 that must be met",
        "Typecheck/lint/tests pass"
      ],
      "priority": 1,
      "passes": false,
      "notes": ""
    },
    {
      "id": "Story-2",
      "title": "Second Story Title",
      "description": "Detailed description of what needs to be done",
      "acceptance_criteria": [
        "Criterion 1 that must be met",
        "Criterion 2 that must be met",
        "Typecheck/lint/tests pass"
      ],
      "priority": 2,
      "passes": false,
      "notes": ""
    }
  ]
}
```

**prd.json fields explained:**

| Field | Purpose |
|-------|---------|
| `branchName` | Git branch for this feature (auto-created by agent). Enables archiving when switching features. |
| `priority` | Integer ordering — lower = higher priority. Stories execute in priority order. |
| `passes` | Boolean state machine — `false` = incomplete, `true` = done |
| `notes` | Freeform field for the AI to record implementation context across iterations |
| `acceptance_criteria` | Array of verifiable conditions. Always include "Typecheck/lint/tests pass". |

### 3. Create .ralph/prompt.md

Incorporate the user's collected requirements:

```markdown
# Project Requirements

## User-Provided Requirements

[INSERT THE USER'S PROJECT REQUIREMENTS HERE]

## Critical Rules

1. **ALWAYS read these files before starting any work:**
   - `.ralph/prompt.md` - Project requirements (this file)
   - `.ralph/progress.txt` - Learnings from previous iterations (Codebase Patterns section at top is most important)
   - `.ralph/AGENTS.md` - Patterns and gotchas

2. **Story completion requirements:**
   - ALL acceptance criteria must be met
   - Quality checks (typecheck, lint, test) must PASS before committing
   - Do NOT commit broken code
   - `prd.json` must be updated with `passes: true` and implementation `notes`
   - Learnings must be consolidated into Codebase Patterns section of `progress.txt`

3. **Never skip verification** — test each component, verify all criteria

4. **Git commit format:** `feat: [Story-ID] - Story Title`

5. **One story per iteration** — implement, verify, commit, then stop

6. **Completion signal:** After committing, check if ALL stories have `passes: true`. If yes, output: `<promise>COMPLETE</promise>`
```

### 4. Create .ralph/progress.txt

```
# Progress Log
# Each iteration appends learnings below

## Codebase Patterns
<!-- Consolidated patterns go here — agents update this section with reusable discoveries -->
<!-- This section is the most important — agents read it first each iteration -->

---
```

The **Codebase Patterns** section at the TOP is critical — it's a self-building knowledge base that gets consolidated (not just appended) each iteration. Later iterations read the most important patterns first.

### 5. Create .ralph/AGENTS.md

```markdown
# Agent Knowledge Base

Patterns, gotchas, and reusable solutions discovered during development.
This file is auto-read by AI coding tools — keep it concise and actionable.

## Working Patterns
- [Reusable patterns added by agents as they discover them]

## Anti-Patterns
- [Things to avoid, added during iterations]

## Troubleshooting
- [Common issues and solutions]

## Story Dependencies
- Story-1: No dependencies
- Story-2: Depends on Story-1
```

**Important:** Agents should only add **reusable patterns** to AGENTS.md, not story-specific implementation details. Story-specific notes go in the story's `notes` field in prd.json.

## Docker Sandbox Setup (if user selected Docker mode)

### 6. Create Dockerfile.ralph

**IMPORTANT: Customize the base image and toolchain layers for the project.** The example below is for a Go project — adapt Layer 1 (system tools), Layer 2 (AI CLI), and Layer 3 (project-specific tooling) as needed.

```dockerfile
# Dockerfile.ralph — Ralph Loop sandbox
#
# Build:  docker compose -f docker-compose.ralph.yml build
# Run:    docker compose -f docker-compose.ralph.yml run --rm ralph

FROM golang:1.24-bookworm  # ← CUSTOMIZE: use project-appropriate base image

# ── Build args ──────────────────────────────────────────────────
ARG GIT_USER_NAME="Ralph Loop"
ARG GIT_USER_EMAIL="ralph@sandbox.local"

# ── Layer 1: System tools ──────────────────────────────────────
RUN apt-get update && apt-get install -y --no-install-recommends \
        jq git make curl ca-certificates unzip \
    && rm -rf /var/lib/apt/lists/*

# ── Layer 2: GitHub Copilot CLI ────────────────────────────────
RUN curl -fsSL https://gh.io/copilot-install | bash

# ── Layer 3: Project-specific tooling ──────────────────────────
# CUSTOMIZE: Add project-specific binaries here (e.g., Vault, Terraform, Node.js)
# Example for HashiCorp Vault:
# ARG VAULT_VERSION=1.20.0
# ARG TARGETARCH
# RUN curl -fsSL "https://releases.hashicorp.com/vault/${VAULT_VERSION}/vault_${VAULT_VERSION}_linux_${TARGETARCH}.zip" \
#        -o /tmp/vault.zip \
#     && unzip /tmp/vault.zip -d /usr/local/bin/ \
#     && rm /tmp/vault.zip

# ── Layer 4: Git config ───────────────────────────────────────
RUN git config --global user.name  "${GIT_USER_NAME}" \
    && git config --global user.email "${GIT_USER_EMAIL}" \
    && git config --global --add safe.directory /workspace

# ── Layer 5: Workspace ────────────────────────────────────────
WORKDIR /workspace
ENTRYPOINT ["bash", ".ralph/ralph-claude.sh"]
```

### 7. Create docker-compose.ralph.yml

```yaml
# docker-compose.ralph.yml — Ralph Loop Docker sandbox

services:
  ralph:
    build:
      context: .
      dockerfile: Dockerfile.ralph
    volumes:
      # Project files — bidirectional, changes persist to host
      - .:/workspace
      # Copilot CLI config (settings, skills, etc.)
      - ~/.copilot:/root/.copilot
      # gh CLI config (for 'gh auth' inside container)
      - ~/.config/gh:/root/.config/gh:ro
      # CUSTOMIZE: Add language-specific cache volumes
      # Go example:
      # - go-mod-cache:/root/go/pkg/mod
      # - go-build-cache:/root/.cache/go-build
      # Node example:
      # - node-modules-cache:/workspace/node_modules
    environment:
      - HOME=/root
      # GH_TOKEN is injected by run-sandbox.sh (extracted from host keychain)
      - GH_TOKEN=${GH_TOKEN:-}
      # CUSTOMIZE: Add project-specific env vars
    stdin_open: true
    tty: true

# CUSTOMIZE: Add named volumes for caches
# volumes:
#   go-mod-cache:
#   go-build-cache:
```

### 8. Create run-sandbox.sh

```bash
#!/bin/bash
# run-sandbox.sh — Launch the Ralph Loop inside a Docker sandbox
#
# Prerequisites:
#   - Docker Desktop running
#   - GitHub Copilot CLI authenticated (~/.copilot must exist)
#   - gh CLI authenticated (for GH_TOKEN extraction)

set -euo pipefail

# Colors
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
BLUE='\033[0;34m'
NC='\033[0m'

print_status() { echo -e "${1}${2}${NC}"; }

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
cd "$SCRIPT_DIR"

# ── Pre-flight checks ──────────────────────────────────────────

print_status "$BLUE" "=== Ralph Loop Docker Sandbox ==="

# 1. Docker available?
if ! command -v docker &> /dev/null; then
    print_status "$RED" "Error: Docker is not installed or not in PATH"
    exit 1
fi

if ! docker info &> /dev/null 2>&1; then
    print_status "$RED" "Error: Docker daemon is not running — start Docker Desktop"
    exit 1
fi

print_status "$GREEN" "✓ Docker is running"

# 2. Copilot config present?
if [[ ! -d "$HOME/.copilot" ]]; then
    print_status "$RED" "Error: ~/.copilot not found"
    print_status "$YELLOW" "Authenticate first: copilot auth login"
    exit 1
fi

print_status "$GREEN" "✓ Copilot config found (~/.copilot)"

# 3. Extract GH_TOKEN from host keychain (required for Copilot CLI inside Docker)
if [[ -z "${GH_TOKEN:-}" ]]; then
    if command -v gh &> /dev/null && gh auth token &> /dev/null; then
        export GH_TOKEN="$(gh auth token)"
        print_status "$GREEN" "✓ GH_TOKEN extracted from gh auth (${#GH_TOKEN} chars)"
    else
        print_status "$RED" "Error: Cannot extract GitHub token"
        print_status "$YELLOW" "Either:"
        print_status "$YELLOW" "  1. Run 'gh auth login' to authenticate the GitHub CLI"
        print_status "$YELLOW" "  2. Set GH_TOKEN env var manually: export GH_TOKEN=ghp_..."
        exit 1
    fi
else
    print_status "$GREEN" "✓ GH_TOKEN already set (${#GH_TOKEN} chars)"
fi

# ── Resolve symlinks ──────────────────────────────────────────
# Absolute symlinks won't resolve inside the container.
# Replace them with real file content.

RESOLVED_LINKS=()

for link in $(find . -maxdepth 1 -type l 2>/dev/null); do
    target=$(readlink "$link")
    if [[ "$target" = /* ]]; then
        if [[ -f "$target" ]]; then
            print_status "$YELLOW" "Resolving symlink: $link → $target"
            cp -L "$link" "${link}.tmp"
            mv "${link}.tmp" "$link"
            RESOLVED_LINKS+=("$link|$target")
        else
            print_status "$YELLOW" "Warning: symlink $link points to missing file $target (skipped)"
        fi
    fi
done

if [[ ${#RESOLVED_LINKS[@]} -gt 0 ]]; then
    print_status "$GREEN" "✓ Resolved ${#RESOLVED_LINKS[@]} symlink(s) for Docker compatibility"
fi

# ── Build & Run ───────────────────────────────────────────────

print_status "$BLUE" "\nBuilding Docker image..."
docker compose -f docker-compose.ralph.yml build

print_status "$BLUE" "\nStarting Ralph Loop in sandbox..."
docker compose -f docker-compose.ralph.yml run --rm ralph
EXIT_CODE=$?

# ── Post-run info ─────────────────────────────────────────────

if [[ ${#RESOLVED_LINKS[@]} -gt 0 ]]; then
    print_status "$YELLOW" "\nNote: The following symlinks were replaced with real files:"
    for entry in "${RESOLVED_LINKS[@]}"; do
        IFS='|' read -r link target <<< "$entry"
        print_status "$YELLOW" "  $link  (was → $target)"
    done
    print_status "$YELLOW" "To restore: ln -sf <target> <link>"
fi

exit $EXIT_CODE
```

### 9. Create .dockerignore

```
# Docker build context exclusions for Ralph Loop sandbox
.git
*.exe
*.so
*.dylib
*.dll
*.test
*.out
```

## After Setup

1. `chmod +x .ralph/ralph-claude.sh`
2. `git init` (if not already a repo)
3. Customize `.ralph/prd.json` with project stories (**REQUIRED**)
4. Review `.ralph/prompt.md` (pre-populated with your requirements)

**If using Docker sandbox:**

5. `chmod +x run-sandbox.sh`
6. Ensure `gh auth login` is done (for GH_TOKEN extraction)
7. Customize `Dockerfile.ralph` Layer 3 for project tooling
8. Customize `docker-compose.ralph.yml` volumes for language caches
9. Run: `./run-sandbox.sh`

**If running directly on host:**

5. Ensure at least one AI CLI is installed (amp, copilot, opencode, or claude)
6. Run: `./.ralph/ralph-claude.sh`
7. Override CLI: `./.ralph/ralph-claude.sh --tool claude`
8. Override iterations: `./.ralph/ralph-claude.sh 30`

## Tips for Writing Good Stories

1. **Size for one context window**: Each story must be completable in ONE iteration. If it feels too big, split it. This is the #1 factor in autonomous coding success.
2. **Order by dependency**: Schema → backend → UI → integration tests → docs. Use the `priority` field.
3. **Be specific and verifiable**: "Add priority column to tasks table with enum values" not "Add priority support"
4. **Always include quality criteria**: Every story should have "Typecheck/lint/tests pass" as a criterion.
5. **Include verification commands**: Criteria should say HOW to verify (e.g., "`go test ./...` passes", "verify in browser")
6. **Document in notes**: Agent updates the story's `notes` field with implementation context for future iterations.

**Right-sized story examples:**
- ✅ "Add priority enum column to tasks table" (schema change + migration)
- ✅ "Create GET /tasks endpoint with priority filter" (one endpoint)
- ❌ "Build the entire task management system" (way too big)
- ❌ "Add priority to tasks" (too vague — schema? UI? API? all three?)

## Lessons from Production Runs

These patterns were discovered running Ralph Loop on real projects:

| Issue | Solution |
|-------|----------|
| Copilot CLI exits 0 on auth errors | `is_auth_error()` checks output for auth patterns |
| Copilot CLI exits 0 on model errors | Generic `^Error:` output detection catches all |
| `~/.copilot` doesn't contain auth tokens | Pass `GH_TOKEN` env var (extracted via `gh auth token`) |
| macOS keychain tokens can't be read in Docker | `run-sandbox.sh` extracts token before Docker build |
| Absolute symlinks break inside Docker | `run-sandbox.sh` resolves them to real files pre-launch |
| Go/Node module downloads slow per run | Named Docker volumes persist caches across runs |
| `--model claude-sonnet-4` may not be available | Omit `--model` to use Copilot's default (works reliably) |
| Later iterations lose context from earlier ones | Codebase Patterns section at TOP of progress.txt (consolidated, not just appended) |
| Loop doesn't know when all stories are done | `<promise>COMPLETE</promise>` sentinel — deterministic exit signal |
| Old runs clutter working directory | Automatic archiving when `branchName` changes in prd.json |
| Story notes lost between iterations | `notes` field in prd.json persists implementation context per story |
| Stories executed in wrong order | `priority` field + `sort_by(.priority)` in jq query |
| Agent works on multiple stories per iteration | Explicit "ONE story per iteration" rule in prompt |
| Agent commits broken code | "Quality checks must PASS before committing" rule in prompt |
