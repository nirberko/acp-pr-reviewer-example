---
title: How I Built an AI-Powered PR Reviewer in 15 Minutes Using ACP
published: false
description: Learn how to create an automated AI code reviewer using ACP (Agent as a Code Protocol) - no Python boilerplate needed, just declarative configuration that runs on GitHub Actions.
tags: ai, github, automation, devops
cover_image: https://dev-to-uploads.s3.amazonaws.com/uploads/articles/og522g9fhl19wmdcxhy7.png
# Use a ratio of 100:42 for best results.
# published_at: 2026-01-16 11:21 +0000
---

## How I Built an AI-Powered PR Reviewer in 15 Minutes Using ACP

*No Python. No complex orchestration. Just declarative configuration.*

There are plenty of AI-powered PR review tools out there, but I wanted to build my own - partly for more control over the review process, and partly just for fun and to learn a new technology. The challenge? Building one traditionally meant writing a lot of imperative code: managing API calls, handling retries, orchestrating workflows, and dealing with error handling. That's a lot of boilerplate for what should be a simple task.

Using **ACP (Agent as a Code Protocol)**, I built a fully functional AI PR reviewer in just 15 minutes that runs automatically on GitHub Actions. Here's how I did it.

## What is ACP?

**ACP (Agent as a Code Protocol)** is Infrastructure as Code for AI agents. Instead of writing imperative Python or JavaScript to manage agent state, retries, and tool wiring, ACP lets you describe your AI agents **declaratively** using a native schema format.

Think of it like Terraform, but for AI workflows. You define *what* you want, not *how* to do it.

ACP is an emerging technology currently in active development and early stages, so expect rapid evolution and potential breaking changes. However, it's already powerful enough to build real-world applications like this PR reviewer, and the declarative approach makes it worth exploring even at this stage.

### Why ACP?

Traditional approach:
```python
# Lots of boilerplate, error handling, retry logic...
async def review_pr(pr_number):
    try:
        pr = await fetch_pr(pr_number)
        files = await fetch_files(pr_number)
        review = await llm.review(code=files)
        await post_review(pr_number, review)
    except Exception as e:
        # handle errors, retries, etc...
```

ACP approach:
```hcl
workflow "review_pr" {
  step "fetch_pr" { ... }
  step "analyze" { agent = agent.reviewer }
  step "submit_review" { ... }
}
```

Clean, readable, and version-controlled.

## The Setup

### Prerequisites

- Python 3.12+
- GitHub repository with Actions enabled
- OpenAI API key
- GitHub Personal Access Token (with `repo` scope)

### Step 1: Install ACP CLI

```bash
pip install acp-cli
```

### Step 2: Create the Project Structure

I organized my ACP configuration into numbered files for clarity:

```
.acp/
├── 00-project.acp      # Project metadata
├── 01-variables.acp    # Input variables
├── 02-providers.acp    # LLM provider config
├── 03-servers.acp      # MCP server connections
├── 04-capabilities.acp # Tool capabilities
├── 05-policies.acp     # Budgets and limits
├── 06-agents.acp       # AI agent definitions
└── 07-workflows.acp    # Workflow orchestration
```

### Step 3: Define Variables

First, I defined what inputs my workflow needs:

```hcl
// .acp/01-variables.acp

variable "openai_api_key" {
  type        = string
  description = "OpenAI API key"
  sensitive   = true
}

variable "github_personal_access_token" {
  type        = string
  description = "GitHub Personal Access Token"
  sensitive   = true
}

variable "owner" {
  type        = string
  description = "GitHub repository owner"
}

variable "repo" {
  type        = string
  description = "GitHub repository name"
}

variable "pr_number" {
  type        = number
  description = "Pull request number to review"
}
```

### Step 4: Configure the LLM Provider

I set up OpenAI with GPT-4o:

```hcl
// .acp/02-providers.acp

provider "llm.openai" "default" {
  api_key = var.openai_api_key
  default_params {
    temperature = 0.3
    max_tokens  = 4000
  }
}

model "gpt4o" {
  provider = provider.llm.openai.default
  id       = "gpt-4o"
  params {
    temperature = 0.2
  }
}
```

### Step 5: Connect to GitHub via MCP

ACP uses **Model Context Protocol (MCP)** to connect to external services. I configured the GitHub MCP server:

```hcl
// .acp/03-servers.acp

server "github" {
  type      = "mcp"
  transport = "stdio"
  command   = ["npx", "@modelcontextprotocol/server-github"]
  auth {
    token = var.github_personal_access_token
  }
}
```

### Step 6: Define Capabilities

Capabilities are the tools my agent can use. I defined three GitHub operations:

```hcl
// .acp/04-capabilities.acp

capability "get_pr" {
  server      = server.github
  method      = "get_pull_request"
  side_effect = "read"
}

capability "list_pr_files" {
  server      = server.github
  method      = "get_pull_request_files"
  side_effect = "read"
}

capability "create_review" {
  server      = server.github
  method      = "create_pull_request_review"
  side_effect = "write"
}
```

### Step 7: Set Policies

I added budget controls to prevent runaway costs:

```hcl
// .acp/05-policies.acp

policy "review_policy" {
  budgets { max_cost_usd_per_run = 1.00 }
  budgets { max_capability_calls = 10 }
  budgets { timeout_seconds = 300 }
}
```

### Step 8: Create the Reviewer Agent

Now for the fun part - defining the AI agent:

```hcl
// .acp/06-agents.acp

agent "reviewer" {
  model = model.gpt4o

  instructions = <<EOF
You are an expert code reviewer. Review pull requests thoroughly.

Focus on:
- Code quality and best practices
- Potential bugs or edge cases
- Performance implications
- Security concerns
- Documentation and readability

Be constructive and specific in your feedback.
Suggest improvements where possible.
EOF

  allow  = [capability.get_pr, capability.list_pr_files, capability.create_review]
  policy = policy.review_policy
}
```

### Step 9: Orchestrate the Workflow

Finally, I wired everything together in a workflow:

```hcl
// .acp/07-workflows.acp

workflow "review_pr" {
  entry = step.fetch_pr

  step "fetch_pr" {
    type       = "call"
    capability = capability.get_pr

    args {
      owner       = input.owner
      repo        = input.repo
      pull_number = input.pr_number
    }

    output "pr_data" { from = result.data }
    next = step.fetch_files
  }

  step "fetch_files" {
    type       = "call"
    capability = capability.list_pr_files

    args {
      owner       = input.owner
      repo        = input.repo
      pull_number = input.pr_number
    }

    output "pr_files" { from = result.data }
    next = step.analyze
  }

  step "analyze" {
    type  = "llm"
    agent = agent.reviewer

    input {
      pr    = state.pr_data
      files = state.pr_files
    }

    output "review" { from = result.response }
    next = step.submit_review
  }

  step "submit_review" {
    type       = "call"
    capability = capability.create_review

    args {
      owner       = input.owner
      repo        = input.repo
      pull_number = input.pr_number
      body        = state.review.response
      event       = "COMMENT"
    }

    output "result" { from = result.data }
    next = step.end
  }

  step "end" { type = "end" }
}
```

Notice how clean this is? Each step:
1. **Fetches PR data** from GitHub
2. **Gets the changed files**
3. **Asks the AI agent to review** (with access to PR and file data)
4. **Submits the review** as a comment

The data flows naturally through `state`, and each step explicitly references the next.

### Step 10: Set Up GitHub Actions

To automate this, I created a GitHub Actions workflow:

```yaml
# .github/workflows/acp-pr-review.yml

name: PR Review

on:
  pull_request:
    types: [opened, synchronize, reopened]

permissions:
  contents: read
  pull-requests: write

jobs:
  review:
    name: AI PR Review
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6

      - name: Set up Python
        uses: actions/setup-python@v6
        with:
          python-version: "3.12"

      - name: Install acp-cli
        run: pip install acp-cli==0.0.3

      - name: Run PR Review
        working-directory: .acp
        run: |
          acp run review_pr \
            --var openai_api_key=${{ secrets.OPENAI_API_KEY }} \
            --var github_personal_access_token=${{ secrets.GITHUB_TOKEN }} \
            --var owner=${{ github.repository_owner }} \
            --var repo=${{ github.event.repository.name }} \
            --var pr_number=${{ github.event.pull_request.number }}
```

I added two secrets in GitHub:
- `OPENAI_API_KEY`: My OpenAI API key
- `GITHUB_TOKEN`: Automatically provided by GitHub Actions

That's it! Now every PR automatically gets reviewed.

## How It Works

When a pull request is opened or updated:

1. **GitHub Actions triggers** the workflow
2. **ACP fetches the PR data** using the GitHub MCP server
3. **ACP gets the list of changed files**
4. **The AI agent reviews** the code with access to both PR metadata and file contents
5. **The review is posted** as a comment on the PR

All orchestration, error handling, and state management is handled by ACP. I just defined the workflow declaratively.

## Testing Locally

You can test the workflow locally before pushing:

```bash
cd .acp

acp run review_pr \
  --var openai_api_key="your-key" \
  --var github_personal_access_token="your-token" \
  --var owner="your-username" \
  --var repo="your-repo" \
  --var pr_number=1
```

## What I Love About This Approach

1. **No boilerplate code**: Everything is declarative configuration
2. **Version controlled**: Agent logic is in `.acp` files, making changes reviewable
3. **Type safe**: ACP validates the schema before execution
4. **Composable**: Agents and workflows can be reused and combined
5. **Cost controlled**: Built-in budget limits prevent surprises
6. **Multi-provider**: Easy to switch between OpenAI, Anthropic, etc.

## Reusability: Using as a Module

One of the coolest features of ACP is that you can package your agent configuration as a reusable module. You can easily import this PR reviewer workflow into your own ACP project using ACP modules - no need to copy files or duplicate configuration!

Just reference it directly from your ACP project:

```hcl
module "simple-acp-module" {
  source  = "github.com/nirberko/acp-pr-reviewer-example//.acp"
  version = "main"  // Git branch name
  
  // Required parameters - pass through our project variables
  openai_api_key               = var.openai_api_key
  github_personal_access_token = var.github_token
  owner                        = var.owner
  repo                         = var.repo
  pr_number                    = var.pr_number
}
```

That's it! ACP modules make it incredibly easy to share and reuse agent configurations across projects or teams, ensuring everyone is using the same, tested workflow.

## Customization

Want to change the review style? Edit the agent instructions in `06-agents.acp`. Need different budget limits? Update `05-policies.acp`. Want to use a different model? Modify `02-providers.acp`.

All changes are in configuration files - no code changes needed.


## The Result

I now have an AI-powered PR reviewer that:
- ✅ Runs automatically on every PR
- ✅ Provides intelligent, constructive feedback
- ✅ Is fully version-controlled and maintainable
- ✅ Took me 15 minutes to set up

## Next Steps

You can extend this further:
- Add support for different review criteria
- Create multiple reviewer agents for different aspects (security, performance, style)
- Integrate with other MCP servers (databases, APIs, etc.)
- Build more complex multi-agent workflows

## Resources

- [ACP GitHub Repository](https://github.com/acp-org/acp)
- [Model Context Protocol](https://modelcontextprotocol.io/)
- [Example Repository](https://github.com/nirberko/acp-pr-reviewer-example)

---

**Have you tried ACP? What AI workflows would you build with it?** Let me know in the comments! 🚀
