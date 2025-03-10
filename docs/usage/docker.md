---
title: Docker
description: Docker Package Manager Support in Renovate
---

# Docker

Renovate supports upgrading dependencies in various types of Docker definition files:

- Docker's `Dockerfile` files
- Docker Compose `docker-compose.yml`, `compose.yml` files
- CircleCI config files
- Kubernetes manifest files
- Ansible configuration files

## How It Works

1. Renovate searches in each repository for any files matching each manager's configured `fileMatch` pattern(s)
1. Matching files are parsed, Renovate checks if the file(s) has any Docker image references (e.g. `FROM` lines in a `Dockerfile`)
1. If the image tag in use "looks" like a version (e.g. `myimage:1`, `myimage:1.1`, `myimage:1.1.0`, `myimage:1-onbuild`) then Renovate checks the Docker registry for upgrades (e.g. from `myimage:1.1.0` to `myimage:1.2.0`)

## Preservation of Version Precision

By default, Renovate preserves the precision level specified in the Docker images.
For example, if the existing image is pinned at `myimage:1.1` then Renovate only proposes upgrades to `myimage:1.2` or `myimage:1.3`.
This means that you will not get upgrades to a more specific versions like `myimage:1.2.0` or `myimage:1.3.0`.
Renovate does not yet support "pinning" an imprecise version to a precise version, e.g. from `myimage:1.2` to `myimage:1.2.0`, but it's a feature we'd like to work on one day.

## Version compatibility

Although suffixes in SemVer indicate pre-releases (e.g. `v1.2.0-alpha.2`), in Docker they typically indicate compatibility, e.g. `1.2.0-alpine`.
By default Renovate assumes suffixes indicate compatibility, for this reason Renovate will not _change_ any suffixes.
Renovate will update `1.2.0-alpine` to `1.2.1-alpine` but never updates to `1.2.1` or `1.2.1-stretch` as that would change the suffix.

If this behavior does not suit a particular package you have, Renovate allows you to customize the `versioning` scheme it uses.
For example, you have a Docker image `foo/bar` that sticks to SemVer versioning.
This means that you need to tell Renovate that suffixes indicate pre-release versions, and not compatibility.

You could then use this `packageRules` array, to tell Renovate to use `semver` versioning for the `foo/bar` package:

```json
{
  "packageRules": [
    {
      "matchDatasources": ["docker"],
      "matchPackageNames": ["foo/bar"],
      "versioning": "semver"
    }
  ]
}
```

Another example is the official `python` image, which follows `pep440` versioning.
You can tell Renovate to use the `pep440` versioning scheme with this set of `packageRules`:

```json
{
  "packageRules": [
    {
      "matchDatasources": ["docker"],
      "matchPackageNames": ["python"],
      "versioning": "pep440"
    }
  ]
}
```

If traditional versioning doesn't work, try Renovate's built-in `loose` `versioning`.
Renovate will perform a best-effort sort of the versions, regardless of whether they have letters or digits.

If both the traditional versioning, and the `loose` versioning do not give the results you want, try the `regex` `versioning`.
This approach uses regex capture group syntax to specify which part of the version string is major, minor, patch, pre-release, or compatibility.
See the docs for `versioning` for documentation and examples of `regex` versioning in action.

## Digest Pinning

We recommend that you pin your Docker images to an exact digest.
By pinning to a digest you ensure your Docker builds are **immutable**: every time you do a `pull` you get the same content.

If you have experience with the way dependency versioning is handled in the JavaScript/npm ecosystem, you might be used to exact versions being immutable.
e.g. if you specify a version like `2.0.1`, you and your colleagues always get the exact same "code".

What you may not know is that Docker's tags are not immutable versions, even if they look like a version.
e.g. you probably expect `myimage:1` and `myimage:1.2` to change over time, but you might incorrectly assume that `myimage:1.2.0` never changes.
Although it probably _shouldn't_, the reality is that any Docker image tag _can_ change content, and potentially break.

Using a Docker digest as the image's primary identifier instead of using a Docker tag will achieve immutability.
It's not easy to work with strings like `FROM node@sha256:d938c1761e3afbae9242848ffbb95b9cc1cb0a24d889f8bd955204d347a7266e`.
Luckily Renovate can update the digests for you, so you don't have to.

To keep things simple, Renovate retains the Docker tag in the `FROM` line, e.g. `FROM node:14.15.1@sha256:d938c1761e3afbae9242848ffbb95b9cc1cb0a24d889f8bd955204d347a7266e`.
Read on to see how Renovate updates Docker digests.

## Digest Updating

If you follow our advice to go from a simple tag like `node:14` to using a pinned digest `node:14@sha256:d938c1761e3afbae9242848ffbb95b9cc1cb0a24d889f8bd955204d347a7266e`, you will get Renovate PRs whenever the `node:14` image is updated on Docker Hub.

Previously this update would have been "invisible" to you - one day you pull code that represents `node:14.15.0` and the next day you get code that represents `node:14.15.1`.
But you can never be sure, especially as Docker caches.
Perhaps some of your colleagues or worse still your build machine are stuck on an older version with a security vulnerability.

By pinning to a digest instead, you will get these updates via Pull Requests, or even committed directly to your repository if you enable branch automerge for convenience.
This ensures everyone on the team uses the latest versions and is in sync.

## Version Upgrading

Renovate also supports _upgrading_ versions in Docker tags, e.g. from `myimage:1.2.0` to `myimage:1.2.1` or `myimage:1.2` to `myimage:1.3`.
If a tag looks like a version, Renovate will upgrade it like a version.

We recommend you use the major.minor.patch tagging scheme e.g. change from `myimage:1` to `myimage:1.1.1`.
This way it's easy to see what the Renovate PR is going to change.
You can see the difference between a PR that upgrades `myimage` from `1.1.1` to `1.1.2`. and a PR that changes the contents of the version you already use (`1.1.1`).

Currently, Renovate will upgrade minor/patch versions (e.g. from `1.2.0` to `1.2.1`) by default, but not upgrade major versions.
If you wish to enable major versions then add the preset `docker:enableMajor` to your `extends` array in your `renovate.json`.

Renovate has some Docker-specific intelligence when it comes to versions.
For example:

### Ubuntu codenames

Renovate understands [Ubuntu release code names](https://wiki.ubuntu.com/Releases) and will offer upgrades to the latest LTS release (e.g. from `ubuntu:xenial` to `ubuntu:focal`).

For this to work you must follow this naming scheme:

- The first term of the full codename is used (e.g. `bionic` for `Bionic Beaver` release)
- The codename is in lowercase

For example, Renovate will offer to upgrade the following `Dockerfile` layer:

```dockerfile
FROM ubuntu:yakkety
```

To

```dockerfile
FROM ubuntu:focal
```

## Configuring/Disabling

If you wish to make changes that apply to all Docker managers, then add them to the `docker` config object.
This is not foolproof, because some managers like `circleci` and `ansible` support multiple datasources that do not inherit from the `docker` config object.

If you wish to override Docker settings for one particular type of manager, use that manager's config object instead.
For example, to disable digest updates for Docker Compose only but leave them for other managers like `Dockerfile`, you would use this:

```json
{
  "docker-compose": {
    "digest": {
      "enabled": false
    }
  }
}
```

The following configuration options are applicable to Docker:

### Disable all Docker Renovation

Add `"docker:disable"` to your `extends` array.

### Disable Renovate for only certain Dockerfiles

Add all paths to ignore into the `ignorePaths` configuration field. e.g.

```json
{
  "extends": ["config:base"],
  "ignorePaths": ["docker/old-files/"]
}
```

### Enable Docker major updates

Add `"docker:enableMajor"` to your `extends` array.

### Disable digest pinning

Add `"default:pinDigestsDisabled"` to your `extends` array.

### Automerge digest updates

Add `"default:automergeDigest"` to your `extends` array.
If you want Renovate to commit directly to your base branch without opening a PR first, add `"default:automergeBranchPush"` to the `extends` array.

### Registry authentication

There are many different registries, and many ways to authenticate to those registries.
We will explain how to authenticate for the most common registries.

#### DockerHub

Here is an example of configuring a default Docker username/password in `config.js`.
The Docker Hub password is stored in a process environment variable.

```js
module.exports = {
  hostRules: [
    {
      hostType: 'docker',
      username: '<your-username>',
      password: process.env.DOCKER_HUB_PASSWORD,
    },
  ],
};
```

You can add additional host rules, read the [hostrules documentation](https://docs.renovatebot.com/configuration-options/#hostrules) for more information.

#### Self-hosted Docker registry

Say you host some Docker images yourself, and use a password to access your self-hosted Docker images.
In addition to self-hosting, you also pull images from Docker Hub, without a password.
In this example you would configure a specific Docker host like this:

```js
module.exports = {
  hostRules: [
    {
      hostType: 'docker',
      matchHost: 'your.host.io',
      username: '<your-username>',
      password: process.env.SELF_HOSTED_DOCKER_IMAGES_PASSWORD,
    },
  ],
};
```

#### Google Container Registry

Assume you are running GitLab CI in the Google Cloud, and you are storing your Docker images in the Google Container Registry (GCR).

Access to the GCR uses Bearer token based authentication.
This token can be obtained by running `gcloud auth print-access-token`, which requires the Google Cloud SDK to be installed.

The token expires after 60 minutes so you cannot store it in a variable for subsequent builds (like you can with `RENOVATE_TOKEN`).

When running Renovate in this context the Google access token must be retrieved and injected into the `hostRules` configuration just before Renovate is started.

_This documentation gives **a few hints** on **a possible way** to achieve this end result._

The basic approach is that you create a custom image and then run Renovate as one of the stages of your project.
To make this run independent of any user you should use a [`Project Access Token`](https://docs.gitlab.com/ee/user/project/settings/project_access_tokens.html) (with Scopes: `api`, `read_api` and `write_repository`) for the project and use this as the `RENOVATE_TOKEN` variable for GitLab CI.
See also the [renovate-runner repository on GitLab](https://gitlab.com/renovate-bot/renovate-runner) where `.gitlab-ci.yml` configuration examples can be found.

To get access to the token a custom Renovate Docker image is needed that includes the Google Cloud SDK.
The Dockerfile to create such an image can look like this:

```Dockerfile
FROM renovate/renovate:32.45.5
# Include the "Docker tip" which you can find here https://cloud.google.com/sdk/docs/install
# under "Installation" for "Debian/Ubuntu"
RUN ...
```

For Renovate to access the Google Container Registry (GCR) it needs the current Google Access Token.
The configuration fragment to do that looks something like this:

```js
hostRules: [
  {
    matchHost: 'eu.gcr.io',
    token: 'MyReallySecretTokenThatExpiresAfter60Minutes',
  },
];
```

One way to provide the short-lived Google Access Token to Renovate is by generating these settings into a `config.js` file from within the `.gitlab-ci.yml` right before starting Renovate:

```yaml
script:
  - 'echo "module.exports = { hostRules: [ { matchHost: ''eu.gcr.io'', token: ''"$(gcloud auth print-access-token)"'' } ] };" > config.js'
  - renovate $RENOVATE_EXTRA_FLAGS
```
