# postgis-wal-g
Docker image for a PostGIS database with wal-g backup and point-in-time recovery

# Usage
When you instantiate a postgis-wal-g container, pass environment variables for wal-g, see the documentation:
https://github.com/wal-g/wal-g.

E.g.

    WALG_S3_PREFIX=s3://com.mycompany.mysystem.dbbackup/
    AWS_ACCESS_KEY_ID=XXXXXXXXXXXXXXXX
    AWS_SECRET_ACCESS_KEY=XXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
    AWS_REGION=eu-central-1

or

    WALG_FILE_PREFIX=/backups/

If you use WALG_FILE_PREFIX, you should make sure the specified directory is backed up to external storage, to
make sure you can recover the backup in case the database server is lost.

You need to pass the environment variables to your container.
If you use docker-compose, We recommend that you put the variables in an env-file and 
pass a reference to that env-file to your container.
This allows you to keep your secret external storage keys only on the deployment server in a 
convenient way.

Your docker-compose.yml would then look like e.g.:

    services:
      postgis:
        env-file: wal-g.env
        ...

# Scheduling full backup
You need an external scheduler that periodically invokes the backup command on the PostGIS container.
This could be e.g. Unix/Linux cron. Your crontab entry then should be like this:

    0 1 * * * /usr/bin/docker exec -i my-db-container backup.sh >> /var/log/my-postgis-backup.log
    
# Scheduling cleanup of older backups
You may also want to clean up older backups. You can specify the number of backups to retain with env
variable `BACKUP_RETAIN_NUMBER`, which you can add to your wal-g config.
Then, add a call to backup-cleanup.sh to your schedule, e.g.:

    0 1 * * * /usr/bin/docker exec -i my-db-container sh -c 'backup.sh && backup-cleanup.sh' >> /var/log/my-postgis-backup.log


# Point-in-time recovery (PITR)

## PITR with docker-compose
Use this sequence of steps if your postgis-wal-g container is managed by docker-compose. 

It is assumed that you have one postgis-wal-g container running, and you can start/stop it with 
`docker-compose start [my-postgis-container]` / `docker-compose stop [my-postgis-container]`

- Stop any client application of your postgis container.
- Start a bash command line on the server where your postgis container runs.
- `cd` to the directory where your docker-compose.yml resides.
- Set a variable with your postgis container name:\
  `export CONTAINER=[my-postgis-container]`\
  E.g. `export CONTAINER=postgis`
- Stop the postgis container:\
  `docker-compose stop $CONTAINER`
- Copy your current data files, just in case you need them later on, if you have enough disk space left:\
  `docker-compose run --rm -v "/var/postgis-data-copy:/data-copy" $CONTAINER sh -c 'cp -Rp $PGDATA /data-copy'`
- Delete all from the postgres data directory. It must be empty for recovery:\
  `docker-compose run --rm $CONTAINER sh -c 'rm -rf $PGDATA/*'` 
- List your backups:\
  `docker-compose run --rm $CONTAINER wal-g backup-list` 
- This gives a list of backups, e.g.\
```
name                          last_modified             wal_segment_backup_start
base_000000010000000400000052 2019-06-11T01:07:18+02:00 000000010000000400000052
base_000000010000000400000082 2019-06-12T01:07:51+02:00 000000010000000400000082
base_000000010000000400000085 2019-06-13T01:07:59+02:00 000000010000000400000085
```
- Recover the desired backup. This may take a while. E.g. recovery of a 3GB backup may take 2 minutes from 
Amazon S3 storage.\
  `docker-compose run --rm $CONTAINER sh -c 'wal-g-script.sh backup-fetch $PGDATA base_000000010000000400000085'`
- Determine the timestamp that you want to recover to, in the format `2019-06-13 10:45:00+02` 
- Create a postgres recovery config file. Replace the `recovery_target_time` timestamp with your timestamp:
```
docker-compose run --rm  $CONTAINER sh -c "
cat <<EOF > /var/lib/postgresql/data/recovery.conf
restore_command = 'wal-g-script.sh wal-fetch %f %p'
standby_mode = off
recovery_target_action = 'promote'
recovery_target_time = '2019-06-13 10:45:00+02'
recovery_target_timeline = latest
EOF"
```
- Run the recovery (in a temporary container):\
  `docker-compose run --rm $CONTAINER`
- Start the postgis container:\
  `docker-compose start $CONTAINER`
- Start your application