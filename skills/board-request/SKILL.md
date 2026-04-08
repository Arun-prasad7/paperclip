---
name: board-request
description: >
  Request board assistance when encountering permission walls or system-level blockers.
  Creates a board request issue assigned to VP of Engineering, who will research
  the exact steps needed and escalate to board users. Use when you need permissions
  for bash commands, webfetch access, or other system-level capabilities.
---

# Board Request Skill

This skill helps agents escalate permission and system-level blockers to the board through a structured workflow.

## When to Use This Skill

Use this skill when you encounter:
- Permission walls (bash command execution, webfetch access, file system access)
- System configuration blockers
- Security constraints that prevent task completion
- Any situation where board-level intervention is needed

## How It Works

1. **Agent calls skill** with details about what permission/access is needed
2. **Skill creates board request issue** assigned to VP of Engineering
3. **VP of Engineering researches** the exact steps the board needs to take
4. **VP of Engineering documents** instructions and reassigns to board user
5. **Board user executes** the required changes
6. **Original blocked issue** is automatically unblocked once resolved

## Usage

When you hit a permission wall during task execution, invoke this skill:

```bash
# Basic usage
/board-request

# You'll be prompted for:
# - requiredPermission: What access/permission do you need?
# - blockedIssueId: Which issue is blocked? (current issue by default)
# - context: Additional details about what you were trying to do
```

## Skill Behavior

The skill performs the following actions:

1. **Validates input:**
   - Checks that blockedIssueId exists and you have access
   - Validates required fields are provided

2. **Creates board request issue:**
   - Title: "Board request: {requiredPermission}"
   - Parent: {blockedIssueId}
   - Assignee: VP of Engineering
   - Status: todo
   - Priority: high
   - Description includes:
     - Requesting agent name
     - Link to blocked issue
     - Permission needed
     - Context about what was being attempted
     - Instructions for VP of Engineering

3. **Updates blocked issue:**
   - Sets status to "blocked"
   - Adds comment linking to board request issue

4. **Returns result:**
   - Success: Links to both issues
   - Error: Clear error message explaining what went wrong

## Implementation

When you invoke this skill, it will:

### Step 1: Get Current Context

```bash
# Get requesting agent info
curl -sS "$PAPERCLIP_API_URL/api/agents/me" \
  -H "Authorization: Bearer $PAPERCLIP_API_KEY"

# Validate blocked issue exists (if not current issue)
curl -sS "$PAPERCLIP_API_URL/api/issues/{blockedIssueId}" \
  -H "Authorization: Bearer $PAPERCLIP_API_KEY"
```

### Step 2: Create Board Request Issue

```bash
curl -sS "$PAPERCLIP_API_URL/api/companies/$PAPERCLIP_COMPANY_ID/issues" \
  -H "Authorization: Bearer $PAPERCLIP_API_KEY" \
  -H "X-Paperclip-Run-Id: $PAPERCLIP_RUN_ID" \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Board request: {requiredPermission}",
    "description": "## Permission Request\n\n**Requested by:** {agent.name}\n**Blocked issue:** [{blockedIssue.identifier}](/{PREFIX}/issues/{blockedIssue.identifier})\n**Permission needed:** {requiredPermission}\n\n## Context\n\n{context}\n\n## Instructions for VP of Engineering\n\nPlease research exactly what the board needs to do to grant this permission:\n\n- Step-by-step instructions\n- Code changes needed\n- Bash commands to execute\n\nAdd these instructions as a document to this issue, then reassign to the board user.",
    "status": "todo",
    "priority": "high",
    "parentId": "{blockedIssueId}",
    "assigneeAgentId": "c3879577-5195-402f-80a1-63add1ddce60",
    "projectId": "{blockedIssue.projectId}",
    "goalId": "{blockedIssue.goalId}"
  }'
```

### Step 3: Update Blocked Issue

```bash
curl -sS "$PAPERCLIP_API_URL/api/issues/{blockedIssueId}" \
  -X PATCH \
  -H "Authorization: Bearer $PAPERCLIP_API_KEY" \
  -H "X-Paperclip-Run-Id: $PAPERCLIP_RUN_ID" \
  -H "Content-Type: application/json" \
  -d '{
    "status": "blocked",
    "comment": "## Blocked: Permission Required\n\nThis issue is blocked pending board approval for required permissions.\n\n**Board request:** [{boardRequestIdentifier}](/{PREFIX}/issues/{boardRequestIdentifier})\n\nWaiting for VP of Engineering to research and escalate to board."
  }'
```

## VP of Engineering Constants

The VP of Engineering agent ID is hardcoded in this skill:
- **Agent ID:** `c3879577-5195-402f-80a1-63add1ddce60`
- **Name:** VP of Engineering
- **Title:** VP of Engineering & CTO

If this agent changes, update the skill implementation.

## Error Handling

The skill handles these error cases:

- **Invalid blockedIssueId:** Returns error if issue doesn't exist or agent lacks access
- **Missing required fields:** Returns error listing which fields are required
- **API failures:** Returns error with details about what failed
- **Network issues:** Returns error suggesting retry

## Examples

### Example 1: Bash Permission

```
Agent: "I need to run bash commands to execute tests but permission is denied"

Skill creates:
- Board request: "Board request: bash command execution"
- Blocked issue updated with link to board request
- VP of Engineering assigned to research and document steps
```

### Example 2: WebFetch Access

```
Agent: "I need webfetch access to call the GitHub API"

Skill creates:
- Board request: "Board request: webfetch access for GitHub API"
- Includes context about which API endpoint is needed
- VP of Engineering will document how to configure webfetch permissions
```

## Testing

To test this skill:

1. Create a test issue
2. Invoke the skill with test parameters
3. Verify board request issue is created
4. Verify blocked issue is updated
5. Check that VP of Engineering receives the assignment
6. Validate all links work correctly

## Notes

- Always provide clear context about what you were trying to accomplish
- Be specific about which permission or capability you need
- If you're not sure what permission name to use, describe the action you were trying to take
- The VP of Engineering will translate your request into specific technical requirements
