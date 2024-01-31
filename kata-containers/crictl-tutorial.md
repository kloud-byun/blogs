# Kata-Containers with crictl


## :clipboard: Introduction
Kata Containers offer a unique blend of lightweight container isolation with the security features of virtual machines. When working with these environments, understanding and managing these containers becomes crucial. This guide introduces `crictl`, a command-line interface for interacting with Container Runtime Interface (CRI) compatible runtimes, and elaborates on how it integrates with Kata Containers. Here, we'll explore how to leverage `crictl` for efficient container management.

## 💡 What is crictl ?
**`crictl`** is a CLI tool designed for Kubernetes environments, specifically for interacting with CRI-compatible container runtimes. Its primary functions include **inspecting and debugging containers and pods** within Kubernetes nodes. It offers diverse capabilities for managing containers, pods, and images, facilitating operations like **starting, stopping, and inspecting containers, as well as managing pods and pulling or pushing container images.**

<details>
  <summary>:mag: "Click to view" Comparison table for crictl alternatvies</summary>

  | Feature/Tool                | crictl       | Docker CLI       | ctr     | kubectl   | nerdctl   |
  |-----------------------------|--------------|------------------|---------|-----------|-----------|
  | **Primary Use**             | Kubernetes inspection/debugging | Docker container management | Minimalist containerd CLI | Kubernetes cluster interaction | Container management with Docker compatibility |
  | **Kubernetes Integration**  | Designed for Kubernetes | Indirect through Docker | Compatible, but not direct | Directly designed for Kubernetes | Compatible with Docker-like experience |
  | **Container Runtime**       | CRI-compatible | Docker-specific | containerd-specific | Kubernetes-focused | containerd with Docker API compatibility |
  | **Operation Types**         | Container/pod management | Broad container operations | Basic container operations | Cluster-wide management | Docker-like operations including build, push, pull |
  | **Kata-Containers Suitability** | High for Kubernetes-based setups | Moderate via Docker | Moderate with containerd | High for Kubernetes clusters | High, especially in Docker-like environments |

</details>

> :memo: _Note:_ this tutorial assumes that you have already set up Kata-Containers with Firecracker, using containerd. If you haven't, visit [how-to-setup-kata-with-firecracker](https://github.com/kata-containers/kata-containers/blob/main/docs/how-to/how-to-use-kata-containers-with-firecracker.md).


## :bookmark_tabs: Table of Contents
1. [Crictl Installation](#floppy_disk-crictl-installation)
2. [CNI Plugins Installation](#floppy_disk-cni-plugins-installation)
3. [Run a crictl demo (w/ busybox)](#zap-run-a-crictl-demo-w-busybox)
4. [Run a crictl demo (w/ custom container)](#zap-run-a-crictl-demo-w-custom-container)
5. [Crictl Verification & Management](#gear-crictl-verification--management)

## :floppy_disk: Crictl Installation <small>[Back to Top](#kata-containers-with-crictl)</small>

1. Download the `crictl` tarball from its GitHub release page.

    _:hourglass: (v1.29.0 as of January 2024)_

    ```bash
    curl -L -o /tmp/crictl.tar.gz "https://github.com/kubernetes-sigs/cri-tools/releases/download/v1.29.0/crictl-v1.29.0-linux-amd64.tar.gz"
    ```

2. Extract the tarball to the /bin directory

    _(kata currently requires sudo privileges to run, so it requires crictl to run as sudo)_
    
    ```bash
    sudo tar -xzf /tmp/crictl.tar.gz -C /bin/
    ```
    _Now you can use `sudo crictl -h` for additional details_

3. Configure crictl by setting the runtime endpoint, image endpoint, and timeout.

    _(I noticed that it's generally safe to provide at least 10 seconds for timeout, to run kata)_


    ```bash
    sudo crictl config --set runtime-endpoint=unix:///run/containerd/containerd.sock
    sudo crictl config --set image-endpoint=unix:///run/containerd/containerd.sock
    sudo crictl config --set timeout=30
    ```
    _After you run these commands, crictl.yaml will be created at `/etc/crictl.yaml` as a configuration file. You may directly modify this file to configure afterwards._


## :floppy_disk: CNI Plugins Installation <small>[Back to Top](#kata-containers-with-crictl)</small>

**CNI (Container Network Interface) plugins** are a set of networking standards/libraries for configuring network interfaces in Linux containers. When you run containers using Kata Containers and crictl, CNI plugins allow these containers to have proper network configurations, facilitating communication between containers, with the host, and with external networks.

> :memo: _Note_: containerd is often automatically configured to use cni plugins as default. You may check that it actually is, via `containerd config dump | grep cni`. It should result in something similar to:
```bash
[root@ctm vagrant]# containerd config dump | grep cni
    [plugins."io.containerd.grpc.v1.cri".cni]
      bin_dir = "/opt/cni/bin"
      conf_dir = "/etc/cni/net.d"
```
> If not, go ahead and modify the paths by editing /etc/containerd/config.toml


1. Download CNI plugins:

    ```bash
    sudo curl -L -o /tmp/cni-plugins.tgz https://github.com/containernetworking/plugins/releases/download/v1.4.0/cni-plugins-linux-amd64-v1.4.0.tgz
    ```
    _:hourglass: (v1.4.0 as of January 2024)_
2. Create the CNI bin directory:

    ```bash
    sudo mkdir -p /opt/cni/bin
    ```
    
3. Extract CNI plugins to the created directory:

    ```bash
    sudo tar -xzf /tmp/cni-plugins.tgz -C /opt/cni/bin
    ```
    _(You may view the different types of network binaries available in this directory now)_

4. Create the CNI configuration file at `/etc/cni/net.d/`. For example:

    ```bash
    echo '{
        "cniVersion": "0.4.0",
        "name": "cni-network",
        "type": "bridge",
        "bridge": "br-cni",
        "isGateway": true,
        "ipMasq": true,
        "ipam": {
            "type": "host-local",
            "subnet": "172.19.0.0/24",
            "routes": [
                { "dst": "0.0.0.0/0" }
            ]
        }
    }' | sudo tee /etc/cni/net.d/10-mynet.conf
    ```
    _(Configuration above will create a bridged network for the pods upon deployment)_

5. Restart containerd.service:

    ```bash
    sudo systemctl restart containerd.service
    ```

<details>
  <summary>:mag: "Click to view" Details on CNI Configuration </summary>

##### When using CNI with containerd, it's crucial to understand how containerd selects the network configuration:

  - **Configuration File Selection**: Containerd uses the CNI configuration files located in `/etc/cni/net.d/`. If multiple configuration files are present, containerd will use the first one in lexicographic (alphabetical) order. For instance, if you have `10-mynet.conf`, `20-mynet.conf`, and `30-mynet.conf`, containerd will use `10-mynet.conf`. This behavior emphasizes the need for careful naming and management of your CNI configuration files.

  - **Network Configuration Example**: The provided configuration sets up a common `bridged` network. This configuration creates a network bridge named `br-cni`, acting as a gateway for the containers. The specified gateway address will be `172.19.0.1`, and the subnet `172.19.0.0/24` provides IPs like `172.19.0.2`, `172.19.0.3`, etc. The limit for the IPs depends on the subnet's size; in this case, it can theoretically accommodate up to 254 pods (`172.19.0.2` to `172.19.0.255`).

  - :lock: **Dynamic Configuration and Security Policies**: I have heard a bit about how advanced CNI plugins like Calico and Cilium might be capable of dynamically selecting the appropriate network configuration for different pods. They can also _potentially_ enforce network security policies and restrictions, potentially integrating with SELinux policies for enhanced security. This level of customization and control requires a complex, multi-tenant Kubernetes environments.

  - :link: **Learning More about CNI**: For a deeper understanding of CNI plugins, their configuration, and how to utilize them effectively within containerized environments, consider visiting the official Container Network Interface GitHub repository and documentation. [Container Network Interface (CNI)](https://github.com/containernetworking/cni)

</details>


## :zap: Run a crictl demo (w/ busybox) <small>[Back to Top](#kata-containers-with-crictl)</small>

### Setting Up the Environment

1. Create a `demo` folder and navigate to it:
    ```bash
    mkdir demo
    cd demo
    ```
2. Create `sandbox_config.json`:
    ```bash
    echo '{
        "metadata": {
            "name": "busybox-pod",
            "uid": "busybox-pod",
            "namespace": "test.kata"
        },
        "hostname": "busybox_host",
        "log_directory": "",
        "dns_config": {},
        "port_mappings": [],
        "resources": {},
        "labels": {},
        "annotations": {},
        "linux": {}
    }' > sandbox_config.json
    ```
   _(This JSON file defines the configuration for the sandbox (pod) where our container will run. It includes metadata like the name, namespace, and hostname.)_

3. Create `container_config.json`:
    ```bash
    echo '{
        "metadata": {
          "name": "busybox-container",
          "namespace": "test.kata"
        },
        "image": {
          "image": "docker.io/library/busybox:latest"
        },
        "command": [
          "sleep",
          "9999"
        ],
        "args": [],
        "working_dir": "/",
        "log_path": "",
        "stdin": false,
        "stdin_once": false,
        "tty": false
    }' > container_config.json
    ```
   _(This JSON file defines the container configuration. It specifies the container image (BusyBox in this case) and the command to execute.)_

### Deploying the Pod and Container

1. Pull the BusyBox image:
    ```bash
    sudo crictl pull busybox
    ```
   _(Downloads the BusyBox image which will be used in our container.)_

2. Create the pod and save its ID:
    ```bash
    POD_ID=$(sudo crictl runp sandbox_config.json)
    ```
   _(Creates the pod using the `sandbox_config.json` and stores its ID in `POD_ID`.)_
   > :memo: _Note_: This is the step when the CNI plugins come into play to automatically spawn the gateway + veth pairs for the microVM; We can go ahead and do `ip addr show` to see the newly created interfaces.

3. Create the container and save its ID:
    ```bash
    CONTAINER_ID=$(sudo crictl create $POD_ID container_config.json sandbox_config.json)
    ```
   _(Creates the container to the created pod (`POD_ID`) using `container_config.json` and `sandbox_config.json`, then stores its ID in `CONTAINER_ID`.)_

4. Start the container:
    ```bash
    sudo crictl start $CONTAINER_ID
    ```

5. Verify your running pod/container:
    ```bash
    sudo crictl ps -a
    ```


> :memo: _Note_: The pod and container IDs are crucial for managing them with crictl commands.


## :zap: Run a crictl demo (w/ custom container) <small>[Back to Top](#kata-containers-with-crictl)</small>

### Setting Up the Custom Container Environment

1. Navigate to the home directory and set up a custom demo environment:
    ```bash
    mkdir demo-custom
    cd demo-custom
    ```

2. Create a new `flask-app` folder:
    ```bash
    mkdir flask-app
    cd flask-app
    ```

3. Create a simple custom flask app, `flask-echo.py`:
    ```bash
    echo 'from flask import Flask
    from datetime import datetime

    app = Flask(__name__)

    @app.route("/")
    def echo():
        return f"<h3>Hello Flask!</h3><p>|    Time: {datetime.now()}</p>"

    if __name__ == "__main__":
        app.run(host="0.0.0.0", port=9099)' > flask-echo.py
    ```
    _(We'll make the flask app run on port 9099)_

4. Add a `requirements.txt` file for dependencies:
    ```bash
    echo 'click==8.0.4
    distro==1.8.0
    Flask==2.0.3
    importlib-metadata==4.8.3
    itsdangerous==2.0.1
    Jinja2==3.0.3
    MarkupSafe==2.0.1
    selinux==0.2.1
    typing_extensions==4.1.1
    Werkzeug==2.0.3
    zipp==3.6.0
    requests' > requirements.txt
    ```

5. Create a `Dockerfile`:
    ```bash
    echo 'FROM python:3.9-slim

    WORKDIR /app

    ADD . /app

    RUN pip install --no-cache-dir -r requirements.txt

    EXPOSE 9099

    CMD ["flask", "run", "--host=0.0.0.0", "--port=9099"]' > Dockerfile
    ```

### Building and Deploying the Container

#### To build and import our customly built container images into crictl, we need a builder tool. We'll go ahead and work with `podman` for this case, as it does not require any additional daemons to run.

1. Install Podman (if not already installed):
    ```bash
    sudo yum install -y podman
    ```

2. Build the container image:
    ```bash
    sudo podman build -t flask-echo:latest $HOME/demo-custom/flask-app/
    ```
    > :memo: _Note_: This command creates a new network interface named `cni-podman0`, which is visible when running `ip addr show`. This occurs because Podman uses CNI for container networking. On first use, Podman automatically sets up a default network bridge (`cni-podman0`) for container communication. This bridge allows containers to connect to each other and the host network, facilitating network isolation and management. However, we only need this interface for the build, and will not be utilizing it in the future after this process.


3. Save the container image to a tar file:
    ```bash
    sudo podman save -o $HOME/demo-custom/flask-app/flask-echo.tar flask-echo:latest
    ```
   _(Exports the container image as a tarball that can be imported.)_

4. Import the image into containerd:
    ```bash
    sudo ctr -n k8s.io images import $HOME/demo-custom/flask-app/flask-echo.tar
    ```

5. Verify the imported image in crictl:
    ```bash
    sudo crictl images
    # you will see the image `localhost/flask-echo`
    # our crictl now has our custom container image
    # and is ready for the deployment
    ```

### Running the Container with crictl

1. Navigate to the demo-custom directory:
    ```bash
    cd $HOME/demo-custom
    ```

2. Create `echo-pod.json` for the pod configuration:
    ```bash
    echo '{
        "metadata": {
            "name": "echo-pod",
            "uid": "echo-pod-uid",
            "namespace": "default"
        },
        "hostname": "echo-host",
        "log_directory": "/tmp",
        "dns_config": {},
        "port_mappings": [],
        "resources": {},
        "labels": {},
        "annotations": {},
        "linux": {}
    }' > echo-pod.json
    ```

3. Create `flask-echo-container.json` for the container configuration:
    ```bash
    echo '{
        "metadata": {
            "name": "flask-echo-container",
            "namespace": "default"
        },
        "image": {
            "image": "localhost/flask-echo:latest"
        },
        "command": [
            "flask",
            "run",
            "--host=0.0.0.0",
            "--port=9099"
        ],
        "envs": [
            {
                "key": "FLASK_APP",
                "value": "flask-echo.py"
            }
        ],
        "working_dir": "/app",
        "log_path": "flask-echo-container.log",
        "stdin": false,
        "stdin_once": false,
        "tty": false,
        "linux": {}
    }' > flask-echo-container.json
    ```

4. Deploy the pod and container:
    ```bash
    POD_ID=$(sudo crictl runp $HOME/demo-custom/echo-pod.json)
    CONTAINER_ID=$(sudo crictl create $POD_ID $HOME/demo-custom/flask-echo-container.json $HOME/demo-custom/echo-pod.json)
    sudo crictl start $CONTAINER_ID
    ```
    > :memo: _Note_: CNI plugins come into play _after_ the pod has been created, to automatically spawn the gateway `(br-cni)` + veth pairs for the microVM; We can go ahead and do `ip addr show` to see the newly created interfaces.

5. Verify your running pod/container:
    ```bash
    sudo crictl ps -a
    ```

6. Verify your kata processes:
    ```bash
    sudo ps -ef | grep -E 'kata|firecracker'
    ```
    _(You'll be able to see two processes to represent the kata+firecracker: First one being `/opt/kata/bin/containerd-shim-kata-v2` and second `/firecracker`. Each pod (microVM/sandbox) generation will create these two pairs of processes upon deployment.)_

## :gear: Crictl Management <small>[Back to Top](#kata-containers-with-crictl)</small>

### Overview

After deploying containers with crictl, it's essential to verify and manage the resources. Here are some oftenly used key commands for monitoring and managing pods, containers, and images in a crictl/Kata Containers environment.

### :mag: Command Reference Table

#### Pod Management
| Command                              | Description                                   | 
|--------------------------------------|-----------------------------------------------|
| `sudo crictl pods`                   | Lists all pods.                               |
| `sudo crictl pods -v`                | Lists all pods with verbose information.      |
| `sudo crictl inspectp $POD_ID`       | Inspects a specific pod.                      |
| `sudo crictl rmp $POD_ID`            | Removes a specific pod (must be stopped).     |
| `sudo kata-runtime exec $POD_ID`     | Executes into the pod's microVM.              |

#### Container Management
| Command                              | Description                                   | 
|--------------------------------------|-----------------------------------------------|
| `sudo crictl ps -a`                  | Shows all containers and their states.        |
| `sudo crictl inspect $CONTAINER_ID`  | Inspects a specific container.                |
| `sudo crictl rm $CONTAINER_ID`       | Removes a specific container (must be stopped).|
| `sudo crictl logs $CONTAINER_ID`     | Checks the logs for a specific container.     |
| `sudo crictl exec -it $CONTAINER_ID sh`| Executes a shell session in a container.   |

#### Image Management
| Command                              | Description                                   | 
|--------------------------------------|-----------------------------------------------|
| `sudo crictl img`                    | Lists all stored images.                      |
| `sudo crictl inspecti $IMAGE_NAME`   | Inspects a specific image.                    |
| `sudo crictl rmi $IMAGE_NAME`        | Removes a specified image.                    |

> :memo: _Note_: `sudo crictl -h` provides information on all the possible commands.

<details>
  <summary>:mag: "Click to view" Simple script to reset crictl environment</summary>



</details>
```

Within the  <summary>:mag: "Click to view" Simple script to reset crictl environment</summary>, I'd like to include a simple script that can pretty much "Reset" the pods for crictl (because otherwise, we'd have to remove one by one) for dev purposes. Let's call it `clear-pods.sh`. I currently have:
```
#!/bin/bash

# Get all pod IDs
POD_IDS=$(sudo crictl pods -q)

# Stop and remove each pod
for pod_id in $POD_IDS; do
 pod_name=$(sudo crictl inspectp $pod_id | jq -r '.status.metadata.name')
 echo "Stopping and removing pod: $pod_name"
 sudo crictl stopp $pod_id
 sudo crictl rmp $pod_id
done

# Clean up network interfaces
# Remove veth pairs associated with the pods
for veth in $(ip link | grep veth | cut -d: -f2); do
 sudo ip link delete $veth
done

# Find all bridge names that start with 'br-'
BRIDGE_NAMES=$(ip link show | awk '/^[0-9]+: br-/{print $2}')

# Iterate over the bridge names
for BRIDGE_NAME in $BRIDGE_NAMES; do
 # Set the bridge down
 sudo ip link set $BRIDGE_NAME down

 # Delete the bridge
 sudo ip link delete $BRIDGE_NAME type bridge
done
