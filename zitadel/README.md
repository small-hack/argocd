# An Argo CD App for Zitadel

[ZITADEL](https://github.com/zitadel/zitadel/tree/main) is an identity management and provider application similar to Keycloak. It helps you manage your users acrross apps and acts as your OIDC provider. It's community have been really nice, so we'll be supporting some Argo CD apps here in favor of depracting Keycloak support down the line eventually. Here's the [Zitadel helm chart](https://github.com/zitadel/zitadel-charts/tree/main) that we're deploying here.

It's important to take a look at the [`defaults.yaml`](https://github.com/zitadel/zitadel/blob/main/cmd/defaults.yaml) to see what the default `ConfigMap` will look like for Zitadel.

This ArgoCD app of apps is designed to be pretty locked down to allow you to use **only** a default service account (that also can't log in through the UI) to do all the heavy lifting with terraform, pulumi, or your own self service script. We support both [cockroachdb](./zitadel_and_cockroachdb) _and_ [postgresql](./zitadel_and_postgresql) as the database backends.

## Sync waves

1. External Secrets for both your database (cockroachdb or postgresql) and zitadel
   - Zitadel database credentials
   - Zitadel `masterkey` secret
   Persistent volume for your database, including backups via [k8up](https://k8up.io)
2. postgres or cockraochdb helm chart
3. zitadel helm chart with ONLY a service account and registration DISABLED

## Usage

To deploy Zitadel and postgresql:
```bash
argocd app create zitadel --upsert --repo https://github.com/small-hack/argocd-apps --path zitadel/zitadel_and_postgresql --sync-policy automated --self-heal --auto-prune --dest-namespace zitadel --dest-server https://kubernetes.default.svc
```

To deploy Zitadel and cockraochdb:
```bash
argocd app create zitadel --upsert --repo https://github.com/small-hack/argocd-apps --path zitadel/zitadel_and_cockroachdb --sync-policy automated --self-heal --auto-prune --dest-namespace zitadel --dest-server https://kubernetes.default.svc
```

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
