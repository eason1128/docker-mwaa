# Complete Setup Guide: AWS MWAA Local Runner with Colima on macOS

This guide provides step-by-step instructions to set up AWS MWAA Local Runner using Docker with Colima (without Docker Desktop) on macOS.

## Prerequisites

- macOS (Intel or Apple Silicon)
- Homebrew installed
- Terminal access

## Step 1: Install Docker and Colima

### 1.1 Install Docker CLI
```bash
brew install docker
```

### 1.2 Install Colima
```bash
brew install colima
```

### 1.3 Install Docker Compose (if not included)
```bash
brew install docker-compose
```

### 1.4 Start Colima
```bash
colima start
```

### 1.5 Verify Installation
```bash
# Check Colima status
colima status

# Check Docker
docker --version
docker ps

# Check Docker Compose
docker-compose --version
```

Expected output for `colima status`:
```
time="..." level=info msg="colima is running using macOS Virtualization.Framework"
time="..." level=info msg="arch: aarch64"
time="..." level=info msg="runtime: docker"
time="..." level=info msg="mountType: sshfs"
time="..." level=info msg="socket: unix:///Users/username/.colima/default/docker.sock"
```

## Step 2: Clone and Setup AWS MWAA Local Runner

### 2.1 Clone Repository
```bash
git clone https://github.com/aws/aws-mwaa-local-runner.git
cd aws-mwaa-local-runner
```

### 2.2 Validate Prerequisites
```bash
./mwaa-local-env validate-prereqs
```

Expected output:
```
Docker is Installed. ✔
docker-compose is Installed. ✔
Python3 is Installed ✔
Pip3 is Installed. ✔
```

## Step 3: Fix PostgreSQL Permission Issue

The default configuration doesn't work with Colima due to SSHFS permission restrictions. We need to modify the Docker Compose configuration.

### 3.1 Create Docker Named Volume
```bash
docker volume create aws-mwaa-postgres-data
```

### 3.2 Backup Original Configuration
```bash
cp docker/docker-compose-local.yml docker/docker-compose-local.yml.backup
```

### 3.3 Modify docker-compose-local.yml

Edit `docker/docker-compose-local.yml` and make the following changes:

**Original problematic line (around line 14):**
```yaml
volumes:
  - "${PWD}/db-data:/var/lib/postgresql/data"
```

**Replace with:**
```yaml
volumes:
  - "aws-mwaa-postgres-data:/var/lib/postgresql/data"
```

**Add volumes section at the end of the file:**
```yaml
volumes:
  aws-mwaa-postgres-data:
    external: true
```

### 3.4 Complete Fixed docker-compose-local.yml

Here's the complete fixed file content:

```yaml
version: "3.7"
services:
  postgres:
    image: postgres:13-alpine
    environment:
      - POSTGRES_USER=airflow
      - POSTGRES_PASSWORD=airflow
      - POSTGRES_DB=airflow
    logging:
      options:
        max-size: 10m
        max-file: "3"
    volumes:
      - "aws-mwaa-postgres-data:/var/lib/postgresql/data"

  local-runner:
    image: amazon/mwaa-local:2_10_3
    restart: always
    depends_on:
      - postgres
    environment:
      - LOAD_EX=n
      - EXECUTOR=Local
    logging:
      options:
        max-size: 10m
        max-file: "3"
    volumes:
      - "${PWD}/dags:/usr/local/airflow/dags"
      - "${PWD}/plugins:/usr/local/airflow/plugins"
      - "${PWD}/requirements:/usr/local/airflow/requirements"
      - "${PWD}/startup_script:/usr/local/airflow/startup"
    ports:
      - "8080:8080"
    command: local-runner
    healthcheck:
      test: ["CMD-SHELL", "[ -f /usr/local/airflow/airflow-webserver.pid ]"]
      interval: 30s
      timeout: 30s
      retries: 3
    env_file:
      - ./config/.env.localrunner

volumes:
  aws-mwaa-postgres-data:
    external: true
```

## Step 4: Build and Start MWAA Environment

### 4.1 Build Docker Image
```bash
./mwaa-local-env build-image
```

This step takes several minutes. Expected output includes:
```
Successfully built [image-id]
Successfully tagged amazon/mwaa-local:2_10_3
```

### 4.2 Start Local Environment
```bash
./mwaa-local-env start
```

Expected output:
```
Creating network "aws-mwaa-local-runner-2_10_3_default" with the default driver
Creating aws-mwaa-local-runner-2_10_3-postgres-1 ... done
Creating aws-mwaa-local-runner-2_10_3-local-runner-1 ... done
Attaching to aws-mwaa-local-runner-2_10_3-postgres-1, aws-mwaa-local-runner-2_10_3-local-runner-1
```

### 4.3 Verify Setup
```bash
# Check running containers
docker ps

# Check logs
docker logs aws-mwaa-local-runner-2_10_3-postgres-1 --tail 5
docker logs aws-mwaa-local-runner-2_10_3-local-runner-1 --tail 10
```

### 4.4 Access Airflow UI
- Open browser to: http://localhost:8080
- Username: `admin`
- Password: `test`

## Step 5: File Structure and Modifications Summary

### Files Modified in aws-mwaa-local-runner:
1. **docker/docker-compose-local.yml** - Modified PostgreSQL volume mount
2. **docker/docker-compose-local.yml.backup** - Created as backup

### Key Changes Made:
- Changed PostgreSQL data storage from host directory bind mount to Docker named volume
- Added external volume definition to compose file

### Directory Structure After Setup:
```
aws-mwaa-local-runner/
├── SETUP_GUIDE_COLIMA.md          # This guide (new)
├── COLIMA_TROUBLESHOOTING.md      # Troubleshooting doc (new)
├── docker/
│   ├── docker-compose-local.yml         # Modified
│   ├── docker-compose-local.yml.backup  # Created
│   └── ...
├── dags/                          # Your DAGs go here
├── plugins/                       # Custom plugins
├── requirements/                  # Python requirements
│   └── requirements.txt
├── startup_script/               # Startup scripts
│   └── startup.sh
└── mwaa-local-env               # Main CLI script
```

## Step 6: Development Workflow

### 6.1 Adding DAGs
Place your DAG files in the `dags/` directory:
```bash
# Example
cp your_dag.py dags/
```

### 6.2 Adding Python Dependencies
Edit `requirements/requirements.txt`:
```bash
# Test requirements installation
./mwaa-local-env test-requirements
```

### 6.3 Custom Plugins
Place plugin files in `plugins/` directory.

### 6.4 Startup Scripts
Modify `startup_script/startup.sh` for custom initialization:
```bash
# Test startup script
./mwaa-local-env test-startup-script
```

## Step 7: Common Operations

### Start Environment
```bash
./mwaa-local-env start
```

### Stop Environment
Press `Ctrl+C` in the terminal running the containers.

### Reset Database
```bash
./mwaa-local-env reset-db
```

### View All Available Commands
```bash
./mwaa-local-env help
```

### Clean Shutdown
```bash
# Stop containers
docker-compose -p aws-mwaa-local-runner-2_10_3 -f ./docker/docker-compose-local.yml down

# Stop Colima (optional)
colima stop
```

## Step 8: Data Management

### Backup PostgreSQL Data
```bash
docker run --rm -v aws-mwaa-postgres-data:/data -v $(pwd):/backup alpine tar czf /backup/postgres-backup.tar.gz -C /data .
```

### Restore PostgreSQL Data
```bash
docker run --rm -v aws-mwaa-postgres-data:/data -v $(pwd):/backup alpine sh -c "cd /data && tar xzf /backup/postgres-backup.tar.gz"
```

### Remove All Data (Reset)
```bash
docker volume rm aws-mwaa-postgres-data
docker volume create aws-mwaa-postgres-data
```

## Troubleshooting

### Issue: Permission Denied Error
If you see `chown: /var/lib/postgresql/data: Permission denied`, ensure you've applied the Docker named volume fix from Step 3.

### Issue: Containers Won't Start
1. Check Colima status: `colima status`
2. Restart Colima: `colima stop && colima start`
3. Check Docker connectivity: `docker ps`

### Issue: Port 8080 Already in Use
```bash
# Find process using port 8080
lsof -i :8080

# Kill process if needed
kill -9 <PID>
```

### Issue: Can't Access Airflow UI
1. Wait 2-3 minutes for full startup
2. Check container health: `docker ps`
3. Check logs: `docker logs aws-mwaa-local-runner-2_10_3-local-runner-1`

## Performance Considerations

### Colima Configuration (Optional)
For better performance, you can configure Colima with more resources:

```bash
# Stop current instance
colima stop

# Start with custom configuration
colima start --cpu 4 --memory 8 --disk 60
```

## Security Notes

- Default Airflow credentials are `admin/test` - change in production
- Colima runs containers in an isolated VM
- PostgreSQL data is stored in Docker volumes, not directly accessible from host

## References

- [Colima GitHub](https://github.com/abiosoft/colima)
- [AWS MWAA Local Runner](https://github.com/aws/aws-mwaa-local-runner)
- [Docker Volumes Documentation](https://docs.docker.com/storage/volumes/)

---

**Last Updated**: July 23, 2025  
**Tested Environment**:
- macOS with Apple Silicon
- Colima v0.x.x
- Docker v24.x.x
- AWS MWAA Local Runner v2.10.3