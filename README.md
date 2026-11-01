![Build Status](https://img.shields.io/badge/Dedicated_Server-Linux-green)
![Terraria](https://img.shields.io/badge/Terraria-8A2BE2)
# Terraria Linux Server Setup Guide

Step-by-step guide to set up a Ubuntu Terraria dedicated server with automated backups, updates, graceful shutdown, and a systemd unit that correctly tracks the real game process.

**Prerequisites**

- Ubuntu server (20.04+)
- A user with sudo privileges

> [!Caution]
> Paths below assume the server user is `your_username` - adjust to match your own setup.

> [!Important]
> **Terraria's dedicated server is distributed directly from terraria.org, not SteamCMD**. App ID `105600` via SteamCMD requires the account to actually *own* the paid game (per LinuxGSM's own documentation); an anonymous SteamCMD login reports a false "Success! fully installed" while downloading zero actual bytes. The direct download is the real, officially-documented path (per the official Terraria Wiki) and needs no Steam account at all.

> [!Important]
> **Terraria does not handle `SIGTERM` gracefully - it exits immediately without saving.** The only real graceful shutdown is typing `exit` into the running console. There's no signal-based fallback worth trying; `stop_server.sh` below goes straight from `exit` to `SIGKILL` if that doesn't work.

> [!Note]
> Each downloaded release unpacks into a **version-numbered folder** (e.g. `1458/Linux/`). Unlike some community guides assume, there **is** a real, working, machine-readable way to check the current version: `https://www.terraria.org/api/get/dedicated-servers-names` returns a JSON array with the latest release first - confirmed live. This guide's scripts use it directly, mirroring the same real-update-detection pattern used for the SteamCMD-based games in this series.

--------------------------------------------------------------------------------
# Step 1: Update and Upgrade Your System

    sudo apt update && sudo apt full-upgrade -y && sudo apt autoremove -y

--------------------------------------------------------------------------------
# Step 2: Install Required Dependencies

    sudo apt install screen openssh-server unzip curl jq ufw -y

--------------------------------------------------------------------------------
# Step 3: Configure UFW

Adjust the port below to match whatever you set in `serverconfig.txt` (Step 5):

    sudo ufw allow from any proto tcp to any port 7777 comment "Terraria Server Port"
    sudo ufw allow from any to any port 22 comment "SSH"
    sudo ufw default deny incoming
    sudo ufw enable
    sudo ufw status

--------------------------------------------------------------------------------
# Step 4: Create a Non Sudo User

    sudo adduser your_username
    sudo reboot

--------------------------------------------------------------------------------
# Step 5: Configure the Server

    mkdir -p /home/your_username/terraria_server
    nano /home/your_username/terraria_server/serverconfig.txt

This file stays outside the version-numbered folders deliberately, so it survives every future update untouched - `start_server.sh` copies it into place before each launch.

    world=/home/your_username/.local/share/Terraria/Worlds/world1.wld
    autocreate=3
    worldname=your_server_name
    difficulty=0
    maxplayers=16
    port=7777
    password=your_server_password
    motd=Welcome to the server!
    upnp=0
    worldevil=random
    seed=

> [!Important]
> `autocreate` only generates a new world if the file at `world=` doesn't already exist - `1`/`2`/`3` set the size (small/medium/large). If you already have a world you want to use, point `world=` at its real path (find it with `find ~ -iname "*.wld" -type f`) and the server loads it directly with no prompts, regardless of what `autocreate` says. Confirmed on a real server: with the file present, `-config serverconfig.txt` boots straight to a running server, zero interactive input needed.

> [!Note]
> `worldevil` (`random`/`corrupt`/`crimson`) and `seed` (blank for random, or a specific value/keyword like `forthewary`) only matter for `autocreate` - both are ignored once `world=` points at an existing file.

Lock down the file, since it holds your server password in plaintext:

    chmod 600 /home/your_username/terraria_server/serverconfig.txt

**Confirmed available console commands**, useful for reference:

    help            Displays a list of commands.
    playing         Shows the list of players.
    exit            Shutdown the server and save.
    exit-nosave     Shutdown the server without saving.
    save            Save the game world.
    kick <player>   Kicks a player from the server.
    ban <player>    Bans a player from the server.
    password        Show password.
    say <words>     Send a message.
    motd            Print MOTD.

--------------------------------------------------------------------------------
# Step 6: Create the Start Script

    cd
    mkdir .scripts
    cd .scripts
    nano start_server.sh

Update `DIRPATH`, `BACKUP_DIR`, and `LOG_DIR` to match your setup:

    #!/bin/bash
    set -o pipefail

    #set -x     # Uncomment to enable debug output.

    INSTANCE_NAME="Terraria"
    DIRPATH="/home/your_username/terraria_server"
    WORLD_DIR="/home/your_username/.local/share/Terraria/Worlds"
    CONFIG_FILE="$DIRPATH/serverconfig.txt"
    BACKUP_DIR="/home/your_username/backups"
    TIMESTAMP=$(date '+%Y-%m-%d_%H%M%S')
    LOG_DIR="/home/your_username/logs"
    LOGFILE="$LOG_DIR/terraria_$TIMESTAMP.log"

    PID_DIR="/home/your_username/.run"
    PID_FILE="$PID_DIR/terraria.pid"
    GAME_PROCESS_NAME="TerrariaServer.bin.x86_64"

    # Returns a JSON array of filenames, latest release first.
    VERSION_API="https://www.terraria.org/api/get/dedicated-servers-names"

    export SCREENDIR="/home/your_username/.screen"

    mkdir -p "$LOG_DIR"
    mkdir -p "$BACKUP_DIR"
    mkdir -p "$PID_DIR"

    # --- SELF-LOCK: prevent overlapping invocations from racing the duplicate check ---
    LOCK_DIR="/home/your_username/.flock"
    LOCK_FILE="$LOCK_DIR/start_server.lock"
    mkdir -p "$LOCK_DIR"

    exec 200>"$LOCK_FILE"
    if ! flock -n 200; then
        echo "$(date '+%Y-%m-%d %H:%M:%S') - Another instance of start_server.sh is already running (lock held). Exiting cleanly."
        exit 0
    fi

    log() {
        echo "$(date '+%Y-%m-%d %H:%M:%S') - $1"
    }

    # Installed versions that actually contain a runnable server binary, newest first.
    # A folder alone is not proof of an install: an interrupted extract can leave an
    # empty or partial version folder behind.
    working_versions() {
        local d
        ls -d "$DIRPATH"/*/ 2>/dev/null | sort -rV | while read -r d; do
            [ -x "${d}Linux/TerrariaServer.bin.x86_64" ] && basename "$d"
        done
    }

    {
        # --- STEP 0: DUPLICATE SCREEN SESSION PROTECTION ---
        if /usr/bin/screen -list | grep -q "\.${INSTANCE_NAME}[[:space:]]"; then
            log "'${INSTANCE_NAME}' is already running in an active Screen session. Nothing to do."
            exit 0
        fi

        # --- STEP 1: BACKUP & LOG ROTATION ---
        # World files live outside DIRPATH entirely (Terraria's own default), not
        # nested under the install directory like the SteamCMD-based games.
        log "Initiating pre-launch Terraria world backup..."
        if [ -d "$WORLD_DIR" ]; then
            tar -czf "$BACKUP_DIR/terraria_backup_$TIMESTAMP.tar.gz" -C "$WORLD_DIR" .
            if [ $? -eq 0 ]; then
                log "Backup successfully created: terraria_backup_$TIMESTAMP.tar.gz"
            else
                log "Warning: Backup compression encountered errors."
            fi
        else
            log "Warning: Worlds directory not found. Skipping backup step."
        fi

        find "$BACKUP_DIR" -name "terraria_backup_*.tar.gz" -type f -mtime +30 -exec rm -f {} \;
        log "Backup rotation check complete (30-day cutoff)."
        find "$LOG_DIR" -name "terraria_*.log" -type f -mtime +30 -exec rm -f {} \;
        log "Log rotation check complete."

        # --- STEP 2: GAME ENGINE SOFTWARE UPDATE ---
        # Mirrors the SteamCMD games' pattern: this runs unconditionally on
        # every launch, regardless of what triggered it (boot, crash-recovery,
        # or update_checker.sh), and only actually downloads if the target
        # version isn't already present locally.
        log "Checking terraria.org for the current server version..."
        LATEST_FILENAME=$(curl -fsS --max-time 20 "$VERSION_API" 2>/dev/null | jq -r '.[0]' 2>/dev/null)
        LATEST_VERSION=$(echo "$LATEST_FILENAME" | sed -E 's/terraria-server-(.+)\.zip/\1/')

        if [ -z "$LATEST_VERSION" ] || [ "$LATEST_VERSION" == "null" ]; then
            log "Warning: Could not reach the version API. Falling back to whatever's already installed locally."
            LATEST_VERSION=$(working_versions | head -n1)
            if [ -z "$LATEST_VERSION" ]; then
                log "Critical Error: Version API unreachable and no working install found locally. Aborting."
                exit 1
            fi
        fi

        GAME_DIR="$DIRPATH/$LATEST_VERSION/Linux"

        # Leftovers from an interrupted earlier update (hidden, so they never count as a version).
        rm -rf "$DIRPATH"/.stage.* 2>/dev/null

        if [ ! -x "$GAME_DIR/TerrariaServer.bin.x86_64" ]; then
            log "Version $LATEST_VERSION not present locally. Downloading..."
            STAGE=$(mktemp -d "$DIRPATH/.stage.XXXXXX")
            # Everything is built in a hidden staging folder and only moved into place once
            # it is complete and verified, so a failed download or extract can never leave
            # a half-installed version folder behind.
            if [ -d "$STAGE" ] \
                && wget -q --tries=3 --timeout=60 "https://terraria.org/api/download/pc-dedicated-server/terraria-server-$LATEST_VERSION.zip" -O "$STAGE/server.zip" \
                && unzip -tq "$STAGE/server.zip" > /dev/null \
                && unzip -q "$STAGE/server.zip" -d "$STAGE/extracted" \
                && [ -f "$STAGE/extracted/$LATEST_VERSION/Linux/TerrariaServer.bin.x86_64" ] \
                && chmod +x "$STAGE/extracted/$LATEST_VERSION/Linux/TerrariaServer.bin.x86_64"; then
                # Each release bundles Windows/Mac/Linux builds together - only
                # Linux is ever used here, so drop the other two.
                rm -rf "$STAGE/extracted/$LATEST_VERSION/Windows" "$STAGE/extracted/$LATEST_VERSION/Mac"
                rm -rf "$DIRPATH/$LATEST_VERSION"   # any earlier broken attempt
                mv "$STAGE/extracted/$LATEST_VERSION" "$DIRPATH/$LATEST_VERSION"
                rm -rf "$STAGE"
                log "Version $LATEST_VERSION downloaded, verified and installed (Linux build only)."
            else
                rm -rf "$STAGE"
                log "Warning: Download/extract of version $LATEST_VERSION failed."
                FALLBACK_VERSION=$(working_versions | head -n1)
                if [ -z "$FALLBACK_VERSION" ]; then
                    log "Critical Error: No working install to fall back on. Aborting."
                    exit 1
                fi
                log "Falling back to the already-installed working version $FALLBACK_VERSION so the server still comes up."
                LATEST_VERSION="$FALLBACK_VERSION"
                GAME_DIR="$DIRPATH/$LATEST_VERSION/Linux"
            fi
        else
            log "Version $LATEST_VERSION already present locally. No download needed."
        fi

        # --- STEP 2.5: PRUNE OLD VERSION FOLDERS ---
        # Keeps the current version plus one previous (for easy manual
        # rollback), removes anything older. Each version's Linux build is
        # tens of MB at most, so this is about avoiding clutter over years
        # of updates, not meaningful disk space.
        log "Pruning old version folders (keeping the 2 newest working versions)..."
        KEEP_VERSIONS=$(working_versions | head -n 2)
        ls -d "$DIRPATH"/*/ 2>/dev/null | while read -r old_dir; do
            if ! echo "$KEEP_VERSIONS" | grep -qx "$(basename "$old_dir")"; then
                log "Removing old or incomplete version folder: $old_dir"
                rm -rf "$old_dir"
            fi
        done
        find "$DIRPATH" -maxdepth 1 -name "terraria-server-*.zip" -type f 2>/dev/null | sort -V | head -n -2 | while read -r old_zip; do
            log "Removing old downloaded zip: $old_zip"
            rm -f "$old_zip"
        done

        # --- STEP 3: START APPLICATION WINDOW ---
        log "Starting Terraria server (version $LATEST_VERSION) inside Screen session..."
        cp "$CONFIG_FILE" "$GAME_DIR/serverconfig.txt"
        cd "$GAME_DIR"
        /usr/bin/screen -dmS "$INSTANCE_NAME" ./TerrariaServer.bin.x86_64 -config serverconfig.txt

        sleep 5

        if /usr/bin/screen -list | grep -q "\.${INSTANCE_NAME}[[:space:]]"; then
            log "Screen session is alive. Verifying the actual game process next..."
        else
            log "Critical Error: Failed to start Terraria screen session."
            exit 1
        fi

        # --- STEP 3.5: CAPTURE THE ACTUAL GAME PROCESS PID ---
        log "Waiting for $GAME_PROCESS_NAME to confirm the server actually started..."
        PID_WAIT_MAX=60
        PID_WAIT_COUNT=0
        GAME_PID=""

        while [ -z "$GAME_PID" ] && [ $PID_WAIT_COUNT -lt $PID_WAIT_MAX ]; do
            GAME_PID=$(pgrep -f "$GAME_PROCESS_NAME" | head -n1)
            if [ -z "$GAME_PID" ]; then
                sleep 1
                ((PID_WAIT_COUNT++))
            fi
        done

        if [ -n "$GAME_PID" ]; then
            echo "$GAME_PID" > "$PID_FILE"
            log "Terraria server started successfully in background (PID $GAME_PID)."
        else
            log "Critical Error: $GAME_PROCESS_NAME never appeared after ${PID_WAIT_MAX}s."
            log "If world= doesn't point at a real existing file, the server hangs on an interactive prompt here instead of starting - check serverconfig.txt."
            exit 1
        fi
    } 2>&1 | tee -a "$LOGFILE"

Make it executable:

    chmod u+x start_server.sh

> [!Note]
> First run downloads the current version fresh - test it manually once before wiring up systemd, same as with every other game in this series:
>
>     bash start_server.sh
>     screen -r Terraria

--------------------------------------------------------------------------------
# Step 7: Create the Stop Script

    nano stop_server.sh

    #!/bin/bash
    set -o pipefail

    INSTANCE_NAME="Terraria"
    GAME_PROCESS_NAME="TerrariaServer.bin.x86_64"

    TIMESTAMP=$(date '+%Y-%m-%d_%H%M%S')
    LOG_DIR="/home/your_username/logs"
    LOGFILE="$LOG_DIR/terraria_stop_$TIMESTAMP.log"

    PID_DIR="/home/your_username/.run"
    PID_FILE="$PID_DIR/terraria.pid"

    export SCREENDIR="/home/your_username/.screen"

    mkdir -p "$LOG_DIR"

    log() {
        echo "$(date '+%Y-%m-%d %H:%M:%S') - $1"
    }

    {
        find "$LOG_DIR" -name "terraria_stop_*.log" -type f -mtime +30 -exec rm -f {} \;
        log "Stop log rotation check complete."

        if /usr/bin/screen -list | grep -q "\.${INSTANCE_NAME}[[:space:]]"; then
            log "Active ${INSTANCE_NAME} session discovered. Initializing graceful countdown..."

            # 'say' broadcasts to connected players - confirmed real via the
            # console's own command list. 'exit' below handles the actual
            # save+shutdown as one atomic action once the countdown finishes.
            /usr/bin/screen -S "$INSTANCE_NAME" -X stuff $'say Server shutting down in 5 minutes! Please prepare.\n'
            sleep 120
            /usr/bin/screen -S "$INSTANCE_NAME" -X stuff $'say Server shutting down in 3 minutes! Find a safe spot.\n'
            sleep 120
            /usr/bin/screen -S "$INSTANCE_NAME" -X stuff $'say Server shutting down in 60 seconds! Please log out.\n'
            sleep 30
            /usr/bin/screen -S "$INSTANCE_NAME" -X stuff $'say Server shutting down in 30 seconds!\n'
            sleep 30

            log "Sending 'exit' (the game's own documented graceful stop command - Terraria does not handle SIGTERM gracefully)..."

            # Sends the literal text "exit" followed by Enter into the console.
            /usr/bin/screen -S "$INSTANCE_NAME" -X stuff $'exit\n'

            log "Waiting for the world to finish saving and the process to exit..."
            MAX_WAIT=120    # seconds (one pass of the loop = 1 second)
            COUNT=0
            while pgrep -f "$GAME_PROCESS_NAME" > /dev/null && [ $COUNT -lt $MAX_WAIT ]; do
                sleep 1
                ((COUNT++))
            done

            SERVER_PID=$(pgrep -f "$GAME_PROCESS_NAME")
            if [ -n "$SERVER_PID" ]; then
                log "Process still alive after 'exit' (${MAX_WAIT}s wait). Force killing as last resort (possible data loss - Terraria doesn't save on SIGKILL either)."
                kill -9 "$SERVER_PID"
            else
                log "Process exited cleanly after 'exit' command."
            fi

            if /usr/bin/screen -list | grep -q "\.${INSTANCE_NAME}[[:space:]]"; then
                log "Screen session still present. Force closing."
                /usr/bin/screen -S "$INSTANCE_NAME" -X quit
            fi

            [ -f "$PID_FILE" ] && rm -f "$PID_FILE"
            log "Terraria engine instance safely terminated."
        else
            log "No active server screen found. Lifecycle step skipped."
        fi
    } 2>&1 | tee -a "$LOGFILE"

Make it executable:

    chmod u+x stop_server.sh

--------------------------------------------------------------------------------
# Step 8: Create an Automated Update Checker (Optional)

    nano update_checker.sh

    #!/bin/bash

    set -o pipefail

    GAME_NAME="Terraria"
    DIRPATH="/home/your_username/terraria_server"
    STOP_SCRIPT="/home/your_username/.scripts/stop_server.sh"
    START_SCRIPT="/home/your_username/.scripts/start_server.sh"
    SCREEN_NAME="Terraria"
    RESTART_POLL_MAX_WAIT=90
    TIMESTAMP=$(date '+%Y-%m-%d_%H%M%S')
    LOG_DIR="/home/your_username/logs"
    LOG_FILE="terraria_update_$TIMESTAMP.log"
    WEBHOOK_URL=""
    IMAGE_URL=""
    DISCORD_FOOTER="Server Maintenance Automation"

    # Returns a JSON array of filenames, latest release first.
    VERSION_API="https://www.terraria.org/api/get/dedicated-servers-names"

    export SCREENDIR="/home/your_username/.screen"

    FULL_LOG_PATH="$LOG_DIR/$LOG_FILE"

    # Installed versions that actually contain a runnable server binary, newest first
    # (a leftover or half-extracted folder does not count).
    working_versions() {
        local d
        ls -d "$DIRPATH"/*/ 2>/dev/null | sort -rV | while read -r d; do
            [ -x "${d}Linux/TerrariaServer.bin.x86_64" ] && basename "$d"
        done
    }

    # If this file exists, the checker refuses to stop the server. It is created
    # when an update did not apply after a restart, so a failed update can never
    # restart the server every 30 minutes. Clear it with: rm "$BLOCK_FLAG"
    BLOCK_FLAG="/home/your_username/.run/update_checker_blocked"

    mkdir -p "$LOG_DIR"

    log() {
        echo "$1"
    }

    send_discord_message() {
        local message="$1"
        local json_payload
        local emoji="🛠️"

        if [[ "$message" == *"No updates found"* ]]; then
            emoji="🛠️ ℹ️"
        elif [[ "$message" == *"Maintenance Complete"* ]]; then
            emoji="🛠️ 🔄"
        elif [[ "$message" == *"Updates Found"* ]]; then
            emoji="🛠️ ✅"
        elif [[ "$message" == *"Maintenance Failed"* ]]; then
            emoji="🛠️ ❌"
        fi

        if [[ "$message" == *"Maintenance Started"* ]]; then
            json_payload=$(cat <<EOF
    {
        "embeds": [{
            "title": "$emoji $message",
            "color": 16711680,
            "image": { "url": "$IMAGE_URL" },
            "footer": { "text": "$DISCORD_FOOTER" }
        }]
    }
    EOF
    )
        else
            json_payload=$(cat <<EOF
    {
        "embeds": [{
            "title": "$emoji $message",
            "color": 16711680,
            "footer": { "text": "$DISCORD_FOOTER" }
        }]
    }
    EOF
    )
        fi

        if [[ -n "$WEBHOOK_URL" ]]; then
            curl -s -H "Content-Type: application/json" -X POST -d "$json_payload" "$WEBHOOK_URL" > /dev/null 2>&1
        fi
    }

    {
        find "$LOG_DIR" -name "terraria_update_*.log" -type f -mtime +3 -exec rm -f {} \;
        log "$(date '+%Y-%m-%d %H:%M:%S') - Update-checker log rotation check complete."

        # --- HONOR THE SAFETY BLOCK ---
        if [ -f "$BLOCK_FLAG" ]; then
            log "$(date '+%Y-%m-%d %H:%M:%S') - Checker is BLOCKED by $BLOCK_FLAG (a previous update did not apply). Not touching the server."
            log "Reason recorded: $(head -c 300 "$BLOCK_FLAG")"
            log "Investigate, then run: rm $BLOCK_FLAG"
            exit 0
        fi

        # --- READ THE TRUE INSTALLED VERSION FROM DISK ---
        # No separate manifest needed - the version-numbered folder itself is
        # the source of truth, and only counts if the real binary is present.
        LOCAL_VERSION=$(working_versions | head -n1)
        if [ -z "$LOCAL_VERSION" ]; then
            log "$(date '+%Y-%m-%d %H:%M:%S') - CRITICAL ERROR: No installed version found under $DIRPATH."
            exit 1
        fi

        # --- QUERY THE REMOTE VERSION API ---
        LATEST_FILENAME=$(curl -fsS --max-time 20 "$VERSION_API" 2>/dev/null | jq -r '.[0]' 2>/dev/null)
        LATEST_VERSION=$(echo "$LATEST_FILENAME" | sed -E 's/terraria-server-(.+)\.zip/\1/')

        if [ -z "$LATEST_VERSION" ] || [ "$LATEST_VERSION" == "null" ]; then
            log "$(date '+%Y-%m-%d %H:%M:%S') - Error: Version API unreachable. Skipping check."
            exit 1
        fi

        if [ "$LATEST_VERSION" != "$LOCAL_VERSION" ]; then
            send_discord_message "Maintenance Started: Updates Found for $GAME_NAME ($LOCAL_VERSION -> $LATEST_VERSION). Initializing patching pipeline."
            log "$(date +'%Y-%m-%d %H:%M:%S') Executing established graceful shutdown script..."
            bash "$STOP_SCRIPT"

            log "$(date +'%Y-%m-%d %H:%M:%S') Server process terminated. Waiting for systemd auto-restart to bring it back (start_server.sh will download $LATEST_VERSION automatically)..."

            POLL_COUNT=0
            while ! screen -list | grep -q "\.${SCREEN_NAME}[[:space:]]" && [ $POLL_COUNT -lt $RESTART_POLL_MAX_WAIT ]; do
                sleep 2
                ((POLL_COUNT++))
            done

            if screen -list | grep -q "\.${SCREEN_NAME}[[:space:]]"; then
                log "$(date +'%Y-%m-%d %H:%M:%S') Systemd auto-restart succeeded — server back online after $((POLL_COUNT*2))s."
            else
                log "$(date +'%Y-%m-%d %H:%M:%S') WARNING: No screen session detected. Falling back to manual start..."
                send_discord_message "Maintenance Failed: Systemd auto-restart did not bring $GAME_NAME back — falling back to manual start."
                bash "$START_SCRIPT"
                sleep 5
                if screen -list | grep -q "\.${SCREEN_NAME}[[:space:]]"; then
                    log "$(date +'%Y-%m-%d %H:%M:%S') Manual fallback start succeeded."
                else
                    log "$(date +'%Y-%m-%d %H:%M:%S') CRITICAL ERROR: Manual fallback also failed. Server is likely DOWN."
                    send_discord_message "Maintenance Failed: Manual fallback start also failed. Server is likely DOWN — manual intervention required."
                    exit 1
                fi
            fi

            # --- VERIFY THE UPDATE ACTUALLY LANDED ---
            NEW_LOCAL_VERSION=$(working_versions | head -n1)
            if [ "$NEW_LOCAL_VERSION" != "$LATEST_VERSION" ]; then
                log "$(date +'%Y-%m-%d %H:%M:%S') ERROR: Update did NOT apply (installed: ${NEW_LOCAL_VERSION:-none}, expected: $LATEST_VERSION)."
                echo "$(date +'%Y-%m-%d %H:%M:%S') update to $LATEST_VERSION did not apply (installed: ${NEW_LOCAL_VERSION:-none})" > "$BLOCK_FLAG"
                log "Checker blocked via $BLOCK_FLAG so it cannot restart the server in a loop."
                send_discord_message "Maintenance Failed: $GAME_NAME update to $LATEST_VERSION did not apply after restart. Auto-updates paused until an admin clears the block."
                exit 1
            fi
            send_discord_message "Maintenance Complete: $GAME_NAME is updated to $LATEST_VERSION and back online."
            log "$(date +'%Y-%m-%d %H:%M:%S') --- MAINTENANCE SEQUENCE COMPLETE ---"
        else
            log "$(date '+%Y-%m-%d %H:%M:%S') - $GAME_NAME engine fully optimized and up to date ($LOCAL_VERSION)."
        fi
    } 2>&1 | tee -a "$FULL_LOG_PATH"

Make it executable, then schedule it:

    chmod u+x update_checker.sh
    crontab -e

Add (every 30 minutes shown):

    */30 * * * * flock -n /home/your_username/.flock/update_checker.lock /home/your_username/.scripts/update_checker.sh

> [!Note]
> If an update does not apply after the restart (for example the download failed), the checker writes `/home/your_username/.run/update_checker_blocked`, sends a Discord alert, and stops touching the server so it cannot restart it every 30 minutes. After you have looked at the cause, clear it with `rm /home/your_username/.run/update_checker_blocked`.

--------------------------------------------------------------------------------
# Step 9: Create a Systemd Service

    sudo nano /etc/systemd/system/Terraria.service

**Base configuration** - replace `your_username` throughout:

    [Unit]
    Description=Your Terraria Dedicated Server
    After=network.target network-online.target
    Wants=network-online.target
    StartLimitIntervalSec=1200
    StartLimitBurst=3

    [Service]
    Type=forking
    User=your_username
    Group=your_username
    WorkingDirectory=/home/your_username
    ExecStart=/home/your_username/.scripts/start_server.sh
    ExecStop=/home/your_username/.scripts/stop_server.sh
    PIDFile=/home/your_username/.run/terraria.pid
    KillMode=control-group
    SendSIGKILL=no
    TimeoutStartSec=180
    TimeoutStopSec=480
    RemainAfterExit=no
    Restart=always
    RestartSec=60
    StandardOutput=journal
    StandardError=journal

    [Install]
    WantedBy=multi-user.target

> [!Important]
> `Restart=always`, not `on-failure` - `on-failure` doesn't count a `SIGKILL`-terminated exit (the stop script's last-resort fallback) as a failure, so it can silently skip auto-restart.

> [!Note]
> `TimeoutStartSec=180` covers a normal launch with an existing world/version loading directly - raise it if you're generating a large world or downloading a fresh version for the first time.

> [!Note]
> `TimeoutStopSec=480` covers the full worst case of `stop_server.sh`: the player-facing countdown (5 min warning through 30s warning = 300s) plus up to 120s waiting for `exit` to save and terminate, plus 60s of margin. Keep it above that sum if you change either number. Without enough headroom here, systemd force-kills the whole process group mid-countdown - silently skipping every warning, not just cutting the last one short.

**Enable, start, and confirm before moving on:**

    sudo systemctl daemon-reload
    sudo systemctl enable Terraria.service
    sudo systemctl start Terraria.service
    sudo systemctl status Terraria.service
    cat /home/your_username/.run/terraria.pid

--------------------------------------------------------------------------------
## Recommended: Harden the Service

Layer this on once the base unit is confirmed working - gives you a simpler fallback if anything here needs debugging.

**Lock down the scripts** (files before directory - your own shell expands `*.sh` before `sudo` runs):

    sudo chown root:your_username /home/your_username/.scripts/*.sh
    sudo chmod 750 /home/your_username/.scripts/*.sh
    sudo chown root:your_username /home/your_username/.scripts
    sudo chmod 750 /home/your_username/.scripts

**Add to `[Service]`:**

    Environment="SCREENDIR=/home/your_username/.screen"
    ExecStartPre=+/bin/mkdir -p /home/your_username/.screen
    ExecStartPre=+/bin/chown -R your_username:your_username /home/your_username/.screen
    ExecStartPre=+/bin/chmod 700 /home/your_username/.screen
    NoNewPrivileges=true
    PrivateTmp=true
    ProtectSystem=strict
    ProtectHome=read-only
    ReadWritePaths=/home/your_username/terraria_server
    ReadWritePaths=/home/your_username/.run
    ReadWritePaths=/home/your_username/.flock
    ReadWritePaths=-/home/your_username/.screen
    ReadWritePaths=/home/your_username/logs
    ReadWritePaths=/home/your_username/backups
    ReadWritePaths=/home/your_username/.local/share/Terraria

> [!Important]
> `+` on `ExecStartPre=` is required - without it, these setup commands run inside the same sandbox as the main process, creating a circular dependency (sandbox needs `.screen` to exist to bind-mount it; nothing outside the sandbox exists yet to create it). Produces `status=217/USER` reproducibly on `ProtectSystem=strict` units. `+` runs the setup fully unsandboxed first, breaking the cycle.

**Reload and confirm:**

    sudo systemctl daemon-reload
    sudo systemctl reset-failed Terraria.service
    sudo systemctl restart Terraria.service
    sudo systemctl status Terraria.service
    sudo systemd-analyze security Terraria.service

--------------------------------------------------------------------------------
# Step 10: Create the Host-Level Maintenance Script (Optional)

Handles OS patching and reboot. Needs root - not your game user's `.scripts`/crontab.

    sudo nano /usr/local/sbin/terraria_maintenance.sh

    #!/bin/bash

    PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin

    LOGFILE="/var/log/terraria_system_maintenance.log"
    SERVICE_NAME="Terraria.service"
    WEBHOOK_URL=""
    IMAGE_URL=""

    send_discord_message() {
        local message="$1"
        local json_payload
        local emoji="🛠️"

        if [[ "$message" == *"No updates found"* ]]; then
            emoji="🛠️ ℹ️"
        elif [[ "$message" == *"Maintenance Complete"* ]]; then
            emoji="🛠️ 🔄"
        elif [[ "$message" == *"Updates Found"* ]]; then
            emoji="🛠️ ✅"
        fi

        if [[ "$message" == *"Maintenance Started"* ]]; then
            json_payload=$(cat <<EOF
    {
        "embeds": [{
            "title": "$emoji $message",
            "color": 16711680,
            "image": { "url": "$IMAGE_URL" },
            "footer": { "text": "System Maintenance Automation" }
        }]
    }
    EOF
    )
        else
            json_payload=$(cat <<EOF
    {
        "embeds": [{
            "title": "$emoji $message",
            "color": 16711680,
            "footer": { "text": "System Maintenance Automation" }
        }]
    }
    EOF
    )
        fi

        if [[ -n "$WEBHOOK_URL" ]]; then
            curl -s -H "Content-Type: application/json" -X POST -d "$json_payload" "$WEBHOOK_URL" > /dev/null 2>&1
        fi
    }

    exec > >(tee -a "$LOGFILE") 2>&1

    echo "=============================================================================="
    echo "$(date +'%Y-%m-%d %H:%M:%S') --- Starting Terraria Maintenance Sequence ---"

    send_discord_message "Maintenance Started: Stopping the Terraria server via systemd..."
    if systemctl stop "$SERVICE_NAME"; then
        echo "$(date +'%Y-%m-%d %H:%M:%S') $SERVICE_NAME stopped successfully."
    else
        echo "$(date +'%Y-%m-%d %H:%M:%S') WARNING: systemctl stop reported a non-zero exit for $SERVICE_NAME."
    fi

    apt-get update > /dev/null

    if apt-get upgrade -s 2>/dev/null | grep -q "^Inst"; then
        send_discord_message "Updates Found. Applying system patches..."
        DEBIAN_FRONTEND=noninteractive apt-get -o Dpkg::Options::="--force-confold" -y full-upgrade
        apt-get autoremove -y
        echo "$(date +'%Y-%m-%d %H:%M:%S') OS updates applied successfully."
    else
        send_discord_message "No updates found. System is clean."
        echo "$(date +'%Y-%m-%d %H:%M:%S') No OS updates found."
    fi

    send_discord_message "Maintenance Complete: Rebooting server host."
    sync
    sleep 10
    reboot

Make it executable:

    sudo chmod +x /usr/local/sbin/terraria_maintenance.sh

**Schedule via root's crontab** (not your game user's):

    sudo crontab -e

Add (5 AM daily shown):

    0 5 * * * flock -n /tmp/terraria_maintenance.lock /usr/local/sbin/terraria_maintenance.sh

--------------------------------------------------------------------------------
# Step 11: Hardening (Optional)

    sudo nano /etc/ssh/sshd_config

Set:

    LoginGraceTime 1m
    PermitRootLogin no
    MaxSessions 4

    sudo systemctl daemon-reload
    sudo systemctl restart ssh.service

![image](https://github.com/user-attachments/assets/f12f25af-807d-4981-9e53-ebe2ab3d2688)

**Change Who Can Use the Switch User (su) Command**

    sudo groupadd restrictedsu
    sudo nano /etc/pam.d/su

Add the line:

    auth       required   pam_wheel.so group=restrictedsu

![image](https://github.com/user-attachments/assets/3d3c941b-aadd-4bdb-b736-e2fb4c7b5c8b)

> [!TIP]
> For remote triggers (control panels, automation), use a forced-command SSH key restricted to one script instead of a normal login key: `command="/home/your_username/.scripts/start_server.sh",restrict ssh-ed25519 ...` in `authorized_keys`.

--------------------------------------------------------------------------------

**Conclusion**

You have successfully set up a hardened Terraria server with automated backups, real-time version-checked updates, graceful shutdown, and a systemd service that correctly tracks the real game process.

**References**
- https://terraria.wiki.gg/wiki/Server
- https://terraria.wiki.gg/wiki/Guide:Setting_up_a_Terraria_server
