# Hermes OSS Contributor Skill

Autonomous open source contribution workflow for Hermes Agent.

## Overview

This skill enables Hermes Agent to:
- Discover open source issues suitable for contribution
- Fork repositories and create feature branches
- Implement fixes and submit pull requests
- Track contributions and learn from review feedback
- Monitor PR status and address review comments

## Usage

```python
# In Hermes Agent
skill_view("oss-contributor")
```

## Features

- **Issue Discovery**: Finds issues labeled "good first issue" or "help wanted"
- **Smart Filtering**: Scores repositories by stars, activity, and tech stack
- **PR Workflow**: Complete fork → branch → commit → push → PR flow
- **Contribution Tracking**: Maintains a log of all contributions and outcomes
- **Review Response**: Monitors PRs for feedback and addresses comments

## Requirements

- GitHub authentication (gh CLI or git + PAT)
- Hermes Agent environment
- Python 3.10+

## License

MIT

## Created For

BlutAgent project - A code reviewer, OSS contributor, and personal assistant.

