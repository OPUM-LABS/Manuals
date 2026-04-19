# Migrate from MinIO to Garage

This tutorial is based on my setup. So don't just copy and paste my code-blocks, as you may have to change some paths to match your setup.

### 1\. Prepare the Host

For Garage i have a fast, local disk for metadata (NVMe/SSD recommended) and a slow one for data (HDD).
For the data i've mounted a share from a NAS.

```Bash
# Metadata & Config (Fast Drive)
mkdir -p /appdata/ente_photos/garage/meta
mkdir -p /appdata/ente_photos/garage/config

# Blob Data (Large Storage)
mkdir -p /mnt/Media/Pictures/garage_data
```
### 2\. Configure Garage (`garage.toml`)

Create `/appdata/ente_photos/garage/config/garage.toml`.
(These here are optimized settings)

```TOML
metadata_dir = "/var/lib/garage/meta"
data_dir = "/var/lib/garage/data"

# LMDB is still the best for stability in v2.x. 
# Experimental "fjall" is faster for massive writes but stay on lmdb for production.
db_engine = "lmdb"

# New for v2.x: Periodically snapshot metadata for safety
metadata_auto_snapshot_interval = "6h"

replication_factor = 1
consistency_mode = "consistent"

# Performance: Enable zstd compression. 
# Level 1 is nearly "free" CPU-wise but saves 10-20% disk space even on encrypted data.
compression_level = 1

# Performance: Increase the RAM buffer for block writes.
# 128MB is a sweet spot for home servers to smooth out HDD write spikes.
block_ram_buffer_max = 134217728 

rpc_bind_addr = "[::]:3901"
rpc_public_addr = "10.1.1.65:3901"
rpc_secret = "e20f3c5fb4bdd099e478939d29098c478f204ab69f87608df756b7b496fb7c07"

[s3_api]
s3_region = "us-east-1"
api_bind_addr = "[::]:3900"
root_domain = ".s3.local"

[s3_web]
bind_addr = "[::]:3902"
root_domain = ".web.local"
```
ℹ️ To generate the `rpc_secret` (think of a node-key) just run this command on a Linux machine:  
```bash
openssl rand -hex 32
```
### 3\. The Docker Compose Stack

Save as `compose.yaml`. This version removes `socat` and connects Ente directly to Garage.

I'm using a mix of macvlan (public facing) and internal docker bridge (only internal). I've also moved to postgres 18, in your case it's maybe 15.

If you want to upgrade, you can use this guide: [Upgrade Postgres to v18](https://github.com/OPUM-LABS/Manuals/blob/main/Ente%20Photos/upgrade-postgres-to-v18-docker.md)

```YAML
services:
  museum:
    image: ghcr.io/ente-io/server
    restart: unless-stopped
    networks:
      ente_internal: null
      macvlan_net:
        ipv4_address: 10.1.1.64
        mac_address: 02:00:00:00:00:64
    dns:
      - 10.10.10.1
    ports:
      - 8080:8080
    environment:
      POSTGRES_USER: ${pg_user}
      POSTGRES_PASSWORD: ${pg_pass}
      POSTGRES_DB: ${pg_db}
      museum_key: ${museum_key}
      museum_hash: ${museum_hash}
      museum_jwt_secret: ${museum_jwt_secret}
    depends_on:
      postgres:
        condition: service_healthy
      garage:
        condition: service_started
    volumes:
      - /appdata/ente_photos/museum/museum.yaml:/museum.yaml:ro
      - /appdata/ente_photos/museum/data:/data:ro
  web:
    image: ghcr.io/ente-io/web
    restart: unless-stopped
    network_mode: service:museum
    environment:
      ENTE_API_ORIGIN: https://photo.domain.ch
      ENTE_ALBUMS_ORIGIN: https://photo-pub.domain.ch
  postgres:
    image: postgres:18
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
      - /appdata/ente_photos/db:/var/lib/postgresql
  garage:
    image: dxflrs/garage:v2.2.0
    container_name: garage
    restart: unless-stopped
    networks:
      ente_internal: null
      macvlan_net:
        ipv4_address: 10.1.1.65
        mac_address: 02:00:00:00:00:65
    dns:
      - 10.10.10.1
    environment:
      - RUST_LOG=info
    volumes:
      - /appdata/ente_photos/garage/config/garage.toml:/etc/garage.toml
      - /appdata/ente_photos/garage/meta:/var/lib/garage/meta
      - /mnt/Media/Pictures/ente_photos/garage_data:/var/lib/garage/data
    ports:
      - 3900:3900 # S3 API
      - 3902:3902 # Web UI
networks:
  ente_internal:
    driver: bridge
    internal: true
  macvlan_net:
    external: true

```
* * *

### 4\. Initialize Garage (The "One-Time" Setup)

After starting the container (`docker compose up -d`), run these commands to set up the cluster layout.

1.  Assign Layout  
    ```Bash  
    # Get Node ID
    docker exec -it garage /garage status
    
    # Assign Capacity (Replace NODE_ID below)
    docker exec -it garage /garage layout assign -z zone1 -c 1T [NODE_ID]
    docker exec -it garage /garage layout apply --version 1
    ```
2.  Create Bucket & Keys  
    ```Bash  
    docker exec -it garage /garage bucket create b2-eu-cen
    docker exec -it garage /garage key create ente-key
    # STOP! Copy the Access Key and Secret Key output now.
    ```
3.  Grant Permissions (Critical)  
    Without this, the key exists but can't touch the bucket.  
      
    ```Bash  
    
    docker exec -it garage /garage bucket allow --read --write --owner b2-eu-cen --key ente-key
    ```
4.  Apply CORS Rules (Critical for Web App)  
    Create a file `cors.json` locally:  
      
    ```JSON  
    {
      "CORSRules": [
        {
          "AllowedHeaders": ["*"],
          "AllowedMethods": ["GET", "PUT", "POST", "DELETE", "HEAD"],
          "AllowedOrigins": ["*"],
          "ExposeHeaders": ["ETag", "Content-Length", "Content-Range", "x-amz-request-id"],
          "MaxAgeSeconds": 3600
        }
      ]
    }
    ```
    Apply it using AWS CLI (most robust method):  
      
    ```Bash  
    docker run --rm -v $(pwd):/aws \
      -e AWS_ACCESS_KEY_ID=YOUR_NEW_ACCESS_KEY \
      -e AWS_SECRET_ACCESS_KEY=YOUR_NEW_SECRET_KEY \
      -e AWS_DEFAULT_REGION=us-east-1 \
      amazon/aws-cli \
      --endpoint-url http://10.1.1.65:3900 \
      s3api put-bucket-cors --bucket b2-eu-cen --cors-configuration file:///aws/cors.json
    ```

* * *

### 5\. Connect Ente (`museum.yaml`)

Update your Ente config to use the **internal** docker network name (`garage`) but the new port (`3900`).

Generally everything in this file will be the the same as before the migration, except the s3 part. This has to point now to the new Garage-Backend.
As in the official docs, only the first bucket is relevant for self-hosting.

```YAML
s3:
  are_local_buckets: true
  b2-eu-cen:
    key: [YOUR_NEW_ACCESS_KEY]
    secret: [YOUR_NEW_SECRET_KEY]
    endpoint: https://photo-s3.domain.ch
    region: us-east-1
    bucket: b2-eu-cen
  wasabi-eu-central-2-v3:
    key: user
    secret: pass
    endpoint: https://photo-s3.domain.ch
    region: us-east-1
    bucket: wasabi-eu-central-2-v3
    compliance: false
  scw-eu-fr-v3:
    key: user
    secret: pass
    endpoint: https://photo-s3.domain.ch
    region: us-east-1
    bucket: scw-eu-fr-v3
```
* * *

### 6\. Configure Caddy (Reverse Proxy)

This block handles the public facing endpoint:

CADDYFILE:
```CADDYFILE

photo-s3.domain.ch {
    header Access-Control-Expose-Headers "ETag, Content-Length, Content-Range"

    reverse_proxy 10.1.1.65:3900 {
        header_up Host {host}
        header_up X-Real-IP {remote_host}
    }
}
```
* * *

### 7\. Data Migration (Rclone)

To migrate the data from MinIO to Garage, the best way is to do it with  rclone.

**DO NOT** copy the raw files with `cp` or something!

### First create the rclone remotes

Here is the step-by-step guide to setting up both `rclone` remotes.

You can either run through the interactive `rclone config` menu step-by-step, or simply paste the raw configuration directly into your config file.

--------------------------------------
#### Method 1: The Fast Way (Direct Config Edit)

If you want to save time, you can just paste the text below directly into your `rclone` configuration file (usually located at `~/.config/rclone/rclone.conf` or `/root/.config/rclone/rclone.conf`).

Open the file:

```Bash
nano ~/.config/rclone/rclone.conf
```
Paste this block at the bottom, save, and exit:

```TOML
[minio]
type = s3
provider = Minio
env_auth = false
access_key_id = admin
secret_access_key = password
endpoint = http://10.1.1.66:9000
acl = private

[garage]
type = s3
provider = Other
env_auth = false
access_key_id = [YOUR_NEW_ACCESS_KEY]
secret_access_key = [YOUR_NEW_SECRET_KEY]
region = us-east-1
endpoint = http://10.1.1.65:3900
acl = private
```

* * *
--------------------------------------
#### Method 2: The Interactive Way (Step-by-Step)

If you prefer to use the wizard, run `rclone config` in your terminal and follow these exact keystrokes.

### 1\. Setup the "Rescue" MinIO (Source)

This points to the temporary Docker container running on port 9000 that we used to translate the raw `xl.meta` files.

1.  **`n`** (New remote)
2.  **name**: `minio`
3.  **Storage**: `s3` (Amazon S3 Compliant Storage Providers)
4.  **provider**: `Minio`
5.  **env\_auth**: `false` (Enter your credentials in the next step)
6.  **access\_key\_id**: `admin` _(This matches the `MINIO_ROOT_USER` from the docker command)_
7.  **secret\_access\_key**: `password` _(This matches the `MINIO_ROOT_PASSWORD`)_
8.  **region**: _(Leave blank, press Enter)_
9.  **endpoint**: `http://10.1.1.66:9000` _(I've moved MinIO to a temporary IP)_
10.  **location\_constraint**: _(Leave blank, press Enter)_
11.  **acl**: `private`
12.  **Edit advanced config?**: `n`
13.  **Keep this "minio" remote?**: `y`

#### 2\. Setup the Garage S3 (Destination)

This points to your permanent Garage cluster running on your MacVLAN IP.

1.  **`n`** (New remote)
2.  **name**: `garage`
3.  **Storage**: `s3` (Amazon S3 Compliant Storage Providers)
4.  **provider**: `Other` (Any other S3 compatible provider)
5.  **env\_auth**: `false`
6.  **access\_key\_id**: `[YOUR_NEW_ACCESS_KEY]`
7.  **secret\_access\_key**: `[YOUR_NEW_SECRET_KEY]`
8.  **region**: `us-east-1`
9.  **endpoint**: `http://10.1.1.65:3900`
10.  **location\_constraint**: _(Leave blank, press Enter)_
11.  **acl**: `private`
12.  **Edit advanced config?**: `n`
13.  **Keep this "garage" remote?**: `y`
14.  **`q`** (Quit config)
    
--------------------------------------
### Verification

Once both are added, you can quickly test them by asking `rclone` to list the buckets on each:

```Bash
# Test MinIO Source
rclone lsd minio:

# Test Garage Destination
rclone lsd garage:
```
If both commands return a list showing `b2-eu-cen`, your connection is perfect and you are ready to copy.

1.  **Start "Rescue" MinIO:** Mount your backup folder to a temporary MinIO container. Here is an example:
    ```YAML
    services:
      minio:
        image: minio/minio
        restart: unless-stopped
        networks:
          macvlan_net:
            ipv4_address: 10.1.1.67
            mac_address: 02:00:00:00:00:67
        dns:
          - 10.10.10.1
        ports:
          - 3200:3200
          - 3201:3201
        environment:
          MINIO_ROOT_USER: $minio_user
          MINIO_ROOT_PASSWORD: $minio_pass
          PUID: 65534
          PGID: 100
        command: server /data --address ":3200" --console-address ":3201"
        volumes:
          - /mnt/Media/Pictures/ente_photos_bak:/data
        post_start:
          - command: >
              sh -c '
              #!/bin/sh
              while ! mc alias set h0 http://minio:3200 $minio_user $minio_pass 2>/dev/null
              do
                echo "Waiting for minio..."
                sleep 0.5
              done
              cd /data
              mc mb -p b2-eu-cen
              mc mb -p wasabi-eu-central-2-v3
              mc mb -p scw-eu-fr-v3
              '
    networks:
      macvlan_net:
        external: true
    ```
    
2.  **Configure Rclone:** Point `[minio]` to the Rescue container and `[garage]` to Garage.
    
3.  **Run the Sync:**
    
    ```Bash
    
    rclone copy minio:b2-eu-cen garage:b2-eu-cen \
      --progress --transfers 32 --checkers 64
    ```

**You should now be fully operational.**
