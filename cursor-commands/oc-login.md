---
description: Login to OpenShift cluster by name from .env.local
---

Login to the OpenShift cluster named "${{CLUSTER_NAME}}" using credentials from `.env.local`.

First, show available clusters by listing the cluster section comments. Each section starts with a `# Name` comment line followed by `OC_URL` (possibly commented out):
```bash
grep -B1 'OC_URL' /Users/dipgupta/Documents/odh/odh-dashboard/.env.local | grep '^#' | grep -v 'OC_URL' | sed 's/^# //'
```

Then, extract credentials for the chosen cluster and login. The cluster section starts with a comment like "# CLUSTER_NAME" followed by the OC_URL, and optionally OC_USER/OC_TOKEN/OC_PASSWORD/OC_LOGIN lines.

Parse the .env.local to find the section matching "${{CLUSTER_NAME}}" (case-insensitive), extract the credentials from the lines immediately following that section header, and run the oc login command.

If the section has `OC_LOGIN=web`, use: `oc login --web --server="$URL"`
If the credentials use OC_TOKEN, use: `oc login "$URL" --token="$TOKEN" --insecure-skip-tls-verify`
If the credentials use OC_USER/OC_PASSWORD, use: `oc login "$URL" -u "$USER" -p "$PASSWORD" --insecure-skip-tls-verify`

Confirm when the login is successful and show which cluster you're connected to.
