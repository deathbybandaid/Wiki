---
title: Prowlarr Configuring  PostgreSQL Database
description: Configuring Prowlarr with a Postgres Database
published: true
date: 2022-01-10T15:42:47.749Z
tags: 
editor: markdown
dateCreated: 2022-01-10T15:38:53.538Z
---

# Prowlarr and Postgres

This document will go over the key items for migrating and setting up Postgres support in Prowlarr.

This guide has been created by [Roxedus](https://github.com/Roxedus)

## Creation of initial database

- We do this also when migrating, this is to ensure Prowlarr sets up the required schema.

### Setting up Postgres

Firstly we need a Postgres instance, this guide is written for using the postgres:14 docker image.

> Do not even think about using the latest tag {.is-danger}

```bash
docker create --name=postgres14 \
    -e POSTGRES_PASSWORD=qstick \
    -e POSTGRES_USER=qstick \
    -e POSTGRES_DB=prowlarr-main \
    -p 5432:5432/tcp \
    -v ..appdata/postgres14:/var/lib/postgresql/data \
    postgres:14
```

Prowlarr needs two databases:

- `prowlarr-main`   This is used to store all configuration and history
- `prowlarr-log`    This is used to store events that produce a logentry

Create these databases using your favorite method, with the same username and password. Roxedus used Adminer as he already had that set up.

### Schema creation

We need to tell Prowlarr to use Postgres, the `config.xml` should already be populated with the entries we need.

```xml
<PostgresUser>qstick</PostgresUser>
<PostgresPassword>qstick</PostgresPassword>
<PostgresPort>5432</PostgresPort>
<PostgresHost>postgres14</PostgresHost>
```

## Migrate data

> If you do not want to migrate a existing SQLite database to Postgres, you can are finished with this guide.{.is-info}

To migrate data we can use [PGLoader](https://github.com/dimitri/pgloader), it does however have some gotchas:

- By default transactions are case-insensitive, we use `--with "quote identifiers"` to make them sensitive.
- The version packaged in Debian and Ubuntu's apt repo are tested as too old for newer versions of Postgres (Roxedus has not tested the packages in other distros)
  Roxedus have [re-built a binary](https://github.com/Roxedus/Pgloader-bin) to enable this support (No code-modification needed, just need to be built with updated dependencies)

Once these handled, it's pretty straight forward, after telling it to not mess with the scheme using `--with "data only"`.

```bash
pgloader --with "quote identifiers" --with "data only" prowlarr.db 'postgresql://qstick:qstick@localhost/prowlarr-main'
```

Or alternatively using the dockerimage producing the binary:

```bash
docker run -v ..prowlarr.db:/prowlarr.db --network=host ghcr.io/roxedus/pgloader --with "quote identifiers" --with "data only" /prowlarr.db "postgresql://qstick:qstick@localhost/prowlarr-main"
```