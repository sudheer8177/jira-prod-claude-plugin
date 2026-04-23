# jira-prod

A Claude Code plugin for PossibleWorks engineering — turns a Jira ticket into a full dev workflow: fetch → plan → code → PR → Coolify preview.

## Features

- `/jira-prd <TICKET-ID>` — End-to-end Jira ticket implementation
  - Fetches ticket details, attachments, and Figma designs
  - Routes to the right repo(s) automatically
  - Plans, codes, reviews, and opens a PR
  - Deploys to Coolify dev environment and returns a preview URL

## Requirements

- Atlassian MCP connected (for Jira access)
- Figma MCP connected (for design assets)
- GitHub CLI (`gh`) installed
- Coolify access configured

## Installation

```
/plugin install jira-prod
```

## Usage

```
/jira-prd PW-123
/jira-prd https://possibleworks.atlassian.net/browse/PW-123
```
