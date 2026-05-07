# Deployment

This doc covers the day-to-day deploy loop for CoreX WMS (production host:
`corexwms.com`, EC2 instance running the `corex-wms` systemd service under
`/opt/corex-wms`).

The first-time bootstrap of a fresh instance is still done with
`deploy/deploy.sh`'s older sibling logic (nginx, systemd, Postgres). This doc
is about *redeploys* -- pulling new code and restarting the service.

---

## 1. Manual deploy (current workflow)

SSH or EC2 Instance Connect into the box, then:

```bash
bash /opt/corex-wms/deploy/deploy.sh
```

What it does:

1. `git stash` any local edits to `deploy/` so server-side tweaks aren't lost.
2. `git pull origin main`.
3. `git stash pop` to restore those local edits (no-op if nothing was stashed).
4. `pip install -r requirements.txt` under Python 3.11.
5. `systemctl restart corex-wms`.
6. Polls `https://corexwms.com/api/health` up to 5 times (4s apart). Exits 0
   on HTTP 200, exits 1 otherwise and prints how to read the journal.

The script is idempotent. Running it when there's nothing new to pull just
re-installs deps and restarts the service.

---

## 2. GitHub Actions auto-deploy (opt-in)

`.github/workflows/deploy.yml` can SSH into the instance and run
`deploy.sh` for you. It is currently **manual-only** (`workflow_dispatch`) --
see the "EC2 Instance Connect caveat" below for why.

### Secrets to set

In the repo on GitHub: **Settings -> Secrets and variables -> Actions ->
New repository secret**. Add:

| Name          | Value                                                                 |
|---------------|-----------------------------------------------------------------------|
| `EC2_SSH_KEY` | The full PEM private key (including BEGIN/END lines) for the deploy user. |
| `EC2_HOST`    | `corexwms.com` (or the instance's public DNS).                         |
| `EC2_USER`    | `ec2-user`                                                             |

### Enabling real SSH on the instance

The workflow uses static-key SSH. If the instance is reached today only via
EC2 Instance Connect (short-lived keys pushed through the AWS API), static
SSH won't work until you:

1. Generate a keypair locally:
   ```bash
   ssh-keygen -t ed25519 -f ~/.ssh/corex_deploy -C "gha-deploy"
   ```
2. On the EC2 instance, append the public key to the deploy user:
   ```bash
   cat ~/.ssh/corex_deploy.pub | ssh ec2-user@<host> \
       "cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
   ```
3. Confirm the security group allows inbound TCP 22 from
   `0.0.0.0/0` (or, better, from GitHub Actions' published IP ranges).
4. Paste the contents of `~/.ssh/corex_deploy` (the private key) into the
   `EC2_SSH_KEY` secret. Keep the local copy off GitHub.

### Flipping auto-deploy on

Once the three secrets exist and you've done one successful manual run of
the workflow (Actions tab -> "Deploy to EC2" -> Run workflow), uncomment the
`push: branches: [main]` block at the top of `deploy.yml`. Every merge to
`main` will then redeploy.

### Why it's manual-only right now

Leaving the `push` trigger off means a missing secret or a misconfigured
security group doesn't spam red X's on every commit. The workflow is there,
ready, the moment SSH is set up.

---

## 3. Rolling back

Every deploy lands one or more commits on `main`. To roll back on the
instance:

```bash
cd /opt/corex-wms
git log --oneline -n 10            # find the last known-good SHA
git checkout <good-sha>
sudo python3.11 -m pip install -r requirements.txt -q
sudo systemctl restart corex-wms
curl -s -o /dev/null -w "%{http_code}\n" https://corexwms.com/api/health
```

To un-break `main` itself, do the revert in your normal git workflow
(`git revert <bad-sha>`, PR, merge). Once `main` is clean, run `deploy.sh`
again and the instance fast-forwards off the detached HEAD back onto
`origin/main`.

### If `git pull` refuses because of local edits

`deploy.sh` already stashes `deploy/` for you. If something *else* is dirty
on the server, stash it yourself first:

```bash
git status
git stash push -m "manual-save-$(date +%s)"
bash /opt/corex-wms/deploy/deploy.sh
git stash list   # your changes are still there
```

---

## 4. Troubleshooting

- **Health check fails after restart.**
  ```bash
  sudo journalctl -u corex-wms -n 200 --no-pager
  ```
  Most commonly: a new env var is required and `.env` wasn't updated, or a
  new migration hasn't run.

- **`pip install` fails.** Check the EC2 instance's free disk:
  `df -h /`. The `/opt` volume fills up with wheels sometimes.

- **nginx returns 502.** The app process exited. Same journal command as
  above; nginx itself is healthy in this scenario.
