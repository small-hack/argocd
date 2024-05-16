# An Argo CD App for Zitadel

[ZITADEL](https://github.com/zitadel/zitadel/tree/main) is an identity management and provider application similar to Keycloak. It helps you manage your users across apps and acts as your OIDC provider. The Zitadel community and maintainers have been really nice, so we'll be supporting some Argo CD apps here instead of using Keycloak. Here's the [Zitadel helm chart](https://github.com/zitadel/zitadel-charts/tree/main) that we're deploying here.

<img width="900" alt="screenshot of Argo CD web interface's tree view of a zitadel app of apps. The main app of apps branches off into the following appsets: external secrets, postgres, s3 provider, s3 PVC, and zitadel web app. Each of those then branches off into a similarly named app." src="https://github.com/small-hack/argocd-apps/assets/2389292/467fd0cf-36a7-47fd-80b8-4bd051ec0157">

<details>
  <summary>More Argo CD Zitadel screenshots</summary>
  
  ### Zitadel web app (official zitadel helm chart)
  <img width="900" alt="screenshot of Argo CD web interface's tree view of a zitadel web app in tree view mode. Includes the following child resources: zitadel config map, zitadel service, zitadel service account, zitadel deployment, zitadel init job, zitadel setup job, zitadel service monitor, zitadel ingress, zitadel role, zitadel role binding. The zitadel service then branches off into zitadel endpoint and zitadel endpointslice. The zitadel deployment branches off into a zitadel replica set which branches off into a zitadel pod. The zitadel init and setup jobs also branch off into their own completed pods, and finally, the zitadel ingress resource branches off into a zitadel TLS certificate" src="https://github.com/small-hack/argocd-apps/assets/2389292/e2bf4838-85cf-4f5b-9e1a-b98756fc357c">

  ### Postgresql cluster
  <img width="900" alt="screenshot of Argo CD web interface's tree view of a zitadel postgresql cluster in tree view. It shows the following secrets and coorsponding certificates: client cert, postgres cert, server secret, zitadel cert. Each of those then has their own cert request resource. Afte rthat there's 3 tls issuers: client ca, selfsigned, and server ca. Next there is the cluster, which branches off into a pvc, pod, secret for the app, secret for the super user, service for read, service for read only, service for read write, service account, pod disruption budget for the primary, role, and role binding" src="https://github.com/small-hack/argocd-apps/assets/2389292/366d40e5-2720-4cd8-a5e0-08025909a60d">

     
</details>

It's important to take a look at the [`defaults.yaml`](https://github.com/zitadel/zitadel/blob/main/cmd/defaults.yaml) to see what the default `ConfigMap` will look like for Zitadel.

This Argo CD app of apps is designed to be pretty locked down to allow you to use **only** a default service account (that can't log in through the UI) to do all the heavy lifting with openTofu, Pulumi, or your own self service script. We support only PostgreSQL as the database backend. PostgreSQL is backed up by [barman]() to a local s3 instance using SeaweedFS. That SeaweedFS instance is then backed up to a remote s3 endpoint of your choosing.

The main magic happens in the [app_of_apps](./app_of_apps) directory.

## Sync waves

1. External Secrets for both your database postgresql and zitadel:
     - Zitadel database credentials
     - Zitadel `masterkey` Secrets
2. SeaweedFS instance.
3. Postgresql cluster via the Cloud Native Postgresql Operator including the database init described [here](https://github.com/zitadel/zitadel/tree/0386fe7f96cc8c9ff178d29c9bbee3bfe0a1a568/cmd/initialise/sql/postgres)
4. Zitadel helm chart with ONLY a service account and registration DISABLED

## Usage

To deploy Zitadel and PostgreSQL _without_ an isolated MinIO tenant for PostgreSQL backups:
```bash
argocd app create zitadel --upsert --repo https://github.com/small-hack/argocd-apps --path zitadel/app_of_apps --sync-policy automated --self-heal --auto-prune --dest-namespace zitadel --dest-server https://kubernetes.default.svc
```

To deploy Zitadel and PostgreSQL with an isolated MinIO tenant for PostgreSQL backups:
```bash
argocd app create zitadel --upsert --repo https://github.com/small-hack/argocd-apps --path zitadel/app_of_apps --sync-policy automated --self-heal --auto-prune --dest-namespace zitadel --dest-server https://kubernetes.default.svc --directory-recursion
```

### No Metrics

We have metrics on by default for zitadel. If you're not deploying prometheus, point your Argo CD add [`zitadel/app_of_apps/no-metrics/`](./app_of_apps/no-metrics).

## Zitadel OIDC for logging into Argo CD with Zitadel as the SSO

Check out this [PR](https://github.com/argoproj/argo-cd/pull/15029)

## Using the Zitadel API

The API docs are [here](https://zitadel.com/docs/category/apis).

For Actions (needed for Argo CD and Zitadel to work nicely) you probably want this [link](https://zitadel.com/docs/category/apis/resources/mgmt/actions)

## Helm testing locally

Zitadel has an official guide for k8s deployments [here](https://zitadel.com/docs/self-hosting/deploy/kubernetes).

## TODO
Find a graceful way to setup SMTP or not. Here's the configuration parameters that would need to be set:

```yaml
DefaultInstance:
  # this configuration sets the default email configuration
  SMTPConfiguration:
    # Configuration of the host
    SMTP:
      # must include the port, like smtp.mailtrap.io:2525. IPv6 is also supported, like [2001:db8::1]:2525
      Host: # ZITADEL_DEFAULTINSTANCE_SMTPCONFIGURATION_SMTP_HOST
      User: # ZITADEL_DEFAULTINSTANCE_SMTPCONFIGURATION_SMTP_USER
      Password: # ZITADEL_DEFAULTINSTANCE_SMTPCONFIGURATION_SMTP_PASSWORD
    TLS: # ZITADEL_DEFAULTINSTANCE_SMTPCONFIGURATION_SMTP_SSL
    # If the host of the sender is different from ExternalDomain set DefaultInstance.DomainPolicy.SMTPSenderAddressMatchesInstanceDomain to false
    From: # ZITADEL_DEFAULTINSTANCE_SMTPCONFIGURATION_SMTP_FROM
    FromName: # ZITADEL_DEFAULTINSTANCE_SMTPCONFIGURATION_SMTP_FROMNAME
  DomainPolicy:
    SMTPSenderAddressMatchesInstanceDomain: false
```

## Writing an action to send a groupsClaim

Just an example of how to make a groups claim for zitadel.

```js
function groupsClaim(ctx, api) {
  if (ctx.v1.user.grants === undefined || ctx.v1.user.grants.count == 0) {
    return;
  }

  let grants = [];
  ctx.v1.user.grants.grants.forEach(claim => {
    claim.roles.forEach(role => {
      grants.push(role)
    })
  })

  api.v1.claims.setClaim('groups', grants)
}
```

## Writing an action to send a claim in an OIDC response

Here's an example action script that we've called nextcloudAdminClaim. It iterates through the user's roles and if one of them is `nextcloud_admins` it sends a claim it calls `nextcloud_admin` back with a true value, else it sends back false.

```js
function nextcloudAdminClaim(ctx, api) {

  if (ctx.v1.user.grants === undefined || ctx.v1.user.grants.count == 0) {
    return;
  }

  ctx.v1.user.grants.grants.forEach(claim => {
    if (claim.roles.includes('nextcloud_admins') {
        api.v1.claims.setClaim('nextcloud_admin', true)
        return;
        }

    if (claim.roles.includes('nextcloud_users') {
        api.v1.claims.setClaim('nextcloud_admin', false)
        return;
        }
    }
}
```
