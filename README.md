# 📊 PM2 Custom Ports Dashboard

A lightweight Bash extension for **PM2** that adds a professional networking column to your process list. It mimics the original PM2 interface while providing real-time information about which **TCP ports** your applications are listening on.

## ✨ Features

* **Native Look & Feel**: Designed to perfectly match the original `pm2 list` table.
* **Automatic Port Detection**: Uses `lsof`, `ss`, or `netstat` to find active listeners for each PID.
* **High Performance**: Global single-pass port scanning and subshell-free parsing.
* **Live Metrics**: Includes CPU, Memory, Uptime, Restarts, and Watching status.
* **Color Coded**: Status and ports are highlighted for quick readability.
* **Smart Sorting**: Processes are automatically sorted by port number.

---

## 🛠️ Prerequisites

This script requires `jq`. Install it on your system (e.g., Raspberry Pi OS, Ubuntu, Debian) using:

```bash
sudo apt update
sudo apt install jq -y
```

*(Optional: `lsof`, `iproute2` (for `ss`), or `net-tools` (for `netstat`) for port detection)*

---

## 🚀 Installation

### 1. Create the Script

Create a new file in your home directory:

```bash
nano ~/pm2-ports.sh
```

Paste the following code:

<details>
<summary><b>Click to show pm2-ports.sh code</b></summary>

```bash
#!/bin/bash

# --- COLOR CONFIGURATION ---
GRAY='\033[90m'
CYAN='\033[36m'
GREEN='\033[32m'
RED='\033[31m'
YELLOW='\033[33m'
NC='\033[0m'

V="${GRAY}│${NC}"

# Uptime Formatting Function
format_uptime() {
    local start_time=$1
    if [[ -z "$start_time" || "$start_time" == "null" || "$start_time" -le 0 ]]; then
        echo "0s"
        return
    fi
    local now
    now=$(date +%s%3N)
    local diff=$(( (now - start_time) / 1000 ))
    if [ $diff -lt 60 ]; then echo "${diff}s"
    elif [ $diff -lt 3600 ]; then echo "$((diff/60))m"
    elif [ $diff -lt 86400 ]; then echo "$((diff/3600))h"
    else echo "$((diff/86400))d"; fi
}

# Table Border Function (Calibrated to character width)
draw_line() {
    local start=$1 mid=$2 end=$3
    echo -ne "${GRAY}${start}"
    printf '─%.0s' {1..4};  echo -ne "${mid}" # id (4)
    printf '─%.0s' {1..14}; echo -ne "${mid}" # name (14)
    printf '─%.0s' {1..9};  echo -ne "${mid}" # mode (9)
    printf '─%.0s' {1..9};  echo -ne "${mid}" # pid (9)
    printf '─%.0s' {1..8};  echo -ne "${mid}" # uptime (8)
    printf '─%.0s' {1..5};  echo -ne "${mid}" # ↺ (5)
    printf '─%.0s' {1..10}; echo -ne "${mid}" # status (10)
    printf '─%.0s' {1..6};  echo -ne "${mid}" # cpu (6)
    printf '─%.0s' {1..10}; echo -ne "${mid}" # mem (10)
    printf '─%.0s' {1..10}; echo -ne "${mid}" # watching (10)
    printf '─%.0s' {1..10}; echo -ne "${end}${NC}\n" # port (10)
}

# Check prerequisites
if ! command -v pm2 >/dev/null 2>&1; then
    echo -e "${RED}Error: pm2 is not installed or not in PATH.${NC}"
    exit 1
fi

if ! command -v jq >/dev/null 2>&1; then
    echo -e "${RED}Error: jq is required but not installed.${NC}"
    exit 1
fi

# Global port detection map (Single pass for performance)
declare -A PID_PORTS

collect_all_ports() {
    local pid port
    if command -v lsof >/dev/null 2>&1; then
        while read -r pid port; do
            [[ -z "$pid" || -z "$port" ]] && continue
            if [[ -n "${PID_PORTS[$pid]}" ]]; then
                [[ ",${PID_PORTS[$pid]}," != *",$port,"* ]] && PID_PORTS[$pid]+=",$port"
            else
                PID_PORTS[$pid]="$port"
            fi
        done < <( (lsof -a -iTCP -sTCP:LISTEN -P -n 2>/dev/null || sudo -n lsof -a -iTCP -sTCP:LISTEN -P -n 2>/dev/null) | awk 'NR>1 { pid = $2; port = $9; sub(/^.*:/, "", port); if (pid != "" && port != "") print pid, port }' )
    elif command -v ss >/dev/null 2>&1; then
        while read -r port pid; do
            [[ -z "$pid" || -z "$port" ]] && continue
            if [[ -n "${PID_PORTS[$pid]}" ]]; then
                [[ ",${PID_PORTS[$pid]}," != *",$port,"* ]] && PID_PORTS[$pid]+=",$port"
            else
                PID_PORTS[$pid]="$port"
            fi
        done < <( (ss -tlnp 2>/dev/null || sudo -n ss -tlnp 2>/dev/null) | awk 'NR>1 { port = $4; sub(/^.*:/, "", port); line = $0; while (match(line, /pid=[0-9]+/)) { pid = substr(line, RSTART + 4, RLENGTH - 4); if (pid != "" && port != "") print port, pid; line = substr(line, RSTART + RLENGTH) } }' )
    elif command -v netstat >/dev/null 2>&1; then
        while read -r port pid; do
            [[ -z "$pid" || -z "$port" ]] && continue
            if [[ -n "${PID_PORTS[$pid]}" ]]; then
                [[ ",${PID_PORTS[$pid]}," != *",$port,"* ]] && PID_PORTS[$pid]+=",$port"
            else
                PID_PORTS[$pid]="$port"
            fi
        done < <( (netstat -tlnp 2>/dev/null || sudo -n netstat -tlnp 2>/dev/null) | awk 'NR>1 { port = $4; sub(/^.*:/, "", port); if (match($0, /[0-9]+\/[^ ]+/)) { pid = substr($0, RSTART, RLENGTH); sub(/\/.*$/, "", pid); if (pid != "" && port != "") print port, pid } }' )
    fi
}

get_all_descendants() {
    local parent=$1
    local children
    children=$(pgrep -P "$parent" 2>/dev/null)
    for child in $children; do
        echo "$child"
        get_all_descendants "$child"
    done
}

find_ports_for_process() {
    local target_pid=$1
    local env_port=$2
    local found_ports=""

    if [[ "$target_pid" -gt 0 ]]; then
        # 1. Check main PID
        if [[ -n "${PID_PORTS[$target_pid]}" ]]; then
            found_ports="${PID_PORTS[$target_pid]}"
        fi

        # 2. Check child PIDs (Process tree / wrappers like npm, yarn, npx, bash)
        local pids_to_check
        pids_to_check=$(get_all_descendants "$target_pid")
        for cpid in $pids_to_check; do
            if [[ -n "${PID_PORTS[$cpid]}" ]]; then
                if [[ -n "$found_ports" ]]; then
                    [[ ",$found_ports," != *",${PID_PORTS[$cpid]},"* ]] && found_ports+=","${PID_PORTS[$cpid]}
                else
                    found_ports="${PID_PORTS[$cpid]}"
                fi
            fi
        done
    fi

    # 3. Fallback to PM2 environment variable (PORT / env.PORT)
    if [[ -z "$found_ports" || "$found_ports" == "N/A" ]]; then
        if [[ -n "$env_port" && "$env_port" != "null" ]]; then
            found_ports="$env_port"
        fi
    fi

    echo "${found_ports:-N/A}"
}

# 1. DATA COLLECTION
temp_data=$(mktemp)
trap 'rm -f "$temp_data"' EXIT

collect_all_ports

pm2 jlist | jq -r '.[] | [.pm_id, .name, (.pm2_env.exec_mode // "" | gsub("_mode$"; "")), (.pid // 0), (.pm2_env.pm_uptime // 0), (.pm2_env.restart_time // 0), (.pm2_env.status // "stopped"), (.monit.cpu // 0), (.monit.memory // 0), (.pm2_env.watch // false), (.pm2_env.env.PORT // .pm2_env.PORT // "")] | @tsv' | while IFS=$'\t' read -r id name mode pid uptime_raw restarts status cpu mem_bytes watch env_port; do
    
    display_port="N/A"
    if [[ "$status" == "online" ]]; then
        display_port=$(find_ports_for_process "$pid" "$env_port")
    fi

    if [[ "$display_port" != "N/A" ]]; then
        first_port=$(echo "$display_port" | cut -d',' -f1)
        port_sort=${first_port:-99999}
    else
        port_sort=99999
    fi

    mem_mb=$(awk -v b="$mem_bytes" 'BEGIN {printf "%.1f", b/1048576}')
    cpu_formatted=$(awk -v c="$cpu" 'BEGIN {printf "%.0f%%", c}')
    uptime=$(format_uptime "$uptime_raw")

    echo "$port_sort|$id|$name|$mode|$pid|$uptime|$restarts|$status|$cpu_formatted|${mem_mb}mb|$watch|$display_port" >> "$temp_data"
done

# 2. TABLE HEADER
echo ""
draw_line "┌" "┬" "┐"
echo -e "${V} id ${V} name         ${V} mode    ${V} pid     ${V} uptime ${V} ↺   ${V} status   ${V} cpu  ${V} mem      ${V} watching ${V} port     ${V}"
draw_line "├" "┼" "┤"

# 3. TABLE ROWS
sort -n -t'|' -k1 "$temp_data" | while IFS='|' read -r port_sort id name mode pid uptime restarts status cpu mem watch display_port; do
    echo -ne "${V} $(printf "%-2.2s" "$id") "
    echo -ne "${V} ${CYAN}$(printf "%-12.12s" "$name")${NC} "
    echo -ne "${V} $(printf "%-7.7s" "$mode") "
    echo -ne "${V} $(printf "%-7.7s" "$pid") "
    echo -ne "${V} $(printf "%-6.6s" "$uptime") "
    echo -ne "${V} $(printf "%-3.3s" "$restarts") "

    if [ "$status" == "online" ]; then
        echo -ne "${V} ${GREEN}$(printf "%-8.8s" "$status")${NC} "
    else
        echo -ne "${V} ${RED}$(printf "%-8.8s" "$status")${NC} "
    fi

    echo -ne "${V} $(printf "%-4.4s" "$cpu") "
    echo -ne "${V} $(printf "%-8.8s" "$mem") "

    if [ "$watch" == "true" ]; then
        echo -ne "${V} ${GREEN}$(printf "%-8.8s" "enabled")${NC} "
    else
        echo -ne "${V} ${GRAY}$(printf "%-8.8s" "disabled")${NC} "
    fi

    echo -e "${V} ${YELLOW}$(printf "%-8.8s" "$display_port")${NC} ${V}"
done

draw_line "└" "┴" "┘"
echo ""
```

</details>

### 2. Make it Executable

```bash
chmod +x ~/pm2-ports.sh
```

### 3. Add the `pm2 ports` Command

To integrate this script directly into PM2, add a wrapper function to your shell profile:

1. Open `.bashrc`:

   ```bash
   nano ~/.bashrc
   ```

2. Append this function at the very end:

   ```bash
   # PM2 ports command extension
   pm2() {
     if [[ "$1" == "ports" ]]; then
       ~/pm2-ports.sh
     else
       command pm2 "$@"
     fi
   }
   ```

3. Apply changes:

   ```bash
   source ~/.bashrc
   ```

---

## 📖 Usage

Simply use the new sub-command as if it were a native part of PM2:

```bash
pm2 ports
```
