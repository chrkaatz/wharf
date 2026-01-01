# AGENTS.md

This document provides essential information for AI agents working with the Wharf codebase.

## Project Overview

**Wharf** is an opinionated web frontend for [Dokku](https://dokku.com/docs/), a Docker-powered Platform-as-a-Service (PaaS). It provides a web-based UI for managing Dokku applications, eliminating the need to use the command line for most day-to-day operations.

### Key Features
- Web-based management of Dokku apps
- Application deployment from GitHub repositories
- Configuration management (environment variables)
- Database management (PostgreSQL and Redis)
- Domain and SSL certificate management (Let's Encrypt)
- Process management and monitoring
- Log viewing
- GitHub webhook integration for auto-deployment

## Architecture

### Technology Stack
- **Framework**: Django (Python web framework)
- **Task Queue**: Celery with Redis as broker
- **Database**: PostgreSQL (via Dokku plugin) or SQLite (development)
- **Cache**: Redis
- **Templates**: Jinja2 (via django-jinja)
- **SSH**: Paramiko for SSH connections to Dokku host
- **Version Control**: GitPython for repository operations

### Project Structure

```
wharf/
├── apps/                    # Main Django application
│   ├── models.py           # Database models (App, TaskLog)
│   ├── views.py            # View functions (main application logic)
│   ├── forms.py            # Django forms
│   ├── urls.py             # URL routing
│   └── templates/          # Jinja2 templates
├── wharf/                  # Django project settings
│   ├── settings.py         # Django configuration
│   ├── tasks.py            # Celery tasks (SSH commands, deployments)
│   ├── auth.py             # Authentication backend and middleware
│   ├── celery.py           # Celery app configuration
│   └── urls.py             # Root URL configuration
└── tests/                  # Test suite
```

## Core Concepts

### 1. Dokku Command Execution

Wharf executes Dokku commands via SSH or a Unix socket daemon:

- **SSH Method**: Uses Paramiko to SSH into the Dokku host and execute commands
- **Daemon Method**: Uses Unix socket (`/var/run/dokku-daemon/dokku-daemon.sock`) for faster execution when available

All commands are executed asynchronously via Celery tasks (`wharf.tasks.run_ssh_command`).

### 2. Command Caching

Dokku commands are cached in Redis to avoid repeated SSH connections:
- Cache keys follow pattern: `cmd:<command>`
- Cache timeout: 1 day (24 hours)
- Cache can be cleared per-app or globally

### 3. Task Execution Flow

1. User triggers action in web UI
2. View function calls `run_cmd_with_log()` which:
   - Creates a Celery task (`tasks.run_ssh_command.delay()`)
   - Creates a `TaskLog` entry in database
   - Redirects to `wait_for_command` view
3. `wait_for_command` view polls task status and displays logs in real-time
4. On completion, redirects to a "check" view that validates output
5. Check view clears relevant cache and redirects to final destination

### 4. Authentication

- Simple authentication via `SettingsBackend` using `ADMIN_LOGIN` and `ADMIN_PASSWORD` environment variables
- All views require authentication except: `webhook`, `favicon.ico`, `status`
- Middleware (`LoginRequiredMiddleware`) enforces authentication

## Key Files and Their Purposes

### `apps/views.py`
Main application logic. Contains view functions for:
- App listing and creation
- Configuration management
- Database operations (PostgreSQL, Redis)
- Domain management
- Deployment operations
- Log viewing
- GitHub webhook handling

**Important patterns:**
- `run_cmd_with_log()`: Execute command and track in database
- `run_cmd_with_cache()`: Execute command with caching
- `generic_config()`: Parse Dokku config output
- `generic_list()`: Parse tabular Dokku output
- `generic_info()`: Parse key-value Dokku output

### `wharf/tasks.py`
Celery tasks for background operations:
- `run_ssh_command`: Execute Dokku commands via SSH or daemon
- `deploy`: Clone/pull Git repo and push to Dokku
- `get_public_key`: Get SSH public key for setup
- `check_ssh`: Verify SSH connectivity

**Important functions:**
- `run_with_daemon()`: Execute via Unix socket (faster)
- `run_process()`: Execute local Git commands
- `handle_data()`: Stream command output to Redis

### `apps/models.py`
Database models:
- `App`: Represents a Dokku application (name, GitHub URL)
- `TaskLog`: Tracks Celery task execution (task_id, app, description, success status, timestamp)

### `wharf/auth.py`
Authentication system:
- `SettingsBackend`: Authenticates against environment variables
- `LoginRequiredMiddleware`: Enforces login requirement

### `wharf/settings.py`
Django configuration:
- Uses environment variables for configuration
- Redis for cache and Celery broker
- PostgreSQL or SQLite for database
- Jinja2 for templates

## Important Patterns and Conventions

### 1. Command Execution Pattern

```python
# Synchronous with cache
result = run_cmd_with_cache("apps:list")

# Asynchronous with logging
return run_cmd_with_log(
    app_name,
    "Description of action",
    "dokku:command %s" % app_name,
    "check_view_name"
)
```

### 2. Check View Pattern

After async command execution, a "check" view validates the output:

```python
def check_action(request, app_name, task_id):
    res = AsyncResult(task_id)
    data = get_log(res)
    # Validate output
    if expected_string not in data:
        raise Exception(data)
    # Clear cache
    clear_cache("command:pattern")
    # Redirect
    return redirect_reverse("destination", args=[app_name])
```

### 3. Cache Management

- Cache keys: `cmd:<dokku command>`
- Clear cache: `clear_cache("command:pattern")` or `cache.delete_many([keys])`
- Refresh all: `cache.clear()`

### 4. Error Handling

- `NoSuchAppException`: Raised when app doesn't exist
- Commands that fail raise exceptions with output in message
- SSH failures redirect to setup page

### 5. URL Patterns

- App-specific: `/apps/<app_name>/<action>`
- Global: `/` (index), `/refresh`, `/create_app`
- Task tracking: `/apps/<app_name>/wait/<task_id>/<after>`
- Logs: `/logs/<task_id>`

## Environment Variables

Key environment variables (see `wharf/settings.py`):

- `ADMIN_LOGIN`: Admin username (default: "admin")
- `ADMIN_PASSWORD`: Admin password (required)
- `DOKKU_SSH_HOST`: Dokku host (auto-detected if not set)
- `DOKKU_SSH_PORT`: SSH port (default: 22)
- `GITHUB_SECRET`: Secret for webhook validation
- `REDIS_URL`: Redis connection URL
- `DATABASE_URL`: Database connection URL (via dj-database-url)
- `SECRET_KEY`: Django secret key

## Testing

Tests are in the `tests/` directory:
- `test_views.py`: View tests
- `test_tasks.py`: Celery task tests
- `test_auth.py`: Authentication tests

Run tests with pytest (configured in `pyproject.toml`).

## Development Setup

1. Use Vagrant for full Dokku environment: `vagrant up`
2. Run Django dev server: `docker-compose up`
3. Access at `http://localhost:8000/`

## Common Tasks for AI Agents

### Adding a New Dokku Command

1. Add view function in `apps/views.py`
2. Use `run_cmd_with_log()` for async execution
3. Create check view to validate output
4. Add URL route in `apps/urls.py`
5. Add template if needed
6. Clear relevant cache keys

### Parsing Dokku Output

- Use `generic_config()` for env var output
- Use `generic_list()` for tabular output
- Use `generic_info()` for key-value output
- Handle "does not exist" and "is not a dokku command" errors

### Working with Celery Tasks

- Tasks are defined in `wharf/tasks.py` with `@app.task` decorator
- Use `.delay()` to execute asynchronously
- Use `AsyncResult(task_id)` to check status
- Logs stored in Redis with key `task:<task_id>`

## Important Notes

1. **SSH Key Management**: Wharf generates its own SSH key (`~/.ssh/id_rsa`) if it doesn't exist. This key must be added to Dokku's authorized_keys.

2. **Command Output**: All command output is stored in Redis and streamed to users via the `wait_for_command` view.

3. **Cache Invalidation**: Always clear relevant cache keys after modifying app state (config, domains, databases, etc.).

4. **Error Messages**: Dokku error messages are parsed and displayed to users. Common patterns:
   - "does not exist" → App/resource not found
   - "is not a dokku command" → Plugin not installed

5. **Plugin Compatibility**: Some features depend on Dokku plugins (postgres, redis, letsencrypt). Code checks for plugin availability.

6. **GitHub Integration**: Webhook endpoint validates HMAC signature and triggers deployment for matching apps.

## Code Style

- Follow existing patterns in the codebase
- Use type hints where present
- Django conventions for views, models, forms
- Jinja2 templates (not Django templates)
- Ruff for linting (configured in `pyproject.toml`)

