cronlock
========

Use a central redis instance to globally lock cronjobs across a distributed system

## Notes

 - Follows locking logic from Redis documentation at http://redis.io/commands/setnx
 - requires a `bash` with `/dev/tcp` enabled. Older Debian/Ubuntu systems disable `/dev/tcp`.
 - requires `md5`

## Install

```bash
wget -O /usr/bin/cronlock --no-check-certificate https://raw.github.com/kvz/cronlock/master/cronlock
chmod 755 /usr/bin/cronlock
```

## Options

 - `CRONLOCK_CONFIG` location of config file. this is optional since all config can also be
 passed as environment variables. default: `/etc/cronlock.conf`, `<DIR>/cronlock.conf`
 - `CRONLOCK_HOST` the redis hostname. default: `localhost`
 - `CRONLOCK_PORT` the redis hostname. default: `6379`
 - `CRONLOCK_GRACE` determines how long a lock should at least persist. default is 40s: `40`.
 This is too make sure that if you have a very small job, and clocks aren't in sync, the same job
 on server2 (slightly behind in time) will just fire right after server1 finished it.
 - `CRONLOCK_RELEASE` determines how long a lock can persist at most. acts as a failsafe so there can be no locks that persist forever in case of failure. default is a day: `86400`
 - `CRONLOCK_KEY` a unique key for this command in the global redis instance. default: a hash of cronlock's arguments
 - `CRONLOCK_PREFIX` redis key prefix used by all keys. default: `cronlock.`
 - `CRONLOCK_VERBOSE` set to `yes` to print debug messages. default: `no`


## Examples

### Test

Just executes `ls al`:

```bash
git clone git://github.com/kvz/cronlock.git
cd cronlock
./cronlock ls -al
```

### Single box

```bash
crontab -e
* * * * * cronlock ls -al
```

In this configuration, `ls -al` will be launched every minute. If the previous
`ls -al` has not finished yet, another one is not started.
This works on 1 server, as the default `CRONLOCK_HOST` of `localhost` is used.

In this setup, `cronlock` works much like Tim Kay's awsome [solo](https://github.com/timkay/solo),
except `cronlock` requires redis, so I recommend using Tim Kay's solution here.

### Distributed

```bash
crontab -e
* * * * * CRONLOCK_HOST=redis.mydomain.com cronlock ls -al
```

In this configuration, a central redis instance is used to track the locking for
`ls -al`. So now you can safely assume that throughout a cluster of 100 servers,
just one instance of `ls -al` is ran every minute. No less, no more.

### Commands with different arguments

By default cronlock uses your command and it's arguments to make a unique identifier
by which the global lock is acquired. However if you want to run: `ls -al` or `ls -a`, but just 1 instance of either, you\'ll want to provide your own key:

```bash
crontab -e
* * * * * CRONLOCK_KEY="ls" cronlock ls -al
* * * * * CRONLOCK_KEY="ls" cronlock ls -a
```

### Per application

```bash
crontab -e
* * * * * CRONLOCK_PREFIX="app1.lock." CRONLOCK_KEY="ls" cronlock ls -al
* * * * * CRONLOCK_PREFIX="app1.lock." CRONLOCK_KEY="ls" cronlock ls -a

* * * * * CRONLOCK_PREFIX="app2.lock." CRONLOCK_KEY="ls" cronlock ls -al
* * * * * CRONLOCK_PREFIX="app2.lock." CRONLOCK_KEY="ls" cronlock ls -a
```

## Exit codes

 - `200` Success (delete succeeded or lock not acquired, but normal execution)
 - `201` Failure (cronlock error)
 - `< 200` Success (acquired lock, executed your command), returns the exit code of your command
