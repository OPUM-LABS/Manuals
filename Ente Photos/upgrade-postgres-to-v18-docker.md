# Upgrade Postgres to v18 (Docker)

### Step 1: Backup While your current postgres instance is running, generate the dump.

```Bash:
docker exec -t $(docker compose ps -q postgres) pg_dumpall -c -U ${pg_user} > /appdata/ente_photos/postgres_full_backup.sql
```

*Note: If `${pg_user}` doesn't resolve in your shell, just type the actual username you use for Ente (often `pguser`).

### Step 2: Preparation

1. **Stop the stack:**

    ```Bash:
    docker compose down
    ```

2. **Move the old data:** Postgres 18 cannot read your old files directly.

    ```Bash:
    mv /appdata/ente_photos/db /appdata/ente_photos/db_v15_bak
    mkdir /appdata/ente_photos/db
    ```

### Step 3: Update your `compose.yaml` 

You need to change the *image* and, most importantly, the *volume mount point*. Postgres 18 now expects to manage the parent directory.

```YAML:
postgres:
  image: postgres:18  # Updated to 18
  restart: unless-stopped
  environment:
    POSTGRES_USER: ${pg_user}
    POSTGRES_PASSWORD: ${pg_pass}
    POSTGRES_DB: ${pg_db}
  networks:
    - ente_internal
  healthcheck:
    test:
      - CMD-SHELL
      - pg_isready -d ${pg_db} -U ${pg_user}
    interval: 5s
    timeout: 5s
    retries: 5
  volumes:
    #                 remove this subdirectory:  ⬇️⬇️⬇️⬇️⬇️
    - /appdata/ente_photos/db:/var/lib/postgresql
```

### Step 4: Restore and Reindex

1. **Start the Database only:**

    ```Bash:
    docker compose up -d postgres
    ```

2. **Import the data:**

   (Wait ~10 seconds for it to initialize first)

    ```Bash:
    cat /appdata/ente_photos/postgres_full_backup.sql | docker exec -i $(docker compose ps -q postgres) psql -U ${pg_user}
    ```

3. **Cleanup" (Collation Fix):** 
Run these to ensure your photo metadata sorts correctly under the new version:

    ```Bash:
    docker exec -it $(docker compose ps -q postgres) psql -U ${pg_user} -d ${pg_db} -c "REINDEX DATABASE ${pg_db};"
    docker exec -it $(docker compose ps -q postgres) psql -U ${pg_user} -d ${pg_db} -c "ALTER DATABASE ${pg_db} REFRESH COLLATION VERSION;"
    ```

4. **Launch the full stack:**

  ```Bash
  docker compose up -d
  ```
