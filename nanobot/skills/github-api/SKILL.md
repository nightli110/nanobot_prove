---
name: github-api
description: "Read GitHub repository information using Python + requests. Use when you need to fetch repository details, issues, PRs, code files, commits, or other GitHub data via the GitHub REST API."
metadata: {"nanobot":{"emoji":"🐙","requires":{"bins":["python"]}}}
---

# GitHub API Skill

Use Python + requests to interact with the GitHub REST API for reading repository information.

## Setup

First, check if requests is installed:
```bash
python -c "import requests" 2>/dev/null || pip install requests
```

## Configuration

The GitHub API token is stored in nanobot's config file at `~/.nanobot/config.json`.

Config format:
```json
{
  "tools": {
    "github": {
      "apiToken": "your_github_personal_access_token_here"
    }
  }
}
```

Public repositories don't need authentication, but rate-limited to 60 requests/hour.
With authentication: 5000 requests/hour.

## Helper Functions

Always use this pattern - read token from config automatically:

```python
import requests
import json
import base64
from pathlib import Path

def get_github_token():
    """Read GitHub token from nanobot config.json."""
    config_path = Path.home() / ".nanobot" / "config.json"
    if not config_path.exists():
        return None
    try:
        with open(config_path, "r", encoding="utf-8") as f:
            config = json.load(f)
        return config.get("tools", {}).get("github", {}).get("apiToken")
    except Exception:
        return None

def _github_headers(token=None):
    """Get headers with optional auth."""
    if token is None:
        token = get_github_token()
    headers = {"Accept": "application/vnd.github.v3+json"}
    if token:
        headers["Authorization"] = f"token {token}"
    return headers
```

## Common Endpoints

### Get Repository Info
```python
def get_repo(owner, repo, token=None):
    url = f"https://api.github.com/repos/{owner}/{repo}"
    response = requests.get(url, headers=_github_headers(token))
    response.raise_for_status()
    return response.json()
```

### List Repository Issues
```python
def list_issues(owner, repo, state="open", token=None, per_page=30):
    url = f"https://api.github.com/repos/{owner}/{repo}/issues"
    params = {"state": state, "per_page": per_page}
    response = requests.get(url, headers=_github_headers(token), params=params)
    response.raise_for_status()
    return response.json()
```

### Get a Single Issue
```python
def get_issue(owner, repo, issue_number, token=None):
    url = f"https://api.github.com/repos/{owner}/{repo}/issues/{issue_number}"
    response = requests.get(url, headers=_github_headers(token))
    response.raise_for_status()
    return response.json()
```

### List Pull Requests
```python
def list_pulls(owner, repo, state="open", token=None, per_page=30):
    url = f"https://api.github.com/repos/{owner}/{repo}/pulls"
    params = {"state": state, "per_page": per_page}
    response = requests.get(url, headers=_github_headers(token), params=params)
    response.raise_for_status()
    return response.json()
```

### Get Repository README
```python
def get_readme(owner, repo, ref=None, token=None):
    url = f"https://api.github.com/repos/{owner}/{repo}/readme"
    params = {}
    if ref:
        params["ref"] = ref
    response = requests.get(url, headers=_github_headers(token), params=params)
    response.raise_for_status()
    data = response.json()
    if data.get("encoding") == "base64":
        return base64.b64decode(data["content"]).decode("utf-8")
    return data.get("content", "")
```

### Get a File or Directory Content
```python
def get_contents(owner, repo, path="", ref=None, token=None):
    url = f"https://api.github.com/repos/{owner}/{repo}/contents/{path}"
    params = {}
    if ref:
        params["ref"] = ref
    response = requests.get(url, headers=_github_headers(token), params=params)
    response.raise_for_status()
    data = response.json()

    if isinstance(data, list):
        # Directory listing
        return data
    else:
        # Single file
        if data.get("encoding") == "base64":
            return {
                "type": "file",
                "name": data["name"],
                "path": data["path"],
                "content": base64.b64decode(data["content"]).decode("utf-8"),
                "sha": data["sha"]
            }
        return data
```

### List Commits
```python
def list_commits(owner, repo, path=None, sha=None, since=None, until=None, token=None, per_page=30):
    url = f"https://api.github.com/repos/{owner}/{repo}/commits"
    params = {"per_page": per_page}
    if path:
        params["path"] = path
    if sha:
        params["sha"] = sha
    if since:
        params["since"] = since
    if until:
        params["until"] = until
    response = requests.get(url, headers=_github_headers(token), params=params)
    response.raise_for_status()
    return response.json()
```

### Get a Single Commit
```python
def get_commit(owner, repo, sha, token=None):
    url = f"https://api.github.com/repos/{owner}/{repo}/commits/{sha}"
    response = requests.get(url, headers=_github_headers(token))
    response.raise_for_status()
    return response.json()
```

### List Repository Tags
```python
def list_tags(owner, repo, token=None, per_page=30):
    url = f"https://api.github.com/repos/{owner}/{repo}/tags"
    params = {"per_page": per_page}
    response = requests.get(url, headers=_github_headers(token), params=params)
    response.raise_for_status()
    return response.json()
```

### List Repository Branches
```python
def list_branches(owner, repo, token=None, per_page=30):
    url = f"https://api.github.com/repos/{owner}/{repo}/branches"
    params = {"per_page": per_page}
    response = requests.get(url, headers=_github_headers(token), params=params)
    response.raise_for_status()
    return response.json()
```

### List Releases
```python
def list_releases(owner, repo, token=None, per_page=30):
    url = f"https://api.github.com/repos/{owner}/{repo}/releases"
    params = {"per_page": per_page}
    response = requests.get(url, headers=_github_headers(token), params=params)
    response.raise_for_status()
    return response.json()
```

### Search Issues/PRs
```python
def search_issues(query, token=None, per_page=30):
    url = "https://api.github.com/search/issues"
    params = {"q": query, "per_page": per_page}
    response = requests.get(url, headers=_github_headers(token), params=params)
    response.raise_for_status()
    return response.json()

# Example queries:
# - "repo:owner/repo is:issue is:open"
# - "repo:owner/repo is:pr is:merged"
# - "repo:owner/repo label:bug"
```

### Get Rate Limit Status
```python
def get_rate_limit(token=None):
    url = "https://api.github.com/rate_limit"
    response = requests.get(url, headers=_github_headers(token))
    response.raise_for_status()
    return response.json()
```

## Usage Example

Complete example to fetch repo info:

```python
import requests
import json
import base64
from pathlib import Path

def get_github_token():
    config_path = Path.home() / ".nanobot" / "config.json"
    if not config_path.exists():
        return None
    try:
        with open(config_path, "r", encoding="utf-8") as f:
            config = json.load(f)
        return config.get("tools", {}).get("github", {}).get("apiToken")
    except Exception:
        return None

def _github_headers(token=None):
    if token is None:
        token = get_github_token()
    headers = {"Accept": "application/vnd.github.v3+json"}
    if token:
        headers["Authorization"] = f"token {token}"
    return headers

def get_repo(owner, repo, token=None):
    url = f"https://api.github.com/repos/{owner}/{repo}"
    response = requests.get(url, headers=_github_headers(token))
    response.raise_for_status()
    return response.json()

# Example usage
repo = get_repo("octocat", "Hello-World")
print(json.dumps(repo, indent=2))
```

## Error Handling

Always handle potential errors:
- 404: Repository or resource not found
- 403: Rate limit exceeded or forbidden
- 401: Authentication required

Example with error handling:
```python
def safe_get(url, token=None):
    try:
        response = requests.get(url, headers=_github_headers(token))
        response.raise_for_status()
        return response.json()
    except requests.exceptions.HTTPError as e:
        if response.status_code == 404:
            return {"error": "Not found"}
        elif response.status_code == 403:
            return {"error": "Rate limit or forbidden"}
        raise
```

## Pagination

For results with more than 30 items, use pagination:
```python
def get_all_pages(url, token=None, per_page=100):
    all_items = []
    page = 1
    while True:
        params = {"per_page": per_page, "page": page}
        response = requests.get(url, headers=_github_headers(token), params=params)
        response.raise_for_status()
        items = response.json()
        if not items:
            break
        all_items.extend(items)
        if len(items) < per_page:
            break
        page += 1
    return all_items
```
