# Docker-Swarm cluster installation procedure

> To follow this procedure, you'll need to remote desktop to the nodes with an administrator access.

## 1. Install Consul & join the Consul cluster

For each node in the cluster, follow the [Consul agent installation procedure](https://github.com/yhan/docker-swarm/blob/master/Install-Consul-agent-on-a-server.md).

## 2. Install Logstash

For each node in the cluster, follow the following steps.

### a) Install Java

1. Create a `c:\java` directory
1. Download the [latest minor release of the JRE 8](https://www.oracle.com/technetwork/java/javase/downloads/jre8-downloads-2133155.html) (take the Windows x64 `.tar.gz` download) into `c:\java`
1. Launch a Powershell console, and `cd` into `c:\java`
1. Extract the archive by running the following Powershell command:

    ```powershell
    tar xzf .\the_name_of_the_jre_download.tar.gz
    ```

1. Delete the `.tar.gz` file

### b) Install Logstash

1. Create a `c:\elastic` directory
1. Download the [latest Logstash release](https://www.elastic.co/downloads/logstash) (take the `.zip` download) into `c:\elastic`
1. Launch a Powershell console, and `cd` into `c:\elastic`
1. Extract the archive by running the following Powershell command:

    ```powershell
    Expand-Archive .\the_logstash_archive_you_downloaded.zip -DestinationPath .
    ```

1. Delete the `.zip` file you downloaded
1. Create the following directories:

    - `c:\elastic\config\logstash`
    - `c:\elastic\logs\logstash`
    - `c:\elastic\data\logstash`

1. Go to another server (e.g. `01lfgdev02`) in the `c:\elastic\config\logstash` directory and copy the `jvm.options`, `log4j2.properties`, `logstash.yml`, `pipelines.yml`, `startup.options`, `docker.conf` & `traefik.conf` files, then paste them into the node's `C:\elastic\config\logstash` directory
1. Open the node's `C:\elastic\config\logstash\logstash.yml` file and set the `xpack.monitoring.elasticsearch.url` YAML property to an array containing the URL(s) of the ElasticSearch cluster (e.g. `["http://dev-elasticsearch.service.consul.la-francaise.com:9200/"]` for the DEV environment or `["http://prod-elasticsearch.service.consul.la-francaise.com:9200/"]` for PROD)
1. Open the node's `C:\elastic\config\logstash\docker.conf` file and set the `output.elasticsearch.hosts` property to an array containing the URL(s) of the ElasticSearch cluster (e.g. `["http://dev-elasticsearch.service.consul.la-francaise.com:9200/"]` for the DEV environment or `["http://prod-elasticsearch.service.consul.la-francaise.com:9200/"]` for PROD)
1. Open the node's `C:\elastic\config\logstash\traefik.conf` file and set the `output.elasticsearch.hosts` property to an array containing the URL(s) of the ElasticSearch cluster (e.g. `["http://dev-elasticsearch.service.consul.la-francaise.com:9200/"]` for the DEV environment or `["http://prod-elasticsearch.service.consul.la-francaise.com:9200/"]` for PROD)
1. Download the latest release of [NSSM](https://nssm.cc/download) and unzip it inside `c:\elastic`
1. Open a Powershell console and run `C:\elastic\nssm-2.24\win64\nssm.exe install` (use the proper version number in the path, of course) 
1. In the NSSM window that shows up:

    - set `Service name` to `logstash`
    - in the `Application` tab
        - set `Path` to `C:\elastic\logstash-6.6.2\bin\logstash.bat` (use the proper version number in the path, of course)
        - set `Startup directory` to `C:\elastic\logstash-6.6.2` (use the proper version number in the path, of course)
        - set `Arguments` to `--path.settings c:/elastic/config/logstash`
    - in the `Details` tab
        - set `Display name` to `logstash`
    - in the `Environment` tab
        - set `Environment variables` to `JAVA_HOME=C:\java\jre1.8.0_202` (use the proper JRE version number in the path, of course)
    - click `Install service`

1. Start the logstash service by running the following Powershell command in administrator mode:

    ```powershell
    Start-Service -Name logstash
    ```

## 3. Install Docker

### a) Make sure Windows is up to date

1. Open a Powershell console in administrator mode and run `sconfig`
2. Select option `6) Download and Install Updates`
3. Follow the procedure to install all Windows updates

### b) Install Docker service

Follow the [Docker installation guide](https://docs.docker.com/install/windows/docker-ee/) (just the first 3 steps under `Install Docker Engine - Enterprise`).

### c) Generate PKI certificates

On your local computer:

1. Create a `c:\Temp\docker-pki` directory
1. If you have Git installed, launch a Git Bash and run `cd /c/Temp/docker-pki`; if you don't have Git, download and install the [OpenSSL binaries](https://www.openssl.org/community/binaries.html), add `openssl.exe` to the PATH, open a CMD console and `cd` into `c:\Temp\docker-pki`
1. Create an OpenSSL configuration file for the certificate authority by creating a `c:\Temp\docker-pki\ca.cnf` file with the following content:

    > Give a descriptive name to your certificate authority (the `CN` parameter below)

    ```ini
    [req]
    prompt = no
    distinguished_name = req_distinguished_name

    [req_distinguished_name]
    O = La Francaise
    CN = Docker Swarm Prod
    ```

1. Create a certificate authority by run the following commands:

    ```bash
    # Create the private key for the certificate authority
    openssl genrsa -out ./ca-key.pem 2048

    # Create certificate authority using private key
    openssl req -x509 -new -nodes -key ./ca-key.pem -days 20000 -out ./ca.pem -config ./ca.cnf
    ```

1. On each node, run `ipconfig` and note the node's IP address down, you'll need it in the next step
1. Create an OpenSSL configuration file for the Docker server certificate for each node by creating a `c:\Temp\docker-pki\node.NODE_NAME.cnf` file with the following content:

    > Replace `NODE_NAME` in the `.cnf` file name and in the following content with the actual computer name of the node. Also, replace `NODE_IP` in the following content with the node's IP address.

    ```ini
    [req]
    prompt = no
    req_extensions = v3_req
    distinguished_name = req_distinguished_name

    [req_distinguished_name]
    CN = NODE_NAME

    [ v3_req ]
    basicConstraints = CA:FALSE
    keyUsage = nonRepudiation, digitalSignature, keyEncipherment
    extendedKeyUsage = serverAuth, clientAuth
    subjectAltName = @alt_names

    [alt_names]
    DNS.1 = NODE_NAME
    DNS.2 = NODE_NAME.groupe-ufg.com
    DNS.3 = NODE_NAME.node.consul.la-francaise.com
    DNS.4 = NODE_NAME.node.dc.consul.la-francaise.com
    DNS.5 = dev-swarm.service.consul.la-francaise.com
    DNS.6 = dev-swarm.service.dc.consul.la-francaise.com
    DNS.7 = localhost
    IP.1 = NODE_IP
    IP.2 = 127.0.0.1
    ```

1. Create and sign a server certificate for each node by running the following commands for each node:

    > Replace `NODE_NAME` in the following content with the actual computer name of the node.

    ```bash
    openssl genrsa -out ./node-key.NODE_NAME.pem 2048
    openssl req -new -key ./node-key.NODE_NAME.pem -out ./node.NODE_NAME.csr -config ./node.NODE_NAME.cnf
    openssl x509 -req -in ./node.NODE_NAME.csr -CA ./ca.pem -CAkey ./ca-key.pem -CAcreateserial -out ./node.NODE_NAME.pem -days 20000 -extensions v3_req -extfile ./node.NODE_NAME.cnf
    ```

1. Create an OpenSSL configuration file for the Docker client certificate by creating a `c:\Temp\docker-pki\client.cnf` file with the following content:

    ```ini
    [req]
    prompt = no
    req_extensions = v3_req
    distinguished_name = req_distinguished_name

    [req_distinguished_name]
    CN = client.dc.consul.la-francaise.com

    [ v3_req ]
    basicConstraints = CA:FALSE
    keyUsage = nonRepudiation, digitalSignature, keyEncipherment
    extendedKeyUsage = serverAuth, clientAuth
    subjectAltName = @alt_names

    [alt_names]
    DNS.1 = client.dc.consul.la-francaise.com
    DNS.2 = client.consul.la-francaise.com
    DNS.3 = localhost
    IP.1 = 127.0.0.1
    ```

1. Create and sign a client certificate by running the following commands:

    ```bash
    openssl genrsa -out ./client-key.pem 2048
    openssl req -new -key ./client-key.pem -out ./client.csr -config ./client.cnf
    openssl x509 -req -in ./client.csr -CA ./ca.pem -CAkey ./ca-key.pem -CAcreateserial -out ./client.pem -days 20000 -extensions v3_req -extfile ./client.cnf
    ```

### c) Configure the Docker agent

On each node:

1. Go to `C:\ProgramData\docker` and create a `certs.d` directory
1. Copy to `C:\ProgramData\docker\certs.d` the following PKI files generated in the previous section (replace `NODE_NAME` in the file names with the actual computer name of the node):
    a. `ca.pem`
    b. `node.NODE_NAME.pem`
    c. `node-key.NODE_NAME.pem`
1. Go to `C:\ProgramData\docker\config` and create a `daemon.json` file with the following content (replace `NODE_NAME` in the file names with the actual computer name of the node):

    ```json
    {
      "hosts": [ "npipe://", "tcp://0.0.0.0:2376" ],
      "tls":true,
      "tlsverify":true,
      "tlscacert": "C:\\ProgramData\\docker\\certs.d\\ca.pem",
      "tlscert": "C:\\ProgramData\\docker\\certs.d\\node.NODE_NAME.pem",
      "tlskey": "C:\\ProgramData\\docker\\certs.d\\node-key.NODE_NAME.pem",
      "allow-nondistributable-artifacts": [ "docker-registry.apps.la-francaise.com"],
      "dns": ["10.100.181.210", "10.100.181.211", "10.100.181.212", "10.100.181.213"],
      "dns-search": ["groupe-ufg.com", "node.consul.la-francaise.com"],
      "log-driver": "gelf"
    }
    ```

    > The `dns` property refers to the IPs of the Consul cluster's nodes. The `gelf` value for the `log-driver` property is the logstash driver, which is configured in the command line (see next step).

1. Open a Powershell console in administrator mode and run the following command:

    ```shell
    sc.exe config docker binpath= "\`"C:\Program Files\docker\dockerd.exe\`" --run-service --log-opt gelf-address=udp://dev-logstash.service.consul.la-francaise.com:12201"
    ```

    > This step updates the Windows service registration of the Docker daemon to  add a command line argument. This is required because the `daemon.json` configuration file (created in the previous step) supports setting the log  driver (GELF in our case) but doesn't support setting the address where GELF  log events should be sent. This can only be provided as a command line argument  on the Docker daemon.

### d) Create the Swarm cluster

> At this point you need to know how many manager and worker nodes you need in your cluster. Take a look the [official documentation](https://docs.docker.com/engine/swarm/key-concepts/) for more information about those concepts.

1. On one of the nodes that you want to use as manager, run the following Powershell command in administrator mode:

    > Replace `<IP>` with the node's IP address. The `default-addr-pool` parameter must be a unique IP address range, not used in the corporate network.

    ```powershell
    docker swarm init --advertise-addr <IP> --listen-addr <IP>:2377 --default-addr-pool 172.25.0.0/16
    ```

1. For all other nodes that must be added as managers:
    a. On the first manager node, open a Powershell console in administrator mode and run `docker swarm join-token manager` and copy the outputed `docker swarm join` command
    b. On each node that must be added as a manager, open a Powershell console in administrator mode, run the `docker swarm join` command copied in the previous step
1. For all nodes that must be added as workers:
    a. On the first manager node, open a Powershell console in administrator mode, run `docker swarm join-token worker` and copy the outputed `docker swarm join` command
    b. On each node that must be added as a worker, open a Powershell console in administrator mode, run the `docker swarm join` command copied in the previous step

## 4. Add TFS Service Endpoints

1. In TFS, open the settings gear top menu and click `Services`
1. Click `New Service Endpoint`, then `Docker Host`
1. Enter the following fields:
    a. `Connection name`: a descriptive name (e.g.: `Docker Swarm Dev`)
    b. `Server URL`: the URL of one of the cluster's managers (e.g.: `tcp://NODE_NAME.node.consul.la-francaise.com:2376`) - this will need to be changed in a later step, once the cluster is registered in Consul
    c. `CA Certificate`: the content of the certificate authority file (`ca.pem`) generated in step `3. b)`
    d. `Certificate`: the content of the client certificate file (`client.pem`) generated in step `3. b)`
    e. `Key`: the content of the client certificate key file (`client-key.pem`) generated in step `3. b)`
1. Click `OK`

## 5. Install infrastructure services

This step will install some infrastructure-related, Docker global services on the cluster, such as Traefik and Regis.

Traefik is the load balancer and reverse proxy that dispatches HTTP traffic to service instances.

Regis is a homemade docker add-on which listens for docker events and registers/unregisters services in Consul when containers start/shut down.

### a) Update the Docker Swarm Infrastructure TFS release

**If the new cluster is intended to replace an existing environment cluster (e.g. DEV or PROD):**

1. Go to the [Docker Swarm Infrastructure release](https://srvtfs/tfs/AM/LT/_apps/hub/ms.vss-releaseManagement-web.cd-workflow-hub?definitionId=113) on TFS
1. Go to the tasks of the proper environment
1. In all tasks (`Deploy stack` and `Watch stack deployment`), under `Advanced Options`, set the `Docker host service connection` to the new TFS service endpoint created at step `4.`
1. Save

**If the new cluster is for a new environment which didn't exist before (e.g. UAT or for automated integration testing):**

1. Clone the [Docker Swarm Infrastructure git repository](https://srvtfs/tfs/AM/LT/_git/Docker.Swarm.Infrastructure) locally
1. Add a `docker-compose` file for the new environment (e.g.: `docker-compose.uat.yml`). You can clone it from an existing environment's `docker-compose` file (e.g.: `docker-compose.dev.yml`), but you'll need to adapt the content of the file according to your new environment's requirements.
1. Commit and push to `origin`
1. Go to the [Docker Swarm Infrastructure release](https://srvtfs/tfs/AM/LT/_apps/hub/ms.vss-releaseManagement-web.cd-workflow-hub?definitionId=113) on TFS
1. Add your new environment to the pipeline. You can clone an existing environment (e.g. `Development`).
1. Go to the tasks of your new environment
1. In all tasks (`Deploy stack` and `Watch stack deployment`), under `Advanced Options`, set the `Docker host service connection` to the new TFS service endpoint created at step `4.`
1. Save

### b) Create Docker secrets for Traefik

1. Get the proper `.pem` and `.key` certificate files from the infrastructure team. If the cluster is for a new environment, you'll probably need newly-generated certificates. Otherwise, just ask them to send you the existing certificates used by the environment being replaced.
1. Copy those files to one of the manager nodes in a temporary directory (e.g.: `c:\temp`)
1. Create the Docker secret by running the following Powershell command in administrator mode:

    > Before running the following command, replace `<DOCKER_SECRET_KEY_NAME>` with the value of the `secrets/https.key/name` property in the environment's `docker-compose` file (e.g.: `ufgca-dev.apps.la-francaise.com.key` in `docker-compose.dev.yml`), `<KEY_FILE>` with the `.key` certificate file, `<DOCKER_SECRET_CERT_NAME>` with the value of the `secrets/https.pem/name` property in the environment's `docker-compose` file (e.g.: `dev.apps.la-francaise.com.pem` for DEV), and `<CERTIFICATE_FILE>` with the `.pem` certificate file.

    ```powershell
    docker secret create <DOCKER_SECRET_KEY_NAME> c:\temp\<KEY_FILE>
    docker secret create <DOCKER_SECRET_CERT_NAME> c:\temp\<CERTIFICATE_FILE>
    ```

    You can simply create a file for each secret you want to create, with the name of the file being the desired name of the secret, drop those files in 
    a temporary directory on one of the manager nodes, then run the following Powershell command:

    ```powershell
    Get-ChildItem "<YOUR_TEMPORARY_DIRECTORY>" | Foreach-Object {
      docker secret create "$($_.Name)" "$($_.FullName)"
    }
    ```

### c) Create data folder for Traefik

Traefik writes log files in a mounted volume, which is mapped to a directory on the host nodes.
As such, go on each node of the cluster and create a `c:\traefik` directory.

### d) Deploy infrastructure services on cluster

1. Go to the [Docker Swarm Infrastructure release](https://srvtfs/tfs/AM/LT/_apps/hub/ms.vss-releaseManagement-web.cd-workflow-hub?definitionId=113) on TFS
1. Run the release (at least for the proper environment)
1. Go to each node of the cluster and deploy Regis by running the following Powershell command in administrator mode:

    ```powershell
    docker login docker-registry.apps.la-francaise.com
    docker run --name regis --restart=always -e Consul__Endpoint="http://$($env:COMPUTERNAME).groupe-ufg.com:8500" -e Consul__ExposedAddress="$($env:COMPUTERNAME).node.consul.la-francaise.com" -e Docker__Endpoint="npipe://./pipe/docker_engine" -u ContainerAdministrator --mount type=npipe,src=\\.\pipe\docker_engine,target=\\.\pipe\docker_engine docker-registry.apps.la-francaise.com/regis:1.1.1
    docker logout docker-registry.apps.la-francaise.com
    ```

    > - The `docker login` command will ask for credentials. You can use your Windows credentials or the `advanced_hooker_user` account (without specifying the domain) to log in the docker image registry.
    > - Make also sure to use the latest version of Regis by checking the [build definition](https://srvtfs/tfs/AM/LT/_build/index?context=allDefinitions&path=%5C&definitionId=506&_a=completed) and updating the version number accordingly in the image name of the following command (the command below uses version `1.1.1`, which is the latest at the time of writing).
    > - After the `docker run` command is finished downloading the image and booting the container up (you'll see `Application started. Press Ctrl+C to shut down.` being printed out in the console window), press CTRL+C to detach from the container process (no worry, the CTRL+C command won't be transmitted inside the container, so the actual Regis process won't stop).
