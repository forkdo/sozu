### Configuring a Cluster via the Sozu Command Line

This document demonstrates how to use the `sozu` command-line tool to configure clusters, frontends, and backends as an alternative to directly editing the `config.toml` file.

**Important Note:** All `sozu` commands require the `--config <your-config-file>` parameter to specify the configuration file. This file contains the path to the command socket that the sozu instance is listening on, allowing the command-line tool to communicate with sozu. In the following examples, we use `--config config.toml`.

#### Original `config.toml` Configuration Example:

```toml
# A unique identifier for our routing rule.
[clusters.komo]

# The protocol this cluster will handle.
protocol = "http"
load_balancing = "ROUND_ROBIN"

# 'frontends' define which requests this cluster will handle.
# It matches requests coming to the listener at 'address' with the specified 'hostname'.
frontends = [
    { address = "0.0.0.0:80", hostname = "dash.bdev.cn" },
    { address = "0.0.0.0:443", hostname = "dash.bdev.cn" }
]

# 'backends' define where to forward the matched requests.
backends = [
    { address = "komodo-core-1:9120" }
]
```

#### Converting `config.toml` Configuration to Command-Line Instructions

The following are the steps to convert the `config.toml` configuration above into equivalent `sozu` command-line instructions:

##### 1. Add a Cluster

First, create a new cluster named `komo` and set its load balancing policy. Note that we use the `--id` and `--load-balancing-policy` parameters.

```bash
sozu --config config.toml cluster add --id komo --load-balancing-policy ROUND_ROBIN
```
**Explanation:**
*   `--config config.toml`: Specifies the sozu configuration file.
*   `cluster add --id komo`: Creates a new cluster with the unique identifier `komo`.
*   `--load-balancing-policy ROUND_ROBIN`: Sets the load balancing algorithm to round-robin.

**Regarding Protocol (`protocol`):**
The `cluster add` command may not directly support setting the protocol. If `http` is not the default, you may need to use the `sozu cluster modify` command or check `sozu cluster add --help` for complete options.

##### 2. Add Frontends

Next, add two frontends for the `komo` cluster to match incoming requests. Note the structure of the command: the protocol (`http` or `https`) comes after `frontend`, and you must use the `id <cluster_id>` subcommand at the end to associate it with the cluster.

```bash
# Add a frontend to handle HTTP (port 80)
sozu --config config.toml frontend http add --address 0.0.0.0:80 --hostname dash.bdev.cn id komo

# Add a frontend to handle HTTPS (port 443)
sozu --config config.toml frontend https add --address 0.0.0.0:443 --hostname dash.bdev.cn id komo
```
**Explanation:**
*   `frontend http add` / `frontend https add`: Defines a frontend addition operation for the HTTP or HTTPS protocol, respectively.
*   `--address ... --hostname ...`: Defines the rules for matching traffic.
*   `id komo`: Associates this frontend rule with the cluster with ID `komo`.

##### 3. Add a Backend

Finally, add the backend service address to the `komo` cluster. Sozu will forward matched requests here. Note that you need to specify the cluster with `--id` and provide a unique identifier for this backend with `--backend-id`.

```bash
sozu --config config.toml backend add --id komo --backend-id komodo-core-1 --address komodo-core-1:9120
```
**Explanation:**
*   `backend add`: Defines a backend addition operation.
*   `--id komo`: Specifies the target cluster ID to add the backend to.
*   `--backend-id komodo-core-1`: Sets a unique identifier for this backend within the cluster.
*   `--address komodo-core-1:9120`: Sets the address and port of the backend server.

---

### Complete Command-Line Configuration Flow

The design philosophy of `sozu` is to **manage listeners, certificates, and routing rules (frontends) separately**. Configuring a complete HTTPS service, as described in a `config.toml` file, requires a series of steps via the command line.

Assume your certificate and private key files are located at:
*   Certificate: `/etc/sozu/certs/certificate.pem`
*   Private Key: `/etc/sozu/certs/key.pem`

The following are the complete steps:

#### Step 1: Create a Cluster
First, create a new cluster named `komo` and set its load balancing policy.

```bash
# 1. Create the cluster
sozu --config config.toml cluster add --id komo --load-balancing-policy ROUND_ROBIN
```

#### Step 2: Add a Certificate for HTTPS
For HTTPS traffic, you must first associate the TLS certificate and private key with the address that will be listening (`0.0.0.0:443`).

```bash
# 2. Add the certificate
sozu --config config.toml certificate add \
  --address 0.0.0.0:443 \
  --certificate /etc/sozu/certs/certificate.pem \
  --certificate-chain /etc/sozu/certs/certificate.pem \
  --key /etc/sozu/certs/key.pem
```
**Explanation:**
*   `certificate add`: Defines a certificate addition operation.
*   `--address 0.0.0.0:443`: Specifies which listening address this certificate is for.
*   `--certificate`, `--certificate-chain`, `--key`: Specify the file paths for the certificate, certificate chain, and private key, respectively. If the certificate and chain are in the same file, you can use the same path for both.

#### Step 3: Associate Frontends
Now, create frontend rules to tell sozu how to route traffic from ports 80 (HTTP) and 443 (HTTPS) based on the `hostname`.

```bash
# 3. Associate the frontends
sozu --config config.toml frontend http add --address 0.0.0.0:80 --hostname dash.bdev.cn id komo
sozu --config config.toml frontend https add --address 0.0.0.0:443 --hostname dash.bdev.cn id komo
```
**Explanation:**
*   `frontend http add` / `frontend https add`: Defines routing rules for HTTP and HTTPS, respectively.
*   `id komo`: Routes traffic matching this rule to the cluster with ID `komo`.

#### Step 4: Associate a Backend
Finally, add the backend service address to the `komo` cluster.

```bash
# 4. Associate the backend
sozu --config config.toml backend add --id komo --backend-id komodo-core-1 --address komodo-core-1:9120
```
**Explanation:**
*   `backend add`: Defines a backend addition operation.
*   `--id komo`: Specifies the target cluster to add the backend to.
*   `--backend-id ... --address ...`: Defines the unique ID and address of the backend.

---
By following these four steps, you have completely reproduced the configuration from the `config.toml` file using the command line.
