# Deploying OTG to AWS EC2

Five containers on one server, started and stopped together with docker compose:

```
internet ──80/443──> nginx (the React site + HTTPS) ──/api──> backend (Spring Boot)
                                                                 ├──> postgres  (volume: pgdata)
                                                                 └──> redis     (volume: redisdata)
                     certbot (renews the HTTPS certificate every 12h)
```

Only nginx is reachable from the internet. Postgres, Redis and the backend are on Docker's
private network.

Throughout this guide, `passes.example.com` stands for your real domain.

---

## 1. AWS: create the server

1. **EC2 → Launch instance**
   - Image: **Ubuntu Server 24.04 LTS**
   - Type: **t3.small (2 GB) or larger**. The images are built on the server, and a 1 GB
     instance runs out of memory building the backend.
   - Storage: 20 GB gp3
   - Key pair: create one, and keep the `.pem` file safe - it's your only way in.
2. **Security group** (the firewall), inbound rules:

   | Port | Source | Why |
   |---|---|---|
   | 22 | **My IP** only | SSH |
   | 80 | Anywhere | HTTP (redirects to HTTPS; certificate checks) |
   | 443 | Anywhere | HTTPS |

   Never open 5432 (Postgres) or 6379 (Redis).
3. **Elastic IP**: allocate one and associate it with the instance, so the IP never changes.

## 2. Point the domain at the server

At your domain registrar, add a DNS record:

| Type | Name | Value |
|---|---|---|
| A | `passes` (or `@` for the root domain) | the Elastic IP |

Check it has propagated before step 6: `nslookup passes.example.com` should print the Elastic IP.

## 3. Prepare the server

SSH in (from the folder with your `.pem`):

```bash
ssh -i otg-key.pem ubuntu@<elastic-ip>
```

Add 2 GB of swap - it's what lets a 2 GB instance survive the image build:

```bash
sudo fallocate -l 2G /swapfile && sudo chmod 600 /swapfile
sudo mkswap /swapfile && sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

Install Docker (Docker's official install script) and let the `ubuntu` user run it:

```bash
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker ubuntu
exit
```

SSH in again so the group change applies, then check: `docker compose version`.

## 4. Get the code

The three folders must sit side by side:

```bash
mkdir ~/otg && cd ~/otg
git clone -b rephrase https://github.com/offthegrid-2026/backend.git otg-backend
git clone -b rephrase https://github.com/offthegrid-2026/frontend.git frontend
# then copy the otg-deploy folder here (git clone it, or scp it from your PC)
```

If the repos are private, use a **read-only deploy key** per repo (GitHub → repo →
Settings → Deploy keys), or a fine-grained personal access token with read access only.

## 5. Create the production `.env`

```bash
cd ~/otg/otg-deploy
cp .env.example .env
nano .env
```

Fill in **new** values - never reuse the dev ones:

- `DOMAIN=passes.example.com`, `CORS_ALLOWED_ORIGINS=https://passes.example.com`
- `NGINX_MODE=http` and `COMPOSE_PROFILES=` (empty) for now
- `DB_PASSWORD`, `REDIS_PASSWORD`, `JWT_SECRET_KEY`: generate each with `openssl rand -base64 48`
- `MAIL_USERNAME` + a **new** Gmail app password created only for production
- `SCANNER_MAIL`: the gate-staff emails, comma-separated
- `RAZORPAY_*`: **test** keys for now
- `CERTBOT_EMAIL`: your email

Lock the file down: `chmod 600 .env`

## 6. First start (HTTP), then get the certificate

```bash
docker compose up -d --build
docker compose ps
```

The first build takes several minutes. Wait until `backend` shows `healthy`, then open
`http://passes.example.com` - the site should load (plain HTTP for now).

Request the certificate:

```bash
( set -a; . ./.env; set +a
  docker compose run --rm --entrypoint certbot certbot certonly \
    --webroot -w /var/www/certbot \
    -d "$DOMAIN" --email "$CERTBOT_EMAIL" --agree-tos --no-eff-email )
```

It should end with `Successfully received certificate`.

The parentheses matter: they load `.env` only for that one command. Without them the values
stay set in your terminal, and docker compose gives terminal values priority over `.env` -
so the next step's edit to `.env` would be silently ignored.

## 7. Switch to HTTPS

In `.env`, set:

```
NGINX_MODE=https
COMPOSE_PROFILES=https
```

Then:

```bash
docker compose up -d
```

`https://passes.example.com` should now load with a padlock, and `http://` should redirect
to it. certbot now renews the certificate automatically; nginx reloads every 6 hours to
pick up the renewed one.

## 8. Razorpay (test mode first)

1. Razorpay Dashboard → **Test mode** → Webhooks → Add webhook:
   - URL: `https://passes.example.com/api/v1/payments/webhook`
   - Secret: exactly the value of `RAZORPAY_WEBHOOK_SECRET`
   - Event: `payment.captured`
2. Do a full test: sign up, buy a pass with a Razorpay test card, check the ticket email
   arrives, download the PDF from the dashboard (the attendee's name must appear on it),
   then scan it with a phone logged in as a scanner.

## 9. Going live

Only after step 8 passes end to end:

1. Razorpay → switch to **Live mode**, generate live keys, and enable **automatic capture**.
2. Add the webhook again in **Live mode** (test and live webhooks are separate), with a
   new secret.
3. Update `.env`: `RAZORPAY_KEY_ID` (`rzp_live_...`), `RAZORPAY_KEY_SECRET`,
   `RAZORPAY_WEBHOOK_SECRET`.
4. `docker compose up -d backend`

---

## Day-to-day operations

All commands run from `~/otg/otg-deploy`.

**Deploy new code**

```bash
git -C ../otg-backend pull && git -C ../frontend pull
docker compose up -d --build
```

**Logs**

```bash
docker compose logs -f backend      # Ctrl+C to stop following
docker compose logs --since 1h nginx
```

**Status / restart**

```bash
docker compose ps
docker compose restart backend
```

**Change ticket prices, seats or sale dates** (there is no admin screen - it's SQL):

```bash
( set -a; . ./.env; set +a; docker compose exec postgres psql -U "$DB_USER" -d "$DB_NAME" )
```

Prices are in **paise** (₹599 = `59900`); always write dates with `+05:30`. Wrap changes in
`BEGIN; ... COMMIT;` and check the result before committing.

**Back up the database** - do this before every deploy, and set up the nightly cron below:

```bash
mkdir -p backups
( set -a; . ./.env; set +a; docker compose exec -T postgres pg_dump -U "$DB_USER" -d "$DB_NAME" ) | gzip > backups/otg-$(date +%F).sql.gz
```

Nightly at 03:00, keeping 14 days (`crontab -e`):

```
0 3 * * * cd /home/ubuntu/otg/otg-deploy && set -a && . ./.env && set +a && docker compose exec -T postgres pg_dump -U "$DB_USER" -d "$DB_NAME" | gzip > backups/otg-$(date +\%F).sql.gz && find backups -name '*.sql.gz' -mtime +14 -delete
```

A backup on the same server doesn't survive losing the server - also copy `backups/` to S3
(`aws s3 sync backups s3://<bucket>/otg-backups`, with an IAM role on the instance).

## Things that must never happen

- `docker compose down -v` - the `-v` **deletes the database volume** (every user, payment
  and ticket). Plain `docker compose down` is safe.
- Committing `.env`, or pasting its contents anywhere.
- Opening ports 5432 or 6379 in the security group.

## Troubleshooting

| Symptom | Check |
|---|---|
| `backend` never becomes healthy | `docker compose logs backend` - usually a missing or wrong value in `.env` |
| Site loads but login/API calls fail | `CORS_ALLOWED_ORIGINS` must exactly match the address in the browser |
| certbot: "Timeout during connect" | DNS not pointing at the Elastic IP yet, or port 80 closed in the security group |
| nginx won't start after switching to https | The certificate wasn't issued - go back to `NGINX_MODE=http` and redo step 6 |
| An edit to `.env` has no effect after `docker compose up -d` | Old values are still set in your terminal (from an earlier `set -a`). Run `unset $(grep -oE '^[A-Z_]+' .env)`, or log out and back in, then `docker compose up -d` again |
| Scanner camera doesn't open on phones | The site must be opened over `https://` |
| Build killed / out of memory | The swap from step 3 is missing, or the instance is under 2 GB |
