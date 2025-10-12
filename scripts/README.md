# Azure DevOps Scripts Validation

This directory contains scripts for interacting with Azure DevOps work items and pull requests, designed to be used by the GitHub Copilot agent.

## 📋 Scripts Overview

### Work Item Management
- `get-workitem.sh` - Retrieve work item details
- `add-comment-to-workitem.sh` - Add comments to work items
- `assign-workitem.sh` - Assign work items to users
- `update-workitem-state.sh` - Update work item state
- `update-workitem-activity.sh` - Update work item activity

### Pull Request Management
- `create-pr-with-required-reviewer.sh` - Create PRs with required reviewers
- `add-required-reviewer.sh` - Add required reviewers to existing PRs
- `pipeline-create-pr.sh` - Pipeline-specific PR creation

### Linking (Azure DevOps REST API)
- `link-branch-to-workitem.sh` - Link Git branches to work items using Azure DevOps REST API
- `link-pr-to-workitem.sh` - Link Pull Requests to work items using Azure DevOps REST API

### Testing
- `test-full-workflow.sh` - Test the complete workflow
- `test-scripts-manual.sh` - Interactive manual testing
- `validate-all-scripts.sh` - Automated validation

## 🔗 Linking Work Items with Azure DevOps REST API

The linking scripts use **Azure DevOps REST API** for reliable work item artifact linking. While Azure CLI is used to get repository and project IDs, the actual linking is done via REST API because `az boards work-item relation` only supports work item relations (parent/child), not artifact links (branches/PRs).

### Why REST API for Linking?

✅ **Direct artifact support** - `ArtifactLink` relation type is only available via REST API  
✅ **Azure CLI for metadata** - We use `az repos show` and `az devops project show` to get IDs  
✅ **Proper URI format** - Correct vstfs:// URIs for branches and PRs  
✅ **Better error handling** - HTTP status codes and response bodies for debugging

### Link a Branch to a Work Item

```bash
./scripts/link-branch-to-workitem.sh <work-item-id> <project-name> <repo-name> <branch-name>
```

**Example:**
```bash
export AZURE_DEVOPS_PAT="your-pat-token"
./scripts/link-branch-to-workitem.sh 123 "My Project" "MyRepo" "copilot/123"
```

### Link a Pull Request to a Work Item

```bash
./scripts/link-pr-to-workitem.sh <work-item-id> <project-name> <repo-name> <pr-id>
```

**Example:**
```bash
export AZURE_DEVOPS_PAT="your-pat-token"
./scripts/link-pr-to-workitem.sh 123 "My Project" "MyRepo" 456
```

### How It Works

Both scripts:
1. Get the repository ID using `az repos show`
2. Get the project ID using `az devops project show`
3. Construct the appropriate artifact URI with URL encoding:
   - Branch: `vstfs:///Git/Ref/{ProjectId}%2F{RepositoryId}%2FGB{BranchName}`
     - `GB` = Git Branch prefix
     - `%2F` = URL-encoded separator (instead of `/`)
     - Branch slashes are also encoded (e.g., `copilot/123` → `copilot%2F123`)
   - PR: `vstfs:///Git/PullRequestId/{ProjectId}%2F{RepositoryId}%2F{PullRequestId}`
     - All separators use `%2F` encoding
4. Link using REST API PATCH request to work items endpoint

The REST API uses the `ArtifactLink` relation type which is specifically designed for linking branches, PRs, and other artifacts to work items.

## 🧪 Testing the Scripts

Before running the agent workflow, you can validate that all scripts work correctly. We provide two testing approaches:

### Option 1: Interactive Manual Testing (Recommended for first-time testing)

This script runs each command step-by-step, showing you the exact commands and asking for confirmation before making changes.

1. **Setup** (one-time):
   ```bash
   cp .env.example .env
   # Edit .env with your credentials
   ```

2. **Run the interactive test**:
   ```bash
   ./scripts/test-scripts-manual.sh
   ```

   This will:
   - ✅ Load your configuration from `.env`
   - ✅ Test `get-workitem.sh` (read work item details)
   - ✅ Test `add-comment-to-workitem.sh` (add a test comment)
   - ⚠️ Ask before running state-changing commands (assign, update state, etc.)
   - 🛑 Pause between tests so you can review the output

### Option 2: Automated Validation

This script automatically tests all read-only operations without user interaction.

```bash
./scripts/validate-all-scripts.sh
```

This will:
- ✅ Validate environment variables
- ✅ Test base64 encoding (critical!)
- ✅ Test URL encoding for project names
- ✅ Test `get-workitem.sh`
- ✅ Test `add-comment-to-workitem.sh`
- ⏭️ Skip state-changing operations

### Setup

1. **Copy the example environment file:**
   ```bash
   cp .env.example .env
   ```

2. **Edit `.env` with your Azure DevOps credentials:**
   ```bash
   # .env
   AZURE_DEVOPS_PAT=your-personal-access-token
   SYSTEM_COLLECTIONURI=https://dev.azure.com/your-org/
   SYSTEM_TEAMPROJECT=Your Project Name
   TEST_WORK_ITEM_ID=123  # Use an existing work item for testing
   ```

3. **Make the validation script executable:**
   ```bash
   chmod +x scripts/validate-all-scripts.sh
   ```

### Run the Validation

```bash
./scripts/validate-all-scripts.sh
```

The script will test:
- ✅ Environment variables are set correctly
- ✅ Organization extraction from SYSTEM_COLLECTIONURI
- ✅ Base64 encoding (critical for authentication)
- ✅ URL encoding for project names with spaces
- ✅ `get-workitem.sh` - Read work item details
- ✅ `add-comment-to-workitem.sh` - Add comment to work item
- ⚠️ Other scripts (read-only mode to avoid modifying work items)

### Expected Output

```
========================================
🧪 Azure DevOps Scripts Validation
========================================

▶ Loading environment variables from .env
✅ Environment variables loaded

========================================
STEP 0: Validate Environment Variables
========================================

✅ AZURE_DEVOPS_PAT is set (52 characters)
✅ SYSTEM_COLLECTIONURI is set: https://dev.azure.com/returngisorg/
✅ SYSTEM_TEAMPROJECT is set: GitHub Copilot CLI
✅ TEST_WORK_ITEM_ID is set: 416
...
```

## 📝 Script Descriptions

### Core Scripts (Used by Agent)

1. **`get-workitem.sh`** - Retrieves work item details
   ```bash
   # Using environment variable
   export SYSTEM_COLLECTIONURI=https://dev.azure.com/your-org/
   export SYSTEM_TEAMPROJECT="Your Project"
   export AZURE_DEVOPS_PAT=your-token
   ./scripts/get-workitem.sh 123
   
   # Or specify project explicitly
   ./scripts/get-workitem.sh 123 "Your Project Name"
   ```

2. **`add-comment-to-workitem.sh`** - Adds a comment to a work item
   ```bash
   ./scripts/add-comment-to-workitem.sh your-org "Your Project" 123 "Your comment here"
   
   # HTML format is supported
   ./scripts/add-comment-to-workitem.sh your-org "Your Project" 123 "<b>Bold text</b><br/>Line break"
   ```

3. **`assign-workitem.sh`** - Assigns a work item to a user
   ```bash
   ./scripts/assign-workitem.sh your-org "Your Project" 123 "user@example.com"
   
   # Or assign to the bot
   ./scripts/assign-workitem.sh your-org "Your Project" 123 "GitHub Copilot CLI"
   ```

4. **`update-workitem-state.sh`** - Updates work item state
   ```bash
   # Common states: "To Do", "Doing", "Done"
   ./scripts/update-workitem-state.sh your-org "Your Project" 123 "Doing"
   ```

5. **`update-workitem-activity.sh`** - Updates work item activity field
   ```bash
   # Valid activities: Development, Design, Documentation, Deployment, Testing, Requirements
   ./scripts/update-workitem-activity.sh your-org "Your Project" 123 "Development"
   ```

6. **`create-pr-with-required-reviewer.sh`** - Creates a PR with required reviewer
   ```bash
   # Get repository ID first
   REPO_ID=$(az repos show --repository "Your Repo" \
     --organization https://dev.azure.com/your-org --output json | jq -r '.id')
   
   # Create PR
   ./scripts/create-pr-with-required-reviewer.sh \
     your-org "Your Project" "$REPO_ID" \
     "feature-branch" "main" \
     "PR Title" "PR Description" \
     "reviewer@example.com"
   ```

### Helper Scripts

7. **`add-required-reviewer.sh`** - Adds a required reviewer to an existing PR

8. **`test-scripts-manual.sh`** - Interactive manual testing of all scripts

9. **`validate-all-scripts.sh`** - Automated validation of all scripts

## 🔒 Security

- **Never commit `.env`** - It contains your PAT token
- `.env` is already in `.gitignore`
- Use `.env.example` as a template
- PAT tokens should have appropriate scopes:
  - Work Items: Read & Write
  - Code: Read & Write
  - Pull Requests: Read & Write

## 🐛 Troubleshooting

### Exit Code 43

If you see exit code 43, it's likely a base64 encoding issue:
- The scripts now handle both Linux (GNU) and macOS (BSD) base64
- This is fixed automatically in the current version

### Project Name with Spaces

Project names with spaces (e.g., "GitHub Copilot CLI") are automatically URL-encoded:
- Uses `jq -sRr @uri` if available
- Falls back to `sed 's/ /%20/g'` otherwise

### Environment Variables

Azure Pipelines uses UPPERCASE variables:
- `SYSTEM_COLLECTIONURI` (not `System_CollectionUri`)
- `SYSTEM_TEAMPROJECT` (not `System_TeamProject`)

Scripts support both formats for compatibility.

## 📚 Agent Workflow Order

The GitHub Copilot agent follows this workflow:

```
0. Extract organization from SYSTEM_COLLECTIONURI
1. Read work item details (get-workitem.sh)
2. Add initial comment 👀🤖 (add-comment-to-workitem.sh)
3. Create branch: copilot/<work-item-id>
4. Assign work item to "GitHub Copilot CLI" (assign-workitem.sh)
5. Update state to "Doing" (update-workitem-state.sh)
6. Analyze and implement changes
7. Push branch to Azure DevOps
8. Create PR with required reviewer (create-pr-with-required-reviewer.sh)
9. Update activity field (update-workitem-activity.sh)
```

## 🎯 Next Steps

After validation succeeds:
1. Commit your changes (excluding `.env`)
2. Push to Azure DevOps
3. Run the pipeline with the agent
4. The agent will use these scripts automatically

For more details, see `../.github/copilot-instructions.md`
