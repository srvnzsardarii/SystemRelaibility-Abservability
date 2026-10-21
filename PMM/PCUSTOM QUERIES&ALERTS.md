# PMM:Custom PostgreSQL Queries and Alerts
How to add custom `postgres_exporter` queries(replication lag,standby count,transaction anomaly) to Percona Monitoring and Management (PMM) on Ubuntu and how to alert on them.
> Verified against PMM 3.x conventions.
> Check the official docs if your version differs:https://docs.percona.com/percona-monitoring-and-management/

## 1.Metrics Overview
| Query name | Metric exposed in PMM | Type | Meaning |
|------------|----------------------|------|---------|
|`pg_replication_lag_seconds`|`pg_replication_lag_seconds_lag_seconds`|Gauge|Replication lag(seconds) on a standby.Returns `0` on a primary. |
|`pg_stat_replication_count`|`pg_stat_replication_count_count`|Gauge|Number of streaming standbys connected to this node (meaningful on the primary).|
|`pg_transaction_anomaly`|`pg_transaction_anomaly_n_tup_ins`,`_n_tup_upd`,`_n_tup_del`(label `relname`)|Counter|Cumulative inserts/updates/deletes on `payment_transactions`.Use `rate()` to detect spikes or drops.|

Naming rule:PMM exposes each metric as `<query_name>_<column_name>`,which is why `lag_seconds` becomes `pg_replication_lag_seconds_lag_seconds`.
The original definition was written as a Kubernetes ConfigMap(CloudNativePG).
On a plain Ubuntu host with PMM there is no ConfigMap:the YAML is simply placed as a file on the PostgreSQL host(below).

## 2.Prerequisites
- PMM Server and PMM Client installed(see the main PMM installation guide).
- PostgreSQL already added with `pmm-admin add postgresql ...`.
- The `pmm` monitoring user can read `pg_stat_replication`,`pg_stat_user_tables` and call `pg_last_xact_replay_timestamp()`(a superuser or `pg_monitor` role member can).
- The table `payment_transactions` exists in the database PMM connects to (see Notes).

## 3.Install the Custom Queries
On each PostgreSQL host (primary and standbys):
```bash
#Choose a resolution directory: high resolution(1s-5s),medium resolution(5s-10s),low-resolution(60s)
DIR=/usr/local/percona/pmm/collectors/custom-queries/postgresql/medium-resolution
sudo cp postgres-custom-queries.yaml $DIR/postgres-custom-queries.yaml
sudo chown pmm-agent:pmm-agent $DIR/postgres-custom-queries.yaml
sudo chmod 660 $DIR/postgres-custom-queries.yaml
sudo systemctl restart pmm-agent
```
The YAML file is in this repo: [`postgres-custom-queries.yaml`](./postgres-custom-queries.yaml).
If the `pmm agent` user does not exist on your system,check the owner of the parent directory (`ls -ld $DIR`) and use the same owner.

## 4.Verify the Metrics
1. In the PMM UI open Explore and choose the VictoriaMetrics/Prometheus data source.
2. Run:
```promql
pg_replication_lag_seconds_lag_seconds
pg_stat_replication_count_count
pg_transaction_anomaly_n_tup_ins
```
3. If nothing appears after 1-2 minutes,check the agent log:
```bash
sudo journalctl -u pmm-agent -n 100 --no-pager
pmm-admin status
```
Common causes: YAML indentation error,wrong file permissions,missing privileges,table `payment_transactions` not found.

## 5. Alerts
Create the rules in PMM:Alerting > Alert rules > New alert rule(data source: the PMM metrics source). 
Thresholds below are starting points;tune them to your workload.
### 5.1.Replication lag too high
```promql
pg_replication_lag_seconds_lag_seconds > 30
```
- Evaluate every: `1m`,For:`5m`
- Severity: `warning` (use `> 300` for `critical`)
### 5.2.Standby disconnected
Applies to the primary only;filter by service/node name so it does not fire on standbys.
```promql
pg_stat_replication_count_count{service_name="pgsql-prod-primary"} < 1
```
- For:`2m`
- Severity:`critical`
Replace `1` with the number of standbys you expect.
### 5.3.Transaction anomaly:unusual delete/update rate
```promql
rate(pg_transaction_anomaly_n_tup_del{relname="payment_transactions"}[5m]) > 5
```
```promql
rate(pg_transaction_anomaly_n_tup_upd{relname="payment_transactions"}[5m]) > 100
```
- For: `5m`
- Severity: `warning`
### 5.4.Transaction anomaly:inserts stopped
```promql
rate(pg_transaction_anomaly_n_tup_ins{relname="payment_transactions"}[15m]) == 0
```
- For: `15m`
- Severity: `critical`
- Consider a time-of-day mute timing if traffic legitimately stops at night.
### 5.5.Notification channel
Configure a contact point(email,Slack,Telegram webhook,etc.) 
under Alerting > Contact points,then route the rules to it under Notification policies.

## 6.Notes and Limitations
- Lag on idle primary:`now() - pg_last_xact_replay_timestamp()` keeps growing on a standby when the primary has no writes,producing false alarms. If this is an issue,combine with a write activity check or compare LSNs via `pg_stat_replication` on the primary.
- `pg_stat_replication_count` on standbys returns `0` unless cascading replication is used.
-Single database scope:`pg_stat_user_tables` is per database.The custom query runs against the database PMM connects to;make sure it is the one containing `payment_transactions`.
- Counters reset** on `pg_stat_reset()` or crash recovery;`rate()` handles this safely.
- Schema-qualified tables:the query matches by `relname` only.If several schemas contain the same table name,add `schemaname` as a LABEL and filter on it.

## 7.Rollback
```bash
sudo rm /usr/local/percona/pmm/collectors/custom-queries/postgresql/medium-resolution/postgres-custom-queries.yaml
sudo systemctl restart pmm-agent
```

Maintainer:Sarvenaz Sardari
