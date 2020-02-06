# Dumbster

> Declarative backups and restores

Dumbster is a tool for making backups and restores of databases and file storages. Deployed using this Helm Chart, it provides an easy way of declaratively specifying backup sources and where to put them. In the event of disaster, the same chart can be used to declaratively spin off a single job that handles the restore.

## Features

- **Scheduling:** Choose a schedule to run backup jobs on (crontab syntax).
- **Multiple sources:**
- **Simple, but flexible:** In many cases you can get by using only one instance of the chart, but you can also create many different ones with different configurations

## Supported sources and destinations

Currently, the following sources are supported for backups:

- PostgreSQL (using `pg_dump` and `pg_restore`)
- MongoDB and Cosmos DB with Mongo API (using `mongodump` and `mongorestore`)

The destination determines where the backup will be saved.

- Azure Files

## Usage with Helm Operator

To use this chart together with the Helm Operator, you can create a resource looking something like the one below. The result after templating is a set of jobs/cronjobs depending on the input values. In this example, backups are made from both the specified PostgreSQL databases, and the MongoDB database.

In an effort to make the API-surface as simple as possible, some configuration values are used accross jobs. If you need more granular control of each backup configuration, you can create multiple `HelmRelease`-resources. As an example, you could create one `HelmRelease` for each service, or only one for all the services.

```yaml
apiVersion: helm.fluxcd.io/v1
kind: HelmRelease
metadata:
  name: dumbster-backup-example
  namespace: default
spec:
  releaseName: dumbster
  chart:
    repository: https://charts.sondregjellestad.space/
    name: dumbster
    version: 0.1.12
  values:
    job:
      name: example-service
    backup:
      enabled: true
      schedule: "@hourly"
      retentionDays: 30
      source:
        postgres:
          enabled: true
          secretName: dumbster-postgres-secret
          databases:
            - posts_db
            - users_db
        mongodb:
          enabled: true
          secretName: dumbster-mongodb-secret
      destination:
        azureFile:
          mountPath: "/backup"
          secretName: dumbster-backup-storage-secret
          shareName: backup
          readOnly: false
```

Even restores could be configured on the same `HelmRelease`, but it is probably better to set it up by itself, as the circumstances under which one usually would perform one are a bit different.

*Example:*

```yaml
apiVersion: helm.fluxcd.io/v1
kind: HelmRelease
metadata:
  name: dumbster-restore-example
  namespace: default
spec:
  releaseName: dumbster
  chart:
    repository: https://charts.sondregjellestad.space/
    name: dumbster
    version: 0.1.12
  values:
    job:
      name: example-service
    restore:
      enabled: true
      source:
        azureFile:
          mountPath: "/backup"
          secretName: dumbster-backup-storage-secret
          shareName: backup
          readOnly: false
      destination:
        mongodb:
          enabled: true
          secretName: dumbster-cosmosdb-secret
          restoreFile: 20200128.dump.gzip
          cosmosDB: true

```

If deployments are made using Flux CD, you can manage the secrets using SOPS or Sealed Secrets.

*`InitContainer`, `.flux.yaml`*



## Secrets

Some of the values required for this chart should be handled as secrets. Native Kubernetes secrets are used to provide these values, and only the name of the secret is provided to the chart.

The values to provide in the secret depends on what configuration you want.

For backups, there are two types of secrets required; one for the source, and one for the destination.

*For managing the secrets, check out SOPS, SealedSecrets...*

You need to create at least one secret for each target used.

### MongoDB

```yaml
apiVersion: v1
kind: Secret
metadata:
    name: dumbster-mongodb-secret
type: Opaque
stringData:
    connectionString: mongodb+srv://username:password@host
```

### PostgreSQL

```yaml
apiVersion: v1
kind: Secret
metadata:
    name: dumbster-postgres-secret
type: Opaque
stringData:
		# URI must be URL-encoded (@ -> %40)
    connectionString: postgresql://backup%40sondre-test-db-postgres:password@sondre-test-db-postgres.postgres.database.azure.com
```

### Azure Files

```yaml
apiVersion: v1
kind: Secret
metadata:
    name: dumbster-backup-storage-secret
type: Opaque
stringData:
    azurestorageaccountkey: s3cre7
    azurestorageaccountname: storage-account-123
    azurestoragesharename: backup
```



## A simple setup with Flux CD and Azure KeyVault



SOPS is included in the Flux image, and can therefore be used by Flux without additional configuration.

In order to access credentials for the KeyVault from within Flux, an initcontainer can be used to extract the necessary values from `/etc/kubernetes/azure.json` on the node. With the values present in e.g. a volume mounted to the Flux container, the secrets can be encrypted like so:

*`.flux.yaml`:*

```yaml
version: 1
patchUpdated:
  generators:
    - command: . /shared/keyvault-creds.sh && sops -d secrets.enc.yaml
    - command: kustomize build .
  patchFile: flux-patch.yaml
```

*`.sops.yaml`:*

```yaml
creation_rules:
  - path_regex: .enc.yaml
    encrypted_regex: "^(data|stringData)$"
    azure_keyvault: "https://sondre-test-keyvault.vault.azure.net/keys/sops-key/048722f2b99c48c4935f7292a49a51bd"
```

One drawback with this approach is that it is necessary to specify all files encrypted with SOPS in `.flux.yaml`.



```bash
$ sops
```



## Doing a restore

Restoring a database from a backup involves, at a low level, putting the dumpfile through a tool like `pg_dump`. 

*The restore process has some issues with Azure Cosmos DB.*

### Manually

If you want to do it all by yourself, you first need to install the appropriate tools and get the dumpfile from to file storage that was used to store it.

After downloading the dump file, consult the documentation for the tool for more information on how to use it.

## Future iterations

There are many ways to do this, which are possibly better.

- A custom operator with appropriate CRDs
- A controller reading annotations
- Dynamically injecting secrets, https://github.com/kiwigrid/k8s-sidecar

