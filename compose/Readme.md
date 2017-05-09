Galaxy Docker Compose
=====================

# Table of Contents <a name="toc" />

- [Usage](#Usage)
- [Build](#Build)
- [Roadmap](#Roadmap)
- [Advanced](#Advanced)
  - [postgres](#postgres)
    - [Configuration](#postgres-Configuration)
  - [proftpd](#proftpd)
    - [Configuration](#proftpd-Configuration)
  - [slurm](#slurm)
    - [Configuration](#slurm-Configuration)
  - [galaxy-init](#galaxy-init)
    - [Configuration](#galaxy-init-Configuration)

# Usage <a name="Usage" /> [[toc]](#toc)

At first you need to install docker with compose.
- [docker](https://docs.docker.com/installation/)
- [docker-compose](https://docs.docker.com/compose/install/)

# Build <a name="Build" /> [[toc]](#toc)

Checkout the repository:
```
git checkout https://github.com/bgruening/docker-galaxy-stable.git
cd docker-galaxy-stable/compose
```

Build the compose containers:
```sh
./buildlocal.sh
```
After successful installation you can run the compose file:
```sh
docker-compose up -d
```


# Roadmap <a name="Roadmap" /> [[toc]](#toc)

We wish to split the monolithic container into its components, i.e. postgres, proftpd, slurm, nginx (more?)
So far postgres and proftpd are working.

# Advanced <a name="Advanced" /> [[toc]](#toc)

## postgres <a name="postgres" /> [[toc]](#toc)
You can theoretically use any external database. The included postgres build is based on the [official postgres container](https://hub.docker.com/_/postgres/) and adds an initialization script which loads a vanilla dump of the initial database state on first startup which is faster than initializing by the migration script.

In case you want to generate an initial database dump yourself:
```sh
./dumpsql.sh
./buildlocal.sh
```

To manually initialize a database without the dump connected via docker-compose.yml, before starting the galaxy container you can run:
```sh
docker-compose run galaxy-web install_db.sh
```
which will perform database migration.

### Updating
To update the database to a new migration level, run 
```sh
docker-compose run galaxy-web install_db.sh
```

### Configuration <a name="postgres-Configuration" /> [[toc]](#toc)
See [official postgres container](https://hub.docker.com/_/postgres/).

## proftpd <a name="proftpd" /> [[toc]](#toc)

Proftpd uses the [ansible galaxy extras project](https://github.com/galaxyproject/ansible-galaxy-extras) to configurate proftpd. Remark: The proftp server is configured to only allow uploads of new files, as it is not supposed to be used as a file sharing server.

### Configuration <a name="proftpd-Configuration" /> [[toc]](#toc)

- *proftpd\_db\_connection*=_database@host_: Configurates the database name and hostname. Hostname can be a linked database container.
- *proftpd\_db\_username*=_dbuser_: User in the database.
- *proftpd\_db\_password*=_dbpass_: Password of the user in the database.
- *proftpd\_files\_dir*=_/export/ftp_: Directory where the user files are to be placed. Should be synchronized with volumes.
- *proftpd\_sql\_auth\_type*=_SHA1_: Authentication type used in the galaxy database (SHA1|PBKDF2).
- *proftpd\_welcome*=_Public Galaxy FTP_: Welcome message.
- *proftpd\_passive\_port\_low*=_30000_, *proftpd\_passive\_port\_high*=_40000_: Passive mode port range. This should be kept small for docker exposing the range (30000-30010 or so), because docker allocates each port separately which makes the process stale. Should be same as exposed ports.
- *proftpd\_use\_sftp*=_false_: Enable sftp.
- *proftpd\_nat\_masquerade*=_false_: Set masquearade to true if host is NAT'ed.
- *proftpd\_masquerade\_address*=_ip_: Refers to the ip that clients use to establish an ftp connection. Can be a command that returns an IP or an IP address and applies only if proftpd\_nat\_masquerade is true. `\`ec2metadata --public-ipv4\`` returns the public ip for amazon's ec2 service.

## slurm <a name="slurm" /> [[toc]](#toc)

The slurm container is an example of how to use an external slurm cluster. The container is set up with a pre-installed virtual python environment ready for galaxy. The default galaxy job_conf.xml is compatible with this container, but will require change for your own external cluster or a more advanced setup.

### Configuration <a name="slurm-Configuration" /> [[toc]](#toc)
See [Running jobs outside of the container](https://github.com/bgruening/docker-galaxy-stable/blob/master/docs/Running_jobs_outside_of_the_container.md) and [Using an external Slurm cluster](https://github.com/bgruening/docker-galaxy-stable#using-an-external-slurm-cluster--toc).

## galaxy-init <a name="galaxy-init" /> [[toc]](#toc)

This container is required to initialize the /export directory structure of the galaxy worker.
On startup, this container will copy all missing directories to /export. In order to update galaxy, simply delete /export/galaxy-central and it will be reinitialized with the current version.
When initialization is complete, this container notifies the galaxy handlers to start up by locking /export/.initdone. You can disable this mechanism by setting DISABLE_SLEEPLOCK=true.

### Configuration <a name="galaxy-init-Configuration" /> [[toc]](#toc)
- *DISABLE\_SLEEPLOCK*="": Disable sleeplock mechanism.


## galaxy-web <a name="galaxy-web" /> [[toc]](#toc)

This container runs the actual webhandler.
As of now, this container also runs nginx, job handlers, cron, docker.
This container will wait until it is notified via the lock on /export/.initdone. You can disable this mechanism by setting NONUSE=sleeplock.

### Configuration <a name="galaxy-web-Configuration" /> [[toc]](#toc)

`/export/` directory structure:
- `galaxy-central/`: Main directory containing the galaxy installation. Configurations and data are symlinked to their export directories.
- `.distribution_config/`: Contains the most current example configuration files (useful for updating/comparing to current configuration in `config/`)
- `config/`: Configuration files for galaxy (symlink from `galaxy-central/config`)
- `database/`: Database directory (symlink from `galaxy-central/database`)
- `display_applications/`: App storage (symlink from `galaxy-central/display_applications`)
- `tool-data/`: Data for tools (symlink from `galaxy-central/tool-data`)
- `tools/`: Tools storage (symlink from `galaxy-central/tools`)

- `ftp/`: Storage for ftp uploads (see [proftpd](#proftpd))
- `nginx_upload_store/`: Temporary upload store for upload from web (required for large uploads)
- `shed_tools/`: Data for shed tools
- `tool_deps/`: Dependencies for tools
- `venv/`: The python virtual environment for galaxy and all jobs
- `welcome/`: Contains files for the welcome page (static content)

- `.initdone`: Lockfile for galaxy-init and galaxy-web synchronization (see *DISABLE\_SLEEPLOCK*)
- `var/`: In case docker is made available, it will be mounted here (TODO)
- `munge.key`: Munge key for slurm
- `slurm.conf`: Configuration for slurm

TODO: Configuration Variables
