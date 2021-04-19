# OpenBadge Hub (Python)

The (python) hub is used for controlling, monitoring, and communicating with the badges. It is written in Python and tested with Python version 2.7 (specifically 2.7.16 on Raspbian Buster).

The hub uses [Docker](https://www.docker.com/) and [Docker Compose](https://docs.docker.com/compose/) to simplify setup and deployment.  Currently the only approach for installing Docker on Raspbian is to use the [Docker convenience script](https://docs.docker.com/engine/install/debian/#install-using-the-convenience-script).  Use `pip` (and specifically `pip3`) from the "Alternative install options" to [install Docker Compose](https://docs.docker.com/compose/install/#install-compose) on Raspbian: `pip3 install docker-compose`.

There are two main modes of operation - 
1. [**Standalone mode**](#standalone-mode) - in standalone mode the hub loads a list of badges from a static files and writes data to local files. This mode is useful for testing, development, and when you only need a single hub.
2. [**Server mode**](#server-mode) - in server mode the hub communicates with an [openbadge-server](https://github.com/stottlerhenke-seattle/openbadge-server). It will read the list of badges and send back information such as badge health and collected data.

**Important notes:**
* When using multiple hubs, it's important to set up NTP properly so that the time on the hubs is synchronized. Unsynchronized clocks will lead to significant data loss and data corruption
* When running the hub on Raspberry Pi, the hub will modify system parameters in order to (significantly) speed up Bluetooth communication. These settings might affect the communication with other Bluetooth devices, and therefore we do not apply these changes to non-raspberry pi machines
* Docker uses different base images for Ubuntu and Raspbian, and therefore there are different `.yml` files for different operating systems


## Standalone mode
To use the hub in **standalone** mode, you need to create two files under the `config` folder:
* `devices.txt` - list of member badges. These badges will collect proximity and audio data
* `devices_beacon.txt` - list of beacons badges. These badges will only broadcast their IDs
* The `.env` file used in [server mode](#server-mode) is also required (but not used) in standalone mode

You can refer to [`config/devices.txt.example`](config/devices.txt.example) and [`config/devices_beacon.txt.example`](config/devices_beacon.txt.example) as a reference for the file structure.

**Important notes:**
* Make sure you provide all members and beacons with the same project id (otherwise they will not see each other)
* By convention, member badges should have an id between 1 and 15999. Beacon ids should be between 16000 and 32000
* Choose different project ids for different projects that are running at the same time. This will prevent id conflicts
* For project id, pick a number between 1 and 254. Avoid using project id 0 since this is the default value for ibeacons

Next, use `docker-compose` to run the hub:

* For [**development**](#development-configurations) on linux, mac or windows:
  ```
  docker-compose -f dev_ubuntu.yml up
  ```
* For [**development**](#development-configurations) on a Raspberry Pi (Raspbian Buster):
  ```
  docker-compose -f dev_raspbian.yml up
  ```
* For **production** on a Raspberry Pi (Raspbian Buster), override the default `docker-compose` command parameters to specify `standalone` mode.
  ```
  docker-compose run hub-raspbian -m standalone pull
  ```
  However, **the production configuration does not easily run standalone mode** because it uses a docker volume for the `config` folder, which must be seeded with `devices.txt` and `devices_beacon.txt`.


## Server mode
To use the hub in **server** mode, create a `.env` file (use the [`env.example`](env.example) file from the root directory as a template) and change the following settings:
  * `HUB_UUID` : a unique identifier for this hub.  If this setting is missing, the hub will use the machine's `hostname`.  This identifier is used by the [openbadge-server](https://github.com/stottlerhenke-seattle/openbadge-server) to restrict access to only known hub devices.
  * `BADGE_SERVER_ADDR` : server address (e.g. my.server.com)
  * `BADGE_SERVER_PORT` : server port
  * `APPKEY` : application authentication key (needs to match `APPKEY` in your [openbadge-server](https://github.com/stottlerhenke-seattle/openbadge-server) `.env` configuration)

Next, use `docker-compose` to run the hub.
* For [**development**](#development-configurations) on linux, mac or windows, override the default `docker-compose` command parameters to specify `server` mode:
  ```
  docker-compose -f dev_ubuntu.yml run hub-ubuntu-dev -m server pull
  ```
* For [**development**](#development-configurations) on a Raspberry Pi (Raspbian Buster), override the default `docker-compose` command parameters to specify `server` mode:
  ```
  docker-compose -f dev_raspbian.yml run hub-raspbian-dev -m server pull
  ```
* For **production**, running in the background on a Raspberry Pi (Raspbian Buster):
  ```
  # build after any changes because the production configuration does not mount the local directories
  docker-compose build
  docker-compose up -d
  ```

In order to get an **interactive shell** for the hub container, you need to overwrite the entrypoint.  For example:
```
# Ubuntu
docker-compose -f dev_ubuntu.yml run --entrypoint /bin/bash hub-ubuntu-dev

# Raspbian development configuration
docker-compose -f dev_raspbian.yml run --entrypoint /bin/bash hub-raspbian-dev

# Raspbian production configuration
docker-compose run --entrypoint /bin/bash hub-raspbian
```

To attach to an interactive shell for an already-running hub container:
```
docker exec -it CONTAINER_NAME /bin/bash
```


## Development configurations
The default [`docker-compose.yml`](docker-compose.yml) configuration uses a **production** configuration for Raspbian ([`Dockerfile_raspbian_prod`](compose/openbadge-hub-py/Dockerfile_raspbian_prod)), which runs in [server mode](#server-mode).  There are two **development** configuration files for both Raspbian ([`dev_raspbian.yml`](dev_raspbian.yml)) and Ubuntu ([`dev_ubuntu.yml`](dev_ubuntu.yml)).

The development configuration files are different in that they:
* Default to [standalone mode](#standalone-mode).
* Mount the local `src`, `data`, `logs` and `config` directories as volumes in docker. This allows
faster builds and easier access to the data generated in this mode.


## Miscellaneous
### Notes on BlueZ
One of the main requirements for the hub is the BlueZ library. Newer version of Ubuntu and Raspbian include a somewhat
new version of BlueZ, so just make sure you use a recent release and ensure BlueZ version is >= 5.29. To check which
version you are using, type:
```
bluetoothd --version
```
