# flyway-migrations

Composite action that runs [Flyway](https://flywaydb.org/) against a PostgreSQL
database that is only reachable through a bastion. It opens a tunnel to the
database on `localhost:5432` and runs `flyway <command>` with the repo's
migrations.

Two ways to reach the database:

## SSM Session Manager (recommended)

The workflow assumes an IAM role with GitHub OIDC and opens a Session Manager
port forward (`AWS-StartPortForwardingSessionToRemoteHost`) through the bastion.
No SSH key, no open port 22. The bastion is found by its `Name` tag, so it keeps
working when the instance is replaced.

```yaml
permissions:
  contents: read
  id-token: write # required for OIDC

jobs:
  migrate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
      - uses: meetlara/flyway-migrations@main
        with:
          aws_role_arn: arn:aws:iam::<account>:role/prd-github-db-migrations
          bastion_name: prd-bastion
          db_host: ${{ secrets.PRD_RDS_HOST }}
          db_port: ${{ secrets.PRD_RDS_PORT }}
          db_user: ${{ secrets.PRD_RDS_MIGRATION_USER }}
          db_password: ${{ secrets.PRD_RDS_MIGRATION_PASSWORD }}
          db_name: ${{ secrets.PRD_RDS_DB_NAME }}
```

The role must allow `ssm:StartSession` on the bastion with that document and
`ec2:DescribeInstances` (see `prd/bastion/github_db_migrations.tf` in
`meetlara/terraform`).

## SSH tunnel (legacy)

Used when `aws_role_arn` is empty. Requires `ssh_host`, `ssh_user` and
`ssh_key`, and port 22 open on the bastion.

## Inputs

| Input | Default | Notes |
| --- | --- | --- |
| `aws_role_arn` | `''` | Enables SSM mode |
| `aws_region` | `us-east-1` | SSM mode |
| `bastion_name` | `''` | `Name` tag of the bastion, required in SSM mode |
| `ssh_host`, `ssh_user`, `ssh_key` | `''` | SSH mode |
| `db_host`, `db_user`, `db_password`, `db_name` | | Required |
| `db_port` | `5432` | |
| `migrations_folder_path` | `db/migrations` | |
| `flyway_config_path` | `db/conf/flyway.conf` | |
| `flyway_extra_params` | `-outOfOrder=true -ignoreMigrationPatterns=*:missing` | |
| `flyway_command` | `migrate` | e.g. `repair` |
| `flyway_version` | `8.5.11` | |
