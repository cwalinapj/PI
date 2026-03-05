et’s turn what you’ve got into a one-stop “deploy to Pi(s)” project that is:

repeatable (same command every time)

safe (won’t nuke the wrong disk)

idempotent (re-running doesn’t break things)

works across many Pis (hosts file / inventory)

Below is a complete blueprint: repo structure, Cobra commands, and a single deploy script that builds → copies → installs → runs provisioning steps (like ZFS).

Target end state (what you’ll run)

From your Mac:

# one Pi
./scripts/deploy-pi.sh --host zoly55@192.168.1.57 --disk-id usb-ASMT_ASM246X_AAAABBBB0012-0:0 --zfs

# many Pis
./scripts/deploy-pi.sh --inventory inventory/pis.txt --disk-id usb-ASMT_ASM246X_AAAABBBB0012-0:0 --zfs

What it does:

builds your Cobra CLI for linux/arm64

scp’s it to the Pi(s)

installs to /usr/local/bin/picloud (and optional alias mycli)

runs picloud zfs init ... (or any other provisioning subcommands)

Repo layout that scales

Inside your ~/pi-cloud (Go module) I’d structure like this:

pi-cloud/
  cmd/
    picloud/
      main.go              # the only entrypoint you build
  internal/
    execx/                 # safe command runner (dry-run, logging)
    detect/                # detect disks by by-id, verify whole-disk, refuse mmcblk0
    zfsops/                # pure logic for zfs init (builds command list)
  pkg/ (optional public)
  scripts/
    deploy-pi.sh           # one-stop deploy from Mac
    ssh-lib.sh             # helpers: ssh/scp, retries, colored logs
  inventory/
    pis.txt                # list of hosts (one per line)
  Makefile
  go.mod
  README.md

Key idea: Cobra command layer is thin; the real work lives in internal/* so it’s testable and reusable.

Fix the “zfs command not found” issue permanently

You want one binary name and one root command tree.

cmd/picloud/main.go

This should call your real root command:

package main

import "pi-cloud/cmd/picloud/cmd"

func main() {
	cmd.Execute()
}
cmd/picloud/cmd/root.go

Your root adds subcommands:

rootCmd.AddCommand(zfsCmd)
// rootCmd.AddCommand(otherCmd)

This eliminates the “I built the wrong ./cmd/... path” issue. You always build ./cmd/picloud.

Cobra: make provisioning commands predictable

Make “provisioning” style commands live under a top-level provision group:

picloud provision zfs init

picloud provision docker install

picloud provision baseline (users/groups/ssh keys/packages)

Example command set:

1) picloud provision disks

Prints safe choices (by-id → sdX → size/model/mounted)

shows ONLY whole disks (…-0:0, not …-part2)

highlights “DANGEROUS: has partitions/mounted”

2) picloud provision zfs init

Does what we already outlined, but with hard safety rules:

requires --disk /dev/disk/by-id/...

refuses partitions (*-part2)

refuses mmcblk0

if multiple ASM246X exist, requires explicit disk

requires --wipe-disk to destroy data

supports --dry-run and --yes

3) picloud provision baseline

Installs packages + sets up system basics.

One-stop deploy script (Mac) that’s safe

Put this at scripts/deploy-pi.sh and run it from anywhere.

What it solves:

builds from the correct module (~/pi-cloud)

installs as picloud

requires disk-id explicitly if you’re doing destructive operations

can run against inventory file

Here’s the “shape” (not overly fancy, just reliable):

#!/usr/bin/env bash
set -euo pipefail

REPO_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")/.." && pwd)"
cd "$REPO_DIR"

PI_HOST=""
INVENTORY=""
DISK_ID=""
DO_ZFS=0
POOL="tank"
MOUNT="/tank"
DRY_RUN=0

usage() {
  echo "Usage:"
  echo "  $0 --host user@ip [--disk-id <by-id>] [--zfs] [--pool tank] [--mount /tank] [--dry-run]"
  echo "  $0 --inventory inventory/pis.txt [--disk-id <by-id>] [--zfs] ..."
  exit 2
}

while [[ $# -gt 0 ]]; do
  case "$1" in
    --host) PI_HOST="$2"; shift 2;;
    --inventory) INVENTORY="$2"; shift 2;;
    --disk-id) DISK_ID="$2"; shift 2;;
    --zfs) DO_ZFS=1; shift;;
    --pool) POOL="$2"; shift 2;;
    --mount) MOUNT="$2"; shift 2;;
    --dry-run) DRY_RUN=1; shift;;
    *) usage;;
  esac
done

if [[ -z "$PI_HOST" && -z "$INVENTORY" ]]; then usage; fi

build() {
  echo "==> Build picloud for Pi (linux/arm64)"
  GOOS=linux GOARCH=arm64 CGO_ENABLED=0 go build -o ./dist/picloud ./cmd/picloud
}

install_one() {
  local host="$1"
  echo "==> Install to $host"
  scp ./dist/picloud "$host:/tmp/picloud"
  ssh "$host" "sudo install -m 0755 /tmp/picloud /usr/local/bin/picloud && picloud --help | head -40"
}

run_zfs_one() {
  local host="$1"
  if [[ -z "$DISK_ID" ]]; then
    echo "ERROR: --disk-id is required when --zfs is set (multiple disks exist on your Pi)."
    echo "On Pi, list candidates: ls -l /dev/disk/by-id/ | grep -i ASM246X"
    exit 1
  fi

  local disk="/dev/disk/by-id/$DISK_ID"

  if [[ "$DRY_RUN" -eq 1 ]]; then
    ssh "$host" "sudo picloud provision zfs init --disk $disk --pool $POOL --mount $MOUNT --dry-run"
  else
    ssh "$host" "sudo picloud provision zfs init --disk $disk --pool $POOL --mount $MOUNT --wipe-disk --create-datasets --run-fio"
  fi
}

main() {
  mkdir -p dist
  build

  if [[ -n "$PI_HOST" ]]; then
    install_one "$PI_HOST"
    [[ "$DO_ZFS" -eq 1 ]] && run_zfs_one "$PI_HOST"
  else
    while IFS= read -r host; do
      [[ -z "$host" || "$host" =~ ^# ]] && continue
      install_one "$host"
      [[ "$DO_ZFS" -eq 1 ]] && run_zfs_one "$host"
    done < "$INVENTORY"
  fi
}

main
Why we require --disk-id

Because your Pi shows 4 identical ASM246X disks. Auto-picking “first match” is how you wipe the wrong drive.

Make the CLI generate commands (and log them)

Inside Go, you want a runner that:

prints every command (structured)

supports --dry-run

returns friendly errors

Example pattern:

internal/execx.Run(cmd, args...)

internal/zfsops.Plan(...) -> []Command

Cobra just parses flags and calls Apply(plan)

This is how you keep it maintainable.

“One stop” means baseline + storage + services

Once ZFS works, you’ll probably add:

provision baseline: apt update, packages (fio, git, curl, nfs-common, etc)

provision docker install: optional Docker + compose plugin

provision service install: systemd unit for something you deploy

provision ssh keys: add keys for passwordless deploy

Next concrete step (so we stop thrashing)

Right now, your blocker is: the Pi-installed binary doesn’t have the zfs subcommand.

Let’s solve that with one rule:

Always build: go build -o dist/picloud ./cmd/picloud

So tell me (or paste) the output on your Mac from:

cd ~/pi-cloud
find ./cmd -maxdepth 3 -type f -name main.go -print

If you don’t have cmd/picloud/main.go yet, we’ll create it and wire it up so the build path is stable forever.

If you paste that find output and the first ~80 lines of your root command file (whatever contains Execute()), I’ll give you the exact edits to:

unify into cmd/picloud entrypoint

ensure zfsCmd is added

make scripts/deploy-pi.sh run end-to-end with --zfs
