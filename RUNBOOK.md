# VETRA Operations Runbook

**Server:** 37.27.248.86 | SSH: `ssh -p 2222 root@37.27.248.86`  
**Service:** `gunicorn-vetra.service` (port 8003)  
**Domain:** vetra.live  
**DB:** vetra_db (PostgreSQL :5432)  
**Last updated:** 2026-04-27  

---

## SLA

| Plan | Response time | Uptime target |
|------|---------------|---------------|
| Free | Best effort | 99% |
| Starter | 24h | 99.5% |
| Pro | 4h | 99.9% |
| Enterprise | 1h | 99.95% |

---

## Service Management

```bash
# Status
systemctl status gunicorn-vetra.service
systemctl status nginx celery-vetra.service

# Restart
systemctl restart gunicorn-vetra.service

# Reload nginx (config changes)
nginx -t && systemctl reload nginx

# Check logs
journalctl -u gunicorn-vetra.service -n 100 --no-pager
tail -f /var/log/vetra/error.log
tail -f /var/log/vetra/access.log

# Django check
cd /var/www/vetra && DJANGO_SETTINGS_MODULE=vetra.settings venv/bin/python manage.py check
```

---

## Common Incidents

### P1: Site down (HTTP 502/503)

1. `systemctl status gunicorn-vetra.service` — is it active?
2. If dead: `systemctl restart gunicorn-vetra.service`
3. If restart fails: `journalctl -u gunicorn-vetra.service -n 50` — check error
4. If DB down: `systemctl status postgresql` — restart if needed
5. Check disk: `df -h` — if full, clean logs `/var/log/vetra/*.log`

### P2: High error rate (Sentry alerts)

1. `tail -f /var/log/vetra/error.log` — identify pattern
2. Check recent deploys: `cd /var/www/vetra && git log --oneline -5`
3. Rollback if needed: `git revert HEAD && systemctl restart gunicorn-vetra.service`

### P3: OTS upgrades failing

1. `cd /var/www/vetra && DJANGO_SETTINGS_MODULE=vetra.settings venv/bin/python manage.py shell`
2. `from archive.models import VetraDocument; print(VetraDocument.objects.filter(ots_confirmed=False).count())`
3. Check Celery: `systemctl status celery-vetra.service`
4. Restart Celery: `systemctl restart celery-vetra.service`

### P4: ClamAV not running

1. `systemctl status clamav-daemon`
2. `systemctl restart clamav-daemon`
3. Update signatures: `freshclam`

### P5: Disk full

```bash
df -h
# Clean old logs
find /var/log/vetra -name "*.log" -mtime +30 -delete
# Clean old media (orphaned files)
cd /var/www/vetra && DJANGO_SETTINGS_MODULE=vetra.settings venv/bin/python manage.py shell -c "
from archive.models import VetraDocument
import os
for doc in VetraDocument.objects.all():
    if doc.file and not os.path.exists(doc.file.path):
        print('missing:', doc.seal)
"
```

---

## Database

```bash
# Connect
psql -U vetra_user -d vetra_db

# Backup (manual)
pg_dump vetra_db > /var/backups/vetra/manual_$(date +%Y%m%d_%H%M%S).sql

# Restore
psql -U vetra_user -d vetra_db < backup.sql
```

Automated daily backup: `/var/www/vetra/scripts/backup.sh` (cron 03:00)

---

## Deploy Procedure

```bash
cd /var/www/vetra
git pull origin main
venv/bin/pip install -r requirements.txt --quiet
DJANGO_SETTINGS_MODULE=vetra.settings venv/bin/python manage.py migrate --no-input
DJANGO_SETTINGS_MODULE=vetra.settings venv/bin/python manage.py collectstatic --no-input --clear
DJANGO_SETTINGS_MODULE=vetra.settings venv/bin/python manage.py check
systemctl restart gunicorn-vetra.service
# Verify
curl -s -o /dev/null -w "%{http_code}" https://vetra.live/
```

---

## Escalation

- Level 1 (on-call): Restart service + check logs
- Level 2 (senior): Code rollback + DB inspection
- Level 3 (Ivan): Architecture-level decisions
- Contact: support@vetra.live | WhatsApp: +385...

---

## Monitoring

- Sentry: configured via `SENTRY_DSN` in `.env`
- Uptime: status.vetra.live (Betteruptime — pending setup)
- Grafana: not yet configured
