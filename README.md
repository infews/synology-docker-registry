# Local Docker Registry on a Synology NAS

This repo configures and runs local a Docker private registry with a simple UI on a Synology NAS running DSM 7.x.

It is built off two existing docker containers, [registry](https://hub.docker.com/_/registry) and  [quiq/docker-registry-ui](https://github.com/Quiq/docker-registry-ui) and  provides a tested out configuration.

These two services use the following directories:

| Directory | Function |
| --------- | -------- |
| `./auth` | Designated for authentication information |
| `./certs` | Should be used for storing TLS certificates when TLS is set up |
| `./data/repo` | The actual registry data / manifests. This should be regularly backed up |
| `./data/ui` | UI repo event database. Probably less important to back up |

## Installation

This is a intricate, multi-step installation process. But it is approachable. It follows these steps:

1. [Assumptions](#assumptions)
1. [Open Ports on the Synology Firewall](#open-ports-on-the-synology-firewall)
1. [Configure DNS](#configure-dns)
1. [Configure Your Router for External Access (optional)](#configure-your-router-for-external-access)
1. [Create Registry Folders and Share](#create-registry-folders-and-share)
1. [Clone This Repo and Configure ](#clone-this-repo-and-configure)
1. [Create the Project in Container Manager](#create-the-project-in-container-manager)
1. [Configure the Synology Reverse Proxy](#configure-the-synology-reverse-proxy)
1. [Verify Services are Running](#verify-services-are-running)

### Assumptions 

The steps below assume the following configuration:

* `registry.mywebsite.com` - a public URL for registry repo (only if you want to expose the repo outside of your LAN)
* `registry.home` - a local (private/internal) domain for registry repo
* The registry will run on internal port 49999
* `registry-ui.home` - a local (private/internal) domain for registry UI
* The registry UI will run on internal port 49998

Feel free to update these values, but keep them consistent in the configuration files (see [Clone This Repo and Configure](#clone-this-repo-and-configure)).

There is no public UI for this registry, as the UI project, [quiq/docker-registry-ui](https://github.com/Quiq/docker-registry-ui), does not currently provide authentication. 

The HTTPS-related steps below are only required if you are opening your registry repo externally. If not, these steps can be skipped. Actually, if you do not know what you are doing, prefer skipping them. If you decide to open your repo externally, consider security risks and their mitigation, particularly implementing an authentication method other than basic authentication. Never open your repo without at least implementing HTTPS/TLS.

You can use the registry directly on ports 49999 (registry repo) and 49998 (UI), or other high ports of your choosing. In that case you only need to open ports using the first step. The rest of steps is there to allow for more convenient usage without port numbers.

### Open Ports on the Synology Firewall

1. Open **Control Panel** / **Security** / **Firewall**
1. If the **Firewall** is not Enabled, move to **Configure DNS**
1. If the **Firewall** is Enabled, edit the Rules to open the two ports (49999, 49998) only to your internal network

### Configure DNS

If you are using a different server for DNS inside your network:

1. Set up a CNAME for `registry.home` to map to the IP address of your NAS
1. Set up a CNAME for `registry-ui.home` to map to the IP address of your NAS

If you are using the Synology NAS DNS Server:

1. Open the **DNS Server** App
1. Create (or ensure there is) a Primary Zone for your network
1. Edit the Resource Record for your Primary Zone  
1. Set up a CNAME for `registry.home` to map to the IP address of your NAS
1. Set up a CNAME for `registry-ui.home` to map to the IP address of your NAS

### Configure Your Router for External Access

This is an optional step if you want external access to your registry. You will need:

* An external domain with a public DNS CNAME (an example of `registry.mywebsite.com` is used here) that will resolve to your router's external IP address. 
* TLS/SSL certificate files for this domain
* A reverse proxy that routes this to your registry
  
Configure the reverse proxy as follows:

* Source:
    * Protocol: HTTPS
    * Hostname: `registry.mywebsite.com`
    * Port: 443
* Destination:
    * Protocol: HTTP
    * Hostname: IP Address of your NAS
    * Port: 49999

### Create Registry Folders and Share

1. There is likely already a folder on your NAS at `/volume1/docker`. If not, go to **File Station** and create a shared folder called `docker`
1. Make this folder available for read/write to a Synology user that you have access with from your developer desktop
1. Mount this folder on your developer desktop

### Clone This Repo and Configure

All of the necessary configuration values are set to `CHANGEME`. Searching your repo using `git grep CHANGEME` should reveal any forgotten values.

From your developer desktop:

1. Clone this repo into a folder under the shared `docker` directory named `registry`
   - This means your clone is now _on_ your NAS at `/volume1/docker/registry` 
   - All following steps are within this directory
1. If you are opening external access to the registry (optional):
   1. Copy your domain's certificate files to `./certs` (`cert.pem` and `privkey.pem`)
1. Create the `./auth/htpasswd` file 
   1. Use [this password generator](https://hostingcanada.org/htpasswd-generator/) to create passwords
   1. Add necessary Synology users as lines in this file. User / password combinations in this file will be able to use the registry API when logged in.
1. Change the configuration of the UI in `config-ui.yml`:
   * `registry_url` - the private/internal domain of your registry, e.g. `http://registry.home`
   * `registry_username` - a user you set up in `./htpasswd`
   * `event_listener_token` - a random token matching the respective information in `config-repo.yml` (see below)
1. Change the configuration of the repo in `config-repo.yml`:
   * `http.host` - the internal host name of your registry, `registry.home`
   * `http.secret` - replace with a short random string, for production use a strong and secure generator, like [random.org](https://www.random.org/strings/)
   * `notifications.endpoints[0].headers.Authorization` change the bearer token to match the `event_listener_token` from `config-ui.yml`
1. Ensure the ports for the two containers are set properly in `./docker-compose.yml` (if you changed them from 49999 and 49998)
    
### Create the Project in Container Manager

Now we are ready to start the two docker-based services using the DSM Container Manager

1. Open **Container Manager**
1. Click **Project** from the sidebar
1. Click the **Create** button; configure as follows:
   * **Project Name**: `docker-registry-and-ui` (or something descriptive for you)
   * **Path**: navigate to `docker/registry` (the directory where you cloned this repo) and hit Select
     * Choose "Use an existing docker-compose.yml to create the project"
     * The `docker-compose.yml` should be visible in the window
   * Click Next

_Container Manager will now download each container image and get them started. The logs will stream into a new window. If there are download and starting issues, errors will appear here first._

4. A **Web Station** dialog will open. 
   * Make sure the checkbox is set to connect the ports in each of the ports specified in `docker-compose.yml`
5. **Web Station** will open, with the **New Portal** dialog open; configure as follows:
   * **Service**: This should be set, but make sure it's the **Project Name** specified in the Container Manager above
   * **Portal Type**: `Name-based`
   * **Hostname**: The local hostname for the _registry_. NOT the UI.
   * **Port**: Check `80/443` (if it is not checked)
6. Save

By now the services should be started. Check in Container Manager that there is a green dot next to your Project.

### Configure the Synology Reverse Proxy

You've just set up the reverse proxy for the registry. But you need to do the same for the UI. (Web Station _only_ does this for this first service specified in a Project's `docker-compose.yml`).

1. Go to **Control Panel** / **Login Portal** / **Advanced**
1. Click the **Reverse Proxy** button
1. Click the **Create** Button; configure as follows:
   * **Reverse Proxy Name**: `registry.home`
   * **Source**
     * **Hostname**: `registry-ui.home`
     * **Port**: 80
   * **Destination**
     * **Hostname**: `registry-ui.home`
     * **Port**: 49998

### Verify Services are Running

* Visit `http://registry.home` 
  * It should resolve on your network
  * It should return a `200 OK` and show a blank page
* Visit `http://registry-ui.home`
  * It should resolve on your network
  * It should return a `200 OK` and show you HTML UI for your registry
* If you've opened this externally, visit `https://registry.mywebsite.com`
  * It should resolve from outside your network
  * It should return a `200 OK` and show a blank page

🎉 Congrats! You now have a private container image registry running! 🎉

## Using this Registry from Container Manager

1. Visit **Container Manager** / **Registry**
1. Click the **Settings** button
1. Click the **Add** button
1. In the **Edit Registry** Dialog
   * **Registry Name**: `registry.home`
   * **Registry URL**: `https://registry.home`
   * **Trust Self-Signed Certificate**: checked
   * **Password**: The clear password from the `httpasswd` file
1. Hit Apply
1. Select your just-added registry and hit the **Use** button

Note that Container Manager can only _directly_ use one registry at a time for actions in the UI (creates, updates, etc.). 

However, a Container Manager Project created with a `docker-compose.yml` specifies the registry in the `image` key. Your Projects will need to specify your registry on the image lines as `registry.home/<project>:tag` in order to use your local registry. 

Projects like this one will still default to pulling from DockerHub (as you did above).

## Starting, stopping and removing the UI

Starting and Stopping the registry and registry UI is done with the **Container Manager** / **Project** UI.
