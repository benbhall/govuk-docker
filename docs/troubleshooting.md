# Troubleshooting

## Diagnose common issues when setting up GOV.UK Docker

Run the following command in `~/govuk/govuk-docker`. Since this script makes use of Ruby Gems, you will need to [install some additional dependencies](../CONTRIBUTING.md#testing) in order to do this.

```
bin/doctor
```

## Diagnose database issues with a project/app not making

If you run `make` on an app inside govuk-docker and face any of the following database issues:

- `ActiveRecord::Migration` is not supported
- duplicate key error

...then it is likely that there is a conflict between the app and one of your existing volumes. This is because database contents are stored in a volume that the container uses, so when the container is recreated, it picks up the same data again.

The easiest resolution is to drop the database, e.g. `govuk-docker run content-store-lite bundle exec rails db:mongoid:drop`, and then try to `make` again.

A blunter solution would be to drop the volume altogether, by running `docker volume rm VOLUME_NAME`, where `VOLUME_NAME` can be derived from `docker volume ls`.

## Diagnose general issues with a project/app not working

* Make sure you run all commands via GOV.UK Docker.

```
govuk-docker-run bundle install
govuk-docker-run rake db:migrate
```

* Try repeating the action. If you got a `GdsApi::HTTPUnavailable` or `GdsApi::TimedOutException` the first time around, it could mean that Publishing API wasn't ready in time, as unfortunately there's no way for govuk-docker to know when each dependency is ready.

* Check if one of the dependencies is the problem.

> A common problem for dependencies is when you've previously `git pull`ed the repo, but haven't run `bundle install` or `rake db:migrate`. The logs for the dependency will show if this is the problem.

```
# check if any dependencies have exited
docker ps -a

# check logs for an exited dependency
govuk-docker logs -f publishing-api-app
```

* Try cleaning up and running your command again.

```
# stop all apps and their dependencies
govuk-docker down

# make sure GOV.UK Docker is up-to-date
git pull
```

## Diagnose issues with `dev.gov.uk` domains not resolving

* Check if `dev.gov.uk` works end-to-end

```
dig app.dev.gov.uk @127.0.0.1

# output should contain...
# app.dev.gov.uk.		0	IN	A	127.0.0.1
```

* Check your `/etc/resolver` config is working

```
scutil --dns

# output should contain...
# domain   : dev.gov.uk
# nameserver[0] : 127.0.0.1
# port     : 53
# flags    : Request A records, Request AAAA records
# reach    : 0x00030002 (Reachable,Local Address,Directly Reachable Address)
```

* Re-run the setup script and look for errors

```
bin/setup
```

## Resolve issues caused by an existing Docker installation

You may get one of the following errors when running `bin/setup`.

```
Error: The `brew link` step did not complete successfully
The formula built, but is not symlinked into /usr/local
Could not symlink bin/docker-compose
Target /usr/local/bin/docker-compose
...
```

```
Error: It seems there is already an App at '/Applications/Docker.app'.
```

This isn't a problem if you already have Docker/Compose installed, and the setup script will continue to run. If you like, you can remove your existing Docker/Compose and run `bin/setup` again.

## Cannot `rails db:prepare` or start console due to "Plugin caching_sha2_password could not be loaded"

In MySQL 8.0 `caching_sha2_password` was made the default over the previous `mysql_native_password`.

This can lead to the following error when ActiveRecord is attempting to connect to the database, for example when running `rails db:prepare` or trying to bring up a rails console.

```
ActiveRecord::ConnectionNotEstablished: Plugin caching_sha2_password could not be loaded: /usr/lib/x86_64-linux-gnu/mariadb19/plugin/caching_sha2_password.so: cannot open shared object file: No such file or directory
```

A workaround is to get MySQL to fall back to using `mysql_native_password` as follows:

- Check that you can see `govuk-docker_mysql-8_1` when running `govuk-docker ps`, if not you will need to start a service that uses mysql (for example Whitehall).
- Bring up a mysql console inside the container: `docker exec -it govuk-docker_mysql-8_1 mysql --user=root --password=root`
- Alter the way the root user identifies itself. `ALTER USER 'root' IDENTIFIED WITH mysql_native_password BY 'root';`
