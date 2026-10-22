# GoodSync backup analysis: /Volumes/Stockage2/Backups/goodsync

Read-only analysis for a keep-or-delete decision. Nothing under `/Volumes/Stockage2` was
written, moved or deleted. Readings taken 2026-10-22 between 17:30 and 17:50 CEST from
MSM3U (`macstudio-m3ultra-regis`); live sources read over ssh, read only.

## Verdict

The tree is a frozen GoodSync PULL mirror of two spokes (MSM4M, MBPM5M) taken between
2026-05-28 and 2026-06-02. Nothing has touched it since: 0 inodes changed after
2026-06-03. About 197.6 of its 198.9 GiB still exists elsewhere today or is regenerable.
The one unique asset of real value is **1.30 GiB of MBPM5M Claude Code transcripts**
(9,254 files) that exist in no other location checked. Recommendation: keep one copy of
`MBPM5M/Claude/projects` (3.66 GiB) under `/Volumes/Stockage2/archive`, delete the rest,
which frees about 195 GiB.

## 1. Structure

| Level | Meaning |
|---|---|
| `goodsync/<SPOKE>/` | origin machine: `MSM4M` (Mac Studio M4 Max), `MBPM5M` (MacBook Pro M5 Max) |
| `<SPOKE>/Claude` | `~/.claude` |
| `<SPOKE>/Development` | `~/Development` (`.git`, `node_modules`, build dirs, hidden files excluded) |
| `<SPOKE>/LLM` | `~/.codex` (Codex CLI state and sessions) |
| `<SPOKE>/LocalShare` | `~/.local/share` |
| `<SPOKE>/<PERIM>/_gsdata_` | per-job GoodSync state and 30-day saved prior versions |
| `goodsync/_gsdata_` | state of a job rooted at `goodsync/` plus `_saved_/MBPM5M` (prior versions) |

GoodSync jobs: 8 pull jobs named `<SPOKE>-<PERIM>` (`/dir=rtol /deletions=no`, 30-day
versioning), created by `~/.local/share/goodsync/bin/pivot-create-pull-jobs.sh` and run by
the `com.rcr.goodsync.pull-dispatch` LaunchAgent. That agent is no longer loaded and its
plist is gone; the last dispatcher log lines are 2026-06-02 19:26 (8/8 exit 0). Context:
`~/.local/share/goodsync/docs/AAR-2026-06-02-pull-topology-migration.md`. The fleet has
since moved to the checkout depot (`~/checkout-depot`, versioned by GoodSync into
`/Volumes/Stockage2/checkout-versioning`), so this mirror has no current role.

## 2. Size and age (du -sk; file mtime from stat; counts from find -type f)

| Subtree | Size (GiB) | Files | Oldest mtime | Newest mtime |
|---|---|---|---|---|
| MSM4M/Development | 89.72 | 772,952 | 2025-09-14 | 2026-06-02 19:04 |
| MBPM5M/Development | 54.33 | 662,986 | 2025-09-14 | 2026-06-02 01:36 |
| `_gsdata_` (root) | 41.99 | 632,463 | 2025-12-26 | 2026-06-02 00:53 |
| MBPM5M/Claude | 4.62 | 60,395 | 2025-11-24 | 2026-06-02 19:26 |
| MBPM5M/LocalShare | 3.29 | 19,628 | 2026-02-10 | 2026-06-02 18:00 |
| MSM4M/LocalShare | 3.05 | 11,641 | 2018-05-15 | 2026-06-02 17:39 |
| MSM4M/LLM | 1.48 | 1,517 | 2026-02-08 | 2026-06-02 19:25 |
| MSM4M/Claude | 0.22 | 14,183 | 2025-12-19 | 2026-06-02 19:24 |
| MBPM5M/LLM | 0.18 | 876 | 2026-03-16 | 2026-06-02 19:26 |
| **Total** | **198.86** | **2,176,641** | | |

Notable inner parts: `MSM4M/Development/claude-mem-backups` 17.56 GiB,
`MBPM5M/Development/claude-mem-backups` 5.04 GiB, `MBPM5M/Development/_gsdata_` 6.12 GiB,
`MBPM5M/Claude/projects` 3.66 GiB. Root `_gsdata_/_saved_/MBPM5M` (41.97 GiB) holds prior
versions, over 99% of them under `Development/GitHub` (Regis-RCR 26.5 GiB, RCR-PRO 6.3 GiB).

Written after 2026-06-01: yes, the last pulls on 2026-06-02 (newest file mtime 19:26).
Written after 2026-06-02: no. `find -newerct '2026-06-03'` over the whole tree returns 0.

## 3. Redundancy

Method: per subtree, 300 random files (fixed seed, `_gsdata_` excluded) checked by relative
path and size on the live origin node over ssh; up to 25 same-size files checksummed with
`shasum -a 256` on both sides. Development got a second sample of 2,000 files. The high
value areas (Claude projects, Codex sessions, claude-mem-backups, unison backups) were
compared exhaustively by path and size. Same size is not equal content except where
checksummed. Locations checked: origin nodes, MSM5M (where repos are migrating), MSM3U
`~/Development` and `~/.claude/projects`, `~/pCloud Drive/claude-state`,
`/Volumes/Stockage2/{archive,_offload-msm3u-2026-10-16,fleet-archive}`, and the Time
Machine snapshot 2026-06-01-104837.

| Subtree | Sample on origin (same / diff / missing) | SHA equal | Where the missing part is | State |
|---|---|---|---|---|
| MSM4M/Development | 1,976 / 4 / 20 of 2,000 | 25/25 | 17 of 20 in `claude-mem-benchmark` (deleted outputs); 0.05% of sampled bytes | fully duplicated, residue under 1% |
| MBPM5M/Development | 1,993 / 0 / 7 of 2,000 | 25/25 | 7/7 present on MSM5M | fully duplicated |
| claude-mem-backups (both) | 4,658 of 4,659 by path and size | n/a | 1 file | fully duplicated |
| Development dirs (635) | all exist on origin, MSM5M or MSM3U | | `docs`, `claude-mem-doctor-tui-impl` on MSM5M or MSM3U only | fully duplicated |
| MBPM5M/Claude | 11 / 3 / 286 of 300 | 10/11 | plugin caches; and `projects/`, see below | partly |
| MSM4M/Claude | 11 / 0 / 289 of 300 | 10/11 | plugin caches only (no `projects/` in this copy) | duplicated or regenerable |
| MBPM5M/LLM | 57 / 42 / 201 of 300 | 22/25 | Codex plugin caches; sessions 79/79 present live | duplicated or regenerable |
| MSM4M/LLM | 77 / 165 / 58 of 300 | 25/25 | Codex plugin caches; sessions 697/697 present live | duplicated or regenerable |
| MBPM5M/LocalShare | 120 / 0 / 180 of 300 | 25/25 | `unison/backups`: 179/179 in offload cold archive, 20/20 SHA equal | fully duplicated |
| MSM4M/LocalShare | 76 / 48 / 176 of 300 | 24/25 | `unison/backups`: 170/171 in offload cold archive, 20/20 SHA equal; uv tool caches | fully duplicated |
| root `_gsdata_` | not comparable | | prior versions, by nature in no live tree | unique, low value |

"diff" rows are files rewritten in place since June (sqlite, state json, growing logs).

**MBPM5M Claude transcripts, the exception.** `MBPM5M/Claude/projects` holds 20,791 files
(3.61 GiB). 20,386 of them are no longer on MBPM5M nor in MSM3U `~/.claude/projects`.
Of those, 11,119 files (2.29 GiB) match by name and size in `~/pCloud Drive/claude-state`.
**9,254 files (1.30 GiB) match nowhere**: 0.72 GiB of claude-mem observer sessions
(machine-generated) and 0.57 GiB of real project sessions (3,013 top-level `.jsonl` plus
5,102 subagent and tool-result files, mtimes 2026-01-09 to 2026-06-01). The Time Machine
copy of the old internal `~/Backups/goodsync` (snapshot 2026-06-01) has no `projects/`
directory: 0/60 sampled unique files present.

Failure domain: `TimeMachine2` and `Stockage2` are two APFS volumes of one physical SSD
(WD_BLACK SN850X 8 TB, disk7), so the Time Machine copy (about 158 GiB, incomplete) is not
independent of this tree, and deleting this tree frees nothing on the TM volume.

## 4. Sensitivity (path and file type only; no content opened)

| Category | Where (examples, by path) | Count |
|---|---|---|
| OAuth or API token stores | `<SPOKE>/LLM/auth.json`, `<SPOKE>/LocalShare/{kilo,opencode}/auth.json` | 6 |
| Certificates with likely private keys (`.pfx`) | `<SPOKE>/Development/ABTemp.pfx`, `.../RCR-PRO/{AB-CMA,LMR-CMA}/certs/*.pfx` | 8 |
| Credentials-named file | `MSM4M/Development/RCR-Private/` | 1 |
| `_PRIVATE` and `RCR-Private` trees | `<SPOKE>/Development/...`, `<SPOKE>/LocalShare/...` | 252 files |
| Conversation transcripts (may hold pasted secrets) | `<SPOKE>/Claude/{projects,history.jsonl*,paste-cache}`, `<SPOKE>/LLM/sessions` | 21,000+ |
| Office and PDF documents | `<SPOKE>/Development/...` (`.pdf`, `.docx`, `.xlsx`) | 428 (94 MiB) |
| SSH | `LocalShare/chezmoi/private_dot_ssh/id_ed25519.pub` only (public key) | 2 |

The volume is **not encrypted** (`diskutil info disk7s2`: Encrypted No, FileVault No). The
token files are copies from June 2026; whether they are still valid was not tested.

## 5. Recommendation per subtree

| Subtree | Action | Frees (GiB) | What would be lost |
|---|---|---|---|
| MSM4M/Development | delete | 89.72 | a June 2026 snapshot of working trees; under 1% of files later deleted at source (mostly `claude-mem-benchmark` outputs) |
| MBPM5M/Development | delete | 54.33 | the same class, plus 6.12 GiB of 30-day prior versions |
| root `_gsdata_` | delete | 41.99 | prior versions (Dec 2025 to Jun 2026) of mostly git-tracked files |
| MBPM5M/Claude/projects | **keep one copy** (move to `archive/claude-sessions/MBPM5M`) | 0 | if deleted: 1.30 GiB of transcripts found nowhere else |
| MBPM5M/Claude, rest | delete | 0.96 | plugin caches, file-history, settings backups |
| MSM4M/Claude | delete | 0.22 | plugin caches and old history backups |
| MSM4M/LLM, MBPM5M/LLM | delete | 1.66 | regenerable caches, June sqlite states, stale `auth.json` tokens |
| MSM4M/LocalShare, MBPM5M/LocalShare | delete | 6.34 | nothing found unique (unison backups are in the offload cold archive) |

Totals: keep `MBPM5M/Claude/projects` and delete the rest frees about **195.2 GiB**;
delete everything frees 198.9 GiB and loses the 1.30 GiB of unique transcripts.

Operational notes for whoever executes the decision: only a process with Full Disk Access
on MSM3U can write Stockage2 (sshd cannot); if deleting, also remove the 8 `<SPOKE>-<PERIM>`
job definitions from GoodSync.app so a manual run cannot recreate the tree; deleting the
stale tokens and certificates also removes them from an unencrypted volume.

## Limits of this reading

Presence was checked by path and size on samples, not by full content comparison; the
Development residue is an estimate (2,000-file samples per node). The pCloud match is by
file name and size in the local pCloud Drive mount, not a checksum. GoodSync job
definitions inside GoodSync.app were not located on disk and are not counted here.

∵ Regis RCR ∴
