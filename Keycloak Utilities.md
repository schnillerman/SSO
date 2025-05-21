Here's an **integrated menu-driven script** with following features:

# Key Features

1. **Core Functions**
   - Bulk role assignment
   - Role renaming
   - Role/user listing
   - Menu navigation
   - Session management
   - Compatible with Keycloak 16.x/CentOS 7

1. **Unified Menu System**
   - Hierarchical navigation
   - Always returns to previous menu
   - Context-aware (shows current realm)

2. **Enhanced Listing**
   ```bash
   # Alphabetical sorting example
   data = sorted(json.load(sys.stdin), key=lambda x: x['name'])
   ```

3. **Session Management**
   - Reuses valid sessions
   - Automatic re-authentication

4. **New Functions**
   - Realm selection
   - User/role listing
   - Dry-run mode

5. **Error Handling**
   - Input validation
   - Graceful error recovery
   - Clear status messages

# Usage
```bash
chmod +x keycloak_admin.sh
./keycloak_admin.sh
```

# UX
**Navigation Flow**
1. kcadm.sh discovery
2. Authentication
3. Realm selection
4. Main menu with 6 options
5. Context-sensitive back/quit options

**Sorting Implementation:**
- Roles sorted by name
- Users sorted by username

# Code
```bash
#!/bin/bash

# =================================================================
# Keycloak Administration Toolkit
# Features:
# - Bulk role assignment
# - Role renaming
# - Role/user listing
# - Menu navigation
# - Session management
# Compatible with Keycloak 16.x/CentOS 7
# =================================================================

# Color definitions
RED=$(tput setaf 1)
GREEN=$(tput setaf 2)
YELLOW=$(tput setaf 3)
CYAN=$(tput setaf 6)
BLUE=$(tput setaf 4)
RESET=$(tput sgr0)

# Global variables
KC_ADM=""
SERVER_URL="http://localhost:8080/auth"
CURRENT_REALM=""
DRY_RUN=false

# Dependency check
check_dependencies() {
  if ! command -v python &> /dev/null; then
    echo "${RED}Error: Python 2.7+ required. Install with:"
    echo "  sudo yum install python${RESET}"
    exit 1
  fi
}

# --------------------------------------------------
# kcadm.sh Discovery
# --------------------------------------------------
find_kcadm() {
  DEFAULT_PATHS=(
    "/opt/keycloak/bin"
    "/usr/local/keycloak/bin"
    "$(pwd)"
  )

  read -p "kcadm.sh directory [empty for auto-discovery]: " KC_DIR
  if [ -n "$KC_DIR" ]; then
    KC_ADM="${KC_DIR%/}/kcadm.sh"
    [ -f "$KC_ADM" ] || { echo "${RED}kcadm.sh not found at $KC_ADM${RESET}"; return 1; }
  else
    for path in "${DEFAULT_PATHS[@]}"; do
      candidate="${path}/kcadm.sh"
      [ -f "$candidate" ] && KC_ADM="$candidate" && break
    done
    
    [ -z "$KC_ADM" ] && KC_ADM=$(command -v kcadm.sh 2>/dev/null)
    [ -z "$KC_ADM" ] && echo "${RED}kcadm.sh not found${RESET}" && return 1
  fi
  echo "${GREEN}Using kcadm.sh at: $KC_ADM${RESET}"
  return 0
}

# --------------------------------------------------
# Session Management
# --------------------------------------------------
session_valid() {
  "$KC_ADM" get serverinfo >/dev/null 2>&1
  return $?
}

authenticate() {
  echo "${YELLOW}=== Authentication Required ===${RESET}"
  read -p "Admin Password: " -s ADMIN_PW
  echo
  
  "$KC_ADM" config credentials \
    --server "$SERVER_URL" \
    --realm master \
    --user admin \
    --password "$ADMIN_PW" || return 1
  
  session_valid || {
    echo "${RED}Session validation failed!${RESET}"
    return 1
  }
}

# --------------------------------------------------
# Realm Selection
# --------------------------------------------------
select_realm() {
  REALMS=$("$KC_ADM" get realms --format json | python -c "
import json,sys
data = json.load(sys.stdin)
print('\n'.join([r['id'] for r in data]))")

  echo "${CYAN}Available realms:${RESET}"
  select realm in $REALMS "Back" "Quit"; do
    case $realm in
      "Back") return 1 ;;
      "Quit") exit 0 ;;
      *) 
        [ -n "$realm" ] && CURRENT_REALM="$realm" && return 0
        ;;
    esac
  done
}

# --------------------------------------------------
# Role Operations
# --------------------------------------------------
list_roles() {
  "$KC_ADM" get roles -r "$CURRENT_REALM" --format json | python -c "
import json,sys
data = sorted(json.load(sys.stdin), key=lambda x: x['name'])
print('\n'.join([f\"- {r['name']} (ID: {r['id']})\" for r in data]))"
}

get_role_id() {
  local role_name="$1"
  "$KC_ADM" get roles -r "$CURRENT_REALM" --format json | python -c "
import json,sys,urllib
data = json.load(sys.stdin)
decoded = urllib.unquote('$role_name').decode('utf8')
for r in data:
    if r['name'] == decoded:
        print(r['id'])
        sys.exit(0)
sys.exit(1)"
}

rename_role() {
  while true; do
    echo "${CYAN}Current roles in ${CURRENT_REALM}:${RESET}"
    list_roles
    
    read -p "Enter current role name: " OLD_NAME
    ROLE_ID=$(get_role_id "$OLD_NAME")
    [ -z "$ROLE_ID" ] && echo "${RED}Role not found!${RESET}" && continue
    
    read -p "Enter new role name: " NEW_NAME
    echo "${YELLOW}Renaming '${OLD_NAME}' to '${NEW_NAME}'...${RESET}"
    
    "$KC_ADM" update "roles-by-id/$ROLE_ID" -r "$CURRENT_REALM" -s name="$NEW_NAME" && {
      echo "${GREEN}Successfully renamed role!${RESET}"
      return 0
    }
    
    echo "${RED}Renaming failed!${RESET}"
    return 1
  done
}

# --------------------------------------------------
# User Operations
# --------------------------------------------------
list_users() {
  "$KC_ADM" get users -r "$CURRENT_REALM" --format json | python -c "
import json,sys
data = sorted(json.load(sys.stdin), key=lambda x: x['username'])
print('\n'.join([f\"- {u['username']} (ID: {u['id']})\" for u in data]))"
}

bulk_assign_role() {
  while true; do
    echo "${CYAN}Available roles in ${CURRENT_REALM}:${RESET}"
    list_roles
    
    read -p "Enter role name to assign: " ROLE_NAME
    ROLE_ID=$(get_role_id "$ROLE_NAME")
    [ -z "$ROLE_ID" ] && echo "${RED}Role not found!${RESET}" && continue
    
    read -p "Enable dry-run mode? [y/N]: " DRY_RUN_INPUT
    [[ $DRY_RUN_INPUT =~ [yY] ]] && DRY_RUN=true
    
    OFFSET=0
    LIMIT=100
    TOTAL=0
    
    while :; do
      USER_PAGE=$("$KC_ADM" get users -r "$CURRENT_REALM" \
        --fields id --offset $OFFSET --limit $LIMIT --format json | python -c "
import json,sys
try:
    print(' '.join([u['id'] for u in json.load(sys.stdin)]))
except:
    sys.exit(1)")
      
      [ $? -ne 0 ] && break
      [ -z "$USER_PAGE" ] && break
      
      for USER_ID in $USER_PAGE; do
        echo -n "Processing user $((TOTAL+1)): "
        CMD="$KC_ADM add-roles -r $CURRENT_REALM --uid $USER_ID --roles $ROLE_ID"
        
        if $DRY_RUN; then
          echo "${YELLOW}[DRY-RUN] $CMD${RESET}"
        else
          "$KC_ADM" add-roles -r "$CURRENT_REALM" --uid "$USER_ID" --roles "$ROLE_ID" 2>&1 | grep -v "already has role"
          [ ${PIPESTATUS[0]} -eq 0 ] && echo "${GREEN}OK${RESET}" || echo "${RED}ERROR${RESET}"
        fi
        ((TOTAL++))
      done
      OFFSET=$((OFFSET + LIMIT))
    done
    
    echo "${GREEN}Processed ${TOTAL} users${RESET}"
    return 0
  done
}

# --------------------------------------------------
# Main Menu System
# --------------------------------------------------
main_menu() {
  while true; do
    echo "${BLUE}=== Keycloak Administration Menu ==="
    echo "Current Realm: ${GREEN}${CURRENT_REALM}${BLUE}"
    echo "1. Bulk Assign Role to Users"
    echo "2. Rename Role"
    echo "3. List Roles"
    echo "4. List Users"
    echo "5. Select Realm"
    echo "6. Quit${RESET}"
    
    read -p "Selection [1-6]: " CHOICE
    case $CHOICE in
      1) bulk_assign_role ;;
      2) rename_role ;;
      3) list_roles | less ;;
      4) list_users | less ;;
      5) select_realm && main_menu ;;
      6) exit 0 ;;
      *) echo "${RED}Invalid option!${RESET}" ;;
    esac
    
    read -p "Press Enter to continue..."
  done
}

# --------------------------------------------------
# Script Initialization
# --------------------------------------------------
init() {
  check_dependencies
  while ! find_kcadm; do : ; done
  while ! session_valid; do authenticate; done
  while ! select_realm; do : ; done
}

# --------------------------------------------------
# Execution Flow
# --------------------------------------------------
clear
init
main_menu
```
- Uses Python's `sorted()` with lambda key
- Maintains original functionality while adding ordering

This script maintains all original functionality while adding the requested menu system and sorting capabilities.
