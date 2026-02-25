---
name: ralph-loop-setup
description: Set up the Ralph Loop autonomous agent iteration system for new projects. Use when users ask to set up Ralph Loop, create an autonomous agent workflow, or want to break down a project into stories that Claude Code agents can complete iteratively.
---

# Ralph Loop Setup Skill

This skill sets up the Ralph Loop autonomous agent iteration system for a new project.

**The skill will prompt you to provide your project requirements** either as a file path or as a direct description.

## What is Ralph Loop?

Ralph Loop is an autonomous agent iteration system that orchestrates Claude Code agents to complete project stories iteratively. It breaks down complex projects into discrete, testable stories with acceptance criteria that agents can implement autonomously.

## Key Components

1. **ralph-claude.sh** - The orchestrator script that runs the loop
2. **prd.json** - Product Requirements Document with stories and acceptance criteria
3. **prompt.md** - Project requirements and instructions for the agent
4. **progress.txt** - Iteration log tracking what's been completed
5. **AGENTS.md** - Knowledge base with patterns, gotchas, and reusable solutions
6. **.claude/settings.local.json** - Permissions for Claude Code

## User Input Collection

**IMPORTANT: Before creating any files, gather project requirements from the user.**

When the user invokes this skill, FIRST ask them for their project requirements using AskUserQuestion:

Ask the user: "How would you like to provide your project requirements?"

Options:
1. **"I have a prompt file"** - User will provide a path to an existing file with requirements
2. **"I'll provide a description now"** - User will type/paste their project description directly

### If user selects "I have a prompt file":
- Ask for the file path
- Read the file contents using the Read tool
- Use those contents as the project requirements

### If user selects "I'll provide a description now":
- Ask them to provide their project description/requirements as a string
- Use that string as the project requirements

Store the collected requirements to use when creating `prompt.md` below.

## Setup Instructions

After collecting project requirements from the user, create the following files in the current project directory:

### 1. Create ralph-claude.sh

```bash
#!/bin/bash

# Ralph Loop - Autonomous Agent Iteration System
# This script orchestrates Claude Code or OpenCode agents to complete project stories

set -e

# Get script directory and change to it
SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
cd "$SCRIPT_DIR"

# Also change to project root (parent directory) for git/terraform operations
PROJECT_ROOT="$(cd "$SCRIPT_DIR/.." && pwd)"

# Configuration
PRD_FILE="$SCRIPT_DIR/prd.json"
PROMPT_FILE="$SCRIPT_DIR/prompt.md"
PROGRESS_FILE="$SCRIPT_DIR/progress.txt"
AGENTS_FILE="$SCRIPT_DIR/AGENTS.md"
MAX_ITERATIONS=20
MAX_RETRIES=5
RETRY_DELAY=60

# AI CLI tools (auto-detect both)
AI_CLI=""
AI_CLI_PRIMARY=""
AI_CLI_SECONDARY=""
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

# Detect and set AI CLI tools (both if available)
detect_ai_cli() {
    local has_opencode=false
    local has_claude=false

    if command -v opencode &> /dev/null; then
        has_opencode=true
    fi

    if command -v claude &> /dev/null; then
        has_claude=true
    fi

    if [[ "$has_opencode" == false && "$has_claude" == false ]]; then
        print_status "$RED" "Error: Neither OpenCode nor Claude CLI is installed"
        print_status "$YELLOW" "Install OpenCode from: https://opencode.ai"
        print_status "$YELLOW" "Install Claude from: https://claude.ai/code"
        exit 1
    fi

    # Set primary and secondary CLI (prefer opencode as primary)
    if [[ "$has_opencode" == true ]]; then
        AI_CLI_PRIMARY="opencode"
        if [[ "$has_claude" == true ]]; then
            AI_CLI_SECONDARY="claude"
            print_status "$GREEN" "Detected both OpenCode and Claude CLI (auto-switch enabled)"
        else
            print_status "$GREEN" "Using OpenCode CLI (single mode)"
        fi
    else
        AI_CLI_PRIMARY="claude"
        print_status "$GREEN" "Using Claude CLI (single mode)"
    fi

    CURRENT_CLI="$AI_CLI_PRIMARY"
    AI_CLI="$CURRENT_CLI"
    print_status "$BLUE" "Starting with: $CURRENT_CLI"
}

# Switch to the other CLI tool
switch_cli() {
    if [[ -z "$AI_CLI_SECONDARY" ]]; then
        print_status "$YELLOW" "No secondary CLI available to switch to"
        return 1
    fi

    if [[ "$CURRENT_CLI" == "$AI_CLI_PRIMARY" ]]; then
        CURRENT_CLI="$AI_CLI_SECONDARY"
    else
        CURRENT_CLI="$AI_CLI_PRIMARY"
    fi

    AI_CLI="$CURRENT_CLI"
    print_status "$GREEN" "Switched to: $CURRENT_CLI"
    return 0
}

# Check if output indicates rate limit or usage limit
is_limit_error() {
    local output="$1"
    local exit_code="$2"

    # Common rate limit / usage limit patterns
    if echo "$output" | grep -qiE "rate.?limit|usage.?limit|limit.?reached|quota.?exceeded|too.?many.?requests|capacity|try.?again.?later|overloaded|max.?usage|conversation.?limit|token.?limit|context.?limit|exceeded.?limit"; then
        return 0
    fi

    # Exit code 75 is sometimes used for rate limiting
    if [[ "$exit_code" -eq 75 ]]; then
        return 0
    fi

    return 1
}

# Check prerequisites
check_prerequisites() {
    print_status "$BLUE" "Checking prerequisites..."

    detect_ai_cli

    # Check for jq
    if ! command -v jq &> /dev/null; then
        print_status "$RED" "Error: jq is not installed"
        print_status "$YELLOW" "Install it with: brew install jq (macOS) or apt install jq (Linux)"
        exit 1
    fi

    # Check for required files
    if [[ ! -f "$PRD_FILE" ]]; then
        print_status "$RED" "Error: $PRD_FILE not found"
        exit 1
    fi

    if [[ ! -f "$PROMPT_FILE" ]]; then
        print_status "$RED" "Error: $PROMPT_FILE not found"
        exit 1
    fi

    # Check if git repo
    if ! git rev-parse --is-inside-work-tree &> /dev/null; then
        print_status "$RED" "Error: Not a git repository"
        print_status "$YELLOW" "Initialize with: git init"
        exit 1
    fi

    print_status "$GREEN" "All prerequisites met!"
}

# Count incomplete stories
count_incomplete_stories() {
    jq '[.stories[] | select(.passes == false)] | length' "$PRD_FILE"
}

# Get next incomplete story
get_next_story() {
    jq -r '.stories[] | select(.passes == false) | .id' "$PRD_FILE" | head -1
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

    print_status "$BLUE" "Working on: $story_id - $story_title"
    print_status "$BLUE" "Using CLI: $CURRENT_CLI"

    local prompt="You are an autonomous agent working on completing a project story.

CRITICAL: Before starting, read these files to understand context:
1. Read $PROMPT_FILE - Project requirements and instructions
2. Read $PROGRESS_FILE - Learnings from previous iterations
3. Read $AGENTS_FILE - Patterns, gotchas, and reusable solutions

WORKING DIRECTORY: $PROJECT_ROOT

CURRENT STORY TO COMPLETE:
ID: $story_id
Title: $story_title
Description: $story_description

ACCEPTANCE CRITERIA (ALL must be met):
- $acceptance_criteria

YOUR WORKFLOW:
1. READ PHASE: Read prompt.md, progress.txt, and AGENTS.md first
2. IMPLEMENT PHASE: Create/modify code, configurations, scripts as needed
3. VERIFY PHASE: Test thoroughly and verify ALL acceptance criteria are met
4. UPDATE STATE PHASE:
   - Update $PRD_FILE: Set passes=true for this story
   - Append learnings to $PROGRESS_FILE
   - Update $AGENTS_FILE with any new patterns or gotchas
5. GIT COMMIT PHASE: Commit all changes with a descriptive message

IMPORTANT RULES:
- Do NOT mark a story as complete unless ALL acceptance criteria are verified
- Document any issues, workarounds, or learnings
- Test each component after implementation
- If you encounter errors, debug and fix them before proceeding"

    cd "$PROJECT_ROOT"
    
    # Capture output and exit code
    local output_file=$(mktemp)
    local exit_code=0
    
    # Use correct CLI syntax based on which tool we're using
    if [[ "$CURRENT_CLI" == "opencode" ]]; then
        # OpenCode uses: opencode run [message..] with -m for model selection
        # Using claude-sonnet-4 via GitHub Copilot provider
        # OPENCODE_PERMISSION accepts JSON config: {"*": "allow"} enables autonomous operation
        OPENCODE_PERMISSION='{"*":"allow"}' opencode run -m "github-copilot/claude-sonnet-4" "$prompt" 2>&1 | tee "$output_file" || exit_code=$?
    else
        # Claude CLI uses: claude --dangerously-skip-permissions -p "$prompt"
        claude --dangerously-skip-permissions -p "$prompt" 2>&1 | tee "$output_file" || exit_code=$?
    fi
    
    local output=$(cat "$output_file")
    rm -f "$output_file"
    
    cd "$SCRIPT_DIR"
    
    # Check if this was a limit error
    if is_limit_error "$output" "$exit_code"; then
        print_status "$YELLOW" "Limit reached on $CURRENT_CLI"
        return 2
    fi
    
    return $exit_code
}

# Main loop
main() {
    print_status "$BLUE" "=========================================="
    print_status "$BLUE" "   Ralph Loop - Autonomous Agent System   "
    print_status "$BLUE" "=========================================="

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

        if [[ -z "$next_story" ]]; then
            print_status "$RED" "Error: Could not determine next story"
            exit 1
        fi

        local retry=1
        local switched_this_story=false
        while [[ $retry -le $MAX_RETRIES ]]; do
            print_status "$BLUE" "Attempt $retry of $MAX_RETRIES for $next_story (using $CURRENT_CLI)"

            run_agent "$next_story"
            local agent_result=$?

            if [[ $agent_result -eq 0 ]]; then
                print_status "$GREEN" "Agent completed successfully"
                break
            elif [[ $agent_result -eq 2 ]]; then
                # Limit reached - try switching CLI
                print_status "$YELLOW" "Limit detected on $CURRENT_CLI"
                
                if switch_cli; then
                    print_status "$GREEN" "Switched to $CURRENT_CLI - retrying immediately"
                    switched_this_story=true
                    # Don't increment retry counter for limit switches
                    continue
                else
                    # No secondary CLI available, fall back to wait and retry
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

        ((iteration++))
    done

    print_status "$YELLOW" "Max iterations reached. Some stories may be incomplete."
    exit 1
}

main "$@"
```

### 2. Create prd.json template

Create a `prd.json` file with the following structure. The user should customize the stories for their project:

```json
{
  "project": "PROJECT_NAME",
  "description": "PROJECT_DESCRIPTION",
  "stories": [
    {
      "id": "Story-1",
      "title": "First Story Title",
      "description": "Detailed description of what needs to be done",
      "acceptance_criteria": [
        "Criterion 1 that must be met",
        "Criterion 2 that must be met",
        "Criterion 3 that must be met"
      ],
      "passes": false
    },
    {
      "id": "Story-2",
      "title": "Second Story Title",
      "description": "Detailed description of what needs to be done",
      "acceptance_criteria": [
        "Criterion 1 that must be met",
        "Criterion 2 that must be met"
      ],
      "passes": false
    }
  ]
}
```

### 3. Create prompt.md with user requirements

Create a `prompt.md` file incorporating the user's project requirements collected earlier:

```markdown
# Project Requirements

## User-Provided Requirements

[INSERT THE USER'S PROJECT REQUIREMENTS HERE - either from their file or their direct input]

## Overview
[You can optionally add clarifying overview based on the user's requirements]

## Architecture
[You can optionally add architectural notes based on the user's requirements]

## Critical Rules

1. **ALWAYS read these files before starting any work:**
   - `prompt.md` - Project requirements (this file)
   - `progress.txt` - Learnings from previous iterations
   - `AGENTS.md` - Patterns and gotchas

2. **Story completion requirements:**
   - ALL acceptance criteria must be met
   - Code/configuration must work without errors
   - Changes must be tested and verified
   - `prd.json` must be updated with `passes: true`
   - Learnings must be appended to `progress.txt`

3. **Never skip verification:**
   - Test each component after implementation
   - Verify all acceptance criteria
   - Check logs for errors
   - Document any issues encountered

## Workflow for Each Iteration

1. **Read Phase** - Read prompt.md, progress.txt, AGENTS.md
2. **Select Phase** - Find first story with passes=false
3. **Implement Phase** - Create code, scripts, configurations
4. **Verify Phase** - Test thoroughly, verify acceptance criteria
5. **Update State Phase** - Update prd.json, progress.txt, AGENTS.md
6. **Git Commit Phase** - Commit with descriptive message

## Useful Commands
[Add project-specific commands here]

## Documentation Links
[Add relevant documentation links]
```

### 4. Create progress.txt

Create an empty `progress.txt` file with a header:

```
# Progress Log
# Each iteration appends learnings below

---
```

### 5. Create AGENTS.md

Create an `AGENTS.md` file with initial structure:

```markdown
# Agent Knowledge Base

This file contains patterns, gotchas, and reusable solutions discovered during project development.

## Working Patterns

### General
- [Add patterns that work well]

## Anti-Patterns

### What to Avoid
- [Add things to avoid]

## Component-Specific Knowledge

### [Component 1]
- [Add component-specific learnings]

## Troubleshooting

### Common Issues
- [Add common issues and solutions]

## Story Dependencies

- Story-1: No dependencies
- Story-2: Depends on Story-1
- [Document story dependencies]

## Iteration Notes

### Iteration 1
- [Notes will be added as iterations complete]
```

### 6. Create .claude/settings.local.json

Create a `.claude/settings.local.json` file with appropriate permissions. Customize based on project needs:

```json
{
  "permissions": {
    "allow": [
      "Bash(git:*)",
      "Bash(npm:*)",
      "Bash(node:*)"
    ]
  }
}
```

## After Setup

1. Make the script executable: `chmod +x ralph-claude.sh`
2. Initialize git if not already: `git init`
3. Customize `prd.json` with your project stories (REQUIRED - add your specific stories)
4. Review and enhance `prompt.md` (already populated with your requirements)
5. Update `.claude/settings.local.json` with needed permissions
6. Run the loop: `./ralph-claude.sh`

**Note**: The `prompt.md` file has been pre-populated with the requirements you provided. You should review it and add any additional architecture details, commands, or documentation links specific to your project.

## Tips for Writing Good Stories

1. **Be Specific**: Each story should have clear, measurable acceptance criteria
2. **Order Dependencies**: Put foundational stories first (e.g., setup before features)
3. **Keep Stories Small**: Smaller stories are easier for agents to complete successfully
4. **Include Verification**: Acceptance criteria should include how to verify completion
5. **Document Learnings**: The agent should update AGENTS.md with patterns and gotchas
