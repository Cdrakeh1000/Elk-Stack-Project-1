# Elk-Stack-Project-1
***AUTOMATED ELK STACK DEPLOYMENT**

For this cloud environment project I was responsible for creating virtual networks, resource groups, network security groups, and virtual machines (three web VMs, a jumpbox, and an Elk server) that were distributed along with a load balancer on Microsoft Azure. I deployed an Elk Stack VM with Kibana using an ansible container which I also developed then installed Filebeat and Metricbeat to monitor vulnerable web servers that my Red Team was using.. 

These files have been tested and used to generate a live ELK deployment on Azure. Thy can be used to recreate the entire deployment pictured.

![VNets](/Users/christopherhaverick/Desktop/Elk_Stack_Project/Images/VNets.png)

![Resource Groups](/Users/christopherhaverick/Desktop/Elk_Stack_Project/Images/Resource Groups.png)

![Security Groups](/Users/christopherhaverick/Desktop/Elk_Stack_Project/Images/Security Groups.png)

![Virtual Machines](/Users/christopherhaverick/Desktop/Elk_Stack_Project/Images/Virtual Machines.png)

![Load Balancer](/Users/christopherhaverick/Desktop/Elk_Stack_Project/Images/Load Balancer.png)





**CLOUD SECURITY VIRTUALIZATION NETWORK DIAGRAM **

![Cloud Security Network Diagram](/Users/christopherhaverick/Desktop/Elk_Stack_Project/diagrams/Cloud Security Network Diagram.png)



**DAY 1: ELK INSTALLATION**

On day 1 I configured an ELK server within my virtual network. Specifically, I:

1. Created a new virtual network called ELKVnet in a new region (West US) within my Red-Team resource group.
2. Created a Peer Network Connection between my two vNets - ELKVnet and RedTeamNet.
3. Created a new VM called ElkVM and deployed this VM into the ELKVnet with its own Security Group. This VM was host to the ELK server.
4. Downloaded and configured the elk-docker container onto the ElkVM.
5. Launched and exposed the elk-docker container to start the ELK server.
6. Implemented indentiy and access management by configuring the new security group so that I can connect ELK via HTTP and view it through the browser.



**PEERING THE vNETS**

![Elk to Red vNET peering](/Users/christopherhaverick/Desktop/Elk_Stack_Project/Images/Elk to Red vNET peering.png)

![Red to Elk vNET peering](/Users/christopherhaverick/Desktop/Elk_Stack_Project/Images/Red to Elk vNET peering.png)

**DOWNLOADING AND CONFIGURING THE CONTAINER**

1. Using Ansible I configured the ElkVM.
2. From the Ansible container I added the ElkVM to Ansibles /etc/ansible/hosts file. 
3. Created a playbook that installs docker and configures the container
4. Ran the playbook to launch the container
5. The ElkVM was created to run my ELK stack. In order to use Ansible to configure this VM I had to add it to the list of machines Ansible can discover and connect to. 



**Logging into /etc/ansible/hosts**

![logging into ansible hosts](/Users/christopherhaverick/Desktop/Elk_Stack_Project/ansible/logging into ansible hosts.png)

![:etc:ansible:hosts](/Users/christopherhaverick/Desktop/Elk_Stack_Project/ansible/:etc:ansible:hosts.png)

6. Once I created the [elk] group, I created a playbook to configure it called install-elk.yml

```
---
- name: Configure Elk VM with Docker
  hosts: elk
  remote_user: sysadmin
  become: true
  tasks:
    # Use apt module
    - name: Install docker.io
      apt:
        update_cache: yes
        force_apt_get: yes
        name: docker.io
        state: present

      # Use apt module
    - name: Install python3-pip
      apt:
        force_apt_get: yes
        name: python3-pip
        state: present

      # Use pip module (It will default to pip3)
    - name: Install python Docker module
      pip:
        name: docker
        state: present

      # Use command module
    - name: Increase virtual memory
      command: sysctl -w vm.max_map_count=262144

      # Use sysctl module
    - name: Use more memory
      sysctl:
        name: vm.max_map_count
        value: "262144"
        state: present
        reload: yes

      # Use docker_container module
    - name: download and launch a docker elk container
      docker_container:
        name: elk
        image: sebp/elk:761
        state: started
        restart_policy: always
        # Please list the ports that ELK runs on
        published_ports:
          -  5601:5601
          -  9200:9200
          -  5044:5044

      # Use systemd module
    - name: Enable service docker on boot
      systemd:
        name: docker
        enabled: yes

```

7. Next I ran the playbook using the command: ansible-playbook install-elk.yml. This will enable the docker service on boot, so that when I restart the ElkVM the docker service starts up automatically. 

![ansible-playbook install-elk yml](/Users/christopherhaverick/Desktop/Elk_Stack_Project/ansible/ansible-playbook install-elk yml.png)

8. Next I restricted access to the ElkVM using Azure's network security groups. The ELK stack's web server runs on port 5601. I opened my virutal network's existing NSG and created an incoming rule for my security group that allows TCP traffic over port 5601 from my public IP address. 

![ElkVM NSG 5601 rule](/Users/christopherhaverick/Desktop/Elk_Stack_Project/Images/ElkVM NSG 5601 rule.png)

9. Finally, I verified that i could access my server by navigating to {ElkVM public IP}:5601/app/kibana

![Day 1 Kibana](/Users/christopherhaverick/Desktop/Elk_Stack_Project/Images/Day 1 Kibana.png)





***DAY 2: FILEBEAT AND METRICBEAT INSTALLATION****

The ELK monitoring server is installed and configured I added Filebeat and Metricbeat. Filebeat is used to collect, parse, and visualize ELK logs which helps to better keep track of organizational goals. I completed the following:

1. Install Filebeat/Metricbeat on the Web VM's
2. Created a Filebeat/Metricbeat configuration file for my DVWA VMs
3. Created a Filebeat/Metricbeat installation play by creating an Ansible playbook that accomplishes the tasks required to install Filebeat/Metricbeat
4. Verified the installation and playbooks by confirming that both the installation and playbooks worked by verifying that the ELK stack is receiving logs.



5. In the Filebeat config file I replaced the IP address with the private IP address of my ELK machine in two places

   ![filebeat config file 1](/Users/christopherhaverick/Desktop/Elk_Stack_Project/ansible/filebeat config file 1.png)

   ![filebeat config file 2](/Users/christopherhaverick/Desktop/Elk_Stack_Project/ansible/filebeat config file 2.png)

   

   6. Next I created a new playbook that installs Filebeat and copied the configuration file to the correct location. 

```
---
- name: installing and launching filebeat
  hosts: webservers
  become: yes
  tasks:

  - name: download filebeat deb
    command: curl -L -O https://artifacts.elastic.co/downloads/beats/filebeat/filebeat-7.6.2-amd64.deb
      
 
  - name: install filebeat deb
    command: dpkg -i filebeat-7.6.2-amd64.deb

  - name: drop in filebeat.yml 
    copy:
      src: /etc/ansible/filebeat-config.yml
      dest: /etc/filebeat/filebeat.yml

  - name: enable and configure system module
    command: filebeat modules enable system

  - name: setup filebeat
    command: filebeat setup

  - name: start filebeat service
    command: service filebeat start

  - name: enable service filebeat on boot
    systemd:
      name: filebeat
      enabled: yes
```

7. After creating and saving this playbook I ran it with the command ansible-playbook filbert-playbook.yml

![ansible-playbook filebeat-playbook yml](/Users/christopherhaverick/Desktop/Elk_Stack_Project/ansible/ansible-playbook filebeat-playbook yml.png)

8. Then I confirmed that the ELK stack was receiving logs from my DVWA machines on Kibana by navigating to 'Add log data' - 'System logs' - 'System logs dashboard'

![Filebeat Day 2](/Users/christopherhaverick/Desktop/Elk_Stack_Project/Images/Filebeat Day 2.png)

9. Next I installed Metricbeat in the same method of installing Filebeat. I went into the metricbeat.config.yml file and changed the IP adress in the output.elasticsearch and setup.kibana parts of the file to the private IP address of the ElkVM. Then I created a metricbeat-playbook.yml file and ran the playbook. 

```
---
- name: Install metric beat
  hosts: webservers
  become: true
  tasks:
    # Use command module
  - name: Download metricbeat
    command: curl -L -O https://artifacts.elastic.co/downloads/beats/metricbeat/metricbeat-7.6.1-amd64.deb

    # Use command module
  - name: install metricbeat
    command: dpkg -i metricbeat-7.6.1-amd64.deb

    # Use copy module
  - name: drop in metricbeat config
    copy:
      src: /etc/ansible/files/metricbeat-config.yml
      dest: /etc/metricbeat/metricbeat.yml

    # Use command module
  - name: enable and configure docker module for metric beat
    command: metricbeat modules enable docker

    # Use command module
  - name: setup metric beat
    command: metricbeat setup

    # Use command module
  - name: start metric beat
    command: service metricbeat start

    # Use systemd module
  - name: enable service metricbeat on boot
    systemd:
      name: metricbeat
      enabled: yes

```

10. Finally I made sure metric data was being received by the ELK server by navigating Kibana by going to 'Add Metric Data' - 'Docker Metrics' - 'Docker Metrics Dashboard'



![metricbeat day 2.1](/Users/christopherhaverick/Desktop/Elk_Stack_Project/Images/metricbeat day 2.1.png)

![metricbeat day 2.2](/Users/christopherhaverick/Desktop/Elk_Stack_Project/Images/metricbeat day 2.2.png)

