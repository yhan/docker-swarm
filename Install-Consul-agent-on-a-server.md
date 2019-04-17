# Install a client agent

1. On the server, create a `c:\consul` directory
2. Download the latest version of [Consul](https://www.consul.io/downloads.html) (take the Windows x64 `.zip` download) into `c:\consul`
3. Launch a Powershell console, and `cd` into `c:\consul`
4. Extract the archive by running the following Powershell command:

```powershell
Expand-Archive .\the_consul_archive_you_downloaded.zip -DestinationPath .
```

5. Delete the `.zip` file you downloaded
6. Go to another server with a client agent (e.g. `01lfgdev02`) in the `c:\consul` directory and copy the `config` directory along with the `ca.pem`, `client.dc.consul.la-francaise.com.key` and `client.dc.consul.la-francaise.com.pem` files, then paste into `C:\consul`
7. Open the server's `C:\consul\config\client.json` file and set the `advertise_addr` JSON property to the server's IP address (run `ipconfig` to get it if needed)
8. Install and start the Consul service by running the following Powershell commands as an administrator:

```powershell
New-Service -Name Consul -BinaryPathName "c:\consul\consul.exe agent -config-dir=c:\consul\config" -StartupType Automatic
Start-Service -Name Consul
```

9. The new node should appear in the [Consul cluster dashboard](https://consul.apps.la-francaise.com/ui/dc/nodes)

# Install a server agent

1. On the server, create a `c:\consul` directory
2. Download the latest version of [Consul](https://www.consul.io/downloads.html) (take the Windows x64 `.zip` download) into `c:\consul`
3. Launch a Powershell console, and `cd` into `c:\consul`
4. Extract the archive by running the following Powershell command:

```powershell
Expand-Archive .\the_consul_archive_you_downloaded.zip -DestinationPath .
```

5. Delete the `.zip` file you downloaded
6. Go to another server with a server agent (e.g. `01lfgprd01`) in the `c:\consul` directory and copy the `config` directory along with the `ca.cer`, `ca.pem`, `server.dc.consul.la-francaise.com.key` and `server.dc.consul.la-francaise.com.pem` files, then paste into `C:\consul`
7. Open the server's `C:\consul\config\server.json` file and set the `advertise_addr` JSON property to the server's IP address (run `ipconfig` to get it if needed)
8. Install and start the Consul service by running the following Powershell commands as an administrator:

```powershell
New-Service -Name Consul -BinaryPathName "c:\consul\consul.exe agent -config-dir=c:\consul\config" -StartupType Automatic
Start-Service -Name Consul
```

9. Join the Consul cluster by locating the IP address of one of the nodes where a server consul agent and by then  running the following Powershell command as an administrator (where `<IP>` is the IP address of the server agent you located):

```powershell
.\consul.exe join <IP>
```

10. The new node should appear in the [Consul cluster dashboard](https://consul.apps.la-francaise.com/ui/dc/nodes)
