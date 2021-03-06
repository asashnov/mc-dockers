# Moolticute docker for building windows release

This docker is used by to build/package moolticute for windows.

# Usage

Download the latest image from docker hub
```bash
➜ docker pull mooltipass/mc-win-builder
Using default tag: latest
latest: Pulling from mooltipass/mc-win-builder
Digest: sha256:5c7be97f9d27853455d96d05cedcacbe89b276706e21cdc9a9ed67047e598ff3
Status: Image is up to date for mooltipass/mc-win-builder:latest
```

Clone the moolticute source repository somewhere on you hdd.
```bash
➜ git clone https://github.com/mooltipass/moolticute
[...]
```

Then you need to tell docker where to map the source code of moolticute.

Start the docker
```bash
➜ docker run -t --name mcbuilder -d -v ABSOLUTE_PATH_TO_YOUR_SRC/moolticute:/moolticute mooltipass/mc-win-builder
f0b28a4f15e940968a317dbb6427a4aef0e869d4d3553fe0d30592c97457882d
```

The docker is running, you can check with:
```bash
➜ docker ps
CONTAINER ID IMAGE COMMAND CREATED STATUS PORTS NAMES
f0b28a4f15e9 mooltipass/mc-win-builder "/start.sh /start" About a minute ago Up About a minute mcbuilder
```

Now it's time to start the build:
```bash
➜ docker exec mcbuilder /bin/bash /scripts/build.sh
[...]
```

And finally to create the packages:
```bash
➜ docker exec mcbuilder /bin/bash /scripts/package.sh
[...]
```

Packages are now available in your moolticute source dir (in the packages folder)
