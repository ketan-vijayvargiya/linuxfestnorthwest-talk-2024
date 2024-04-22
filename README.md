# linuxfestnorthwest-talk-2024

This repository contains the code behind my talk, _TLS in the homelab: the easy way and the hard way_, from _LinuxFest Northwest 2024_. More details [here](https://ketanvijayvargiya.com/2-tls-in-the-homelab-the-easy-way-and-the-hard-way/).

## Instructions

### Setup

First, find a box and a domain. I did the following for the talk but there are numerous alternatives:

- A cheap VM from Hetzner.
- A free domain, _wagondime.duckdns.org_, from [Duck DNS](https://www.duckdns.org/) and pointed its IP addresses to the VM.

Second, install Docker and run `docker compose up -d`.

### Step CA

Once you spin up the Step CA container, run `docker compose logs step-ca` and you'll see 2 lines like these:

```sh
step-ca-1  | 👉 Your CA administrative password is: ...
...
step-ca-1  | 2024/04/07 01:13:51 X.509 Root Fingerprint: ...
```

Hang on to the admin password as I don't think you'll be able to access it later. You can also note down the root fingerprint or regenerate it later by running:

```sh
docker compose exec step-ca step certificate fingerprint certs/root_ca.crt
```

### Step CLI

Once Step CA is ready, [setup the Step CLI](https://smallstep.com/docs/step-cli/) on some _client_ machine. (Just ensure that Step CA is accessible from whatever _client_ machine you pick. You could use Tailscale, for instance. For the talk, I reused the same Hetzner box as the _client_, whcih is why you'll see `localhost` in the next command.)

Bootstrap the CLI:

```sh
step ca bootstrap \
    --ca-url localhost:9000 \
    --fingerprint=$CA_FINGERPRINT
```

### 'whoami-1': Free certificate from Let's Encrypt

```sh
curl https://whoami-1.wagondime.duckdns.org
```

### 'whoami-2': Custom certificate from Step CA

First, get the root certificate on the _client_:

```sh
step ca root root_ca.crt
```

Then:

```sh
curl --cacert root_ca.crt https://whoami-2.wagondime.duckdns.org
```

### 'whoami-3': Custom certificate from Step CA, with mTLS


First, get a client certificate on the _client_:

```sh
step ca certificate \
    demo client.crt client.key \
    --ca-url=localhost:9000 \
    --provisioner='admin'
```

Optionally, if you want to import the client certificate into an iOS or macOS certificate store, convert the 2 files to _p12_ format:

```sh
step certificate p12 --legacy client.p12 client.crt client.key
```

The following should now work:

```sh
curl \
	--cert client.crt \
	--key client.key \
	--cacert root_ca.crt \
	https://whoami-3.wagondime.duckdns.org
```
