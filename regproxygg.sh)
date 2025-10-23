#!/bin/bash
# =====================================================================
# GCP SOCKS5 FARM v4.6
# - Tối đa 3 projects (ưu tiên sẵn có; thiếu tự tạo & link billing)
# - Mỗi project: 8 VM (4 Tokyo + 4 Osaka) = 24 proxy
# - Debian 12 + Dante bind theo IFACE thật (không dùng 0.0.0.0 cho external)
# - VPC tự tạo: socks-vpc (auto subnet) + firewall theo tag "socks5"
# - Tất cả log theo project: /tmp/<projectId>.log
# =====================================================================

set -u
PREFIX="proxygen"
NEED=3
TOKYO_ZONE="asia-northeast1-b"
OSAKA_ZONE="asia-northeast2-a"
PROXY_USER="dailem"
PROXY_PASS="dailem2002"
PROXY_PORT="1080"
FIREWALL_NAME="allow-socks5-1080"
NETWORK_NAME="socks-vpc"
: "${BILLING_ACCOUNT_ID:=}"   # có thể để trống; script sẽ tự dò

say(){ echo -e "[ $(date '+%F %T') ] $*"; }
warn(){ echo -e "[WARN] $*" >&2; }
err(){ echo -e "[ERR ] $*" >&2; }
need(){ command -v "$1" >/dev/null 2>&1 || { err "Thiếu lệnh: $1"; exit 1; }; }

need gcloud; need awk; need sed; need grep; need tr

active_acct(){ gcloud auth list --filter=status:ACTIVE --format="value(account)" 2>/dev/null | head -n1; }
default_project(){ gcloud config get-value core/project 2>/dev/null | tr -d '\r'; }
project_has_billing(){ gcloud beta billing projects describe "$1" --format="value(billingEnabled)" 2>/dev/null | grep -q "^True$"; }
project_billing_account(){ gcloud beta billing projects describe "$1" --format="value(billingAccountName)" 2>/dev/null; }
pick_open_billing(){ gcloud beta billing accounts list --filter="open=true" --format="value(name)" 2>/dev/null | head -n1; }
list_billing_accounts(){ gcloud beta billing accounts list --format='table(name,displayName,open)' 2>/dev/null || true; }
normalize_billing_id(){ local x="${1:-}"; x="${x##billingAccounts/}"; echo "$x"; }

enable_apis(){
  local pid="$1"
  gcloud services enable serviceusage.googleapis.com --project="$pid" --quiet >/dev/null 2>&1 || true
  gcloud services enable compute.googleapis.com      --project="$pid" --quiet >/dev/null 2>&1 || true
  gcloud services enable iam.googleapis.com          --project="$pid" --quiet >/dev/null 2>&1 || true
}
grant_roles(){
  local pid="$1" who="user:$(active_acct)"
  [[ -z "$who" ]] && return 0
  gcloud projects add-iam-policy-binding "$pid" --member="$who" --role="roles/compute.admin" --quiet >/dev/null 2>&1 || true
  gcloud projects add-iam-policy-binding "$pid" --member="$who" --role="roles/iam.serviceAccountUser" --quiet >/dev/null 2>&1 || true
  gcloud projects add-iam-policy-binding "$pid" --member="$who" --role="roles/serviceusage.serviceUsageAdmin" --quiet >/dev/null 2>&1 || true
}
prepare_existing_project(){
  local pid="$1"
  project_has_billing "$pid" || return 1
  enable_apis "$pid" || true
  grant_roles "$pid" || true
  return 0
}
check_project_ready(){
  local pid="$1"
  gcloud projects describe "$pid" --format="value(projectId)" --quiet >/dev/null 2>&1 || return 1
  project_has_billing "$pid" || return 1
  gcloud services list --enabled --project="$pid" --filter="NAME=compute.googleapis.com" --format="value(NAME)" --quiet \
    | grep -q "compute.googleapis.com" || return 1
  gcloud compute instances list --project="$pid" --limit=1 --quiet >/dev/null 2>&1 || return 1
  return 0
}

ensure_network(){
  local pid="$1"
  # Tạo VPC auto subnet nếu chưa có
  gcloud compute networks describe "$NETWORK_NAME" --project="$pid" --quiet >/dev/null 2>&1 && return 0
  say "[$pid] tạo network $NETWORK_NAME (auto subnet)…"
  gcloud compute networks create "$NETWORK_NAME" \
    --project="$pid" --subnet-mode=auto --bgp-routing-mode=regional --quiet >/dev/null 2>&1 || true
}

ensure_firewall(){
  local pid="$1"
  gcloud compute firewall-rules describe "$FIREWALL_NAME" --project="$pid" --quiet >/dev/null 2>&1 && return 0
  say "[$pid] tạo firewall $FIREWALL_NAME…"
  gcloud compute firewall-rules create "$FIREWALL_NAME" \
    --project="$pid" --allow="tcp:${PROXY_PORT}" \
    --direction=INGRESS --priority=1000 \
    --network="$NETWORK_NAME" \
    --target-tags="socks5" --quiet >/dev/null 2>&1 || warn "[$pid] tạo firewall lỗi (bỏ qua)."
}

discover_billing_from_projects(){
  gcloud projects list --format='value(projectId)' 2>/dev/null \
  | while read -r pid; do
      [[ -z "$pid" ]] && continue
      enabled="$(gcloud beta billing projects describe "$pid" --format='value(billingEnabled)' 2>/dev/null || true)"
      [[ "$enabled" != "True" ]] && continue
      acct="$(gcloud beta billing projects describe "$pid" --format='value(billingAccountName)' 2>/dev/null || true)"
      [[ -n "$acct" ]] && echo "$acct"
    done | awk '!seen[$0]++'
}

choose_billing_candidates(){
  local DEF="$1"
  [[ -n "${BILLING_ACCOUNT_ID:-}" ]] && echo "$(normalize_billing_id "$BILLING_ACCOUNT_ID")"
  if [[ -n "$DEF" ]]; then
    project_has_billing "$DEF" && project_billing_account "$DEF"
  fi
  discover_billing_from_projects
  pick_open_billing
}

# ---- Startup Script (Dante) ----
SS_FILE="/tmp/startup_socks5.sh"
cat >"$SS_FILE" <<'EOS'
#!/bin/bash
set -euo pipefail
export DEBIAN_FRONTEND=noninteractive

apt-get update -y || true
apt-get install -y dante-server curl iptables || true

USER_NAME="mr.quang"
USER_PASS="2703"
PORT="1080"

id -u "$USER_NAME" &>/dev/null || useradd -m -s /usr/sbin/nologin "$USER_NAME" || true
echo "${USER_NAME}:${USER_PASS}" | chpasswd || true

# Lấy interface mặc định (GCE Debian 12 thường là ens4)
IFACE="$(ip -o -4 route show to default 2>/dev/null | awk '{print $5}' | head -n1)"
[ -z "$IFACE" ] && IFACE="ens4"

# Bật START=yes theo Debian
cat >/etc/default/danted <<EOCFG
START=yes
CONFIGFILE=/etc/danted.conf
EOCFG

# Cấu hình Dante (bind đúng IFACE)
cat >/etc/danted.conf <<EOC
logoutput: syslog
internal: ${IFACE} port = ${PORT}
external: ${IFACE}
clientmethod: none
socksmethod: username
user.notprivileged: nobody

client pass {
  from: 0.0.0.0/0 to: 0.0.0.0/0
}
socks pass {
  from: 0.0.0.0/0 to: 0.0.0.0/0
  command: connect
}
EOC

systemctl enable danted || true
systemctl restart danted || systemctl start danted || true

iptables -I INPUT -p tcp --dport ${PORT} -j ACCEPT 2>/dev/null || true

EXT_IP="$(curl -s --max-time 5 ifconfig.me || hostname -I | awk '{print $1}')"
echo "${EXT_IP}:${PORT}:${USER_NAME}:${USER_PASS}" >/root/_proxy.txt 2>/dev/null || true

if [ -n "${BOT_TOKEN:-}" ] && [ -n "${USER_ID:-}" ]; then
  curl -fsS "https://api.telegram.org/bot${BOT_TOKEN}/sendMessage" \
    --data-urlencode "chat_id=${USER_ID}" \
    --data-urlencode "text=${EXT_IP}:${PORT}:${USER_NAME}:${USER_PASS}" >/dev/null || true
fi
EOS
chmod +x "$SS_FILE" || true

create_projects_to_reach_three(){
  local candidates="$1"; local need_more="$2"; local created=()
  [[ "$need_more" -le 0 ]] && { echo ""; return 0; }
  for i in $(seq 1 "$need_more"); do
    local pid="${PREFIX}-$(date +%s)$RANDOM"
    say "[create $i/$need_more] tạo project: $pid"
    gcloud projects create "$pid" --name="$pid" --quiet >/dev/null || { err "Tạo project lỗi: $pid"; exit 1; }
    local ok=0 acct norm
    while read -r acct; do
      [[ -z "$acct" ]] && continue
      norm="$(normalize_billing_id "$acct")"
      say "[create] thử link billing: $norm"
      if gcloud beta billing projects link "$pid" --billing-account="$norm" --quiet >/dev/null 2>&1; then
        ok=1; break
      fi
    done <<< "$candidates"
    [[ "$ok" -eq 1 ]] || { err "Không link billing được cho $pid."; exit 1; }
    say "[create] bật API"; enable_apis "$pid"
    say "[create] cấp quyền cho $(active_acct)"; grant_roles "$pid"
    created+=("$pid")
  done
  echo "${created[*]}"
}

create_vm(){
  local pid="$1" name="$2" zone="$3"
  gcloud compute instances describe "$name" --zone="$zone" --project="$pid" --quiet >/dev/null 2>&1 && {
    say "[$pid] $name đã tồn tại → skip"; return 0; }
  # Không ẩn stderr/stdout: để log dự án thấy lỗi nếu có (VD: không có default network)
  gcloud compute instances create "$name" \
    --project="$pid" --zone="$zone" --machine-type="e2-micro" \
    --image-family="debian-12" --image-project="debian-cloud" \
    --boot-disk-size="10GB" --boot-disk-type="pd-balanced" \
    --tags="socks5" \
    --network="$NETWORK_NAME" \
    --metadata=enable-oslogin=true \
    --metadata-from-file=startup-script="$SS_FILE" \
    --quiet
}

batch_project(){
  local pid="$1" lg="/tmp/${pid}.log"
  {
    say "[$pid] chuẩn hoá quyền & API…"; prepare_existing_project "$pid" || true
    say "[$pid] kiểm tra điều kiện…"
    if ! check_project_ready "$pid"; then err "[$pid] chưa sẵn sàng (billing/API/quyền). Bỏ qua."; exit 1; fi
    ensure_network "$pid"
    ensure_firewall "$pid"
    say "[$pid] tạo 4 Tokyo + 4 Osaka…"
    create_vm "$pid" "proxy-tokyo-1" "$TOKYO_ZONE" &
    create_vm "$pid" "proxy-tokyo-2" "$TOKYO_ZONE" &
    create_vm "$pid" "proxy-tokyo-3" "$TOKYO_ZONE" &
    create_vm "$pid" "proxy-tokyo-4" "$TOKYO_ZONE" &
    create_vm "$pid" "proxy-osaka-1" "$OSAKA_ZONE" &
    create_vm "$pid" "proxy-osaka-2" "$OSAKA_ZONE" &
    create_vm "$pid" "proxy-osaka-3" "$OSAKA_ZONE" &
    create_vm "$pid" "proxy-osaka-4" "$OSAKA_ZONE" &
    wait
    say "[$pid] chờ IP (45s)…"; sleep 45
    say "[$pid] PROXY:"
    gcloud compute instances list \
      --project="$pid" \
      --filter="name~'^proxy-(tokyo|osaka)'" \
      --format="get(networkInterfaces[0].accessConfigs[0].natIP)" \
      --quiet \
      | sed '/^$/d' | awk -v p="${PROXY_PORT}" -v u="${PROXY_USER}" -v s="${PROXY_PASS}" '{print $1":"p":"u":"s}'
  } >"$lg" 2>&1
}

# ================= MAIN =================
say "Active account: $(active_acct || echo none)"
DEF="$(default_project || true)"

# (A) chọn danh sách candidate project: default -> prefix -> phần còn lại
CAND=()
[[ -n "$DEF" ]] && CAND+=("$DEF")
ALL="$(gcloud projects list --format='value(projectId)' 2>/dev/null)"
PRIORITY=(); OTHERS=()
while read -r p; do
  [[ -z "$p" || "$p" == "$DEF" ]] && continue
  [[ "$p" == ${PREFIX}-* ]] && PRIORITY+=("$p") || OTHERS+=("$p")
done <<< "$ALL"
CAND+=("${PRIORITY[@]}" "${OTHERS[@]}")

# (B) lấy tối đa 3 project sẵn sàng
PICK=()
for pid in "${CAND[@]}"; do
  prepare_existing_project "$pid" || true
  check_project_ready "$pid" || continue
  PICK+=("$pid")
  [[ ${#PICK[@]} -ge $NEED ]] && break
done

# (C) tạo thêm nếu thiếu
if [[ ${#PICK[@]} -lt $NEED ]]; then
  say "Đang tự dò Billing Accounts…"
  CAND_BILL="$(choose_billing_candidates "$DEF" | awk 'NF' | awk '!seen[$0]++')"
  if [[ -z "${CAND_BILL:-}" ]]; then
    err "Không tìm thấy Billing Account (có thể thiếu quyền Billing Viewer)."
    say "Các billing nhìn thấy (nếu có quyền):"; list_billing_accounts || true
    exit 1
  fi
  NEED_MORE=$(( NEED - ${#PICK[@]} ))
  say "Sẽ tạo thêm $NEED_MORE project, thử lần lượt các billing tìm được…"
  read -r -a CREATED <<<"$(create_projects_to_reach_three "$CAND_BILL" "$NEED_MORE")"
  PICK+=("${CREATED[@]}")
fi

say "Dùng 3 project: ${PICK[*]}"

# (D) tạo proxy song song
for pid in "${PICK[@]}"; do batch_project "$pid" & done
wait

say "=== TỔNG HỢP PROXY (ip:port:user:pass) ==="
for pid in "${PICK[@]}"; do
  echo "# $pid"
  if [[ -s "/tmp/${pid}.log" ]]; then
    awk '/PROXY:/{flag=1;next} /chờ IP|Chờ IP|^\[/{next} flag && NF' "/tmp/${pid}.log" \
      | sed '/^\[/d' | sed '/^$/d'
  else
    warn "[$pid] Không có log."
  fi
done

say "Xem log: /tmp/<projectId>.log"
