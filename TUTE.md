## Setup dapp tools

This guide differs slightly from the official setup guide to enable a more portible development system. We want to work in a disposable environment so the best thing is to use docker to manage dapptools. The main tradeoff we are making is that dapptools uses the root user within the Docker container to write files but this can easily be addressed by using a cleanup script that resets file permissions.

The components of our installation

| file                 | desc                                                        |
| -------------------- | ----------------------------------------------------------- |
| `Dockerfile`         | Dockerfile NIX/dapptools environment                        |
| `docker-compose.yml` | Coordinate volumes and services we need for the environment |

### Dockerfile

To set it up we can create a simple docker file:

```Dockerfile
FROM nixos/nix

RUN mkdir -p /app
RUN mkdir -p /nix

ENV USER=me
ENV UID=1000
ENV GID=1000

RUN addgroup -S -g $GID $USER && adduser -S -u $UID $USER -G $USER
RUN chown -R me:me /app
RUN chown -R me:me /nix

RUN apk --no-cache add curl jq git

USER $USER
WORKDIR /app

RUN echo $(curl -sS https://api.github.com/repos/dapphub/dapptools/releases/latest | jq -r .tarball_url) > version
RUN nix-env -iA dapp -f $(cat version) && \
nix-env -iA seth -f $(cat version) && \
nix-env -iA hevm -f $(cat version) && \
nix-env -iA ethsign -f $(cat version)

RUN nix-env -f https://github.com/dapphub/dapptools/archive/master.tar.gz -iA solc-static-versions.solc_0_8_6
RUN git config --global user.email ""
RUN git config --global user.name "anon"

ENV LANG=en_GB.UTF-8
ENV PATH=/home/me/.nix-profile/bin:$PATH

```

This defines our dapptools environment and allows us to run dapptools within Docker. That means we can have a portable installation all you need is docker as opposed to an the esoteric nix installation.

One caveat is that this Dockerfile assumes you are the user 1000 and that you have the group 1000. If you have different user ids you will want to pass args to your docker build.

## docker-compose.yml

Docker compose makes it easy to interact with our dockerfile.

```yaml
services:
  dt:
    build: .
    volumes:
      - ..:/app
    user: me
```

Once you have created the Dockerfile and the `docker-compose.yml` file run the following command to build your env localy. This will take time on first run.

```bash
docker-compose build
```

Once built you can check everything is working by running:

```bash
docker-compose run dt dapp --version
```

If it is working you should receive output like the following:

```bash
Creating dapptools_dt_run ... done
dapp 0.35.0
solc, the solidity compiler commandline interface
Version: 0.8.6+commit.11564f7e.Linux.g++
hevm 0.49.0
```

## Setting up remappings

Dapp tools works best with remappings so that imports are remapped to locally installed modules. You will want to use the dapp tools `ds-test` module for testing. We should add remappings to compensate for this.

First we create our `.dapprc` file to hold our env vars:

```
export DAPP_REMAPPINGS=$(cat remappings.txt)
```

Then we create our `remappings.txt` file:

```
ds-test/=lib/ds-test/src/
```

This means we can import files directly from the ds-test module using `ds-test`

## Setting up gitignore

create a simple gitignore

```
/node_modules
/out
```

## Create a git repo

```bash
git init
git add -A
git commit -m 'init'
```

## Install the ds-test module

Ensure your git repo is clean.

```bash
docker-compose run dt dapp install ds-test
```

## Create a contract to test

Create a folder for our contracts:

```
mkdir -p src
```

Create a simple contract:

```solidity
// SPDX-License-Identifier: GPL-3.0-or-later
pragma solidity ^0.8.6;

contract App {
    string greeting;

    constructor(string memory _greeting) {
        greeting = _greeting;
    }

    function greet() public view returns (string memory) {
        return greeting;
    }
}

```

And a simple test:

```solidity
// SPDX-License-Identifier: GPL-3.0-or-later
pragma solidity ^0.8.6;

import "ds-test/test.sol";

import "./App.sol";

contract AppTest is DSTest {
    App app;

    function setUp() public {
        app = new App("Hello World");
    }

    function test_basic_sanity() public {
        assertEq(app.greet(), "Hello World");
    }
}

```

Run the test command:

```
docker-compose run dt dapp test
```
