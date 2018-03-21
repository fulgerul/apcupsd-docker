APCUPSd-Docker
==============

This Docker container connects to the local APC UPS or a remote apcupsd instance. Even if running in a container it can notify the host and trigger shutdown (or other) actions, if needed. All without special privileges.

It can also be used for any other arbitrary commands and every apcupsd trigger action.

See `Configuration Example` below.

Requirements
------------

- Bash

Usage
-----

```sh
docker run -v /tmp/apcupsd-docker:/tmp/apcupsd-docker gersilex/apcupsd-docker
```

You can read the status from the stdout output, as the container starts `apcupsd -b` and shows INFO loglevel information.

The `/etc/apcupsd/doshutdown` script will be executed when a condition (Low Battery, Low Lifetime left, Timeout exceeded) is reached while being in battery operation (See `/etc/apcupsd/apcupsd.conf` for more information and tweaking).

To be able do signal the Docker host on which the container runs, you should either map your local device into the container (if the UPS is connected to this host) or map a folder from your host into the container and use the included `doshutdown` script. The script will write a `1` into a file with the name `trigger` in that folder. Monitor it on the host with cron or inotify and gracefully shut down your server, when the content is `1`.
Don't forget to remove the file before shutdown to omit shutdown-loops after booting again.

The `host-trigger-check.sh` contains a cron-compatible script that will run an included bash function, if it reads a '1' in the `/tmp/apcupsd-docker/trigger` file on the host. Read the shell script for instructions on how to use it. It's recommended to run this every minute.

Configuration Example
---------------------

An example `apcupsd.conf` file is provided, along with the original comments. Minimal working example:

- UPS connected to a remote apcupsd instance on the host `nas`. Will execute `doshutdown` if 5% battery is left or 10 minutes remaining. Docker host has cron running and checks for shutdown flag every minute.

```apache
# /etc/apcupsd/apcupsd.conf

UPSTYPE net
DEVICE nas:3551
BATTERYLEVEL 5
MINUTES 10
```

```sh
# /root/apcupsd/host-trigger-check.sh (excerpt)
[...]

TRIGGERFILE="/tmp/apcupsd-docker/trigger"

# Put everything you want to do on a shutdown condition instide this function.
action(){
  echo "Detected '1' in '$TRIGGERFILE'."

  # Plan shutdown in 5 minutes
  shutdown -P +5

  echo "Stopping all Docker containers..."
  docker -q | xargs --no-run-if-empty docker stop --time 300

  # Shutdown now, if we finish early with previous command
  shutdown -P now
}

[...]
```

```sh
# /var/spool/cron/gersilex (generated by 'crontab -e')

* * * * * /root/apcupsd/host-trigger-check.sh
```

License
-------

MIT