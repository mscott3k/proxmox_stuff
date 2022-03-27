# proxmox_stuff

Steps taken to install pve7.x


- **Install Proxmox VE (pve)**
  - reboot

- ***AS root***

    - **Update sources**
      - Add no-subscription source
        ``` 
        nano /etc/apt/sources.list 
        ```
        - add:
          ``` 
          deb http://download.proxmox.com/debian stretch pve-no-subscription 
          ```
      - Then remove the enterprise source:
        ``` 
        nano /etc/apt/sources.list.d/pve-enterprise.list 
        ```
        - comment out repo:
          ``` 
          deb https://enterprise.proxmox.com/debian stretch pve-enterprise 
          ```
      - Then update the machine:
        ``` 
        apt-get update && apt-get dist-upgrade -y 
        ```
      - reboot

    - ***GUI:* Enable firewall (on at least datacenter level)**. 
      - Even if you do not use firewall in Proxmox you must enable it (just set default policy to ACCEPT), because Docker will use netfilter.

    - **Prepare iptables for Docker:** 
      ``` 
      iptables -N DOCKER-USER; iptables -I DOCKER-USER -j ACCEPT 
      ```
      - **Make the change permanent:**
        ``` 
        nano /etc/network/interfaces 
        ```
          - add:
            ``` 
            auto lo
            iface lo inet loopback
            # iptable config for Docker
            pre-up iptables -N DOCKER-USER; iptables -I DOCKER-USER -j ACCEPT
            ```

    - **Create zfs volumes:**
      ```
      zfs create -o mountpoint=/var/lib/docker rpool/docker-root
      zfs create -o mountpoint=/var/lib/docker/volumes rpool/docker-volumes
      chmod -R 770 /var/lib/docker
      ```

    - **Setup zfs storage driver for docker**
      ```
      mkdir /etc/systemd/system/docker.service.d
      nano /etc/systemd/system/docker.service.d/storage-driver.conf
      ```
      - add:
        ```
        [Service]
        ExecStart=
        ExecStart=/usr/bin/dockerd --storage-driver=zfs -H fd://
        ```
        
    - **install needed/helpful apt packages:**
      ```
      apt install --install-recommends sudo git htop lm-sensors net-tools nethogs cmake
      ```
      - enable ```lm-sensors```:
        ```
        sensors-detect
        ```
      - [btop++](https://github.com/aristocratos/btop#prerequisites)

    - **Setup additional user**
      ```
      useradd -m mscott
      usermod --shell /bin/bash mscott
      usermod -aG users,sudo,root mscott
      passwd mscott
      ```

    - ***GUI:* Setup user, groups, pools, permissions:**
      - Users:
        - Add linux PAM user(s) created above
      - Groups:
        - Create groups needed
        - ex: admin, assign to root (/) and propagate in permissions. Users in this group have complete control
      - Pools:
        - Optional
        - Create pools to delegate permissions to users/groups
      - Permissions:
        - Assign group/user/API Token permissions

- ***AS [user]***
  - **Install [Docker](https://docs.docker.com/engine/install/debian/):**
    - Update the apt package index and install packages to allow apt to use a repository over HTTPS:
      ```  
      sudo apt-get update
      sudo apt-get install ca-certificates curl gnupg lsb-release
      ```    
    - Add Docker’s official GPG key:
      ```
      curl -fsSL https://download.docker.com/linux/debian/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
      ```
    - Use the following command to set up the stable repository. To add the nightly or test repository, add the word nightly or test (or both) after the word stable in the commands below. Learn about nightly and test channels.
      ```
      echo \
      "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/debian \
      $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
      ````
    - Install Docker Engine
      ```
      sudo apt-get update
      sudo apt-get install docker-ce docker-ce-cli containerd.io -y
      ```
    - Add user to group
      ```
      sudo usermod -aG docker $USER
      ```
    - Logout to init group changes

  - **Install Portainer**
    - Setup zfs dataset
      ```
      sudo zfs create rpool/docker-volumes/portainer_data
      sudo chmod -R 770 /var/lib/docker
      ```
    - Create docker volume
      ```
      docker volume create portainer_data
      ```
    - Create portainer container
      ```
      docker run -d -p 8000:8000 -p 9000:9000 --name=portainer --restart=always -v /var/run/docker.sock:/var/run/docker.sock -v portainer_data:/data portainer/portainer-ce
      ```
    - Setup portainer
      - Go to ```http://[serverIP]:9000``` and configure
  
  
### Sources
 - https://www.servethehome.com/creating-the-ultimate-virtualization-and-container-setup-with-management-guis/
 - https://www.servethehome.com/setup-docker-on-proxmox-ve-using-zfs-storage/
 - https://forum.proxmox.com/threads/tutorial-proxmox-with-docker-and-portainer.77275/
 - https://docs.docker.com/engine/install/debian/
