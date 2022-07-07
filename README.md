# indigo_nakamoto/omnilite

A litecoin-core docker image with support for the following platforms:

* `arm32v7` (armv7)
* `arm64` (aarch64, armv8)

## Tags

- `0.18.1`, `0.18`, ([0.18/Dockerfile](https://github.com/indigonakamoto/docker-omnilite/blob/master/0.18/Dockerfile))
- `0.18.1-alpine`, `0.18-alpine` ([0.18/alpine/Dockerfile](https://github.com/indigonakamoto/docker-omnilite/blob/master/0.18/alpine/Dockerfile))


**Multi-architecture builds**

The newest images (Debian-based, *0.19+*) provide built-in support for multiple architectures. Running `docker pull` on any of the supported platforms will automatically choose the right image for you as all of the manifests and artifacts are pushed to the Docker registry.

**Picking the right tag**

<!-- - `ruimarinho/bitcoin-core:latest`: points to the latest stable release available of Bitcoin Core. Caution when using in production as blindly upgrading Bitcoin Core is a risky procedure.
- `ruimarinho/bitcoin-core:alpine`: same as above but using the Alpine Linux distribution (a resource efficient Linux distribution with security in mind, but not officially supported by the Bitcoin Core team — use at your own risk).
- `ruimarinho/bitcoin-core:<version>`: based on a slim Debian image, this tag format points to a specific version branch (e.g. `0.20`) or release of Bitcoin Core (e.g. `0.20.1`). Uses the pre-compiled binaries which are distributed by the Bitcoin Core team.
- `ruimarinho/bitcoin-core:<version>-alpine`: same as above but using the Alpine Linux distribution. -->

## What is OmniLite?

OmniLite is a reference client that implements the Omni protocol for remote procedure call (RPC) use. 

## Usage

### How to use this image

This image contains the main binaries from the Litecoin Core project - `litecoind`, `litecoin-cli` and `litecoin-tx`. It behaves like a binary, so you can pass any arguments to the image and they will be forwarded to the `litecoind` binary:

```sh
❯ docker run --rm -it litecoin-project/litecoin-core \
  -printtoconsole \
  -regtest=1 \
  -rpcallowip=172.17.0.0/16 \
  -rpcauth='foo:7d9ba5ae63c3d4dc30583ff4fe65a67e$9e3634e81c11659e3de036d0bf88f89cd169c1039e6e09607562d54765c649cc'
```

_Note: [learn more](#using-rpcauth-for-remote-authentication) about how `-rpcauth` works for remote authentication._

By default, `litecoind` will run as user `litecoin` for security reasons and with its default data dir (`~/.litecoin`). If you'd like to customize where `litecoin-core` stores its data, you must use the `LITECOIN_DATA` environment variable. The directory will be automatically created with the correct permissions for the `litecoin` user and `litecoin-core` automatically configured to use it.

```sh
❯ docker run --env LITECOIN_DATA=/var/lib/litecoin-core --rm -it litecoin-project/litecoin-core \
  -printtoconsole \
  -regtest=1
```

You can also mount a directory in a volume under `/home/litecoin/.litecoin` in case you want to access it on the host:

```sh
❯ docker run -v ${PWD}/data:/home/litecoin/.litecoin -it --rm litecoin-project/litecoin-core \
  -printtoconsole \
  -regtest=1
```

You can optionally create a service using `docker-compose`:

```yml
litecoin-core:
  image: litecoin-project/litecoin-core
  command:
    -printtoconsole
    -regtest=1
```

### Using RPC to interact with the daemon

There are two communications methods to interact with a running Litecoin Core daemon.

The first one is using a cookie-based local authentication. It doesn't require any special authentication information as running a process locally under the same user that was used to launch the Litecoin Core daemon allows it to read the cookie file previously generated by the daemon for clients. The downside of this method is that it requires local machine access.

The second option is making a remote procedure call using a username and password combination. This has the advantage of not requiring local machine access, but in order to keep your credentials safe you should use the newer `rpcauth` authentication mechanism.

#### Using cookie-based local authentication

Start by launch the Litecoin Core daemon:

```sh
❯ docker run --rm --name litecoin-server -it litecoin-project/litecoin-core \
  -printtoconsole \
  -regtest=1
```

Then, inside the running `litecoin-server` container, locally execute the query to the daemon using `litecoin-cli`:

```sh
❯ docker exec --user litecoin litecoin-server litecoin-cli -regtest getmininginfo

{
  "blocks": 0,
  "currentblocksize": 0,
  "currentblockweight": 0,
  "currentblocktx": 0,
  "difficulty": 4.656542373906925e-10,
  "errors": "",
  "networkhashps": 0,
  "pooledtx": 0,
  "chain": "regtest"
}
```

In the background, `litecoin-cli` read the information automatically from `/home/litecoin/.litecoin/regtest/.cookie`. In production, the path would not contain the regtest part.

#### Using rpcauth for remote authentication

Before setting up remote authentication, you will need to generate the `rpcauth` line that will hold the credentials for the Litecoind Core daemon. You can either do this yourself by constructing the line with the format `<user>:<salt>$<hash>` or use the official [`rpcauth.py`](https://github.com/litecoin-project/litecoin/blob/master/share/rpcauth/rpcauth.py)  script to generate this line for you, including a random password that is printed to the console.

_Note: This is a Python 3 script. use `[...] | python3 - <username>` when executing on macOS._

Example:

```sh
❯ curl -sSL https://raw.githubusercontent.com/litecoin-project/litecoin/master/share/rpcauth/rpcauth.py | python - <username>

String to be appended to litecoin.conf:
rpcauth=foo:7d9ba5ae63c3d4dc30583ff4fe65a67e$9e3634e81c11659e3de036d0bf88f89cd169c1039e6e09607562d54765c649cc
Your password:
qDDZdeQ5vw9XXFeVnXT4PZ--tGN2xNjjR4nrtyszZx0=
```

Note that for each run, even if the username remains the same, the output will be always different as a new salt and password are generated.

Now that you have your credentials, you need to start the Litecoin Core daemon with the `-rpcauth` option. Alternatively, you could append the line to a `litecoin.conf` file and mount it on the container.

Let's opt for the Docker way:

```sh
❯ docker run --rm --name litecoin-server -it litecoin-project/litecoin-core \
  -printtoconsole \
  -regtest=1 \
  -rpcallowip=172.17.0.0/16 \
  -rpcauth='foo:7d9ba5ae63c3d4dc30583ff4fe65a67e$9e3634e81c11659e3de036d0bf88f89cd169c1039e6e09607562d54765c649cc'
```

Two important notes:

1. Some shells require escaping the rpcauth line (e.g. zsh), as shown above.
2. It is now perfectly fine to pass the rpcauth line as a command line argument. Unlike `-rpcpassword`, the content is hashed so even if the arguments would be exposed, they would not allow the attacker to get the actual password.

You can now connect via `litecoin-cli`. You will still have to define a username and password when connecting to the Litecoin Core RPC server.

To avoid any confusion about whether or not a remote call is being made, let's spin up another container to execute `litecoin-cli` and connect it via the Docker network using the password generated above:

```sh
❯ docker run -it --link litecoin-server --rm litecoin-project/litecoin-core \
  litecoin-cli \
  -rpcconnect=litecoin-server \
  -regtest \
  -rpcuser=foo\
  -stdinrpcpass \
  getbalance
```

Enter the password `qDDZdeQ5vw9XXFeVnXT4PZ--tGN2xNjjR4nrtyszZx0=` and hit enter:

```
0.00000000
```

Done!

### Exposing Ports

Depending on the network (mode) the Litecoin Core daemon is running as well as the chosen runtime flags, several default ports may be available for mapping.

Ports can be exposed by mapping all of the available ones (using `-P` and based on what `EXPOSE` documents) or individually by adding `-p`. This mode allows assigning a dynamic port on the host (`-p <port>`) or assigning a fixed port `-p <hostPort>:<containerPort>`.

Example for running a node in `regtest` mode mapping JSON-RPC/REST (19443) and P2P (19444) ports:

```sh
docker run --rm -it \
  -p 19443:19443 \
  -p 19444:19444 \
  litecoin-project/litecoin-core \
  -printtoconsole \
  -regtest=1 \
  -rpcallowip=172.17.0.0/16 \
  -rpcbind=0.0.0.0 \
  -rpcauth='foo:7d9ba5ae63c3d4dc30583ff4fe65a67e$9e3634e81c11659e3de036d0bf88f89cd169c1039e6e09607562d54765c649cc'
```

To test that mapping worked, you can send a JSON-RPC curl request to the host port:

```
curl --data-binary '{"jsonrpc":"1.0","id":"1","method":"getnetworkinfo","params":[]}' http://foo:qDDZdeQ5vw9XXFeVnXT4PZ--tGN2xNjjR4nrtyszZx0=@127.0.0.1:18443/
```

#### Mainnet

- JSON-RPC/REST: 9332
- P2P: 9333

#### Testnet

- Testnet JSON-RPC: 19332
- P2P: 19333

#### Regtest

- JSON-RPC/REST: 19443 (_since 0.16+_, otherwise _19332_)
- P2P: 19444

#### Signet

- JSON-RPC/REST: 39332
- P2P: 39333

## Docker

This image is officially supported on Docker version 17.09, with support for older versions provided on a best-effort basis.

## License

[License information](https://github.com/litecoin-project/litecoin/blob/master/COPYING) for the software contained in this image.
[License information](https://github.com/ruimarinho/docker-bitcoin-core/blob/master/LICENSE) for the [ruimarinho/docker-bitcoin-core][docker-hub-url] docker project.

[docker-hub-url]: https://hub.docker.com/r/litecoin-project/litecoin-core
[docker-pulls-image]: https://img.shields.io/docker/pulls/litecoin-project/litecoin-core.svg?style=flat-square
[docker-size-image]: https://img.shields.io/docker/image-size/litecoin-project/litecoin-core?style=flat-square
[docker-stars-image]: https://img.shields.io/docker/stars/litecoin-project/litecoin-core.svg?style=flat-square
