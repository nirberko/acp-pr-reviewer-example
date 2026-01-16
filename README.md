# ACP PR Reviewer Example

An example project demonstrating how to use [ACP (Agent as a Code Protocol)](https://github.com/acp-org/acp) to create an AI-powered pull request reviewer that automatically reviews code changes using GitHub Actions.

## What is ACP?

**ACP (Agent as a Code Protocol)** is an Infrastructure as Code approach for AI agents. Instead of writing imperative code to manage agent state, retries, and tool wiring, ACP lets you describe your AI agents declaratively using a native schema format.

ACP provides:
- **Native Schema**: Type-safe `.acp` format with explicit references
- **Multi-Provider Support**: Use OpenAI, Anthropic, or other LLM providers
- **MCP Integration**: Connect to external tools via Model Context Protocol servers
- **Policy Enforcement**: Set budgets, timeouts, and capability limits per agent
- **Workflow Orchestration**: Coordinate multi-step agent workflows with conditional routing

Learn more about ACP at: https://github.com/acp-org/acp

## About This Repository

This repository demonstrates a practical use case of ACP: **automated PR reviews using AI agents**. The example includes:

- A complete ACP specification defining an AI code reviewer agent
- Integration with GitHub's MCP server for accessing PR data
- A GitHub Actions workflow that automatically runs reviews on pull requests
- A multi-step workflow that fetches PR data, analyzes code, and submits reviews

## How It Works

The PR reviewer workflow consists of four steps:

1. **Fetch PR Data**: Retrieves the pull request information using GitHub MCP capabilities
2. **Fetch PR Files**: Gets the list of changed files in the pull request
3. **Analyze**: Uses an AI agent (GPT-4o) to review the code changes and provide feedback
4. **Submit Review**: Posts the AI-generated review as a comment on the pull request

All of this is defined declaratively in `.acp` files, making it easy to version control, review, and modify the agent configuration.

## Project Structure

```
.acp/
├── 00-project.acp      # Project metadata and version
├── 01-variables.acp    # Variable definitions (API keys, tokens)
├── 02-providers.acp    # LLM provider configurations
├── 03-servers.acp      # MCP server connections (GitHub)
├── 04-capabilities.acp # Capability definitions for GitHub operations
├── 05-policies.acp     # Budget and policy constraints
├── 06-agents.acp       # AI agent definitions (reviewer)
└── 07-workflows.acp    # Workflow definitions (review_pr)

.github/
└── workflows/
    └── acp-pr-review.yml  # GitHub Actions workflow
```

## Prerequisites

- Python 3.12 or higher
- GitHub repository with Actions enabled
- OpenAI API key
- GitHub Personal Access Token (with `repo` scope for PR access)

## Setup

### 1. Install ACP CLI

```bash
pip install acp-cli
```

### 2. Configure Secrets

In your GitHub repository, add the following secrets (Settings → Secrets and variables → Actions):

- `OPENAI_API_KEY`: Your OpenAI API key
- `GITHUB_TOKEN`: Automatically provided by GitHub Actions (or use a PAT with `repo` scope)

### 3. Verify ACP Configuration

Validate your ACP specification:

```bash
acp validate .acp/*.acp
```

### 4. Test Locally (Optional)

You can test the workflow locally before setting up GitHub Actions:

```bash
cd .acp

acp run review_pr \
  --var openai_api_key="your-openai-key" \
  --var github_personal_access_token="your-github-token" \
  --var owner="your-username" \
  --var repo="your-repo" \
  --var pr_number=1
```

## Usage

### Automatic Reviews via GitHub Actions

Once set up, the PR reviewer will automatically run when:

- A new pull request is opened
- A pull request is synchronized (new commits pushed)
- A pull request is reopened

The workflow runs automatically via the GitHub Actions configuration in `.github/workflows/acp-pr-review.yml`.

### Manual Execution

You can also run the reviewer manually from the command line:

```bash
cd .acp

acp run review_pr \
  --var openai_api_key="$OPENAI_API_KEY" \
  --var github_personal_access_token="$GITHUB_TOKEN" \
  --var owner="acp-org" \
  --var repo="acp-pr-reviewer-example" \
  --var pr_number=123
```

## Configuration

### Customizing the Reviewer

Edit `.acp/06-agents.acp` to modify the reviewer's instructions:

```acp
agent "reviewer" {
  model = model.gpt4o

  instructions = <<EOF
Your custom review instructions here...
EOF

  allow  = [capability.get_pr, capability.list_pr_files, capability.create_review]
  policy = policy.review_policy
}
```

### Adjusting Policies

Modify `.acp/05-policies.acp` to set budgets and constraints:

```acp
policy "review_policy" {
  budgets { max_cost_usd_per_run = 0.50 }
  budgets { timeout_seconds = 60 }
}
```

### Changing Models

Update `.acp/02-providers.acp` to use different LLM models or providers.

## Features

- 🤖 **AI-Powered Reviews**: Uses GPT-4o to provide intelligent code feedback
- 🔗 **GitHub Integration**: Seamlessly connects to GitHub via MCP
- ⚙️ **Declarative Configuration**: All agent logic defined in readable `.acp` files
- 🔄 **Automated Workflow**: Runs automatically on PR events via GitHub Actions
- 💰 **Cost Controls**: Built-in budget limits and policy enforcement
- 📊 **Version Controlled**: Agent configurations are code, making changes reviewable

## How ACP Differs from Traditional Approaches

**Traditional Approach:**
```python
# Imperative code managing state, retries, error handling...
async def review_pr(pr_number):
    pr = await fetch_pr(pr_number)
    files = await fetch_files(pr_number)
    review = await llm.review(code=files)
    await post_review(pr_number, review)
```

**ACP Approach:**
```acp
workflow "review_pr" {
  step "fetch_pr" { ... }
  step "fetch_files" { ... }
  step "analyze" { agent = agent.reviewer }
  step "submit_review" { ... }
}
```

The ACP approach provides:
- **Type Safety**: Schema validation catches errors early
- **Reusability**: Agents and workflows are composable
- **Visibility**: Clear data flow through explicit references
- **Maintainability**: Configuration changes don't require code changes

## Contributing

Contributions are welcome! This is an example repository demonstrating ACP capabilities. Feel free to:

- Improve the review quality and prompts
- Add support for more review criteria
- Extend the workflow with additional steps
- Add examples of other ACP patterns

## Resources

- [ACP Documentation](https://github.com/acp-org/acp)
- [ACP Schema Reference](https://github.com/acp-org/acp)
- [Model Context Protocol (MCP)](https://modelcontextprotocol.io/)

## License

This project is licensed under the MIT License - see the LICENSE file for details.

