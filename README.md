# S3 backup

An Ansible role to backup an S3 bucket to another bucket or SFTP server.

## Requirements

* The `rclone` command must be installed and executable by the user running the role.

Please see [rclone's documentation](https://rclone.org/install/) on how to install the command.

## Dependencies

The following collections must be installed:

* community.general

## Role Variables

This role requires one dictionary as configuration, `s3_backup`:

```yaml
    s3_backup:
      rclonePath: "/usr/local/bin/terminus"
      debug: true
      stopOnFailure: false
      sources: {}
      remotes: {}
      backups: []
```

Where:
* `rclonePath` is the full path to the `rclone` executable. Optional, defaults to `rclone`.
* `debug` is `true` to enable debugging output. Optional, defaults to `false`.
* `stopOnFailure` is `true` to stop the entire role if any one backup fails. Optional, defaults to `false`.
* `sources` is a dictionary of sites and environments. Required.
* `remotes` is a dictionary of remote upload locations. Required.
* `backups` is a list of backups to perform. Required.

### Specifying Sources

In this role, "sources" specify the source from which to download backups. Each must have a unique key which is later used in the `s3_backup.backups` list.

```yaml
s3_backup:
  sources:
    their-s3-bucket:
      type: "s3"
      bucket: "their-s3-bucket"
      provider: "AWS"
      accessKeyFile: "/path/to/aws-s3-key.txt"
      secretKeyFile: "/path/to/aws-s3-secret.txt"
      region: "us-east-1"
      retryCount: 3
      retryDelay: 30
```

Where, in each entry:
* `bucket` is the name of the S3 bucket. 
* `provider` is the S3 provider. [See rclone's S3 documenation on `--s3-provider`](https://rclone.org/s3/#s3-provider) for possible values. Optional, defaults to `AWS`.
* `accessKeyFile` is the path to a file containing the access key. Optional if `accessKey` is specified.
* `accessKey` is the value of the access key necessary to access the bucket. Ignored if `accessKeyFile` is specified.
* `secretKeyFile` is the path to a file containing the secret key. Optional if `secretKey` is specified.
* `secretKey` is the value of the access key necessary to secret the bucket. Ignored if `secretKeyFile` is specified.
* `region` is the AWS region in which the bucket resides. Required if using AWS S3, may be optional for other providers.
* `endpoint` is the S3 endpoint to use. Optional if using AWS, required for other providers.
* `retryCount` is the number of time to retry `platform` commands if they fail. Optional, defaults to `3`.
* `retryDelay` is the time in seconds to wait before retrying a failed `platform` command. Optional, defaults to `30`.

### Specifying remotes

In this role "remotes" are upload destinations for backups. This role supports S3 or SFTP as remotes. Each remote must have a unique key which is later used in the `s3_backup.backups` list.

```yaml
    - hosts: servers
      vars:
        s3_backup:
          remotes:
            example-s3-bucket:
              type: "s3"
              bucket: "my-s3-bucket"
              provider: "AWS"
              accessKeyFile: "/path/to/aws-s3-key.txt"
              secretKeyFile: "/path/to/aws-s3-secret.txt"
              hostBucket: "my-example-bucket.s3.example.com"
              s3Url: "https://my-example-bucket.s3.example.com"
              region: "us-east-1"
              acl: "private"
            sftp.example.com:
              type: "sftp"
              host: "sftp.example.com"
              user: "example_user"
              keyFile: "/config/id_example_sftp"
              pubKeyFile: "/config/id_example_sftp.pub"
```

For `s3` type remotes:
* `bucket` is the name of the S3 bucket. 
* `provider` is the S3 provider. [See rclone's S3 documenation on `--s3-provider`](https://rclone.org/s3/#s3-provider) for possible values. Optional, defaults to `AWS`.
* `accessKeyFile` is the path to a file containing the access key. Optional if `accessKey` is specified.
* `accessKey` is the value of the access key necessary to access the bucket. Ignored if `accessKeyFile` is specified.
* `secretKeyFile` is the path to a file containing the secret key. Optional if `secretKey` is specified.
* `secretKey` is the value of the access key necessary to secret the bucket. Ignored if `secretKeyFile` is specified.
* `region` is the AWS region in which the bucket resides. Required if using AWS S3, may be optional for other providers.
* `endpoint` is the S3 endpoint to use. Optional if using AWS, required for other providers.
* `acl` is the ACL with which to upload the files. [See the rclone's S3 documentation on `--s3-acl`](https://rclone.org/s3/#s3-acl) for possible values. Optional, defaults to `private`.

For `sftp` type remotes:
* `host` is the hostname of the SFTP server. Required.
* `user` is the username necessary to login to the SFTP server. Required.
* `keyFile` is the path to a file containing the SSH private key. Required.
* `pubKeyFile` si the path to a file containing the SSH public key. Required for database backups, ignored for file backups.

### Specifying backups

The `s3_backup.backups` list specifies the database backups perform, referencing the `s3_backup.sources` and `s3_backup.remotes` sections for connectivity details.

```yaml
s3_backup:
  backups:
    - name: "example.com database"
      source: "example.com"
      path: "path/in/source/bucket"
      disabled: false
      targets: []
```

Where:
* `name` is the display name of the backup. Optional, but makes the logs easier.
* `source` is the name of the key under `s3_backups.sources` from which to generate the backup. Required.
* `path` is the path in the source bucket from which to get files. Optional.
* `disabled` is `true` to disable (skip) the backup. Optional, defaults to `false`.
* `targets` is a list of remotes and additional destination information about where to upload backups. Required.

### Backup targets

Backup targets reference a key in `s3_backup.remotes`, and combine that with additional information used to upload this specific backup.

```yaml
s3_backup:
  backups:
    - name: "example.com database"
      source: "example.com"
      path: "path/in/target/bucket"
      disabled: false
      targets:
        - remote: "example-s3-bucket"
          path: "example.com/database"
          disabled: true
        - remote: "sftp.example.com"
          path: "backups/example.com/database"
          disabled: false
```

Where:
* `remote` is the key under `s3_backup.remotes` to use when uploading the backup. Required.
* `path` is the path on the remote to upload the backup. Optional.
* `disabled` is `true` to skip uploading to the specifed `remote`. Optional, defaults to `false`.

### Ping URL on completion

When a backup completes, you have the option to ping an URL via HTTP:

```yaml
s3_backup:
  backups:
    - name: "example.com database"
      source: "example.com"
      path: "path/in/target/bucket"
      healthcheckUrl: "https://pings.example.com/path/to/service"
      disabled: false
      targets:
        - remote: "example-s3-bucket"
          path: "example.com/database"
          disabled: true
        - remote: "sftp.example.com"
          path: "backups/example.com/database"
          disabled: false
```

Where:
* `healthcheckUrl` is the URL to ping when the backup completes successfully. Optional.

## Example Playbook

```yaml
    - hosts: servers
      vars:
        s3_backup:
          sources:
            their-s3-bucket:
              type: "s3"
              bucket: "their-s3-bucket"
              provider: "AWS"
              accessKeyFile: "/path/to/aws-s3-key.txt"
              secretKeyFile: "/path/to/aws-s3-secret.txt"
              region: "us-east-1"
              retryCount: 3
              retryDelay: 30
          remotes:
            example-s3-bucket:
              type: "s3"
              bucket: "my-s3-bucket"
              provider: "AWS"
              accessKeyFile: "/path/to/aws-s3-key.txt"
              secretKeyFile: "/path/to/aws-s3-secret.txt"
              hostBucket: "my-example-bucket.s3.example.com"
              s3Url: "https://my-example-bucket.s3.example.com"
              region: "us-east-1"
              acl: "private"
            sftp.example.com:
              type: "sftp"
              host: "sftp.example.com"
              user: "example_user"
              keyFile: "/config/id_example_sftp"
              pubKeyFile: "/config/id_example_sftp.pub"
            backups:
              - name: "example.com database"
                source: "example.com"
                path: "path/in/target/bucket"
                healthcheckUrl: "https://pings.example.com/path/to/service"
                disabled: false
                targets:
                  - remote: "example-s3-bucket"
                    path: "example.com/database"
                    disabled: true
                  - remote: "sftp.example.com"
                    path: "backups/example.com/database"
                    disabled: false
      roles:
         - { role: ten7.s3_backup }
```

## License

GPL v3

## Author Information

This role was created by [TEN7](https://ten7.com/).
