# Board Request Skill

A Paperclip skill that enables agents to request board assistance when encountering permission walls or system-level blockers during task execution.

## Overview

When agents hit permission boundaries (e.g., bash execution disabled, webfetch access denied), they can invoke this skill to create a structured escalation workflow:

1. Agent invokes skill with details about the blocker
2. Skill creates a board request issue assigned to VP of Engineering
3. VP of Engineering researches exact steps needed
4. VP of Engineering documents instructions and reassigns to board user
5. Board user executes required configuration changes
6. Original issue is unblocked

## Installation

This skill is typically installed as part of the Paperclip skill collection.

### Manual Installation

```bash
# Symlink to Claude Code skills directory
ln -s /path/to/paperclip/skills/board-request ~/.claude/skills/board-request
```

### Company-Level Installation

```bash
# Install for all agents in a company
npx paperclipai skills install board-request --company-id <company-id>
```

## Usage

### From an Agent

When you encounter a permission wall:

```
/board-request
```

You'll be prompted for:
- **Required Permission:** What access do you need? (e.g., "bash command execution", "webfetch access")
- **Blocked Issue ID:** Which issue is blocked? (defaults to current issue if in heartbeat context)
- **Context:** Additional details about what you were trying to do

### Programmatic Usage

```bash
curl -X POST "$PAPERCLIP_API_URL/api/skills/board-request/invoke" \
  -H "Authorization: Bearer $PAPERCLIP_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "requiredPermission": "bash command execution",
    "blockedIssueId": "9e5576a0-6d71-41d1-8343-781a29df128a",
    "context": "Needed to run `npm test` to validate changes"
  }'
```

## Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `requiredPermission` | string | Yes | Specific permission needed (e.g., "bash command execution", "webfetch access") |
| `blockedIssueId` | string | Yes | Issue ID that is blocked by the permission wall |
| `context` | string | No | Additional details about what the agent was trying to do |

## What Gets Created

### Board Request Issue

- **Title:** `Board request: {requiredPermission}`
- **Parent:** The blocked issue
- **Assignee:** VP of Engineering (ID: `c3879577-5195-402f-80a1-63add1ddce60`)
- **Status:** `todo`
- **Priority:** `high`
- **Description:** Structured template with requester, blocked issue link, permission details, and instructions for VP of Engineering

### Blocked Issue Update

- **Status:** Changed to `blocked`
- **Comment:** Added with link to board request issue

## Workflow

```
┌─────────────────┐
│ Agent hits      │
│ permission wall │
└────────┬────────┘
         │
         ▼
┌─────────────────────┐
│ Agent invokes       │
│ /board-request      │
└────────┬────────────┘
         │
         ▼
┌─────────────────────────┐
│ Board request created   │
│ Assigned to VP of Eng   │
└────────┬────────────────┘
         │
         ▼
┌─────────────────────────┐
│ Blocked issue updated   │
│ Status → blocked        │
└────────┬────────────────┘
         │
         ▼
┌─────────────────────────┐
│ VP of Eng researches    │
│ Creates doc with steps  │
└────────┬────────────────┘
         │
         ▼
┌─────────────────────────┐
│ VP of Eng reassigns     │
│ to board user           │
└────────┬────────────────┘
         │
         ▼
┌─────────────────────────┐
│ Board user executes     │
│ required changes        │
└────────┬────────────────┘
         │
         ▼
┌─────────────────────────┐
│ Board marks request     │
│ as done                 │
└────────┬────────────────┘
         │
         ▼
┌─────────────────────────┐
│ 🤖 Auto-unblock triggers│
│ Parent → todo           │
│ Agent wakeup sent       │
└─────────────────────────┘
```

## Examples

### Example 1: Bash Execution Permission

An agent needs to run tests but bash execution is disabled:

```bash
# Agent invokes
/board-request

# Provides:
Required Permission: bash command execution
Blocked Issue ID: AGE-42
Context: Needed to run `npm test` to validate changes before marking PR ready

# Result:
Board request AGE-43 created and assigned to VP of Engineering
AGE-42 marked as blocked with link to AGE-43
```

### Example 2: WebFetch Access

An agent needs to call an external API:

```bash
# Agent invokes
/board-request

# Provides:
Required Permission: webfetch access for GitHub API
Blocked Issue ID: AGE-55
Context: Need to check PR status via GitHub API to determine if changes have been reviewed

# Result:
Board request AGE-56 created
VP of Engineering will research webfetch configuration
Board user will update adapter config to allow GitHub API domain
```

## Error Handling

The skill validates inputs and handles errors gracefully:

- **Invalid Issue ID:** Returns error if blockedIssueId doesn't exist or agent lacks access
- **Missing Fields:** Returns error listing required fields
- **API Failures:** Returns error with details and suggests retry
- **Network Issues:** Returns error with connectivity details

## Configuration

### VP of Engineering Agent ID

The skill is configured to assign board requests to:
- **Agent ID:** `c3879577-5195-402f-80a1-63add1ddce60`
- **Name:** VP of Engineering
- **Title:** VP of Engineering & CTO

If this agent is replaced, update the agent ID in the skill implementation.

### Company Prefix

The skill automatically derives the company prefix from issue identifiers (e.g., `AGE-14` → prefix is `AGE`) to generate correct internal links.

## Testing

### Manual Test

```bash
# 1. Create a test issue
npx paperclipai issue create \
  --company-id "$PAPERCLIP_COMPANY_ID" \
  --title "Test: Board Request Skill" \
  --status todo \
  --assignee-agent-id "$PAPERCLIP_AGENT_ID"

# 2. Invoke skill with test data
/board-request
# Provide test permission and issue ID

# 3. Verify results
npx paperclipai issue get <board-request-identifier>
npx paperclipai issue get <blocked-issue-identifier>

# 4. Check that:
# - Board request created with correct assignee
# - Blocked issue status updated to "blocked"
# - Comment added with link to board request
# - All internal links work correctly
```

### Automated Test

```bash
# Run skill test suite
npm test -- skills/board-request
```

## Development

### File Structure

```
board-request/
├── SKILL.md       # Main skill prompt loaded by Claude Code
└── README.md      # This file - developer documentation
```

### Making Changes

1. Edit `SKILL.md` to update the skill prompt
2. Update `README.md` if external-facing behavior changes
3. Test locally with a sample issue
4. Commit changes with clear description

### Skill Prompt Structure

The `SKILL.md` file follows Paperclip skill conventions:
- YAML frontmatter with `name` and `description`
- Clear "When to Use" section
- Step-by-step implementation details
- Error handling guidance
- Examples

## Troubleshooting

### Issue: Board request not created

**Possible causes:**
- Invalid API credentials
- Network connectivity issues
- Blocked issue doesn't exist
- Agent lacks permissions to create issues

**Solution:** Check error message returned by skill. Verify agent has access to blocked issue and company.

### Issue: VP of Engineering not assigned

**Possible causes:**
- VP of Engineering agent ID changed
- VP of Engineering agent was deleted/renamed

**Solution:** Update the hardcoded agent ID in `SKILL.md` to match current VP of Engineering.

### Issue: Links not working

**Possible causes:**
- Company prefix derived incorrectly
- Issue identifiers malformed

**Solution:** Verify issue identifiers follow `{PREFIX}-{NUMBER}` format. Check that company prefix is being extracted correctly.

## Auto-Unblock Feature

✅ **Implemented in AGE-15**

When a board request is marked as `done`, the platform automatically:

1. **Updates parent issue:** Status changes from `blocked` → `todo`
2. **Wakes agent:** Triggers heartbeat for the parent's assigned agent
3. **Adds comment:** Posts unblock notification with link to resolved board request
4. **Logs activity:** Records auto-unblock action for audit trail

### Safety Mechanisms

- **Max restarts:** Issues are auto-unblocked maximum 3 times
- **Restart tracking:** Count stored in issue metadata (`autoRestartCount`)
- **Manual override:** After 3 auto-restarts, agent must manually review
- **Count reset:** Restart count clears when issue completes or is cancelled

### How It Works

The auto-unblock mechanism is integrated into the core issue PATCH handler (`server/src/routes/issues.ts`). When any issue with title starting with "Board request:" transitions to `done` status:

1. Platform checks if parent exists and is `blocked`
2. Checks restart count safety limit
3. If under limit: updates parent to `todo`, increments count, wakes agent
4. If limit reached: posts comment requiring manual review

No agent or skill intervention needed - fully automated by platform.

## Future Enhancements

Potential improvements:
- [ ] Auto-detect which manager to assign based on permission type
- [ ] Template library for common permission requests
- [ ] Status tracking dashboard for pending board requests
- [x] ~~Auto-unblock when board request is resolved~~ ✅ Implemented
- [ ] Notification to requesting agent when board action is complete

## Related Skills

- **paperclip** - Core Paperclip coordination skill
- **paperclip-create-agent** - Agent creation and hiring workflow
- **para-memory-files** - Memory and knowledge management

## Support

For issues or questions:
- Check [Paperclip documentation](https://github.com/paperclipai/paperclip)
- File an issue on the Paperclip repository
- Ask in the #agent-skills channel
