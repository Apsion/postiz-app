# Deploying Postiz to Dokploy

Complete guide for deploying Postiz to your Dokploy server at `dokploy.apsion.com`.

**Target URL**: https://postiz.apsion.com

---

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Preparation](#preparation)
3. [DNS Configuration](#dns-configuration)
4. [Dokploy Setup](#dokploy-setup)
5. [Environment Configuration](#environment-configuration)
6. [Deployment](#deployment)
7. [Verification](#verification)
8. [Post-Deployment](#post-deployment)
9. [Troubleshooting](#troubleshooting)
10. [Maintenance](#maintenance)

---

## Prerequisites

### Required

- ✅ Access to Dokploy server at https://dokploy.apsion.com
- ✅ Domain `postiz.apsion.com` with DNS access
- ✅ Resend account for email delivery
- ✅ OAuth apps created for desired social platforms

### Optional

- Cloudflare R2 bucket (for cloud storage - local storage is default and works great for 2 users)

### Recommended

- Git repository with Postiz code (for auto-deploy)
- SSH access to Dokploy server (for troubleshooting)
- Database backup strategy

---

## Preparation

### 1. Generate Secure Secrets

```bash
# Generate PostgreSQL password
openssl rand -base64 32

# Generate JWT secret
openssl rand -base64 64
```

**Save these values** - you'll need them for the environment configuration.

### 2. Create Production Environment File

```bash
# Copy the template
cp .env.production.template .env.production

# Edit with your values
# DO NOT commit this file to git!
```

Fill in all required values in `.env.production`:

**Critical values to update:**
- `POSTGRES_PASSWORD` - Use generated password from step 1
- `JWT_SECRET` - Use generated secret from step 1
- `DATABASE_URL` - Update password to match POSTGRES_PASSWORD
- `RESEND_API_KEY` - Your Resend API key
- `EMAIL_FROM_ADDRESS` - Your sending email address
- Social platform OAuth credentials (as needed)

**Optional (defaults to local storage):**
- `CLOUDFLARE_*` - Your R2 credentials if using cloud storage (6 variables)

### 3. Verify Docker Compose Configuration

```bash
# Test the docker-compose file syntax
docker compose -f docker-compose.prod.yaml config

# This should output the processed configuration without errors
```

---

## DNS Configuration

### 1. Get Dokploy Server IP

If you don't know your Dokploy server IP:

```bash
# Option 1: Ping the Dokploy domain
ping dokploy.apsion.com

# Option 2: DNS lookup
nslookup dokploy.apsion.com

# Option 3: Check with your hosting provider
```

### 2. Create DNS Record

In your DNS provider (where you manage apsion.com):

**Create A Record:**
- **Type**: A
- **Name**: `postiz` (or `postiz.apsion.com` depending on provider)
- **Value**: Your Dokploy server IP address
- **TTL**: 300 (or default)

**Alternative - CNAME Record:**
- **Type**: CNAME
- **Name**: `postiz`
- **Value**: `dokploy.apsion.com`
- **TTL**: 300 (or default)

### 3. Verify DNS Propagation

```bash
# Check DNS resolution
nslookup postiz.apsion.com

# Should return your Dokploy server IP
# May take 5-30 minutes to propagate
```

**Wait for DNS to propagate** before proceeding to Dokploy setup.

---

## Dokploy Setup

### 1. Login to Dokploy

1. Navigate to https://dokploy.apsion.com
2. Enter your credentials
3. Access the dashboard

### 2. Create New Project

1. Click **"Create Project"** (or Projects → New Project)
2. Fill in:
   - **Name**: `Postiz`
   - **Description**: `Social Media Scheduling Platform`
3. Click **"Create"**

### 3. Add Docker Compose Service

Within your new project:

1. Click **"Add Service"** → **"Docker Compose"**
2. Configure source:

**Option A: Git Repository (Recommended)**
- **Source Type**: Git Repository
- **Repository URL**: Your Postiz git repo URL
- **Branch**: `main`
- **Compose File Path**: `docker-compose.prod.yaml`
- Enable **Auto Deploy** (optional - deploys on git push)

**Option B: Raw Compose**
- **Source Type**: Raw Compose
- Paste contents of `docker-compose.prod.yaml`

3. Click **"Next"** or **"Create"**

---

## Environment Configuration

### 1. Configure Environment Variables

In your Dokploy service:

1. Navigate to **Environment** tab
2. Use the built-in editor to add variables

**Copy and paste from your `.env.production` file**.

**Critical variables to verify:**

```bash
# Database (internal docker network)
DATABASE_URL=postgresql://postiz:YOUR_PASSWORD@postiz-postgres:5432/postiz
REDIS_URL=redis://postiz-redis:6379
POSTGRES_PASSWORD=YOUR_PASSWORD

# Security
JWT_SECRET=YOUR_LONG_RANDOM_STRING

# URLs
FRONTEND_URL=https://postiz.apsion.com
NEXT_PUBLIC_BACKEND_URL=https://postiz.apsion.com/api
BACKEND_INTERNAL_URL=http://localhost:3000

# Storage (local storage - files in Docker volume)
STORAGE_PROVIDER=local
UPLOAD_DIRECTORY=/uploads

# Optional: Cloudflare R2 (comment out if using local storage)
#CLOUDFLARE_ACCOUNT_ID=your-account-id
#CLOUDFLARE_ACCESS_KEY=your-access-key
#CLOUDFLARE_SECRET_ACCESS_KEY=your-secret-key
#CLOUDFLARE_BUCKETNAME=postiz-media
#CLOUDFLARE_BUCKET_URL=https://postiz-media.your-account-id.r2.cloudflarestorage.com
#CLOUDFLARE_REGION=auto

# Email
RESEND_API_KEY=re_your_resend_key
EMAIL_FROM_ADDRESS=noreply@apsion.com
EMAIL_FROM_NAME=Postiz

# System
IS_GENERAL=true
NX_ADD_PLUGINS=false

# Add all your social platform OAuth credentials
X_API_KEY=...
LINKEDIN_CLIENT_ID=...
# etc.
```

3. Click **"Save"**

**Note**: Don't configure the domain yet - you need to deploy first to create the services!

---

## Deployment

### 1. Initial Deploy

1. In your Dokploy service, click **"Deploy"**
2. Monitor the deployment logs in real-time

**Expected build process:**
```
1. Pulling code from git (if using git source)
2. Building docker image from Dockerfile.dev
   - Installing Node.js dependencies (pnpm install)
   - Generating Prisma client
   - Building all apps (backend, frontend, workers, cron)
   - This takes 5-10 minutes
3. Starting containers
   - postiz-postgres (PostgreSQL)
   - postiz-redis (Redis)
   - postiz-app (main application)
4. Running health checks
5. Starting PM2 processes (backend, frontend, workers, cron)
```

**Wait for**: "Deployment completed successfully" or "PM2 successfully started"

### 2. Monitor Build Logs

Watch for these key messages:

✅ `pnpm install` completes
✅ `prisma generate` completes
✅ `pnpm run build` completes
✅ `nginx started`
✅ `PM2 process manager started`
✅ All health checks passing

🚫 **If you see errors**, check the [Troubleshooting](#troubleshooting) section.

### 3. Configure Domain (After Initial Deploy)

Once the initial deployment completes successfully:

1. Navigate to **Domains** tab
2. Click **"Add Domain"**
3. Fill in:
   - **Service**: Select `postiz-app` (from dropdown - should now be available)
   - **Host**: `postiz.apsion.com`
   - **Container Port**: `5000`
   - **HTTPS**: ✅ Enabled
   - **Certificate**: `letsencrypt`
   - **Path**: `/` (leave default)
4. Click **"Add"** or **"Save"**

**Important Notes**:
- The `postiz-app` service must be selected - this is the container running nginx on port 5000
- Don't select `postiz-postgres` or `postiz-redis` - those are internal services only
- If the Service dropdown is empty, go back and ensure the deployment completed successfully

### 4. Wait for SSL Certificate

- Let's Encrypt certificate issuance takes 1-2 minutes after domain is added
- Dokploy will show certificate status in Domains tab
- You may see "Certificate Pending" initially - this is normal
- Once issued, you'll see a green checkmark or "Valid" status

---

## Verification

### 1. Check Service Health

In Dokploy:

1. Navigate to **Monitoring** tab
2. Verify all containers are running:
   - ✅ `postiz-app` - Running
   - ✅ `postiz-postgres` - Running (healthy)
   - ✅ `postiz-redis` - Running (healthy)

### 2. Check Application Logs

1. Navigate to **Logs** tab
2. Select **postiz-app** container
3. Look for PM2 process messages:
   ```
   PM2 [backend] started
   PM2 [frontend] started
   PM2 [workers] started
   PM2 [cron] started
   ```

### 3. Access Application

1. Open browser to: https://postiz.apsion.com
2. You should see:
   - ✅ Valid SSL certificate (green lock icon)
   - ✅ Postiz login/registration page
   - ✅ No browser console errors

**If you see errors**, check the [Troubleshooting](#troubleshooting) section.

### 4. Test Database Connection

From the application homepage, try to register an account:

1. Click **"Sign Up"** or **"Register"**
2. Fill in user details
3. Submit form

**Expected behavior:**
- ✅ Registration succeeds
- ✅ Email sent (if RESEND_API_KEY configured)
- ✅ User can login (after email verification or auto-activation)

This confirms:
- Database connection working
- Backend API working
- Frontend routing working
- Email delivery working (if configured)

---

## Post-Deployment

### 1. Create Admin Account

1. Navigate to https://postiz.apsion.com
2. Click **"Register"** or **"Sign Up"**
3. Create account with your email
4. Verify email (check inbox for Resend email)
5. Login with credentials

**Note**: First user is automatically admin.

### 2. Configure Application Settings

After logging in:

1. Complete onboarding flow (if shown)
2. Create an organization
3. Configure organization settings:
   - Name
   - Logo (tests Cloudflare R2 upload)
   - Timezone
   - Language

### 3. Test Social Platform Connections

1. Navigate to **Integrations** or **Channels**
2. Click **"Add Channel"** or **"Connect"**
3. Select a social platform (e.g., X/Twitter)
4. Follow OAuth flow
5. Verify connection successful

This tests:
- OAuth credentials working
- Callback URLs configured correctly
- Social platform API integration

### 4. Test Post Scheduling

1. Create a test post
2. Add content and media (tests Cloudflare R2 upload)
3. Select connected channel
4. Schedule for future time
5. Verify:
   - Post appears in scheduled posts
   - Workers process the job (check logs)
   - Post publishes at scheduled time

### 5. Setup Monitoring (Optional)

If using Sentry:

1. Add environment variables:
   - `NEXT_PUBLIC_SENTRY_DSN`
   - `SENTRY_DSN`
2. Redeploy application (click "Deploy" in Dokploy)
3. Verify errors appear in Sentry dashboard

### 6. Configure Backups

In Dokploy:

1. Navigate to your service → **Volumes** tab
2. Find `postgres-data` volume
3. Click **"Backup Settings"**
4. Configure:
   - Backup frequency: Daily
   - Retention: 7 days
   - Destination: S3 compatible storage
5. Run manual backup to test

---

## Troubleshooting

### Build Failures

#### Error: "Out of memory during build"

**Symptom**: Build fails with Node.js heap errors

**Solution**:
- Dockerfile already uses `--max-old-space-size=4096` (4GB)
- Ensure Dokploy server has **at least 6GB RAM**
- Check Dokploy resource limits aren't restricting container

#### Error: "Prisma client generation failed"

**Symptom**: Build fails during `pnpm install` or `prisma generate`

**Solution**:
1. Verify `DATABASE_URL` in environment variables
2. Ensure PostgreSQL container is accessible
3. Check Prisma schema is valid: `libraries/nestjs-libraries/src/database/prisma/schema.prisma`

#### Error: "Build failed - package not found"

**Symptom**: pnpm can't find package or dependency resolution fails

**Solution**:
1. Clear Docker build cache in Dokploy
2. Redeploy
3. If using git source, verify code is up to date

### Runtime Failures

#### Error: "Can't connect to database"

**Symptom**: Backend logs show PostgreSQL connection errors

**Solution**:
1. Verify `DATABASE_URL` uses `postiz-postgres` (container name), NOT `localhost`
2. Correct: `postgresql://postiz:password@postiz-postgres:5432/postiz`
3. Wrong: `postgresql://postiz:password@localhost:5432/postiz`
4. Check password matches `POSTGRES_PASSWORD`
5. Verify PostgreSQL container is healthy (Monitoring tab)

#### Error: "Can't connect to Redis"

**Symptom**: Workers or backend can't connect to Redis

**Solution**:
1. Verify `REDIS_URL` uses `postiz-redis` (container name), NOT `localhost`
2. Correct: `redis://postiz-redis:6379`
3. Wrong: `redis://localhost:6379`
4. Verify Redis container is healthy (Monitoring tab)

#### Error: "Frontend shows 404 or blank page"

**Symptom**: https://postiz.apsion.com shows error or doesn't load

**Solution**:
1. Check nginx is running: `docker exec postiz-app ps aux | grep nginx`
2. Check frontend is running: `docker exec postiz-app pm2 status`
3. Verify port 5000 is exposed in docker-compose
4. Check Traefik routing in Dokploy Domains tab
5. Wait 1-2 minutes for Traefik to reload configuration

#### Error: "Backend API returns 502 or 504"

**Symptom**: Frontend loads but API calls fail

**Solution**:
1. Verify `NEXT_PUBLIC_BACKEND_URL=https://postiz.apsion.com/api`
2. Check backend is running: `docker exec postiz-app pm2 logs backend`
3. Verify nginx reverse proxy config (should be automatic)
4. Check backend health: `curl http://localhost:5000/api/` from server

### Deployment Issues

#### Error: "Service is required" when adding domain

**Symptom**: Dokploy shows "Services not found" or requires service selection when adding domain

**Solution**:
1. This is specific to Docker Compose deployments (multiple services)
2. You must select **which service** the domain routes to
3. Select `postiz-app` from the Service dropdown
4. The service name comes from `docker-compose.prod.yaml`:
   ```yaml
   services:
     postiz-app:  # <-- This is the service name to select
       ...
   ```
5. Do NOT select `postiz-postgres` or `postiz-redis` - those are internal services
6. If dropdown is empty, the compose deployment hasn't started yet - deploy first, then add domain

#### Error: "Domain shows 404 or 502"

**Symptom**: https://postiz.apsion.com returns error

**Solution**:
1. Wait 1-2 minutes for Traefik to update routes
2. Verify domain configuration in Dokploy points to port `5000`
3. Check container port is correctly exposed: `docker ps | grep postiz-app`
4. Verify DNS is resolving to correct IP: `nslookup postiz.apsion.com`

#### Error: "SSL certificate not issued"

**Symptom**: Certificate shows as "Pending" or browser shows SSL error

**Solution**:
1. Verify DNS is pointing to Dokploy server IP
2. Ensure domain is accessible via HTTP first
3. Check Let's Encrypt logs in Dokploy
4. Let's Encrypt may take 1-2 minutes - be patient
5. If still failing after 5 minutes:
   - Verify port 80 and 443 are open on server firewall
   - Check domain isn't rate limited by Let's Encrypt
   - Try removing and re-adding domain in Dokploy

### Application Issues

#### Error: "Email not sending"

**Symptom**: Users don't receive verification emails

**Solution**:
1. Verify `RESEND_API_KEY` is set and correct
2. Check `EMAIL_FROM_ADDRESS` is verified in Resend
3. Check backend logs for email errors: `docker exec postiz-app pm2 logs backend | grep -i email`
4. Verify Resend API key has correct permissions
5. Check Resend dashboard for bounce/delivery logs

#### Error: "File upload fails"

**Symptom**: Can't upload images, avatars, or media

**Solution for local storage** (default):
1. Verify `STORAGE_PROVIDER=local`
2. Verify `UPLOAD_DIRECTORY=/uploads`
3. Check uploads volume is mounted in docker-compose
4. Verify container has write permissions to /uploads
5. Check disk space: `df -h`

**Solution for Cloudflare R2** (if enabled):
1. Verify all 6 Cloudflare environment variables are set:
   - `CLOUDFLARE_ACCOUNT_ID`
   - `CLOUDFLARE_ACCESS_KEY`
   - `CLOUDFLARE_SECRET_ACCESS_KEY`
   - `CLOUDFLARE_BUCKETNAME`
   - `CLOUDFLARE_BUCKET_URL`
   - `CLOUDFLARE_REGION=auto`
2. Verify `STORAGE_PROVIDER=cloudflare`
3. Check R2 bucket exists and is accessible
4. Verify R2 API keys have correct permissions (read + write)
5. Ensure you're using a SEPARATE bucket (not your backup bucket)
6. Check backend logs for R2 errors

#### Error: "Social platform OAuth fails"

**Symptom**: Can't connect social accounts

**Solution**:
1. Verify OAuth credentials are set for that platform
2. Check OAuth app callback URL is set to: `https://postiz.apsion.com/api/integration/social/{platform}/callback`
3. Verify OAuth app is active/approved in platform developer console
4. Check backend logs for OAuth errors
5. Common issues:
   - Callback URL mismatch
   - App not approved by platform
   - Credentials for wrong environment (dev vs prod)

#### Error: "Workers not processing jobs"

**Symptom**: Scheduled posts don't publish

**Solution**:
1. Check workers are running: `docker exec postiz-app pm2 logs workers`
2. Verify Redis connection (workers use Redis for queue)
3. Check worker logs for errors
4. Restart workers: `docker exec postiz-app pm2 restart workers`
5. Verify jobs are in Redis: `docker exec postiz-redis redis-cli KEYS "*"`

### Performance Issues

#### Issue: "Application is slow"

**Solution**:
1. Check resource usage in Dokploy Monitoring tab
2. Verify sufficient RAM (recommended: 4GB minimum)
3. Check CPU usage - may need to increase limits
4. Review database query performance
5. Consider scaling:
   - Increase container resources
   - Add database indexes
   - Use Redis caching more aggressively

#### Issue: "Database queries slow"

**Solution**:
1. Check PostgreSQL logs: `docker exec postiz-postgres tail -f /var/log/postgresql/postgresql.log`
2. Run `VACUUM ANALYZE` on database
3. Check disk I/O performance
4. Consider increasing PostgreSQL resources
5. Review slow query logs

---

## Maintenance

### Updating Application

#### Option 1: Git Auto-Deploy (Recommended)

If you enabled Auto Deploy:

1. Push changes to git repository
2. Dokploy automatically detects changes
3. Triggers new deployment
4. Application updates with zero downtime (via health checks)

#### Option 2: Manual Redeploy

1. Navigate to your service in Dokploy
2. Click **"Deploy"** button
3. Monitor deployment logs
4. Verify new version deployed successfully

### Database Backups

#### Manual Backup

```bash
# Backup database to file
docker exec postiz-postgres pg_dump -U postiz postiz > backup_$(date +%Y%m%d).sql

# Compress backup
gzip backup_$(date +%Y%m%d).sql
```

#### Restore from Backup

```bash
# Decompress
gunzip backup_20250101.sql.gz

# Restore
docker exec -i postiz-postgres psql -U postiz postiz < backup_20250101.sql
```

#### Automated Backups

Use Dokploy's built-in backup feature:

1. Navigate to service → **Volumes** tab
2. Find `postgres-data` volume
3. Configure S3-compatible backup destination
4. Set schedule (e.g., daily at 2 AM)
5. Set retention policy (e.g., keep 7 days)

### Uploads Backup (Local Storage)

If using local file storage, back up the uploads volume:

**Option 1: Using Dokploy (Recommended)**
1. Navigate to service → **Volumes** tab
2. Find `uploads` volume
3. Configure backup (same as database backups)

**Option 2: Manual Backup**
```bash
# First, find the actual volume name
docker volume ls | grep uploads
# Example output: postiz_uploads or similar

# Backup uploads to tar file (replace VOLUME_NAME with actual name)
docker run --rm -v VOLUME_NAME:/data -v $(pwd):/backup alpine tar czf /backup/uploads_$(date +%Y%m%d).tar.gz -C /data .

# Restore uploads from tar file
docker run --rm -v VOLUME_NAME:/data -v $(pwd):/backup alpine tar xzf /backup/uploads_20250101.tar.gz -C /data
```

### Monitoring

#### View Application Logs

```bash
# All PM2 processes
docker exec postiz-app pm2 logs

# Specific process
docker exec postiz-app pm2 logs backend
docker exec postiz-app pm2 logs frontend
docker exec postiz-app pm2 logs workers
docker exec postiz-app pm2 logs cron

# Follow logs in real-time
docker exec postiz-app pm2 logs --lines 100
```

#### View Database Logs

```bash
# PostgreSQL logs
docker logs postiz-postgres --tail 100 --follow

# Redis logs
docker logs postiz-redis --tail 100 --follow
```

#### Monitor Resources

Use Dokploy dashboard:
- CPU usage per container
- Memory usage per container
- Disk usage
- Network I/O

Set up alerts for:
- High CPU usage (>80%)
- High memory usage (>80%)
- Disk space low (<10% free)

### Scaling

#### Vertical Scaling (Increase Resources)

1. Navigate to service → **Advanced** tab
2. Adjust:
   - **CPU**: Increase cores
   - **Memory**: Increase RAM limit
3. Redeploy service

#### Horizontal Scaling (Multiple Instances)

For production with high traffic:

1. **Separate workers**:
   - Create dedicated service for workers
   - Point to same PostgreSQL and Redis
   - Scale workers independently

2. **Database scaling**:
   - Use external managed PostgreSQL (AWS RDS, etc.)
   - Enable connection pooling
   - Consider read replicas

3. **Redis scaling**:
   - Use external managed Redis (AWS ElastiCache, etc.)
   - Enable Redis cluster mode

### Security Updates

#### Update Dependencies

```bash
# Update all npm dependencies
pnpm update --latest

# Update specific package
pnpm update package-name

# Rebuild and redeploy
git commit -am "chore: update dependencies"
git push
```

#### Update Base Images

Rebuild with latest base images:

1. Dockerfile uses `node:22.20-alpine`
2. Docker Compose uses `postgres:17-alpine` and `redis:7-alpine`
3. These auto-update when you rebuild

Force rebuild:
```bash
# In Dokploy, click "Rebuild" instead of "Deploy"
# This ignores cache and pulls latest base images
```

### Rollback

#### Rollback to Previous Deployment

In Dokploy:

1. Navigate to **Deployments** tab
2. View deployment history
3. Click **"Rollback"** on a previous successful deployment
4. Confirm rollback

#### Emergency Restore

If deployment is completely broken:

1. Stop current deployment
2. Restore database from backup (see Database Backups)
3. Deploy last known-good version
4. Verify application working
5. Investigate what went wrong

---

## Advanced Configuration

### Environment Variable Management

For different environments (staging, production):

1. Create separate projects in Dokploy
2. Use different environment files
3. Maintain separate databases

### Custom Domain Setup

To add additional domains (e.g., www.postiz.apsion.com):

1. Add DNS record pointing to Dokploy server
2. In Dokploy service → **Domains** tab
3. Click **"Add Domain"**
4. Add: `www.postiz.apsion.com`
5. Same settings as main domain

### Webhook Integration

Setup automated deployments:

1. Navigate to service → **Settings** tab
2. Copy webhook URL
3. Add to git repository:
   - GitHub: Settings → Webhooks → Add webhook
   - GitLab: Settings → Webhooks → Add webhook
   - Bitbucket: Settings → Webhooks → Add webhook
4. Trigger: On push to main branch
5. Now git pushes auto-deploy

### Health Check Customization

Modify health checks in `docker-compose.prod.yaml`:

```yaml
healthcheck:
  test: ['CMD', 'wget', '--quiet', '--tries=1', '--spider', 'http://localhost:5000/api/health']
  interval: 30s      # How often to check
  timeout: 10s       # How long to wait for response
  retries: 3         # How many failures before unhealthy
  start_period: 90s  # Grace period on startup
```

---

## Support & Resources

### Documentation

- **Postiz Official Docs**: https://docs.postiz.com/
- **Dokploy Documentation**: https://docs.dokploy.com/
- **Docker Compose Reference**: https://docs.docker.com/compose/

### Getting Help

If you encounter issues not covered in this guide:

1. Check Dokploy logs for detailed error messages
2. Review application logs via PM2
3. Search Postiz GitHub issues
4. Join Postiz Discord community (if available)
5. Contact Dokploy support

### Useful Commands

```bash
# Check all running containers
docker ps

# View container resource usage
docker stats

# Execute command in container
docker exec postiz-app <command>

# View PM2 process status
docker exec postiz-app pm2 status

# Restart all PM2 processes
docker exec postiz-app pm2 restart all

# Database connection test
docker exec postiz-postgres psql -U postiz -c "SELECT 1"

# Redis connection test
docker exec postiz-redis redis-cli ping

# View disk usage
df -h

# View memory usage
free -h
```

---

## Checklist: Successful Deployment

Use this checklist to verify your deployment:

- [ ] DNS resolves to Dokploy server
- [ ] HTTPS certificate issued (green lock)
- [ ] Application loads at https://postiz.apsion.com
- [ ] Can register new user account
- [ ] Email verification working (if enabled)
- [ ] Can login successfully
- [ ] Can create organization
- [ ] Can upload images (Cloudflare R2 working)
- [ ] Can connect social account (OAuth working)
- [ ] Can create post
- [ ] Can schedule post
- [ ] Workers processing jobs
- [ ] Cron tasks running
- [ ] Database backups configured
- [ ] Monitoring/alerts setup (if using)
- [ ] All 4 PM2 processes running (backend, frontend, workers, cron)
- [ ] PostgreSQL container healthy
- [ ] Redis container healthy

---

## Next Steps

After successful deployment:

1. 🎉 **Celebrate!** Your Postiz instance is live.
2. 📊 **Monitor** application performance and logs
3. 🔒 **Security** - Review and harden configuration
4. 📧 **Users** - Invite your 2 users to register
5. 🔗 **Integrate** - Connect all desired social platforms
6. 📅 **Schedule** - Start creating and scheduling posts
7. 💾 **Backup** - Verify automated backups working
8. 📈 **Optimize** - Monitor and tune performance as needed

**Enjoy your self-hosted Postiz platform!** 🚀
