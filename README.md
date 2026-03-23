# airflow-basic-setup

Docker-based Apache Airflow setup for Windows (Docker Desktop).

---

## Prerequisites

| Tool | Notes |
|------|-------|
| [Docker Desktop for Windows](https://docs.docker.com/desktop/install/windows-install/) | Enable WSL 2 backend (recommended) |
| Git | To clone DAG repositories |

---

## Directory layout

All Airflow data lives under `C:\airflow`:

```
C:\airflow\
    dags\        ← clone your DAG git repos here as sub-directories
    logs\
    plugins\
    config\
```

Create the directories before starting:

```powershell
New-Item -ItemType Directory -Force `
    C:\airflow\dags, `
    C:\airflow\logs, `
    C:\airflow\plugins, `
    C:\airflow\config
```

---

## Configuration

Copy the `.env` file to `C:\airflow` (or keep it next to `docker-compose.yml`) and fill in the two secret values:

```powershell
# Generate a Fernet key
docker run --rm apache/airflow:2.9.3 python -c `
    "from cryptography.fernet import Fernet; print(Fernet.generate_key().decode())"

# Generate a webserver secret key
docker run --rm apache/airflow:2.9.3 python -c `
    "import secrets; print(secrets.token_hex(32))"
```

Paste the outputs into `.env`:

```env
AIRFLOW__CORE__FERNET_KEY=<generated-fernet-key>
AIRFLOW__WEBSERVER__SECRET_KEY=<generated-secret-key>
```

---

## First-time start

```powershell
# 1. Initialise the metadata database and create the admin user
docker compose up airflow-init

# 2. Start all services in the background
docker compose up -d

# 3. Open the Airflow UI
start http://localhost:8080
# Default credentials: admin / admin  ⚠️  Change this password after first login!
```

---

## Adding DAG repositories

Each git repository becomes a sub-directory under `C:\airflow\dags\`.
Airflow will automatically pick up every Python DAG file it finds anywhere in that folder tree.

```powershell
# Clone a first DAG repo
git clone https://github.com/your-org/dags-team-a.git C:\airflow\dags\team-a

# Clone a second DAG repo
git clone https://github.com/your-org/dags-team-b.git C:\airflow\dags\team-b
```

### Keeping DAGs up-to-date

Create a scheduled Windows Task (or a simple PowerShell script) to pull the latest changes:

```powershell
# update-dags.ps1 – run this on a schedule (e.g. every 5 minutes via Task Scheduler)
Get-ChildItem -Path C:\airflow\dags -Directory | ForEach-Object {
    if (Test-Path (Join-Path $_.FullName ".git")) {
        Write-Host "Pulling $($_.Name)..."
        git -C $_.FullName pull --ff-only
    }
}
```

Schedule it with Task Scheduler:

```powershell
$action  = New-ScheduledTaskAction -Execute "powershell.exe" `
               -Argument "-NonInteractive -File C:\airflow\update-dags.ps1"
$trigger = New-ScheduledTaskTrigger -Daily -At "00:00"
$settings = New-ScheduledTaskSettingsSet -ExecutionTimeLimit (New-TimeSpan -Hours 1)
Register-ScheduledTask -TaskName "AirflowDagSync" `
    -Action $action -Trigger $trigger -Settings $settings

# Then add a repetition interval so it runs every 5 minutes throughout the day:
$task = Get-ScheduledTask -TaskName "AirflowDagSync"
$task.Triggers[0].Repetition.Interval = "PT5M"
$task | Set-ScheduledTask
```

---

## Common operations

```powershell
# View logs
docker compose logs -f airflow-scheduler
docker compose logs -f airflow-webserver

# Stop all containers (data is preserved)
docker compose down

# Stop and remove all data (including the Postgres volume)
docker compose down -v

# Upgrade Airflow image – update AIRFLOW_IMAGE_NAME in .env, then:
docker compose pull
docker compose up airflow-init   # run migrations
docker compose up -d
```

---

## Services overview

| Service | Description | Port |
|---------|-------------|------|
| `postgres` | Metadata database | – |
| `airflow-init` | One-shot DB migration + admin user creation | – |
| `airflow-scheduler` | Parses DAGs and triggers task instances | – |
| `airflow-webserver` | Web UI | `8080` |
