# Commit subject cleanup (2026-10-26)

Reviewed 1569 current bringup commits across eight repositories.
Five common-device subjects and one local-manifest subject were normalized.
All common-device commit trees, author identities/dates and raw message-body bytes
remain identical. Change-Id, Signed-off-by and upstream links remain intact.
Every rewritten commit has a valid local GPG signature. No cherry-picks were squashed.

| Previous subject | Normalized subject |
| --- | --- |
| sdm660-common: Drop legacy ANT remanents | sdm660-common: Drop legacy ANT remnants |
| sdm660-common: tetheroffload: Version 1.1 | sdm660-common: tetheroffload: Update to version 1.1 |
| sdm660-common: remove lineage-priv keys inherit | sdm660-common: Remove duplicate private-key inheritance |
| mata: Use faux /metadata partition | sdm660-common: Use faux /metadata partition |
| sdm660-common: rro_overlays: Clean up pinner list | sdm660-common: overlay: Remove obsolete libRScpp pin entry |
| bringup: Record confirmed hotspot spawn failure and post-boot fixes | X00TD: Record confirmed hotspot spawn failure and post-boot fixes |

The metadata fix is still derived from LuK1337's original mata commit;
the local subject now describes the ASUS tree where its paths were adapted.
Upstream full message bodies are retained, with subject adaptations recorded
explicitly in commit-subject-cleanup.json. A changed parent also changes hashes
of later commits even when their subject and source tree are unchanged.

Kernel head e572bc741676 and all 1,540 kernel commits are unchanged.
Commit-message cleanup alone requires no rebuild and does not alter existing
ROM ZIPs or images. The pending hotspot/cpuset kernel rebuild is separate.

The user approved replacing the two affected branch histories after creating
remote backup refs at backup/axion-before-subject-cleanup-20261026.
Each replacement uses an explicit SHA-pinned force-with-lease.
The original common head is e1f6c62c52a92b1bb1efe0248e0616c70195f01b.
Candidate common head: 84b1432de1ee99574ba979e1aa4f4c2485a30043.
See the JSON mapping for every old/new commit ID.
