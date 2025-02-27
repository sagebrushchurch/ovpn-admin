# ovpn-admin

Simple web UI to manage OpenVPN users, their certificates & routes in Linux. While backend is written in Go, frontend is based on Vue.js.

Originally created in [Flant](https://flant.com/) for internal needs & used for years, then updated to be more modern and [publicly released](https://blog.flant.com/introducing-ovpn-admin-web-interface-for-openvpn/) in March'21. Your contributions are welcome!

## Features

* Adding OpenVPN users (generating certificates for them);
* Revoking/restoring users certificates;
* Generating ready-to-user config files;
* Providing metrics for Prometheus, including certificates expiration date, number of (connected/total) users, information about connected users;
* (optionally) Specifying CCD (`client-config-dir`) for each user;
* (optionally) Operating in a master/slave mode (syncing certs & CCD with other server);
* (optionally) Specifying/changing password for additional authorization in OpenVPN;
* (optionally) Specifying the Kubernetes LoadBalancer if it's used in front of the OpenVPN server (to get an automatically defined `remote` in the `client.conf.tpl` template).

### Screenshots

Managing users in ovpn-admin:
![ovpn-admin UI](https://raw.githubusercontent.com/flant/ovpn-admin/master/img/ovpn-admin-users.png)

An example of dashboard made using ovpn-admin metrics:
![ovpn-admin metrics](https://raw.githubusercontent.com/flant/ovpn-admin/master/img/ovpn-admin-metrics.png)

## Installation

### Disclaimer

This tool uses external calls for `bash`, `coreutils` and `easy-rsa`, thus **Linux systems only are supported** at the moment.

### 1. Docker

There is a ready-to-use [docker-compose.yaml](https://github.com/flant/ovpn-admin/blob/master/docker-compose.yaml), so you can just change/add values you need and start it with [start.sh](https://github.com/flant/ovpn-admin/blob/master/start.sh).

Requirements. You need [Docker](https://docs.docker.com/get-docker/) and [docker-compose](https://docs.docker.com/compose/install/) installed.

Commands to execute:

```bash
git clone https://github.com/flant/ovpn-admin.git
cd ovpn-admin
./start.sh
```

### 2. Building from source

Requirements. You need Linux with the following components installed:
- [golang](https://golang.org/doc/install)
- [packr2](https://github.com/gobuffalo/packr#installation)
- [nodejs/npm](https://nodejs.org/en/download/package-manager/)

Commands to execute:

```bash
git clone https://github.com/flant/ovpn-admin.git
cd ovpn-admin
./bootstrap.sh
./build.sh
./ovpn-admin 
```

(Please don't forget to configure all needed params in advance.)

### 3. Prebuilt binary (WIP)

You can also download and use prebuilt binaries from the [releases](https://github.com/flant/ovpn-admin/releases) page — just choose a relevant tar.gz file.

To use password authentication (the `--auth` flag) you have to install [openvpn-user](https://github.com/pashcovich/openvpn-user/releases). This tool should be available in your `$PATH` and its binary should be executable (`+x`).

## Usage

```
usage: ovpn-admin [<flags>]

Flags:
  --help                       show context-sensitive help (try also --help-long and --help-man)

  --listen.host="0.0.0.0"      host for ovpn-admin
  (or $OVPN_LISTEN_HOST)

  --listen.port="8080"         port for ovpn-admin
  (or $OVPN_LISTEN_PROT)

  --role="master"              server role, master or slave
  (or $OVPN_ROLE)

  --master.host="http://127.0.0.1"  
  (or $OVPN_MASTER_HOST)       URL for the master server

  --master.basic-auth.user=""  user for master server's Basic Auth
  (or $OVPN_MASTER_USER)
 
  --master.basic-auth.password=""  
  (or $OVPN_MASTER_PASSWORD)   password for master server's Basic Auth

  --master.sync-frequency=600  master host data sync frequency in seconds
  (or $OVPN_MASTER_SYNC_FREQUENCY)

  --master.sync-token=TOKEN    master host data sync security token
  (or $OVPN_MASTER_TOKEN)

  --ovpn.network="172.16.100.0/24"  
  (or $OVPN_NETWORK)           NETWORK/MASK_PREFIX for OpenVPN server

  --ovpn.server=HOST:PORT:PROTOCOL ...  
  (or $OVPN_SERVER)            HOST:PORT:PROTOCOL for OpenVPN server
                               can have multiple values

  --ovpn.server.behindLB       enable if your OpenVPN server is behind Kubernetes
  (or $OVPN_LB)                Service having the LoadBalancer type

  --ovpn.service="openvpn-external"  
  (or $OVPN_LB_SERVICE)        the name of Kubernetes Service having the LoadBalancer
                               type if your OpenVPN server is behind it

  --mgmt=main=127.0.0.1:8989 ...  
  (or $OVPN_MGMT)              ALIAS=HOST:PORT for OpenVPN server mgmt interface;
                               can have multiple values

  --metrics.path="/metrics"    URL path for exposing collected metrics
  (or $OVPN_METRICS_PATH)

  --easyrsa.path="./easyrsa/"  path to easyrsa dir
  (or $EASYRSA_PATH)

  --easyrsa.index-path="./easyrsa/pki/index.txt"  
  (or $OVPN_INDEX_PATH)        path to easyrsa index file

  --ccd                        enable client-config-dir
  (or $OVPN_CCD)

  --ccd.path="./ccd"           path to client-config-dir
  (or $OVPN_CCD_PATH)

  --templates.clientconfig-path=""  
  (or $OVPN_TEMPLATES_CC_PATH) path to custom client.conf.tpl

  --templates.ccd-path=""      path to custom ccd.tpl
  (or $OVPN_TEMPLATES_CCD_PATH)

  --auth.password              enable additional password authorization
  (or $OVPN_AUTH)

  --auth.db="./easyrsa/pki/users.db"
  (or $OVPN_AUTH_DB_PATH)      database path for password authorization

  --debug                      enable debug mode
  (or $OVPN_DEBUG)

  --verbose                    enable verbose mode
  (or $OVPN_VERBOSE)
 
  --version                    show application version
```

## Further information

Please feel free to use [issues](https://github.com/flant/ovpn-admin/issues) and [discussions](https://github.com/flant/ovpn-admin/discussions) to get help from maintainers & community.
