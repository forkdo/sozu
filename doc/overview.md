# Sozu Tutorial

## Introduction

Sozu is a fast, reliable, and programmable HTTP and TCP reverse proxy written in Rust. It supports advanced routing, load balancing, TLS termination, and dynamic configuration.

## 1. Rust Environment Setup

Before installing Sozu, ensure you have the latest stable version of Rust installed on your system. We recommend using `rustup` for installation:

1.  **Install `rustup`**: Visit the [official rustup website](https://rustup.rs) or run the following command in your terminal:

    ```bash
    curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
    ```

2.  **Configure Environment**: Follow the installer's instructions, which typically involve running `source $HOME/.cargo/env` to add `cargo` (Rust's package manager) to your PATH.

3.  **Verify Installation**: Run `rustc --version` and `cargo --version` to verify that Rust is installed successfully.

## 2. Sozu Installation

You can choose to install Sozu from crates.io via `cargo install` or build it from the source code.

### 2.1 Install via `cargo install`

Sozu is published on [crates.io](https://crates.io/). Installation is straightforward:

```bash
car go install sozu
```

After installation, the `sozu` executable will be available in your `~/.cargo/bin` directory.

### 2.2 Build from Source

If you want to build Sozu from the source code (for example, for development or to use the latest version), follow these steps:

1.  **Clone the Repository**: If you haven't already, clone the Sozu repository:

    ```bash
    git clone https://github.com/sozu-proxy/sozu.git
    cd sozu
    ```

2.  **Build**: Navigate to the `bin` directory and use the `cargo build` command. For a production build, be sure to use the `--release` flag to enable optimizations:

    ```bash
    cd bin
    cargo build --release --locked
    ```

    -   The `--release` flag tells Cargo to enable compiler optimizations, generating a more performant binary. Use this only for production builds.
    -   The `--locked` flag forces Cargo to adhere to the dependency versions specified in `Cargo.lock`, preventing dependency breakage.

The executable will be located at `target/release/sozu` after the build is complete.

## 3. How to Use Sozu

Sozu can run as a standalone process or be integrated into a Docker container or Systemd service.

### 3.1 Running Sozu

-   **If installed via `cargo install`**: The `sozu` command should already be in your `$PATH`.

    ```bash
    sozu start -c <path/to/your/config.toml>
    ```

-   **If built from source**: The executable is in the `target/release` directory.

    ```bash
    ./target/release/sozu start -c <path/to/your/config.toml>
    ```

You can edit the reverse proxy's configuration in the `config.toml` file to declare new clusters, frontends, and backends.

**Tip**: You can use the `sozu` binary as a CLI to interact with the reverse proxy. See the [command-line documentation](https://github.com/sozu-proxy/sozu/blob/main/doc/configure_cli.md) for more information.

### 3.2 Running with Docker

The Sozu repository provides a multi-stage `Dockerfile` based on `alpine:edge`.

1.  **Build the Docker Image**:

    ```bash
    docker build -t sozu .
    ```

    You can also build an image for a specific Alpine version:

    ```bash
    docker build --build-arg ALPINE_VERSION=3.14 -t sozu:main-alpine-3.14 .
    ```

2.  **Run the Docker Container**:

    ```bash
    docker run \
      --ulimit nofile=262144:262144 \
      --name sozu-proxy \
      -v /run/sozu:/run/sozu \
      -v /path/to/your/config.toml:/etc/sozu/config.toml \
      -v /my/state/:/var/lib/sozu \
      -p 8080:80 \
      -p 8443:443 \
      sozu start -c /etc/sozu/config.toml
    ```

    -   `-v /path/to/your/config.toml:/etc/sozu/config.toml`: Mounts your custom `config.toml` file into the container.
    -   `-v /my/state/:/var/lib/sozu`: Mounts your initial configuration state JSON file if you have one.

### 3.3 Systemd Integration

The Sozu repository provides Systemd unit files for easy integration as a service.

1.  **Copy the Unit File**: Copy `sozu/os-build/systemd/sozu.service` to the `/etc/systemd/system/` directory:

    ```bash
    sudo cp sozu/os-build/systemd/sozu.service /etc/systemd/system/
    ```

2.  **Reload Systemd**:

    ```bash
    sudo systemctl daemon-reload
    ```

3.  **Start the Service**:

    ```bash
    sudo systemctl start sozu.service
    ```

4.  **Enable on Boot**:

    ```bash
    sudo systemctl enable sozu.service
    ```

## 4. Sozu Configuration

Sozu's core configuration is managed through the `config.toml` file. It consists of three main sections: `global` parameters, protocol definitions (like `https`, `http`, `tcp`), and the `clusters` section.

### 4.1 Configuration File (`config.toml`) Structure Overview

Here is an example of a Sozu configuration file:

```toml
# Global parameters
command_socket = "./command_folder/sock"
saved_state = "./state.json"
log_level = "info"
log_target = "stdout"
worker_count = 2
handle_process_affinity = false
max_connections = 500
buffer_size = 16384
activate_listeners = true

# Listener configuration
[[listeners]]
protocol = "http"
address = "0.0.0.0:8080"
# public_address = "1.2.3.4:80" # For logs and forwarding headers
# expect_proxy = false # Expect PROXY protocol header

# HTTPS listener example
[[listeners]]
protocol = "https"
address = "0.0.0.0:8443"
certificate = "/path/to/certificate.pem"
key = "/path/to/key.pem"
certificate_chain = "/path/to/certificate_chain.pem"
# tls_versions = ["TLS_V12", "TLS_V13"]
# answer_404 = "/path/to/my-404-answer.http" # Custom error page

# Cluster configuration
[clusters]
[clusters.MyWebsiteCluster]
protocol = "http" # https proxy also uses http protocol
# load_balancing_policy="roundrobin" # Load balancing policy: "roundrobin" (default) or "random"
# https_redirect = true # Force redirect of http traffic to https
# sticky_session = true # Optional, enable sticky sessions

frontends = [
  { address = "0.0.0.0:8080", hostname = "your_domain.com" },
  # For HTTPS frontends, certificate and key are also required
  { address = "0.0.0.0:8443", hostname = "your_domain.com", certificate = "/path/to/certificate.pem", key = "/path/to/key.pem", certificate_chain = "/path/to/certificate_chain.pem" }
]
backends  = [
  { address = "127.0.0.1:8000" }, # Backend server address
  { address = "127.0.0.1:8001" }
]

# Metrics configuration
[metrics]
address = "127.0.0.1:8125"
# tagged_metrics = false
# prefix = "sozu"
```

### 4.2 Global Parameters

Global parameters are set in the `[global]` section and affect both the main and worker processes:

| Parameter                | Description                                                | Possible Values                           |
|--------------------------|------------------------------------------------------------|-------------------------------------------|
| `saved_state`            | Path from which Sozu attempts to load state on startup     |                                           |
| `log_level`              | Logging level                                              | `debug`, `trace`, `error`, `warn`, `info` |
| `log_target`             | Logging output destination                                 | `stdout`, `tcp`, or `udp` address         |
| `access_logs_target`     | Access log output destination (if activated)               | `stdout`, `tcp`, or `udp` address         |
| `command_socket`         | Path to the Unix command socket                            |                                           |
| `worker_count`           | Number of worker processes                                 |                                           |
| `handle_process_affinity`| Bind worker processes to CPU cores                         | `true`, `false`                           |
| `max_connections`        | Maximum number of concurrent connections                   |                                           |
| `buffer_size`            | Request buffer size used by worker processes (in bytes)    |                                           |
| `activate_listeners`     | Automatically start listeners                              | `true`, `false`                           |
| `front_timeout`          | Maximum inactivity time for frontend sockets               |                                           |
| `connect_timeout`        | Maximum inactivity time for connection requests            |                                           |
| `request_timeout`        | Maximum inactivity time for requests                       |                                           |
| `zombie_check_interval`  | Interval for checking zombie sessions                      |                                           |
| `pid_file_path`          | Path to the file where the PID is stored                   |                                           |

### 4.3 Listeners

The `[[listeners]]` section defines a set of listening sockets that accept client connections. You can define any number of listeners.

-   **Common Parameters**:
    ```toml
    [[listeners]]
    protocol = "http" # or "https", "tcp"
    address = "0.0.0.0:8080"
    # public_address = "1.2.3.4:80" # Optional, for logs and forwarding headers
    # expect_proxy = true # Optional, configure client socket to receive PROXY protocol header
    ```

-   **HTTP and HTTPS Listener Specific Options (Custom Error Pages)**:
    You can define custom responses for HTTP and HTTPS listeners, such as 404 Not Found or 503 Service Unavailable. These responses can be plain text files containing HTML and some template variables (e.g., `%REQUEST_ID%`).

    ```toml
    # Send a 404 response when Sozu doesn't know the requested domain or path
    answer_404 = "/path/to/my-404-answer.http"
    # Send a 503 response when no backend servers are available
    answer_503 = "/path/to/my-503-answer.http"
    ```

-   **HTTPS Listener Specific Options**:
    ```toml
    # Supported TLS versions. Possible values: "SSL_V2", "SSL_V3", "TLS_V12", "TLS_V13". Defaults to "TLS_V12" and "TLS_V13".
    tls_versions = ["TLS_V12", "TLS_V13"]
    # Defines the name of the sticky session cookie if the cluster has sticky_session activated. Defaults to "SOZUBALANCEID".
    sticky_name = "SOZUBALANCEID"
    ```

    Rustls-based HTTPS listener specific options:
    ```toml
    cipher_list = [
        # TLS 1.3 cipher suites
        "TLS13_AES_256_GCM_SHA384",
        # ... other cipher suites
    ]
    ```

### 4.4 Clusters

Declare your list of clusters in the `[clusters]` section:

```toml
[clusters]
[clusters.MyWebsiteCluster]
protocol = "http" # or "tcp". HTTPS proxy also uses http protocol
# load_balancing_policy="roundrobin" # Load balancing policy: "roundrobin" (default) or "random"
# https_redirect = true # Force redirect of http traffic to https
# sticky_session = true # Optional, enable sticky sessions

frontends = [
  { address = "0.0.0.0:8080", hostname = "your_domain.com" },
  # For HTTPS frontends, certificate and key are also required
  { address = "0.0.0.0:8443", hostname = "your_domain.com", certificate = "/path/to/certificate.pem", key = "/path/to/key.pem", certificate_chain = "/path/to/certificate_chain.pem" }
]
backs  = [
  { address = "127.0.0.1:8000" }, # Backend server address
  { address = "127.0.0.1:8001" }
]
```

### 4.5 Metrics

Sozu reports its status to other network components via a UDP socket and implements the `statsd` protocol.

Configuration:
```toml
[metrics]
address = "127.0.0.1:8125" # Address and port of the statsd service
# tagged_metrics = false # Use InfluxDB's statsd protocol to add tags
# prefix = "sozu" # Prefix for metric keys
```

### 4.6 PROXY Protocol

The PROXY protocol is used to pass the client's real IP address and port information through a chain of proxies. Sozu supports PROXY protocol version 2.

-   **Configure Sozu to Expect a PROXY Protocol Header**:
    ```toml
    [[listeners]]
    address = "0.0.0.0:80"
    expect_proxy = true
    ```

    This makes the client connection receive a PROXY protocol header before any other data is read.

-   **Configure Sozu to Send a PROXY Protocol Header to the Backend**:
    ```toml
    [[listeners]]
    address = "0.0.0.0:81"

    [clusters]
    [clusters.NameOfYourTcpCluster]
    send_proxy = true
    frontends = [
      { address = "0.0.0.0:81" }
    ]
    ```
    **Note**: Only applicable to TCP clusters (HTTP and HTTPS proxies will use forwarding headers).

-   **Configure Sozu to Forward a PROXY Protocol Header**:
    ```toml
    [[listeners]]
    address = "0.0.0.0:80"
    expect_proxy = true

    [clusters]
    [clusters.NameOfYourCluster]
    send_proxy = true
    frontends = [
      { address = "0.0.0.0:80" }
    ]
    ```

    Sozu will receive the PROXY protocol header from the client connection, validate it, and then forward it to the upstream backend. This allows a proxy chain to work without losing client connection information.
    **Note**: Only applicable to TCP clusters.

## 5. Dynamic Backend Management (Service Discovery)

In dynamic environments like Docker, the IP addresses of backend services can change frequently. Although `config.toml` resolves IP addresses at startup, you can use the Sozu command-line tool (CLI) to **dynamically add, update, or remove backends** without restarting Sozu.

This process requires an external mechanism (such as a simple script, a service discovery tool, or a custom program) to:

1.  **Monitor** your backend services (e.g., by listening to container start/stop events via the Docker API, or querying Docker's built-in DNS service).
2.  **Obtain** the current IP address and port of the backend services.
3.  **Use the `sozu` CLI tool** to send commands to the running Sozu instance.

Here is an example of using the `sozu` CLI to manage backends:

### 5.1 Start Sozu to Accept Commands

Ensure that the `command_socket` is configured correctly in your `config.toml` so that the CLI tool can connect to the running Sozu instance. For example:

```toml
command_socket = "/var/lib/sozu/command.sock"
```

### 5.2 Find a Docker Container's IP Address

Assuming your Docker network is named `my-network` and your backend service is named `my-backend-app`, you can use the `docker inspect` command to get its IP address.

```bash
# Get the IP address of a single running container
docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' my-backend-app

# Assuming your backend service is exposed on port 8080
export BACKEND_IP=$(docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' my-backend-app)
export BACKEND_PORT=8080
export BACKEND_ADDRESS="${BACKEND_IP}:${BACKEND_PORT}"
```

In a `docker-compose` environment, if services are under the same `user-defined bridge network`, the service name can often be resolved directly as a hostname. However, if Sozu resolves IPs at startup, you still need to get the IP first.

### 5.3 Dynamically Add a Backend

Use the `sozu backend add` command to add a new backend to a specified cluster.

```bash
# Assuming your Sozu config path is /etc/sozu/config.toml
# and the command_socket is set in that file
# Assuming your cluster ID is 'komo'
CONFIG_PATH="/etc/sozu/config.toml"
CLUSTER_ID="komo"
NEW_BACKEND_ID="backend-1" # Provide a unique ID for the new backend
NEW_BACKEND_ADDRESS="172.17.0.5:9120" # Replace with your backend's actual IP:port

sozu --config "${CONFIG_PATH}" backend add --id "${CLUSTER_ID}" --backend-id "${NEW_BACKEND_ID}" --address "${NEW_BACKEND_ADDRESS}"
```

-   `--config`: Specifies the path to the Sozu configuration file, which should contain the path to the `command_socket`.
-   `backend add`: Indicates the action to add a backend.
-   `--id`: Specifies the cluster ID to add the backend to.
-   `--backend-id`: Provides a unique identifier for this backend instance.
-   `--address`: Specifies the IP address and port of the backend server.

### 5.4 Dynamically Remove a Backend

Use the `sozu backend remove` command to remove a backend from a specified cluster.

```bash
# Assuming your Sozu config path is /etc/sozu/config.toml
CONFIG_PATH="/etc/sozu/config.toml"
CLUSTER_ID="komo"
OLD_BACKEND_ID="backend-1" # The ID of the backend to remove
OLD_BACKEND_ADDRESS="172.17.0.5:9120" # Replace with the actual IP:port of the backend to remove

sozu --config "${CONFIG_PATH}" backend remove --id "${CLUSTER_ID}" --backend-id "${OLD_BACKEND_ID}" --address "${OLD_BACKEND_ADDRESS}"
```

-   `--config`: Specifies the path to the Sozu configuration file.
-   `backend remove`: Indicates the action to remove a backend.
-   `--id`: Specifies the cluster ID to remove the backend from.
-   `--backend-id`: The ID of the backend to remove.
-   `--address`: The IP address and port of the backend server to remove.

### 5.5 Check Sōzu Worker Status

You can use the `sozu status` command to check the running status of the Sōzu workers.

```bash
# Assuming your Sozu config path is /etc/sozu/config.toml
CONFIG_PATH="/etc/sozu/config.toml"
sozu --config "${CONFIG_PATH}" status --json
```

This will return the current worker status in JSON format. To see detailed cluster, frontend, and backend configurations, see the next section.

### 5.6 Automation Script (Concept)

To achieve true service discovery, you need to write a continuously running script (e.g., using Python, Bash, or Go) that:

1.  **Periodically** or by **event-listening** to the Docker API, detects changes in backend containers.
2.  **Compares** the list of backends currently in Sozu with the list of backends actually running in Docker.
3.  **Executes** `sozu backend add` or `sozu backend remove` commands to synchronize the two lists.

This script will act as the "control plane" in your service discovery solution.

### 5.7 View Dynamically Added Frontends and Backends

After dynamically adding a `frontend` or `backend` via the command line, you might want to verify that they have been loaded successfully. Sōzu's routing structure is `frontend` -> `cluster` -> `backend`. You can track and view this dynamically added data in two steps:

1.  **Find the `cluster ID` for a `frontend`**

    Use the `sozu frontend list --json` command to list all configured frontends. In the returned JSON data, find the `frontend` you are interested in and note its `cluster_id`.

    ```bash
    # Replace <path/to/your/config.toml> with your config file path
    sozu --config <path/to/your/config.toml> frontend list --json
    ```

2.  **Find the `backends` for a `cluster ID`**

    Once you have the `cluster ID`, use the `sozu cluster list --id <CLUSTER_ID>` command to view the details of that specific cluster, which includes a list of all its `backends`.

    ```bash
    # Replace <path/to/your/config.toml> and <YOUR_CLUSTER_ID>
    sozu --config <path/to/your/config.toml> cluster list --id <YOUR_CLUSTER_ID> --json
    ```

Through this two-step process, you can clearly see the complete mapping from `frontend` to `backend` and confirm that your dynamic configuration has been applied.

We hope this tutorial helps you to better understand and use Sozu!