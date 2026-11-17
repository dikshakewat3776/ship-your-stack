# Node.js (Express) on a VPS with Jenkins, PM2, and zero-downtime releases

Deploy an **Express** (or similar Node HTTP) app using a **Capistrano-style** layout: timestamped releases under `releases/`, a **`current` symlink**, **shared** data (logs, uploads, public), **PM2 cluster** mode, and a **Jenkins** scripted pipeline that atomically switches releases, reloads PM2, runs a **health check**, and **rolls back** on failure.

---

## Architecture

| Piece | Role |
|--------|------|
| Jenkins agent | Checkout, build, copy to release dir, switch symlink, PM2 reload |
| `releases/<timestamp>/` | Immutable deploy artifact per release (`npm ci`, app code) |
| `current` → release | Symlink PM2 `cwd` reads; switch is atomic (`ln -sfn`) |
| `shared/` | Persistent `logs`, `uploads`, `public` symlinked into each release |
| PM2 | Cluster mode on `./bin/www` (or your entry script) |
| Secrets (on server) | Split env files loaded into PM2 env (not in git) |

Typical base path on the server:

```
/var/lib/jenkins/deployments/my-app/
├── releases/
│   ├── 20250317143022/
│   └── ...
├── shared/
│   ├── logs/
│   ├── uploads/
│   └── public/
├── current -> releases/20250317143022/
└── ecosystem.config.js   # generated or maintained per deploy
```

Secrets live outside the repo, e.g. `/opt/my-app/secrets/`:

- `db.env`
- `application.env`
- `vendor.env`

(Adjust file names; provide `*.env.template` in the repo for operators.)

---

## Prerequisites

**Jenkins**

- Agent with **Node.js**, **npm**, **PM2**, **curl**, **rsync**, and git.
- Credentials / network access to your **Git** remote.
- Pipeline job with parameters: `BRANCH_NAME`, `COMMIT_ID` (optional SHA), `DEPLOY_ENV` (e.g. `qa`, `staging`, `production`).

**Application**

- Express (or compatible) entry: common pattern **`./bin/www`** bootstrapping `app`.
- `package-lock.json` for **`npm ci --omit=dev`**.
- Health route returning **200** (e.g. `GET http://localhost:8090/api/v1/health`).
- Optional: `npm test`, `npm run eslint` (pipeline may treat failures as warnings).

**Server paths** (customize in pipeline):

| Variable | Example |
|----------|---------|
| `deployBase` | `/var/lib/jenkins/deployments/my-app` |
| `secretsBase` | `/opt/my-app/secrets` |
| `pm2AppName` | `my-app` |
| `healthCheckUrl` | `http://localhost:8090/api/v1/health` |
| `clusterInstances` | `2` |
| `maxReleases` | `5` |

**Branch policy:** Enforce production deploys only from `master` or `main` (match your Git default).

---

## Pipeline stages (overview)

1. **Validate parameters** — env + branch rules for production.
2. **Checkout** — Git SCM; optional fixed `COMMIT_ID` or branch tip; record deployed SHA.
3. **Setup Node** — verify `node` / `npm` versions.
4. **Setup shared** — `mkdir -p shared/{logs,uploads,public}`.
5. **Create release** — `releases/YYYYMMDDHHMMSS`; remember **previous** release name from `current` for rollback.
6. **Copy code** — `rsync` workspace → release (exclude `node_modules`, `logs`; exclude `.git` in prod if desired).
7. **Install** — `npm pkg delete scripts.prepare` (avoids **husky** in prod), then **`npm ci --omit=dev`**.
8. **Tests / lint** — optional; non-blocking in many setups (`|| true`).
9. **Link shared** — symlink `logs`, `uploads`, `public` into release.
10. **Validate secrets** — warn or fail if env files missing under `secretsBase`.
11. **PM2 ecosystem** — write `ecosystem.config.js` loading secret files into `env`, set `NODE_ENV`, `NODE_CONFIG_DIR` to `current` if using [node-config](https://github.com/node-config/node-config).
12. **Atomic switch** — `ln -sfn <newRelease> current`.
13. **Zero-downtime reload** — `set +x`; source secret files; `pm2 reload ecosystem.config.js --update-env` (or `pm2 start` first time).
14. **Health check** — after sleep, `curl` health URL; if not **200**, **repoint `current`** to previous release, `pm2 reload`, delete failed release, **fail the build**.
15. **Cleanup** — keep last **N** releases; delete older directories.

---

## PM2 ecosystem pattern

Generated at deploy time so **`cwd`** is always `${deployBase}/current` and secrets merge into `env`:

```js
const fs = require('fs');
const path = require('path');

const secretsDir = '/opt/my-app/secrets';
const envFiles = ['db.env', 'application.env', 'vendor.env'];
const secretEnv = {};

envFiles.forEach((file) => {
  const filePath = path.join(secretsDir, file);
  if (!fs.existsSync(filePath)) return;
  fs.readFileSync(filePath, 'utf8')
    .split('\n')
    .filter((line) => line.trim() && !line.startsWith('#'))
    .forEach((line) => {
      const idx = line.indexOf('=');
      if (idx <= 0) return;
      let val = line.substring(idx + 1).trim();
      if (
        (val.startsWith('"') && val.endsWith('"')) ||
        (val.startsWith("'") && val.endsWith("'"))
      ) {
        val = val.slice(1, -1);
      }
      secretEnv[line.substring(0, idx).trim()] = val;
    });
});

module.exports = {
  apps: [
    {
      name: 'my-app',
      script: './bin/www',
      cwd: '/var/lib/jenkins/deployments/my-app/current',
      instances: '2',
      exec_mode: 'cluster',
      watch: false,
      env: Object.assign({}, secretEnv, {
        NODE_ENV: 'qa',
        NODE_CONFIG_DIR: '/var/lib/jenkins/deployments/my-app/current/config',
      }),
    },
  ],
};
```

Use **`pm2 reload`** with **`--update-env`** after symlink changes so workers pick up new code and env without a hard outage window.

---

## Jenkins pipeline (reference skeleton)

Save as `Jenkinsfile` (declarative-style stages are often **scripted** with `node { }` for this pattern). Replace placeholders:

```groovy
node {
    def appName = 'my-app'
    def deployBase = '/var/lib/jenkins/deployments/my-app'
    def releasesPath = "${deployBase}/releases"
    def sharedPath = "${deployBase}/shared"
    def currentLink = "${deployBase}/current"
    def ecosystemFile = "${deployBase}/ecosystem.config.js"
    def appRepoUrl = 'git@github.com:org/my-app.git'
    def branchName = params.BRANCH_NAME ?: 'main'
    def commitId = params.COMMIT_ID ?: ''
    def deployEnv = params.DEPLOY_ENV ?: 'qa'
    def pm2AppName = 'my-app'
    def secretsBase = '/opt/my-app/secrets'
    def maxReleases = 5
    def clusterInstances = 2
    def healthCheckUrl = 'http://localhost:8090/api/v1/health'
    def timestamp = ''
    def releasePath = ''
    def previousRelease = ''

    stage('Validate Parameters') {
        if (deployEnv == 'production' && branchName != 'main') {
            error("Production must deploy from 'main'")
        }
    }

    stage('Checkout') {
        dir('app-src') {
            checkout scm: [
                $class: 'GitSCM',
                branches: [[name: commitId?.trim() ? commitId : "*/${branchName}"]],
                userRemoteConfigs: [[url: appRepoUrl]]
            ]
        }
        if (!commitId?.trim()) {
            commitId = sh(script: 'cd app-src && git rev-parse HEAD', returnStdout: true).trim()
        }
        sh '''
            shopt -s dotglob
            cp -r app-src/* app-src/.* . 2>/dev/null || true
            rm -rf app-src
        '''
    }

    // ... Setup Node, Setup Shared, Create Release, Copy, npm ci, tests, link shared,
    //     validate secrets, write ecosystem, atomic symlink, pm2 reload ...

    stage('Health Check') {
        def healthStatus = sh(script: """
            sleep 15
            curl -s -o /dev/null -w "%{http_code}" ${healthCheckUrl}
        """, returnStdout: true).trim()
        if (healthStatus != '200') {
            if (previousRelease) {
                sh """
                    ln -sfn ${releasesPath}/${previousRelease} ${currentLink}
                    pm2 reload ${ecosystemFile} --update-env
                    rm -rf ${releasePath}
                """
                error("Health check failed; rolled back to ${previousRelease}")
            }
            error('Health check failed; no previous release')
        }
    }
}
```

In your real pipeline, copy the full stage bodies from your working job; the guide above documents **behavior**, not every `sh` block.

**Security:** Use `set +x` while **sourcing** secret files in shell so values do not print in Jenkins logs.

---

## Production install detail: Husky

Dev installs often define `"prepare": "husky install"`. **`npm ci --omit=dev`** does not install husky, so **delete `scripts.prepare`** before `npm ci` in the release directory:

```bash
npm pkg delete scripts.prepare
npm ci --omit=dev
```

---

## Manual rollback

```bash
ls -1t /var/lib/jenkins/deployments/my-app/releases/
ln -sfn /var/lib/jenkins/deployments/my-app/releases/<TIMESTAMP> \
        /var/lib/jenkins/deployments/my-app/current
pm2 reload /var/lib/jenkins/deployments/my-app/ecosystem.config.js --update-env
```

---

## Verification

1. Jenkins build green; summary shows release timestamp and commit SHA.  
2. `pm2 status my-app` — cluster instances online.  
3. Health URL returns **200** on the app port (and via Nginx if used).  
4. `readlink -f .../current` points at the new release directory.  
5. Logs and uploads under `shared/` persist across deploys.

---

## Troubleshooting

| Symptom | Likely cause |
|---------|----------------|
| Health check 502/000 after deploy | App not listening on expected port; secrets missing; crash on boot |
| Rollback loop | Health URL wrong path/port; app needs longer warm-up than `sleep 15` |
| `husky install` failure on CI | `prepare` script not removed before `npm ci --omit=dev` |
| PM2 serves old code | Symlink not updated before reload; `cwd` not using `current` |
| Missing env at runtime | Secret files absent or not parsed into ecosystem / shell `set -a` |
| Cluster memory spike | Too many `instances`; tune or use `max_memory_restart` in ecosystem |
| `.git` in prod release | Checkout rsync excludes `.git` for prod only — intentional for security |

---

## Security reminders

- Never store production secrets in the Git repo or Jenkins job config as plaintext if avoidable; use server-side files or a secret store.  
- Restrict Jenkins and SSH access to deploy paths and `/opt/.../secrets`.  
- Prefer removing `.git` from production releases.  
- Rotate credentials if they appeared in build logs.

---

## Quick checklist

1. `deployBase`, `shared/`, and `secretsBase` exist on the agent.  
2. Health endpoint matches `healthCheckUrl` and listens on localhost before Nginx.  
3. `ecosystem.config.js` uses `cwd: .../current` and cluster settings you expect.  
4. Pipeline keeps **N** releases and records **previous** release before symlink switch.  
5. Failed health check triggers symlink rollback and removes the bad release directory.