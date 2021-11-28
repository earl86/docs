---
title: Upgrading Postgres Overview
menu:
  docs_{{ .version }}:
    identifier: guides-postgres-upgrading-overview
    name: Overview
    parent: guides-postgres-upgrading
    weight: 10
menu_name: docs_{{ .version }}
section_menu_id: guides
---

> New to KubeDB? Please start [here](/docs/README.md).

{{< notice type="warning" message="This is an Enterprise-only feature. Please install [KubeDB Enterprise Edition](/docs/setup/install/enterprise.md) to try this feature." >}}

# Upgrading Postgres version Overview

This guide will give you an overview of how KubeDB enterprise operator upgrades the version of `Postgres` database.

## Before You Begin

- You should be familiar with the following `KubeDB` concepts:
  - [Postgres](/docs/guides/postgres/concepts/postgres.md)
  - [PostgresOpsRequest](/docs/guides/postgres/concepts/opsrequest.md)

## How Upgrade Process Works

The following diagram shows how KubeDB enterprise operator used to upgrade the version of `Postgres`. Open the image in a new tab to see the enlarged version.

<figure align="center">
  <img alt="Stash Backup Flow" src="/docs/guides/postgres/upgrading/overview/images/pg-upgrading.png">
<figcaption align="center">Fig: Upgrading Process of Postgres</figcaption>
</figure>

The upgrading process consists of the following steps:

1. At first, a user creates a `Postgres` cr.

2. `KubeDB` community operator watches for the `Postgres` cr.

3. When it finds one, it creates a `StatefulSet` and related necessary stuff like secret, service, etc.

4. Then, in order to upgrade the version of the `Postgres` database the user creates a `PostgresOpsRequest` cr with the desired version.

5. `KubeDB` enterprise operator watches for `PostgresOpsRequest`.

6. When it finds one, it halts the `Postgres` object so that the `KubeDB` community operator doesn't perform any operation on the `Postgres` during the upgrading process.

7. By looking at the target version from `PostgresOpsRequest` cr, `KubeDB` enterprise operator takes one of the following steps:
    - either update the images of the `StatefulSet` for upgrading between patch/minor versions.
    - or creates a new `StatefulSet` using targeted image for upgrading between major versions.

8. After successful upgradation of the `StatefulSet` and its `Pod` images, the `KubeDB` enterprise operator updates the image of the `Postgres` object to reflect the updated cluster state.

9. After successful upgradation of `Postgres` object, the `KubeDB` enterprise operator resumes the `Postgres` object so that the `KubeDB` community operator can resume its usual operations.

In the next doc, we are going to show a step by step guide on upgrading of a Postgres database using upgrade operation.