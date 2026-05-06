# WARNING
This repo is public. Never commit anything into this repo that could ever be considered sensitive.

---

# platform-schema

JSON Schema for the `platform.yaml` file authored in service repositories.
The schema describes how a service expresses its identity, workload shape, and
per-environment runtime configuration in a single declarative file. The
platform pipeline validates each service's `platform.yaml` against this schema
and translates it into the values consumed by the shared helm chart at deploy
time.

## Layout

```
schema.json        Top-level router — dispatches on `apiVersion` to the
                   matching per-major contract. This is the URL service
                   repositories point at.
v1/
└── schema.json    The v1 contract (resolved by the router when
                   `apiVersion: platform.mrq/v1`).
```

A future `v2/` will live alongside `v1/`; the router gains a second dispatch
branch and both majors validate in parallel until the older one is retired.

## Consuming the schema

Service repositories point at the router URL once and never touch it again
when bumping majors — only `apiVersion` changes:

```
https://raw.githubusercontent.com/lindar-joy/platform-schema/master/schema.json
```

Declare the URL via a [yaml-language-server modeline][modeline] on the first
line of `platform.yaml`:

```yaml
# yaml-language-server: $schema=https://raw.githubusercontent.com/lindar-joy/platform-schema/master/schema.json
apiVersion: platform.mrq/v1
metadata:
  name: my-service
  owner: my-team
spec:
  type: backend-api
  replicas: 2
  compute:
    requests: { memory: 512Mi, cpu: 250m }
    limits:   { memory: 1Gi,   cpu: 1000m }
```

The [Red Hat YAML extension][rh-yaml] for VSCode and most other yaml-aware
editors respect the modeline and provide schema-driven completion, hover
documentation, and inline validation — no per-repo `.vscode/settings.json`
required.

[modeline]: https://github.com/redhat-developer/yaml-language-server#using-inlined-schema
[rh-yaml]: https://marketplace.visualstudio.com/items?itemName=redhat.vscode-yaml

## Versioning

The schema follows major-only pinning. Within a major version, **all changes
are additive**: new optional fields, new defaults that do not flip existing
behaviour, bug fixes. Service repos pinned to `platform.mrq/v1` pick up
improvements automatically and never see a breaking change without explicitly
bumping their `apiVersion`.

Breaking changes — renamed fields, removed fields, required-field additions,
default changes that flip deployed behaviour — ship as a new major version
(`v2/schema.json`) alongside the existing one. Both run in parallel during a
deprecation window committed to upfront, before the older major is retired.

## Validating locally

The pipeline is the authoritative validator, but any JSON Schema validator
works for local checks. Examples:

```sh
# Node — ajv-cli (load the router and the per-major schemas it refs into)
npx ajv-cli@latest validate \
  -s schema.json \
  -r v1/schema.json \
  -d path/to/platform.yaml \
  --spec=draft2020

# Python — check-jsonschema
pip install check-jsonschema
check-jsonschema --schemafile schema.json path/to/platform.yaml
```
