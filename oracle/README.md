[<img src='../assets\Initia-Oracle-Banner.png' alt='banner' width= '99.9%'>]()

# Overview
**[Slinky](https://github.com/skip-mev/slinky/)** is a general-purpose price oracle leveraging ABCI++.

## Components

The Slinky Oracle consists of two main elements:

1. **On-Chain Component**
    - Retrieves price data from the sidecar with each block.
    - Forwards these prices to the blockchain through vote extensions.
    - Compiles prices from all validators involved.

2. **Sidecar Process**
    - Dedicated to polling price information from various providers.
    - Delivers this data to the on-chain component.

### Navigation Menu
- [Overview](#overview)
  - [Components](#components)
    - [Navigation Menu](#navigation-menu)
  - [Hardware requirements](#hardware-requirements)
  - [Config](#config)
  - [Installation guide](#installation-guide)
    - [1. Install required packages](#1-install-required-packages)
    - [2. Install Go](#2-install-go)
    - [3. Install `slinky` binary](#3-install-slinky-binary)
    - [4. Set Up Variables](#4-set-up-variables)
    - [5. Update oracle configuration](#5-update-oracle-configuration)
    - [6. Create a Service File](#6-create-a-service-file)
    - [7. Start the Oracle](#7-start-the-oracle)
    - [8. Validating Prices](#8-validating-prices)
    - [9. Enable Oracle Vote Extension](#9-enable-oracle-vote-extension)
  - [Monitoring](#monitoring)
    - [Oracle Service Metrics](#oracle-service-metrics)
    - [Oracle Application Metrics](#oracle-application-metrics)
  - [Useful commands](#useful-commands)
    - [Monitor server load](#monitor-server-load)
    - [Check logs of the oracle](#check-logs-of-the-oracle)
    - [Restart the oracle](#restart-the-oracle)
    - [Upgrade the oracle](#upgrade-the-oracle)
    - [Stop the oracle](#stop-the-oracle)
    - [Delete the oracle from the server](#delete-the-oracle-from-the-server)

## Hardware requirements
```py
- Memory: 16 GB RAM
- CPU: 4 cores
- Bandwidth: 1 Gbps
- Linux amd64 arm64 (Ubuntu LTS release)
```

## Config

To operate the oracle sidecar, it is essential to have valid configuration files for both the oracle component and the market component. The oracle executable recognizes flags that specify each configuration file. Below are recommended default settings suitable for the existing Initia Devnet environment.

1. **Oracle Component**
    - Determines how often to poll price providers.
    - Manages multiplexing behavior of websockets and more.
    - This configuration has been tested by Skip and is safe to use.
    - The recommended oracle component configuration can be found in `config/core/oracle.json` within the Slinky repo.

2. **Market Component**
    - Determines which markets the sidecar should fetch prices for.
    - The desired markets will be stored on-chain and pulled by the sidecar.
    - To properly configure the sidecar, you must point the sidecar to the GRPC port on a node (typically port 9090). This can be done by:
        - Adding the `--market-map-endpoint` flag when starting the sidecar.
        - Modifying the `oracle.json` component as shown below:
        
        ```json
        {
          "market_map_endpoint": "http://localhost:9090"
        }
        ```

## Installation guide

### 1. Install required packages
```bash
sudo apt update && \
sudo apt install curl git jq build-essential gcc unzip wget -y
```

### 2. Install Go
```bash
cd $HOME && \
ver="1.22.2" && \
wget "https://golang.org/dl/go$ver.linux-amd64.tar.gz" && \
sudo rm -rf /usr/local/go && \
sudo tar -C /usr/local -xzf "go$ver.linux-amd64.tar.gz" && \
rm "go$ver.linux-amd64.tar.gz" && \
echo "export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin" >> $HOME/.bash_profile && \
source $HOME/.bash_profile && \
go version
```

### 3. Install `slinky` binary
```bash
cd $HOME && \
ver="v0.4.3" && \
git clone https://github.com/skip-mev/slinky.git && \
cd slinky && \
git checkout $ver && \
make build && \
mv build/slinky /usr/local/bin/
```

### 4. Set Up Variables

Set up the necessary environment variables for the oracle sidecar. The `NODE_GRPC_ENDPOINT` should point to the Initia node (validator) gRPC endpoint, which typically runs on port 9090. The `ORACLE_CONFIG_PATH` should point to the local path of the oracle configuration file. The `ORACLE_GRPC_PORT` is the default port for the GRPC server, and the `ORACLE_METRICS_ENDPOINT` is the default Prometheus metrics endpoint.

```bash
echo 'export NODE_GRPC_ENDPOINT="0.0.0.0:9090"' >> ~/.bash_profile
echo 'export ORACLE_CONFIG_PATH="$HOME/slinky/config/core/oracle.json"' >> ~/.bash_profile
echo 'export ORACLE_GRPC_PORT="8080"' >> ~/.bash_profile
echo 'export ORACLE_METRICS_ENDPOINT="0.0.0.0:8002"' >> ~/.bash_profile
source $HOME/.bash_profile
```

### 5. Update oracle configuration
```bash
sed -i "s|\"url\": \".*\"|\"url\": \"$NODE_GRPC_ENDPOINT\"|" $ORACLE_CONFIG_PATH
sed -i "s|\"prometheusServerAddress\": \".*\"|\"prometheusServerAddress\": \"$ORACLE_METRICS_ENDPOINT\"|" $ORACLE_CONFIG_PATH
sed -i "s|\"port\": \".*\"|\"port\": \"$ORACLE_GRPC_PORT\"|" $ORACLE_CONFIG_PATH
```

### 6. Create a Service File

Create a systemd service file to manage the Initia Oracle.

```bash
sudo tee /etc/systemd/system/initia-oracle.service > /dev/null <<EOF
[Unit]
Description=Initia Oracle
After=network.target

[Service]
User=$USER
Type=simple
ExecStart=$(which slinky) --oracle-config-path $ORACLE_CONFIG_PATH
Restart=on-failure
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
EOF
```

### 7. Start the Oracle

Start the oracle service. After starting, the oracle gRPC endpoint will be available on `$ORACLE_GRPC_PORT`, which is by default port `8080`, and the Prometheus metrics endpoint will be available on `$ORACLE_METRICS_PORT`, which is by default port `8002`.

```bash
sudo systemctl daemon-reload && \
sudo systemctl enable initia-oracle && \
sudo systemctl restart initia-oracle && \
sudo journalctl -u initia-oracle -f -o cat
```

If everything was configured correctly, the logs should look like this:

```py
May 16 13:13:47 slinky[1731461]: {"level":"info","ts":"2024-05-16T13:13:47.885Z","caller":"oracle/oracle.go:163","msg":"oracle updated prices","pid":1731461,"process":"oracle","last_sync":"2024-05-16T13:13:47.885Z","num_prices":65}
May 16 13:13:47 slinky[1731461]: {"level":"info","ts":"2024-05-16T13:13:47.887Z","caller":"marketmap/fetcher.go:116","msg":"successfully fetched market map data from module; checking if market map has changed","pid":1731461,"process":"provider_orchestrator"}
May 16 13:13:48 slinky[1731461]: {"level":"info","ts":"2024-05-16T13:13:48.135Z","caller":"oracle/oracle.go:163","msg":"oracle updated prices","pid":1731461,"process":"oracle","last_sync":"2024-05-16T13:13:48.135Z","num_prices":65}
May 16 13:13:48 slinky[1731461]: {"level":"info","ts":"2024-05-16T13:13:48.385Z","caller":"oracle/oracle.go:163","msg":"oracle updated prices","pid":1731461,"process":"oracle","last_sync":"2024-05-16T13:13:48.385Z","num_prices":65}
May 16 13:13:48 slinky[1731461]: {"level":"info","ts":"2024-05-16T13:13:48.635Z","caller":"oracle/oracle.go:163","msg":"oracle updated prices","pid":1731461,"process":"oracle","last_sync":"2024-05-16T13:13:48.635Z","num_prices":65}
May 16 13:13:48 slinky[1731461]: {"level":"info","ts":"2024-05-16T13:13:48.886Z","caller":"oracle/oracle.go:163","msg":"oracle updated prices","pid":1731461,"process":"oracle","last_sync":"2024-05-16T13:13:48.886Z","num_prices":65}
May 16 13:13:49 slinky[1731461]: {"level":"info","ts":"2024-05-16T13:13:49.134Z","caller":"oracle/oracle.go:163","msg":"oracle updated prices","pid":1731461,"process":"oracle","last_sync":"2024-05-16T13:13:49.134Z","num_prices":65}
May 16 13:13:49 slinky[1731461]: {"level":"info","ts":"2024-05-16T13:13:49.386Z","caller":"oracle/oracle.go:163","msg":"oracle updated prices","pid":1731461,"process":"oracle","last_sync":"2024-05-16T13:13:49.386Z","num_prices":65}
May 16 13:13:49 slinky[1731461]: {"level":"info","ts":"2024-05-16T13:13:49.634Z","caller":"oracle/oracle.go:163","msg":"oracle updated prices","pid":1731461,"process":"oracle","last_sync":"2024-05-16T13:13:49.634Z","num_prices":65}
May 16 13:13:49 slinky[1731461]: {"level":"info","ts":"2024-05-16T13:13:49.885Z","caller":"oracle/oracle.go:163","msg":"oracle updated prices","pid":1731461,"process":"oracle","last_sync":"2024-05-16T13:13:49.885Z","num_prices":65}
May 16 13:13:50 slinky[1731461]: {"level":"info","ts":"2024-05-16T13:13:50.133Z","caller":"oracle/oracle.go:163","msg":"oracle updated prices","pid":1731461,"process":"oracle","last_sync":"2024-05-16T13:13:50.133Z","num_prices":65}
May 16 13:13:50 slinky[1731461]: {"level":"info","ts":"2024-05-16T13:13:50.384Z","caller":"oracle/oracle.go:163","msg":"oracle updated prices","pid":1731461,"process":"oracle","last_sync":"2024-05-16T13:13:50.383Z","num_prices":65}
May 16 13:13:50 slinky[1731461]: {"level":"info","ts":"2024-05-16T13:13:50.633Z","caller":"oracle/oracle.go:163","msg":"oracle updated prices","pid":1731461,"process":"oracle","last_sync":"2024-05-16T13:13:50.633Z","num_prices":65}
```

### 8. Validating Prices

Upon launching the oracle, you should observe successful price retrieval from the provider sources. Additionally, you have the option to execute the test client script available in the Slinky repository. This script requests the Oracle gRPC endpoint for the latest currency pairs every 10 seconds.

```bash
cd $HOME/slinky && \
make run-oracle-client
```

If everything was configured correctly, the logs should look like this:

```py
2024/05/16 13:09:55 Calling Prices RPC...
2024/05/16 13:09:55 Currency Pair, Price: (AAVE/USD, 8521408563)
2024/05/16 13:09:55 Currency Pair, Price: (ADA/USD, 4513805522)
2024/05/16 13:09:55 Currency Pair, Price: (AEVO/USD, 802320928)
2024/05/16 13:09:55 Currency Pair, Price: (AGIX/USD, 9239495798)
2024/05/16 13:09:55 Currency Pair, Price: (ALGO/USD, 1770708283)
2024/05/16 13:09:55 Currency Pair, Price: (APE/USD, 1201500000)
2024/05/16 13:09:55 Currency Pair, Price: (APT/USD, 8320064025)
2024/05/16 13:09:55 Currency Pair, Price: (ARB/USD, 963735494)
2024/05/16 13:09:55 Currency Pair, Price: (ARKM/USD, 2290116046)
2024/05/16 13:09:55 Currency Pair, Price: (ASTR/USD, 883253301)
2024/05/16 13:09:55 Currency Pair, Price: (ATOM/USD, 8336334533)
2024/05/16 13:09:55 Currency Pair, Price: (AVAX/USD, 3419267707)
2024/05/16 13:09:55 Currency Pair, Price: (AXL/USD, 1009203681)
2024/05/16 13:09:55 Currency Pair, Price: (BCH/USD, 4495898359)
2024/05/16 13:09:55 Currency Pair, Price: (BLUR/USD, 3677470988)
2024/05/16 13:09:55 Currency Pair, Price: (BNB/USD, 5752300920)
2024/05/16 13:09:55 Currency Pair, Price: (BONK/USD, 2447979191)
2024/05/16 13:09:55 Currency Pair, Price: (BTC/USD, 6591886754)
2024/05/16 13:09:55 Currency Pair, Price: (COMP/USD, 5482192877)
2024/05/16 13:09:55 Currency Pair, Price: (CRV/USD, 4140000000)
2024/05/16 13:09:55 Currency Pair, Price: (DOGE/USD, 15217086834)
2024/05/16 13:09:55 Currency Pair, Price: (DOT/USD, 6829381752)
2024/05/16 13:09:55 Currency Pair, Price: (DYDX/USD, 2003801520)
2024/05/16 13:09:55 Currency Pair, Price: (DYM/USD, 2632052821)
2024/05/16 13:09:55 Currency Pair, Price: (EOS/USD, 7981192476)
2024/05/16 13:09:55 Currency Pair, Price: (ETC/USD, 2693187274)
2024/05/16 13:09:55 Currency Pair, Price: (ETH/USD, 2974684873)
2024/05/16 13:09:55 Currency Pair, Price: (FET/USD, 22224444777)
2024/05/16 13:09:55 Currency Pair, Price: (FIL/USD, 5745898359)
2024/05/16 13:09:55 Currency Pair, Price: (GRT/USD, 2969187675)
2024/05/16 13:09:55 Currency Pair, Price: (HBAR/USD, 1098939575)
2024/05/16 13:09:55 Currency Pair, Price: (ICP/USD, 1213685474)
2024/05/16 13:09:55 Currency Pair, Price: (IMX/USD, 2379825930)
2024/05/16 13:09:55 Currency Pair, Price: (INJ/USD, 2315626250)
2024/05/16 13:09:55 Currency Pair, Price: (JTO/USD, 4618847539)
2024/05/16 13:09:55 Currency Pair, Price: (JUP/USD, 11374049619)
2024/05/16 13:09:55 Currency Pair, Price: (LDO/USD, 1532613045)
2024/05/16 13:09:55 Currency Pair, Price: (LINK/USD, 13727691076)
2024/05/16 13:09:55 Currency Pair, Price: (LTC/USD, 8168767507)
2024/05/16 13:09:55 Currency Pair, Price: (MANA/USD, 4245698279)
2024/05/16 13:09:55 Currency Pair, Price: (MATIC/USD, 6774709883)
2024/05/16 13:09:55 Currency Pair, Price: (MKR/USD, 2730146018)
2024/05/16 13:09:55 Currency Pair, Price: (NEAR/USD, 8033213285)
2024/05/16 13:09:55 Currency Pair, Price: (NTRN/USD, 67036814)
2024/05/16 13:09:55 Currency Pair, Price: (OP/USD, 2383653461)
2024/05/16 13:09:55 Currency Pair, Price: (ORDI/USD, 3757723089)
2024/05/16 13:09:55 Currency Pair, Price: (PEPE/USD, 101694677871)
```

### 9. Enable Oracle Vote Extension

To utilize the Slinky oracle data in the Initia node, the Oracle setting should be enabled in the `config/app.toml` file. You can do this programmatically with the following script. Make sure to execute this command on the server where your Initia node is running and set the correct variable values if needed.

```bash
ORACLE_GRPC_ENDPOINT="0.0.0.0:8080"
ORACLE_CLIENT_TIMEOUT="500ms"
NODE_APP_CONFIG_PATH="$HOME/.initia/config/app.toml"

sed -i '/\[oracle\]/!b;n;c\
enabled = "true"' $NODE_APP_CONFIG_PATH

sed -i "/oracle_address =/c\oracle_address = \"$ORACLE_GRPC_ENDPOINT\"" $NODE_APP_CONFIG_PATH

sed -i "/client_timeout =/c\client_timeout = \"$ORACLE_CLIENT_TIMEOUT\"" $NODE_APP_CONFIG_PATH

sed -i '/metrics_enabled =/c\metrics_enabled = "false"' $NODE_APP_CONFIG_PATH
```

The configuration file located at `$NODE_APP_CONFIG_PATH` should have these fields:

```py
###############################################################################
###                                  Oracle                                 ###
###############################################################################
[oracle]
# Enabled indicates whether the oracle is enabled.
enabled = "true"

# Oracle Address is the URL of the out-of-process oracle sidecar. This is used to
# connect to the oracle sidecar when the application boots up. Note that the address
# can be modified at any point, but will only take effect after the application is
# restarted. This can be the address of an oracle container running on the same
# machine or a remote machine.
oracle_address = "0.0.0.0:8080"

# Client Timeout is the time that the client is willing to wait for responses from 
# the oracle before timing out.
client_timeout = "500ms"

# MetricsEnabled determines whether oracle metrics are enabled. Specifically,
# this enables instrumentation of the oracle client and the interaction between
# the oracle and the app.
metrics_enabled = "false"
```

## Monitoring

### Oracle Service Metrics

Slinky exposes metrics on the `/metrics` endpoint on port `8002` by default. These metrics are in the Prometheus format and can be scraped by Prometheus or any other monitoring system that supports the Prometheus format.

The Slinky repository contains a sidecar Grafana dashboard that can be used to monitor Slinky Metrics. Instructions can be found here: [Set Up Grafana Dashboard](https://github.com/skip-mev/slinky/blob/main/metrics.md#set-up)

### Oracle Application Metrics

If metrics for the oracle are enabled on the Initia node, they will be exposed on the node's Prometheus port. These metrics provide information about validators' reporting activity for prices and their votes.

Metrics can also be checked here: [Slinky Service Metrics](https://github.com/skip-mev/slinky/blob/main/service/metrics/README.md)


## Useful commands
### Monitor server load
```bash 
sudo apt update
sudo apt install htop -y
htop
```

### Check logs of the oracle
```bash
sudo journalctl -u initia-oracle -f -o cat
```

### Restart the oracle
```bash
sudo systemctl restart initia-oracle
```

### Upgrade the oracle
```bash
ORACLE_VERSION="v0.4.3"

cd $HOME && \
git clone https://github.com/skip-mev/slinky.git && \
cd slinky && \
git checkout $ORACLE_VERSION && \
make build && \
mv build/slinky /usr/local/bin/

# Restart the oracle
sudo systemctl restart initia-oracle && sudo journalctl -u initia-oracle -f -o cat
```

### Stop the oracle
```bash
sudo systemctl stop initia-oracle
```

### Delete the oracle from the server
```bash
sudo systemctl stop initia-oracle
sudo systemctl disable initia-oracle
sudo rm /etc/systemd/system/initia-oracle.service
rm -rf $HOME/slinky
sudo rm /usr/local/bin/slinky
```
