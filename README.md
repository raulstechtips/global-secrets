# global-secrets

Helm chart for Kubernetes Secrets shared across namespaces.
The Infisical operator fetches the credential values. Reflector copies the
generated Secrets to other namespaces.

## Ownership

| Component | Responsibility |
| --- | --- |
| This chart | Defines `InfisicalStaticSecret` resources, key mappings, Secret types, and reflection settings. |
| `cluster-config` | Selects the chart version and supplies environment values. |
| Infisical | Stores the credential values. |
| Infisical operator | Creates and refreshes the source Kubernetes Secrets. |
| Reflector | Copies the selected Secrets across namespaces. |

The chart does not install the Infisical operator or Reflector.
It does not create a second connection or authentication resource.
It references the existing `InfisicalAuth`.

## Prerequisites

- Helm 3.
- An installed Infisical operator with the `v1beta1` `InfisicalStaticSecret`
  CRD, including `targets[].secretType`.
- An existing `InfisicalAuth` with access to the configured project,
  environment, and paths.
- Reflector, if reflection is enabled for any Secret.
- An existing release namespace. The development release uses `default`.

The chart creates no resources with its default values.
Environment-specific configuration belongs in
`cluster-config/apps/dev/global-secrets.yaml`.

## Configure the chart

Set the shared Infisical settings once:

```yaml
infisical:
  authRef:
    name: infisical-prod-universal-auth
    namespace: default
  projectSlug: k3s-ntpo
  environmentSlug: prod
  refreshInterval: 60s
```

Then add one entry for each source Secret:

```yaml
secrets:
  - name: github-registry-secret
    secretPath: /registry
    type: kubernetes.io/dockerconfigjson
    keys:
      .dockerconfigjson: DOCKER_CONFIG_JSON
    reflection:
      enabled: true
  - name: harbor-registry-secret
    secretPath: /registry/harbor
    type: kubernetes.io/dockerconfigjson
    keys:
      .dockerconfigjson: DOCKER_CONFIG_JSON
    reflection:
      enabled: true
```

`keys` maps a Kubernetes Secret key on the left to an Infisical key name
on the right. These are names, not credential values.
Only mapped keys are included in the generated Secret.
The chart supports source key names with dots and hyphens.

| Value | Required | Behavior |
| --- | --- | --- |
| `infisical.authRef.name` | When Secrets are configured | Name of the shared authentication resource. |
| `infisical.authRef.namespace` | When Secrets are configured | Namespace of that resource. |
| `infisical.projectSlug` | When Secrets are configured | Infisical project to read. |
| `infisical.environmentSlug` | When Secrets are configured | Infisical environment to read. |
| `infisical.refreshInterval` | Yes | Defaults to `60s`; seconds must be at least 5. Minute, hour, day, and week units are also accepted. |
| `secrets[].name` | Yes | Name of both the static-secret resource and its target Secret. Names must be unique. |
| `secrets[].secretPath` | Yes | Absolute Infisical folder path. |
| `secrets[].keys` | Yes | Non-empty map of target keys to source key names. |
| `secrets[].type` | No | Defaults to `Opaque`. Docker configuration Secrets require a `.dockerconfigjson` mapping. |
| `secrets[].reflection.enabled` | No | Defaults to `false`. When true, enables automatic reflection to all namespaces. |

The release namespace is used for every source Secret.
The target creation policy is `Owner`: deleting its `InfisicalStaticSecret`
also deletes the generated Secret.

Reflector annotations are set on the generated Secret, not just on the
Infisical custom resource. An existing unrelated Secret with the same name
in another namespace can prevent automatic reflection there.

The Harbor `DOCKER_CONFIG_JSON` value was created from
`/jenkins/registries/harbor` credentials for `harbor.dev.raulstechtips.dev`.
It is a stored copy. Rotating `Username` or `Password` in the Jenkins folder
does not regenerate this JSON. Update the JSON source when those credentials
change.

## Verify locally

From this repository, run:

```text
helm lint --strict charts
helm lint --strict charts -f charts/ci/registry-values.yaml
helm unittest charts
helm template global-secrets charts --namespace default \
  -f ../cluster-config/apps/dev/global-secrets.yaml
```

The development render must contain two `InfisicalStaticSecret` resources.
Their targets must be `default/github-registry-secret` and
`default/harbor-registry-secret`, both with type
`kubernetes.io/dockerconfigjson`. The render contains key references only.

## Publish and select a version

The Jenkinsfile uses the existing `clusterConfigurablePipeline`.
It checks the chart, scans it, and publishes it to:

```text
oci://ghcr.io/raulstechtips/charts/global-secrets
```

The source chart starts at `0.0.0`; the shared pipeline calculates build
versions. `cluster-config/argo/dev/global-secrets.yaml` initially selects
the intended first release, `0.1.0`. That release must be published before
it can be synchronized. Alternatively, select a successfully published
main build.

Merge the cluster-config application definition before the chart's first
main build so `argoDeploy` has a target file to update. Main and release
builds use the existing pipeline behavior to update that file in Git.
The application has no automated sync policy, so publishing or updating
its version does not deploy the Secrets.

## Migrate the existing GitHub registry Secret

Perform these steps only when ready to deploy. Creating or merging the PRs
does not perform them.

The legacy resource is
`default/infisical-secret-github-registry-secret`, kind `InfisicalSecret`.
It currently owns `default/github-registry-secret`. Do not delete it with
the default cascading deletion policy.

1. Publish the selected chart version and make sure the two Infisical
   source paths contain their `DOCKER_CONFIG_JSON` values.
2. Let the root Argo application adopt the new
   `argo/dev/global-secrets.yaml` definition. Do not sync the child yet.
3. Open **Argo CD > Applications > global-secrets > App Details**.
   Confirm that its source is the new Helm chart, its values source is
   `apps/dev/global-secrets.yaml`, and automated sync is disabled.
   Do not continue while the old source or automated sync is active.
4. Confirm that the existing source Secret is present:

   ```text
   kubectl --namespace default get secret github-registry-secret
   ```

5. Delete only the legacy custom resource and preserve its dependent Secret:

   ```text
   kubectl --namespace default delete \
     infisicalsecrets.secrets.infisical.com \
     infisical-secret-github-registry-secret --cascade=orphan
   ```

6. Confirm that `github-registry-secret` still exists and no longer has an
   owner reference to the deleted custom resource:

   ```text
   kubectl --namespace default get secret github-registry-secret \
     -o jsonpath='{.metadata.ownerReferences}'
   ```

   If the old owner reference remains, stop before syncing the new chart.
   Do not print the Secret data.
7. In **Argo CD > global-secrets**, select **Sync**, inspect the two
   `InfisicalStaticSecret` resources, then synchronize them. Do not select
   prune as part of this first ownership transfer.
8. Check the operator resources and source Secret types:

   ```text
   kubectl --namespace default get infisicalstaticsecrets
   kubectl --namespace default get secret \
     github-registry-secret harbor-registry-secret
   ```

   Both source Secrets must exist with type `kubernetes.io/dockerconfigjson`.
   Check each static-secret resource's status for a successful synchronization.
9. Check copies in a namespace that needs registry access:

   ```text
   kubectl --namespace jenkins get secret \
     github-registry-secret harbor-registry-secret
   ```

   Check Reflector events or logs if a copy is missing. Resolve any existing
   same-name Secret conflict before claiming reflection is complete.

The old files under `cluster-config/apps/global-secrets` remain as migration
reference. Do not reapply their old Application or custom resource after
the ownership transfer.
