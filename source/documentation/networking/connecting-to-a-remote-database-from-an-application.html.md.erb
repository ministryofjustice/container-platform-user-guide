---
title: Connecting to a Remote Database From an Application
last_reviewed_on: 2026-10-09
review_in: 6 months
---

# <%= current_page.data.title %>

## Connecting from an application

To connect  your application to an RDS outside of the Container Platform, that RDS must be configured to allow the connection, see the relevant guide for your situation: 

- [Connecting to a Cloud Platform database](/documentation/networking/connecting-to-a-cloud-platform-database.html)
- [Connecting to a Modernisation Platform database](/documentation/networking/connecting-to-a-modernisation-platform-database.html)

### Credentials

> **This is an interim approach, not a finished pattern.**
>
> The Container Platform does not yet provide a secrets management mechanism. There is no
> External Secrets Operator and no sealed secrets, so credentials cannot be held in Git and
> have to be created directly against the cluster.
>
> The team is working on a proper mechanism. What follows is a placeholder to unblock you
> until that lands, and it will be replaced when it does.
>
> **Please talk to us in
> [#container-platform-alpha-users](https://moj.enterprise.slack.com/archives/C0BKZHMQNRK)
> before you start**, whether you want to use the approach below, have a different one in
> mind, or just have questions. We would rather hear from you early, and knowing how teams
> are handling this shapes what we build.

Read the values from the RDS secret in your Cloud Platform namespace and create a secret in
your Container Platform namespace:

```bash
CP2=live.cloud-platform.service.justice.gov.uk
get() { kubectl --context $CP2 -n [your cloud platform namespace] get secret [your rds secret] -o jsonpath="{.data.$1}" | base64 -d; }

kubectl -n [your container platform namespace] create secret generic [your app]-db \
  --from-literal=PGHOST="$(get rds_instance_address)" \
  --from-literal=PGPORT=5432 \
  --from-literal=PGDATABASE="$(get database_name)" \
  --from-literal=PGUSER="$(get database_username)" \
  --from-literal=PGPASSWORD="$(get database_password)" \
  --from-literal=PGSSLMODE=verify-full
```

Using the standard `PG*` variable names means most PostgreSQL clients pick them up without
you building a connection string, so no credential needs to appear in your code.

> Because this secret is created by hand, Argo CD does not manage it. It will not be
> recreated if the namespace is rebuilt, and it is invisible to code review. Keep a note of
> how to recreate it.

Never commit credentials to a repository or put them in plain manifests.

### Verifying the database certificate

Amazon RDS presents a certificate signed by a private Amazon CA, not a public one. It is not
in the trust store of a normal container image, so a client that verifies properly will
reject the connection until you supply the CA.

The Container Platform does not currently distribute CA bundles. cert-manager runs on the
cluster, but it issues certificates for your own services rather than telling your
application which external authorities to trust. You supply the bundle yourself.

Download the bundle for your region from the
[certificate bundles table](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/UsingWithRDS.SSL.html#UsingWithRDS.SSL.CertificatesAllRegions)
in the AWS documentation, commit it to your deployment directory, and mount it as a
ConfigMap. The certificates are public, so a ConfigMap is correct and a Secret is not.

```yaml
      containers:
        - name: my-app
          env:
            - name: PGSSLMODE
              value: verify-full
            - name: PGSSLROOTCERT
              value: /etc/rds/rds-ca.pem
          envFrom:
            - secretRef:
                name: my-app-db
          volumeMounts:
            - name: rds-ca
              mountPath: /etc/rds
              readOnly: true
      volumes:
        - name: rds-ca
          configMap:
            name: my-app-rds-ca
```

Set `PGSSLMODE` in the deployment rather than relying on the secret alone. If it is only in
the secret, anyone recreating that secret without it silently drops to `prefer`, which
accepts any certificate.

`verify-full` is worth the extra step. `require` encrypts the connection but does not check
who the server is, so it is no defence against anything on the path between the two
platforms.

### Query safety

Use parameterised queries. Building SQL by concatenating user input is the most common cause
of serious vulnerabilities in applications of this shape, and the network work above does
nothing to protect you from it.

### A working example

The [test app](https://github.com/ministryofjustice/container-platform-test-app) connects to
a Cloud Platform database using this pattern, and its deployment lives in
[container-platform-environments](https://github.com/ministryofjustice/container-platform-environments/tree/main/namespaces/octo/test-app).

## Troubleshooting

**An application fails with `certificate verify failed` or `SSL error`.**

The RDS certificate authority is not in your image's trust store. Mount the bundle for your
region and point `PGSSLROOTCERT` at it, as described in
[Verifying the database certificate](#verifying-the-database-certificate). Check the file is
actually present in the container, and that you took the bundle for the right region.

**The connection works but your application ignores the database.**

Check the image you deployed contains the database code. Pinning an image by digest is
right, but a digest from before the feature was added will start cleanly and silently do
nothing.

## Getting help

Ask in [#container-platform-alpha-users](https://moj.enterprise.slack.com/archives/C0BKZHMQNRK).

## See Also

- [Connecting to a Remote Database With A Port Forwarding Pod](/documentation/networking/connecting-to-a-remote-database-with-port-forward.html)