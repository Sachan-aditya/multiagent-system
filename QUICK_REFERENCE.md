# Quick Reference - Commands to Run on Your EC2

## 🚀 FASTEST WAY - Copy/Paste One of These:

### Option 1: Complete Inspection (5 minutes)
```bash
echo "===== DOCKER STATUS =====" && sudo docker ps -a && echo "" && \
echo "===== USERS =====" && PGPASSWORD='N7dtRPZK64K1' psql -h dev-sarvdhan-psql.crlmxwxdweyk.ap-south-1.rds.amazonaws.com -U sarvdhan_api -d sarvdhan_chat_api -c "SELECT id, email, created_at FROM users;" && echo "" && \
echo "===== CHAT ROOMS =====" && PGPASSWORD='N7dtRPZK64K1' psql -h dev-sarvdhan-psql.crlmxwxdweyk.ap-south-1.rds.amazonaws.com -U sarvdhan_api -d sarvdhan_chat_api -c "SELECT id, loan_id, status FROM loan_chat_rooms ORDER BY id;" && echo "" && \
echo "===== RECENT MESSAGES =====" && PGPASSWORD='N7dtRPZK64K1' psql -h dev-sarvdhan-psql.crlmxwxdweyk.ap-south-1.rds.amazonaws.com -U sarvdhan_api -d sarvdhan_chat_api -c "SELECT id, loan_id, sender_display_name, LEFT(body, 60), created_at FROM loan_messages ORDER BY created_at DESC LIMIT 20;" && echo "" && \
echo "===== CHAT APP LOGS =====" && sudo docker logs --tail 150 sarvdhan-chat-app
```

### Option 2: Database Only (2 minutes)
```bash
PGPASSWORD='N7dtRPZK64K1' psql -h dev-sarvdhan-psql.crlmxwxdweyk.ap-south-1.rds.amazonaws.com -U sarvdhan_api -d sarvdhan_chat_api << 'EOF'
\dt
SELECT relname, n_live_tup FROM pg_stat_user_tables ORDER BY n_live_tup DESC;
SELECT id, email FROM users;
SELECT id, loan_id, status FROM loan_chat_rooms;
SELECT loan_id, COUNT(*) FROM loan_messages GROUP BY loan_id;
EOF
```

### Option 3: Logs Only (1 minute)
```bash
echo "=== CHAT APP ===" && sudo docker logs --tail 100 sarvdhan-chat-app && \
echo "" && echo "=== UI ===" && sudo docker logs --tail 100 tripayui && \
echo "" && echo "=== OPENFIRE ===" && sudo docker logs --tail 100 sarvdhan-openfire
```

---

## Individual Commands

### Docker

**Check if containers running:**
```bash
sudo docker ps
```

**Get logs from chat app:**
```bash
sudo docker logs --tail 200 sarvdhan-chat-app
```

**Get logs from UI:**
```bash
sudo docker logs --tail 100 tripayui
```

**Get logs from Openfire:**
```bash
sudo docker logs --tail 100 sarvdhan-openfire
```

**Watch logs live:**
```bash
sudo docker logs -f --tail 50 sarvdhan-chat-app
```

---

### Database - All Tables

**List all tables:**
```bash
PGPASSWORD='N7dtRPZK64K1' psql -h dev-sarvdhan-psql.crlmxwxdweyk.ap-south-1.rds.amazonaws.com -U sarvdhan_api -d sarvdhan_chat_api -c "\dt"
```

**Row counts in each table:**
```bash
PGPASSWORD='N7dtRPZK64K1' psql -h dev-sarvdhan-psql.crlmxwxdweyk.ap-south-1.rds.amazonaws.com -U sarvdhan_api -d sarvdhan_chat_api -c "SELECT relname, n_live_tup FROM pg_stat_user_tables ORDER BY n_live_tup DESC;"
```

---

### Database - Specific Tables

**Users (8 rows):**
```bash
PGPASSWORD='N7dtRPZK64K1' psql -h dev-sarvdhan-psql.crlmxwxdweyk.ap-south-1.rds.amazonaws.com -U sarvdhan_api -d sarvdhan_chat_api -c "SELECT id, email, created_at FROM users;"
```

**Tenants (2 rows):**
```bash
PGPASSWORD='N7dtRPZK64K1' psql -h dev-sarvdhan-psql.crlmxwxdweyk.ap-south-1.rds.amazonaws.com -U sarvdhan_api -d sarvdhan_chat_api -c "SELECT * FROM tenants;"
```

**Chat Rooms (7 rows):**
```bash
PGPASSWORD='N7dtRPZK64K1' psql -h dev-sarvdhan-psql.crlmxwxdweyk.ap-south-1.rds.amazonaws.com -U sarvdhan_api -d sarvdhan_chat_api -c "SELECT id, loan_id, room_jid, status, created_at FROM loan_chat_rooms;"
```

**Chat Members (35 rows):**
```bash
PGPASSWORD='N7dtRPZK64K1' psql -h dev-sarvdhan-psql.crlmxwxdweyk.ap-south-1.rds.amazonaws.com -U sarvdhan_api -d sarvdhan_chat_api -c "SELECT loan_id, COUNT(*) as members FROM loan_chat_members GROUP BY loan_id;"
```

**Chat Messages (334 rows - latest 30):**
```bash
PGPASSWORD='N7dtRPZK64K1' psql -h dev-sarvdhan-psql.crlmxwxdweyk.ap-south-1.rds.amazonaws.com -U sarvdhan_api -d sarvdhan_chat_api -c "SELECT id, loan_id, sender_display_name, LEFT(body, 80), created_at FROM loan_messages ORDER BY created_at DESC LIMIT 30;"
```

**Roles (3 rows):**
```bash
PGPASSWORD='N7dtRPZK64K1' psql -h dev-sarvdhan-psql.crlmxwxdweyk.ap-south-1.rds.amazonaws.com -U sarvdhan_api -d sarvdhan_chat_api -c "SELECT * FROM roles;"
```

**Permissions (9 rows):**
```bash
PGPASSWORD='N7dtRPZK64K1' psql -h dev-sarvdhan-psql.crlmxwxdweyk.ap-south-1.rds.amazonaws.com -U sarvdhan_api -d sarvdhan_chat_api -c "SELECT * FROM permissions;"
```

**Refresh Tokens (72 rows):**
```bash
PGPASSWORD='N7dtRPZK64K1' psql -h dev-sarvdhan-psql.crlmxwxdweyk.ap-south-1.rds.amazonaws.com -U sarvdhan_api -d sarvdhan_chat_api -c "SELECT COUNT(*) as total, COUNT(DISTINCT user_id) as unique_users FROM refresh_tokens;"
```

---

### Useful Analytics

**Messages per loan:**
```bash
PGPASSWORD='N7dtRPZK64K1' psql -h dev-sarvdhan-psql.crlmxwxdweyk.ap-south-1.rds.amazonaws.com -U sarvdhan_api -d sarvdhan_chat_api -c "SELECT loan_id, COUNT(*) as message_count, MAX(created_at) as last_message FROM loan_messages GROUP BY loan_id ORDER BY message_count DESC;"
```

**Members per loan:**
```bash
PGPASSWORD='N7dtRPZK64K1' psql -h dev-sarvdhan-psql.crlmxwxdweyk.ap-south-1.rds.amazonaws.com -U sarvdhan_api -d sarvdhan_chat_api -c "SELECT loan_id, COUNT(*) FROM loan_chat_members WHERE removed_at IS NULL GROUP BY loan_id;"
```

**Activity in last 24 hours:**
```bash
PGPASSWORD='N7dtRPZK64K1' psql -h dev-sarvdhan-psql.crlmxwxdweyk.ap-south-1.rds.amazonaws.com -U sarvdhan_api -d sarvdhan_chat_api -c "SELECT COUNT(*) as messages_today FROM loan_messages WHERE created_at > NOW() - INTERVAL '24 hours';"
```

**User with most messages:**
```bash
PGPASSWORD='N7dtRPZK64K1' psql -h dev-sarvdhan-psql.crlmxwxdweyk.ap-south-1.rds.amazonaws.com -U sarvdhan_api -d sarvdhan_chat_api -c "SELECT sender_display_name, COUNT(*) as message_count FROM loan_messages GROUP BY sender_display_name ORDER BY message_count DESC LIMIT 10;"
```

---

## 📋 What Each Command Shows

| Command | Purpose | Key Info |
|---------|---------|----------|
| `docker ps` | Container status | Are all 3 containers UP? |
| `docker logs` | Application logs | Any ERROR or WARN messages? |
| `\dt` | List tables | Should be 17 tables |
| `pg_stat_user_tables` | Row counts | Messages: 334, Rooms: 7, etc. |
| `users` | User accounts | How many users? Are they created? |
| `loan_chat_rooms` | Chat rooms | How many rooms? What status? |
| `loan_messages` | Chat messages | Latest message timestamp? Count? |
| `roles` | Access control | Admin, sales, bot defined? |

---

## Connection String Reference

**PostgreSQL:**
- Host: `dev-sarvdhan-psql.crlmxwxdweyk.ap-south-1.rds.amazonaws.com`
- Port: `5432`
- Database: `sarvdhan_chat_api`
- Username: `sarvdhan_api`
- Password: `N7dtRPZK64K1`

**Redis:**
- Host: `master.sarvdhan-redis.rqmnod.aps1.cache.amazonaws.com`
- Port: `6379`

**Openfire:**
- URL: `http://localhost:9090`
- Admin Console: `http://localhost:9090/admin`

**Chat API:**
- URL: `https://chat-api.sarvdhan.com`
- Local: `http://localhost:8080`

**UI:**
- URL: `http://<your-ec2-public-ip>`
- Local: `http://localhost`

---

## 🎯 Fastest Diagnosis (Copy-Paste This)

```bash
echo "DOCKER STATUS:" && \
sudo docker ps --format "table {{.Names}}\t{{.Status}}" && \
echo "" && \
echo "DATABASE CONNECTIVITY:" && \
PGPASSWORD='N7dtRPZK64K1' psql -h dev-sarvdhan-psql.crlmxwxdweyk.ap-south-1.rds.amazonaws.com -U sarvdhan_api -d sarvdhan_chat_api -c "SELECT COUNT(*) as message_count FROM loan_messages;" && \
echo "" && \
echo "RECENT ERRORS:" && \
sudo docker logs --tail 50 sarvdhan-chat-app 2>&1 | grep -i "error\|exception" | head -5
```

This will tell you in 10 seconds:
1. Are all containers running?
2. Can we reach the database?
3. Are there any critical errors?
