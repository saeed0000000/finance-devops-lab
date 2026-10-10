# PostgreSQL PITR Lab

## Purpose

Restore PostgreSQL from a base backup and replay archived WAL
until the named restore point `pitr_before_t0`.

The restore runs in a separate Pod and PVC. It does not replace
the original `postgres-0` data directory.

## Prerequisites

The following resources must exist in the cluster:

- PVC `postgres-base-backups`
- PVC `postgres-wal-archive`
- Base backup directory:
  `base-20261008-121335`
- Archived WAL containing the restore point:
  `pitr_before_t0`

This manifest is specific to the current training environment.
Update the backup path and recovery settings when using another backup.

## Run

```bash
kubectl apply -f labs/pitr/postgres-pitr-restore-v2.yaml

kubectl logs postgres-pitr-restore-v2 -c prepare-restore
kubectl logs postgres-pitr-restore-v2 -c postgres
```

## Verify recovered data

```bash
kubectl exec postgres-pitr-restore-v2 -c postgres -- \
  psql -U finance -d finance -c "
SELECT 'BEFORE-T0 marker' AS marker, COUNT(*) AS rows
FROM users
WHERE email = 'pitr-before-t0@example.local'
UNION ALL
SELECT 'AFTER-T0 marker', COUNT(*)
FROM users
WHERE email = 'pitr-after-t0@example.local';
"
```

Expected result:

- `BEFORE-T0 marker`: 1
- `AFTER-T0 marker`: 0

## Verify recovery state

```bash
kubectl exec postgres-pitr-restore-v2 -c postgres -- \
  psql -U finance -d finance -x -c "
SELECT
    pg_is_in_recovery() AS in_recovery,
    pg_get_wal_replay_pause_state() AS replay_pause_state;
"
```

Expected result:

- `in_recovery`: `t`
- `replay_pause_state`: `paused`

## Cleanup

After verification, remove the temporary restore resources if they
are no longer needed:

```bash
kubectl delete pod postgres-pitr-restore-v2
kubectl delete pvc postgres-pitr-restore-v2
```

Do not delete the source base backup or WAL archive as part of this cleanup.
Do not delete or modify the original `postgres-0` data PVC.

## Limitations

This lab demonstrates PITR within the existing Kind cluster.
The base backup and WAL archive remain inside the same lab environment.
It does not yet demonstrate disaster recovery from a complete cluster or
storage failure.
