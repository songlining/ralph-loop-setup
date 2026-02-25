# Ralph Loop Setup Skill - Installation & Update Guide

This guide covers how to install and update the ralph-loop-setup skill in Claude Code.

## Initial Installation

### Step 1: Add Local Marketplace

First, add your local skills directory as a marketplace:

```bash
claude plugin marketplace add /Users/larry.song/work/hashicorp/local_skills/ralph-loop-setup
```

This registers the directory as a source for plugins. The marketplace configuration is stored in:
- `/Users/larry.song/work/hashicorp/local_skills/ralph-loop-setup/.claude-plugin/marketplace.json`

### Step 2: Install the Plugin

Install the ralph-loop-setup skill from your local marketplace:

```bash
claude plugin install ralph-loop-setup@local-ralph-skills
```

### Step 3: Verify Installation

Check that the plugin is installed:

```bash
claude plugin marketplace list
```

You should see `local-ralph-skills` in the list of configured marketplaces.

## Using the Skill

Once installed, the skill is automatically available. You can trigger it by asking:

- "Set up Ralph Loop for my project"
- "Create an autonomous agent workflow"
- "Help me set up iterative agent development"
- "Initialize Ralph Loop in this directory"

The skill will create:
- **ralph-claude.sh** - The orchestrator script
- **prd.json** - Product Requirements Document template
- **prompt.md** - Project requirements template
- **progress.txt** - Iteration log
- **AGENTS.md** - Agent knowledge base
- **.claude/settings.local.json** - Claude Code permissions

## Updating the Skill

When you make changes to the skill files, follow these steps to update the installed version:

### Method 1: Update via Marketplace (Recommended)

```bash
# Update the marketplace to pick up changes
claude plugin marketplace update local-ralph-skills

# Update the plugin
claude plugin update ralph-loop-setup
```

### Method 2: Reinstall

```bash
# Uninstall the current version
claude plugin uninstall ralph-loop-setup

# Reinstall from marketplace
claude plugin install ralph-loop-setup@local-ralph-skills
```

### Method 3: Development Mode (No Installation)

For active development, use the skill directly without installation:

```bash
# Load skill from directory for this session only
claude --plugin-dir /Users/larry.song/work/hashicorp/local_skills/ralph-loop-setup/ralph-loop-setup
```

This approach loads the skill dynamically, so any changes are immediately available without reinstalling.

## Validation

Before updating, you can validate your changes:

```bash
claude plugin validate /Users/larry.song/work/hashicorp/local_skills/ralph-loop-setup/ralph-loop-setup
```

This checks that the skill structure and metadata are correct.

## Repackaging (Optional)

If you need to create a distributable `.skill` file after updates:

```bash
cd /Users/larry.song/work/hashicorp/local_skills/ralph-loop-setup
zip -r ralph-loop-setup.skill ralph-loop-setup/
```

This creates an updated `ralph-loop-setup.skill` file that can be shared with others.

## Marketplace Configuration

The marketplace is configured in `.claude-plugin/marketplace.json`:

```json
{
  "name": "local-ralph-skills",
  "owner": {
    "name": "Larry Song",
    "email": "larry.song@example.com"
  },
  "metadata": {
    "description": "Ralph Loop autonomous agent iteration system skills",
    "version": "1.0.0"
  },
  "plugins": [
    {
      "name": "ralph-loop-setup",
      "description": "Set up the Ralph Loop autonomous agent iteration system for new projects",
      "source": "./",
      "strict": false,
      "skills": [
        "./ralph-loop-setup"
      ]
    }
  ]
}
```

## Skill Directory Structure

```
ralph-loop-setup/
├── .claude-plugin/
│   └── marketplace.json           # Marketplace configuration
├── ralph-loop-setup/
│   └── SKILL.md                   # Skill definition and instructions
├── ralph-loop-setup.skill         # Packaged skill (ZIP archive)
└── INSTALLATION.md                # This file
```

## Quick Reference

| Action | Command |
|--------|---------|
| Add marketplace | `claude plugin marketplace add /path/to/ralph-loop-setup` |
| Install skill | `claude plugin install ralph-loop-setup@local-ralph-skills` |
| Update marketplace | `claude plugin marketplace update local-ralph-skills` |
| Update skill | `claude plugin update ralph-loop-setup` |
| Uninstall skill | `claude plugin uninstall ralph-loop-setup` |
| Validate skill | `claude plugin validate /path/to/ralph-loop-setup/ralph-loop-setup` |
| List marketplaces | `claude plugin marketplace list` |
| Dev mode (no install) | `claude --plugin-dir /path/to/ralph-loop-setup/ralph-loop-setup` |

## Troubleshooting

### Skill not triggering

- Ensure the plugin is installed: `claude plugin marketplace list`
- Try reinstalling: `claude plugin uninstall ralph-loop-setup && claude plugin install ralph-loop-setup@local-ralph-skills`
- Check the skill description in SKILL.md matches your use case

### Changes not appearing

- Update the marketplace: `claude plugin marketplace update local-ralph-skills`
- Update the plugin: `claude plugin update ralph-loop-setup`
- Or use dev mode: `claude --plugin-dir /path/to/ralph-loop-setup/ralph-loop-setup`

### Validation errors

- Run validation: `claude plugin validate /path/to/ralph-loop-setup/ralph-loop-setup`
- Check SKILL.md has proper YAML frontmatter (name and description)
- Ensure all referenced files exist in the skill directory

## Sharing the Skill

To share the skill with others:

1. **Package the skill**:
   ```bash
   cd /path/to/ralph-loop-setup
   zip -r ralph-loop-setup.skill ralph-loop-setup/
   ```

2. **Share the .skill file**:
   - Send `ralph-loop-setup.skill` to others
   - They can install it by creating their own marketplace or using `--plugin-dir`

3. **Or share the source directory**:
   - Others can add your directory as a marketplace
   - Or package it themselves

## What Ralph Loop Creates

When you use this skill, it sets up the following in your project:

| File | Purpose |
|------|---------|
| `ralph-claude.sh` | Main orchestrator script that runs the agent loop |
| `prd.json` | Product Requirements Document with stories and acceptance criteria |
| `prompt.md` | Project requirements and agent instructions |
| `progress.txt` | Log of learnings from each iteration |
| `AGENTS.md` | Knowledge base with patterns and gotchas |
| `.claude/settings.local.json` | Claude Code permissions for the project |

## Running Ralph Loop

After setup:

1. Make the script executable: `chmod +x ralph-claude.sh`
2. Initialize git if needed: `git init`
3. Customize `prd.json` with your project stories
4. Customize `prompt.md` with your requirements
5. Update permissions in `.claude/settings.local.json`
6. Run: `./ralph-claude.sh`

## Support

For issues or questions:
- Check the skill documentation in `ralph-loop-setup/SKILL.md`
- Review the generated files after running the skill
- Validate the skill structure before updating
