# Rethinking Releases: Moving GhostBSD Toward a Pijul-Style Channel Model

Traditional releases work well when you have a few large milestones per year.  
But as GhostBSD evolves faster and the infrastructure around ZFS and pkgbase becomes more modular, fixed releases start to feel outdated.  

Instead of publishing “GhostBSD 25.02,” we could take a page from **Pijul’s model**, treating every system state as a signed sequence of patches that live inside a **channel**.

---

## 1. Channels Instead of Releases

A **channel** replaces the idea of a static release.  
Instead of installing “24.10,” users choose a channel:

- `stable` – tested patches, long-term supported baseline  
- `testing` – candidate patches before promotion to stable  
- `unstable` – continuous integration and experimental work  
- `lts` (optional) – extended support baseline

Each channel advances by applying **patches** that modify the system relative to a known baseline image.  
The ISO simply bootstraps into a chosen channel; it no longer defines the release.

---

## 2. Patches as First-Class Citizens

Each patch is:

- Signed and content-addressed (by hash)  
- Distributed through a small channel index  
- Applied atomically in a ZFS boot environment  
- Reversible with one command

Users no longer upgrade from 24.10 to 24.11.  
They move from one verified patch set to the next.

```bash
gbsd channel show
gbsd update
gbsd rollback
````

Each system identifies itself in a reproducible way:

```
channel=stable
patch=VQ2M7H6
baseline=20251015
```

This allows precise reproducibility and instant rollback through ZFS boot environments.

---

## 3. How Updates Work

1. The system reads the channel index and verifies signatures.
2. It determines which new patches apply beyond the current patch set.
3. Patches are staged in a new boot environment and validated.
4. On success, the new environment becomes active for the next boot.
5. If anything fails, rollback is automatic.

No partial upgrades. No confusion over versions. No stale ISOs.

---

## 4. Publishing Artifacts

Every channel maintains:

* **Channel index:** ordered patch list with signatures
* **Patch bundles:** compressed archives containing diffs, scripts, and metadata
* **Baselines:** full images (such as monthly snapshots) for new installations
* **Package repository:** built from the patch set for that channel
* **Manifests:** machine-readable SBOMs and checksums

Example names:

```
GhostBSD.stable.202511091745.iso
baseline.stable.20251015.img
patches/stable/VQ/2M/VQ2M7H6.tar.zst
channels/stable/channel.index
```

---

## 5. Security and Reproducibility

* Every patch and channel head is **cryptographically signed** using minisign or signify.
* Channel indices include a timestamp and signature to prevent rollback.
* CI builds are fully reproducible with `SOURCE_DATE_EPOCH`.
* Users can **pin** systems to a specific patch ID for deterministic rebuilds.

---

## 6. ghostbsd-build in a Channel World

Only small changes are required in `build.sh`:

```sh
CHANNEL="${CHANNEL:-stable}"

_stamp() {
  if [ -n "${SOURCE_DATE_EPOCH:-}" ]; then
    date -u -r "$SOURCE_DATE_EPOCH" "$1"
  else
    date -u "$1"
  fi
}

BUILD_STAMP="$(_stamp +%Y%m%d%H%M)"
ISO_NAME="GhostBSD.${CHANNEL}.${BUILD_STAMP}.iso"
```

After building packages and images, the system signs and publishes a `channel.index`:

```sh
create_channel_index "${CHANNEL}" > channel.index
minisign -S -s channel_private.key -m channel.index
```

---

## 7. Migration Path

1. Introduce channels alongside normal releases.

2. Build monthly **baselines** similar to current ISOs.

3. Let advanced users opt into the channel model:

   ```bash
   gbsd migrate --channel stable
   ```

4. Over time, de-emphasize traditional version numbers.
   For example, “GhostBSD 25.02” becomes **stable@patch:XYZ1234**.

---

## 8. Benefits

* Continuous delivery without a release bottleneck
* Signed, auditable provenance of every change
* Atomic upgrades and instant rollbacks
* Reproducible systems for research and CI
* Reduced need for frequent ISO rebuilds

---

## 9. Trade-Offs

* Requires new tooling (`gbsd-channel`, patch signing, server automation)
* Demands more discipline when preparing patches
* Some users prefer periodic named releases, so communication will matter

---

**In short:**
GhostBSD could evolve into a living system that is always current and always verifiable, without the artificial rhythm of fixed releases.
Every change becomes a signed patch in a channel, and the “release” is simply the patch set you trust today.

