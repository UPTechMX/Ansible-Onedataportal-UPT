# ansible-onedataportal
[Ansible](https://docs.ansible.com) playbooks for automatic deployment of the One Data Portal (Portal Satu Data) comprising of [CKAN](https://ckan.org) version 2.8.3 and [Oskari](https://www.oskari.org/) version 1.55.2.

## Requirements
### Target server
* A fresh installed [Ubuntu Server 18.04](https://ubuntu.com/download/server) (minimum 4 GB RAM and 20 GB of free space) with a user that is part of the `sudo` group (IMPORTANT: using the root user is not recommended since by default the server provisioning playbook `0_provision_server.yml` will disable root user login for better security). Below is an example of a user called `username` that is created and added to the `sudo` group. 
  ```
  adduser username
  usermod -aG sudo username
  ```
* The server is fully updated and configured with a static IP address (below is an example setting the server IP address to 192.168.1.200 on the `enp0s3` network interface, check the available interface name with `ifconfig`)
  ```
  sudo apt update && sudo apt upgrade
  sudo nano /etc/netplan/50-cloud-init.yaml
  ```
  ```
  network:
     ethernets:
         enp0s3:
             dhcp4: no
             addresses: [192.168.1.200/24]
             gateway4: 192.168.1.1
             nameservers:
                 addresses: [8.8.8.8,8.8.4.4]
     version: 2
  ```
  ```
  sudo netplan apply
  sudo reboot
  ```
* Ansible installed on the target server
  ```
  sudo apt update
  sudo apt install -y software-properties-common
  sudo apt-add-repository --yes --update ppa:ansible/ansible
  sudo apt install -y ansible
  ansible --version
    ansible 2.9.6
      config file = /etc/ansible/ansible.cfg
      configured module search path = [u'/home/user/.ansible/plugins/modules', u'/usr/share/ansible/plugins/modules']
      ansible python module location = /usr/lib/python2.7/dist-packages/ansible
      executable location = /usr/bin/ansible
      python version = 2.7.17 (default, Nov  7 2019, 10:07:09) [GCC 7.4.0]
  ```
  
## How to run
SSH to the target server and perform the following steps:
* Clone this repo (install `git` package if necessary)
  ```
  # clone the ansible playbook repo to /opt
  cd /opt
  # become root user
  sudo su
  git clone https://github.com/geoenvo/ansible-onedataportal.git
  cd ansible-onedataportal
  ```
* Configure the mandatory playbook variables
  * ```nano vars_provision_server.yml```
    ```
    server_timezone: set this to the server's timezone (default is Asia/Makassar)
    ```
  * ```nano vars_postgresql.yml```
    ```
    (Optional, recommended) postgres_user_password: change the password for the default postgres role
    ```
  * ```nano vars_ckan.yml```
    ```
    ckan_site_url: set this to the IP address or domain name of the server where CKAN will be accessed (default is http://192.168.1.200)
    (Optional, recommended) ckan_admin.username: set the username for the CKAN admin user (default is admin)
    (Optional, recommended) ckan_admin.password: set the default password for the CKAN admin user
    (Optional) ckan_max_resource_size: change the default resource file size upload limit (default is 100 MB)
    (Optional) ckan_max_image_size: change the default image file size upload limit (default is 10 MB)
    ```
  * ```nano vars_oskari.yml```
    ```
    oskari_domain: set this to the IP address or domain name of the server where Oskari will be accessed (default is http://192.168.1.200:8080)
    ```
* Run the playbooks
    * Recommended method: run the install all playbook to install everything in order
      * This will run the playbooks above in correct order from 0 to 6
        ```
        cd /opt/ansible-onedataportal
        # become root user
        sudo su
        ansible-playbook -K -i inventory install_all.yml
          BECOME password: enter the user's sudo password
        ```
    * Or one by one
      * Must be run in the order of the filename
        ```
        cd /opt/ansible-onedataportal
        # become root user
        sudo su
        ansible-playbook -K -i inventory 0_provision_server.yml
          BECOME password: enter the user's sudo password
        ansible-playbook -K -i inventory 1_install_postgresql.yml
        ansible-playbook -K -i inventory 2_install_solr.yml
        ansible-playbook -K -i inventory 3_install_ckan.yml
        ansible-playbook -K -i inventory 4_install_ckan_extras.yml
        ansible-playbook -K -i inventory 5_install_oskari.yml
        ```
    * NOTE: to speed up installation copy `solr-6.5.1.tgz`, `jetty-9.4.12-oskari.zip`, `geoserver-2.13.2-war.zip`, and `oskari-map.war` to /tmp to skip the long tasks for downloading and building
    * Optional: install the development tools for customizing/building Oskari components
      * Refer to https://oskari.org/documentation/backend/setup-development
      * PostgreSQL and PostGIS must be installed first if they haven't been installed above
        ```
        cd /opt/ansible-onedataportal
        # become root user
        sudo su
        ansible-playbook -K -i inventory 1_install_postgresql.yml
          BECOME password: enter the user's sudo password
        ansible-playbook -K -i inventory 6_install_oskari_development.yml
        ```
* Some manual configuration might be needed to achieve the following (this is optional):
  * Configure firewall to further secure the server
  * Reverse proxy a (sub)domain with SSL certificate
  * Change the default passwords for CKAN/Oskari/GeoServer
  * Modify advanced CKAN and Oskari settings
      * Default CKAN configuration file path is `/etc/ckan/ckan/production.ini`
      * Default Oskari configuration file path is `/opt/oskari/oskari-server/resources/oskari-ext.properties`
  * Using and updating a non-english localization
  * Building Oskari frontend and server components
  * Check this repo's wiki for more details
* Reboot the server
  ```
  sudo reboot
  ```
* Access the One Data Portal at the server's IP address or domain name on a web browser:
  * By default the CKAN Data Portal is accessible at http://the.server.ip.address (reverse proxied by NGINX from http://the.server.ip.address:8090)
  * CKAN DataPusher service at http://the.server.ip.address:8800/
  * pycsw service at http://the.server.ip.address:8000/?service=CSW&version=2.0.2&request=GetCapabilities
  * Solr admin at http://the.server.ip.address:8983/solr/
  * Oskari Geoportal at http://the.server.ip.address:8080
  * GeoServer admin at http://the.server.ip.address:8082/geoserver/web
* Done!
