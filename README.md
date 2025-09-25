# Checking-Palo-Alto-firewall-for-disk-usage

### Architecture:
* PANORAMA
* SERVER LINUX

### Part 1. Create Account on PANORAMA

<img width="1607" height="267" alt="image" src="https://github.com/user-attachments/assets/7982842e-0520-49cd-9110-e8c9c6955e40" />

### Part 2. Create API KEY for this Account The key is valid as long as the account exists and you do not change its password.

Run CMD and paste this command:

```bash
curl -sk "https://192.168.10.44/api/?type=keygen&user=super_admin&password=PASSWORD_FOR_THIS_ACCOUNT"
```

### Part 3. Create directory on Linux server

```bash
mkdir -p /home/super_admin/check_disk
```

### Part 4. Create script

```bash
touch /home/super_admin/check_disk/panorama_check_disk.sh
chmod 600 /home/super_admin/check_disk/panorama_check_disk.sh
```

### Part 5. Paste script
```bash
#!/usr/bin/env bash
# Zbiera zajętość partycji z WSZYSTKICH podłączonych urządzeń (PA) widocznych w Panorama
# i zapisuje CSV: serial,hostname,ip_address,filesystem,size,used,avail,use_percent,mountpoint

set -euo pipefail

##########################  KONFIGURACJA DOMYŚLNA  ##########################
PAN_HOST="${PAN_HOST:-192.168.10.44}"
API_KEY="${API_KEY:-API-KEY}"
INSECURE=${INSECURE:-true}              # true => curl -k (self-signed)
TIMEOUT=${TIMEOUT:-30}
OUTPUT_CSV="${OUTPUT_CSV:-./panorama_disk_usage.csv}"
FILTER_REGEX="${FILTER_REGEX:-}"         # opcjonalny regex na hostname/serial/IP
#############################################################################

usage() {
  cat <<EOF
Użycie: $(basename "$0") [--filter REGEX] [--output PLIK]
Zmienne: PAN_HOST, API_KEY, INSECURE(true/false), TIMEOUT, OUTPUT_CSV, FILTER_REGEX.
Przykład: FILTER_REGEX='TEST|00795100' ./$(basename "$0")
EOF
}

# Parametry CLI
while [[ $# -gt 0 ]]; do
  case "$1" in
    -h|--help) usage; exit 0 ;;
    --filter) FILTER_REGEX="${2:-}"; shift 2 ;;
    --output) OUTPUT_CSV="${2:-}"; shift 2 ;;
    *) echo "Nieznana opcja: $1"; usage; exit 1 ;;
  esac
done

if [[ -z "${API_KEY// }" ]]; then
  echo "Błąd: brak API_KEY." >&2; exit 1
fi

TMPDIR="$(mktemp -d /tmp/panorama_disk.XXXXXX)"
trap 'rm -rf "$TMPDIR"' EXIT

CURL_BASE=( --silent --show-error --max-time "$TIMEOUT" )
[[ "$INSECURE" == true ]] && CURL_BASE+=( -k )
API_URL="https://${PAN_HOST}/api/"

ts() { date +'%F %T'; }
log() { echo "[$(ts)] $*"; }

##################################
# 1) Pobierz listę urządzeń
##################################
log "Pobieram listę PODŁĄCZONYCH urządzeń z Panorama: ${PAN_HOST}"

DEVRESP="$TMPDIR/devices.xml"
HTTP_CODE=$(curl "${CURL_BASE[@]}" --get \
  --data-urlencode "cmd=<show><devices><connected></connected></devices></show>" \
  --data "type=op" \
  --data-urlencode "key=${API_KEY}" \
  --write-out "%{http_code}" \
  --output "$DEVRESP" \
  "$API_URL") || { echo "Błąd: curl przy pobieraniu listy urządzeń."; exit 2; }

if [[ "$HTTP_CODE" != "200" ]]; then
  echo "Błąd HTTP przy pobieraniu listy urządzeń: $HTTP_CODE"
  sed -n '1,200p' "$DEVRESP" || true
  exit 3
fi

# --- PARSOWANIE XML: rozbij na linie, potem wyciągaj wszystkie wystąpienia ---
read_arrays() {
  local x_in="$DEVRESP"
  local x="$TMPDIR/devices.pretty.xml"
  # Wstaw nową linię między '><' – każdy tag w osobnym wierszu
  sed 's/></>\n</g' "$x_in" > "$x"

  # Zbierz wartości; każda para trafia w osobny wiersz
  mapfile -t SERIALS   < <(sed -n 's/^<serial>\([^<]*\)<\/serial>$/\1/p'     "$x")
  mapfile -t HOSTNAMES < <(sed -n 's/^<hostname>\([^<]*\)<\/hostname>$/\1/p' "$x")
  mapfile -t IPS       < <(sed -n 's/^<ip-address>\([^<]*\)<\/ip-address>$/\1/p' "$x")
  if [[ ${#IPS[@]} -eq 0 ]]; then
    mapfile -t IPS     < <(sed -n 's/^<ip>\([^<]*\)<\/ip>$/\1/p' "$x")
  fi
}
read_arrays

if [[ ${#SERIALS[@]} -eq 0 ]]; then
  echo "Nie znaleziono żadnych podłączonych urządzeń (parser nie znalazł tagów)."
  sed -n '1,200p' "$DEVRESP" || true
  exit 4
fi

# Uzupełnij długości tablic
normalize_array_lengths() {
  local want=$1; local -n arr=$2
  local cur=${#arr[@]}
  if (( cur < want )); then for _ in $(seq 1 $((want-cur))); do arr+=("-"); done; fi
}
normalize_array_lengths ${#SERIALS[@]} HOSTNAMES
normalize_array_lengths ${#SERIALS[@]} IPS

# Opcjonalny filtr urządzeń
if [[ -n "$FILTER_REGEX" ]]; then
  NEW_S=() NEW_H=() NEW_I=()
  for i in "${!SERIALS[@]}"; do
    s="${SERIALS[$i]}"; h="${HOSTNAMES[$i]}"; ip="${IPS[$i]}"
    if [[ "$s" =~ $FILTER_REGEX || "$h" =~ $FILTER_REGEX || "$ip" =~ $FILTER_REGEX ]]; then
      NEW_S+=("$s"); NEW_H+=("$h"); NEW_I+=("$ip")
    fi
  done
  SERIALS=("${NEW_S[@]}"); HOSTNAMES=("${NEW_H[@]}"); IPS=("${NEW_I[@]}")
  [[ ${#SERIALS[@]} -eq 0 ]] && { echo "Po filtrze '${FILTER_REGEX}' brak urządzeń."; exit 0; }
fi

log "Znaleziono ${#SERIALS[@]} urządzeń do przetworzenia."
echo "serial,hostname,ip_address,filesystem,size,used,avail,use_percent,mountpoint" > "$OUTPUT_CSV"

##################################
# 2) show system disk-space
##################################
fetch_and_parse_disk() {
  local serial="$1" hostname="$2" ipaddr="$3"
  local out_xml="$TMPDIR/${serial}_disk.xml"
  local cmd_xml='<show><system><disk-space></disk-space></system></show>'

  local http
  http=$(curl "${CURL_BASE[@]}" --get \
    --data-urlencode "cmd=${cmd_xml}" \
    --data "type=op" \
    --data-urlencode "key=${API_KEY}" \
    --data-urlencode "target=${serial}" \
    --write-out "%{http_code}" \
    --output "$out_xml" \
    "$API_URL") || { echo "[$serial] curl nie powiódł się"; return 1; }

  if [[ "$http" != "200" ]]; then
    echo "[$serial] Błąd HTTP: $http"
    sed -n '1,120p' "$out_xml" || true
    return 1
  fi

  # Usuń tagi XML i zostaw treść (z układem kolumn df)
  local result_txt
  if grep -q "<result" "$out_xml"; then
    result_txt=$(sed 's/<[^>]*>//g' "$out_xml" | sed 's/^[[:space:]]\+//g')
  else
    result_txt=$(cat "$out_xml")
  fi

  local txtfile="$TMPDIR/${serial}_disk.txt"
  printf "%s\n" "$result_txt" > "$txtfile"

  # Znajdź nagłówek tabeli df
  local header_line
  header_line=$(grep -nE '^Filesystem[[:space:]]+Size[[:space:]]+Used[[:space:]]+Avail[[:space:]]+Use%[[:space:]]+Mounted on' "$txtfile" | head -n1 | cut -d: -f1 || true)
  [[ -z "$header_line" ]] && header_line=$(grep -nE 'Size[[:space:]]+Used[[:space:]]+Avail[[:space:]]+Use%' "$txtfile" | head -n1 | cut -d: -f1 || true)
  if [[ -z "$header_line" ]]; then
    echo "[$serial] Nie znaleziono nagłówka disk-space — zrzut (pierwsze 80 linii):"
    sed -n '1,80p' "$txtfile" || true
    return 1
  fi

  local start=$((header_line + 1))
  awk -v s="$start" 'NR>=s { if ($0 ~ /^[[:space:]]*$/) exit; print }' "$txtfile" \
  | while IFS= read -r line; do
      filesystem=$(echo "$line" | awk '{print $1}')
      size=$(echo "$line" | awk '{print $2}')
      used=$(echo "$line" | awk '{print $3}')
      avail=$(echo "$line" | awk '{print $4}')
      usep=$(echo "$line" | awk '{print $5}')
      mountpoint=$(echo "$line" | awk '{for(i=6;i<=NF;i++){ if(i>6) printf " "; printf $i }}')
      [[ -z "$mountpoint" ]] && mountpoint="-"
      _csv() { echo "$1" | sed 's/,/;/g'; }
      printf '%s,%s,%s,%s,%s,%s,%s,%s,%s\n' \
        "$(_csv "$serial")" "$(_csv "$hostname")" "$(_csv "$ipaddr")" \
        "$(_csv "$filesystem")" "$(_csv "$size")" "$(_csv "$used")" "$(_csv "$avail")" "$(_csv "$usep")" "$(_csv "$mountpoint")" \
        >> "$OUTPUT_CSV"
    done
  return 0
}

##################################
# 3) Iteracja
##################################
for i in "${!SERIALS[@]}"; do
  s="${SERIALS[$i]}"; h="${HOSTNAMES[$i]}"; ip="${IPS[$i]}"
  [[ -z "$s" ]] && continue
  log "Przetwarzam: serial=$s, host=$h, ip=$ip"
  if ! fetch_and_parse_disk "$s" "$h" "$ip"; then
    echo "[$s] Wystąpił błąd — przechodzę dalej."
  fi
done

log "Gotowe. CSV zapisane w: $OUTPUT_CSV"
```

### Part 6. Modyfying file (change API-KEY, IP-address). 

### Part 7. Add modyfing permission to script

```bash
chmod +x /home/super_admin/check_disk/panorama_check_disk.sh
```

### Part 8. Testing manual

```bash
./home/super_admin/check_disk/panorama_check_disk.sh
```

<img width="1307" height="203" alt="image" src="https://github.com/user-attachments/assets/bd849925-4696-47d2-bd14-f68d14fe1d52" />

### Part 9. Check file csv. 
```bash
column -s, -t panorama_disk_usage.csv | less -S
```
<img width="1453" height="692" alt="image" src="https://github.com/user-attachments/assets/20d1aa87-9738-48fa-8471-e90ce82030f2" />


### Part 10. Add to crontab
```bash
(crontab -l 2>/dev/null; echo "15 1 * * * /home/super_admin/check_disk/panorama_check_disk.sh >/dev/null 2>&1") | crontab -
```
