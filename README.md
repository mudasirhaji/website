Deploy a Website on an Apache Web Server on Multiple EC2 Instances using Ansible Playbook

Why Ansible?


No matter your role, or what your automation goals are, Ansible can help you demonstrate value, connect teams, and deliver efficiencies for your organization. Ansible delivers simple IT automation that ends repetitive tasks and frees up DevOps teams for more strategic work.


Agenda
In this blog, we will learn how to deploy a website on multiple AWS EC2 Instances using Ansible Playbooks without the need to install the service on each server separately through the power of Automation which is the ultimate need for DevOps practice and ‘Automate everything’ is the key principle of DevOps. 

Pre-requisites:
AWS account
Basic Knowledge of Ansible & Ansible playbooks
Basic Linux Knowledge
Project Architecture: The below image shows the reference architecture of our project which we would follow in this blog.

Steps Involved:
1. Create a couple of Security Groups under AWS VPC first for our Ansible Control Master Server and then another for Ansible nodes.Log into the AWS console, choose your preferred region, head back to VPC, and select security groups on the left navigation bar as shown below:

Click on Create security group:

 Provide the basic details of your security group such as name, and description, and select the default VPC as shown below:

Under Inbound rules allow ssh from anywhere for this demonstration and click on Create security group.

Now let’s create another security group that would be common for all three Ansible nodes on which we would deploy the web server.

Under Inbound Rules allow HTTP from anywhere and allow SSH from the master security group only and click on Create security group as depicted below:


2. Now that we have created both our security groups it's time to create the Ansible master EC2 Instance.
    Go to the EC2 dashboard, and click on launch instance:


Provide the name for your EC2 Instance:



Under AMI, select Amazon Linux 2023 AMI which is free-tier eligible:



Under Instance type, choose t2.micro which provides 1 vCPU, 1 GiB Memory, and is also free tier eligible.




Under key pair which allows us to ssh into the EC2 instance, we can either use an existing or create new pair. For this demo, we will create a new key pair.




Provide the name for your new key pair and click on Create key pair. Make sure you save this key pair as you can download it only once.


Now edit your Network Settings. Select the default VPC, select the subnet of your choice, or leave with no preference. Auto-assign public IP should be enabled and under security, groups choose the existing security group for the ansible-master server we created in step-1.



Rest of the settings we can keep them at default and click on the Launch instance.

The success message shows the completion of the EC2 Instance, now click on Connect to Instance link as shown:

Now on the Connect to instance windows, copy the following commands and paste them into your terminal in order to ssh into the server:

chmod 400 devops.pem
ssh -i "devops.pem" ec2-user@ec2-52-90-115-27.compute-1.amazonaws.com



Terminal Output:




3. Create a key-pair on the Ansible master server that would be used to establish a connection between the Ansible master and Ansible nodes with the command:

                              ssh-keygen -t rsa -b 2048




4. Now after creating the SSH key pairs we need to import the public key into the EC2 console which would allow the Ansible nodes to use that key in order to establish communication with the Ansible master server.
Copy the public key by following the below steps as shown in the output:




After copying the public key, go to the EC2 management console, and on the left navigation bar under Network & Security, select Key Pairs. Then under Actions click on Import key pair.





Provide the name of the new key being imported and paste the public key from the terminal. Then click on Import key pair.


5. Launch Ansible Node Servers:
    Now we will launch three EC2 Instances for this demo.
    Go to the EC2 dashboard and click on Launch instances.
    Provide the name and under the Number of instances type 3.



Select Amazon Linux 2023 AMI and Instance type as t2.micro.
Here we have to make sure that under the key pair, we to need select ansible-public-key which we created earlier in our demo.

Under Network settings select the default VPC, Auto-assign public IP as enabled. Make sure to select the existing security group ansible-nodes-sg which we created in earlier steps.

Keep the rest of the settings at default and launch the instances.

After successfully launching, we can view the instances and tag them appropriately as shown below:


6. Test the connectivity between Ansible master and node servers:
     To test the ssh connection between the master and worker/slave nodes, we would log into the master Ansible server, and take the private IP of the first node server from the console as shown below:


Go to the terminal of the Ansible master server and type the following command as shown in the below screenshot.



Check the private IP which should be the same one that we copied earlier from the AWS console.
In the same way, test the connectivity between the other two servers and the master server.

Note: Since we are able to connect between the Ansible Master Server and all other nodes, it is to be noted that this connection is made possible by the SSH itself and not Ansible as so far we haven’t installed the Ansible in the master server.

7. Install Ansible on the Master Server:
First, we will update our Ansible master server with the below command:
                                    sudo yum update -y

Since we are using Amazon Linux 2023 AMI, the installation steps would be different than that for Amazon Linux2 AMI.
Copy and paste the below commands to install the Ansible:
                         curl https://bootstrap.pypa.io/get-pip.py -o get-pip.py
                         python3 get-pip.py --user
                         python3 -m pip install --user ansible



To check if Ansible was successfully installed check the version with the below command:

                                    
8. Create an Ansible Inventory file:
                             Ansible automates tasks on managed nodes or “hosts” in your infrastructure, using a list or group of lists known as inventory. You can pass host names at the command line, but most Ansible users create inventory files. Your inventory defines the managed nodes you automate, with groups so you can run automation tasks on multiple hosts at the same time.
              Within the inventory file, you can organize your servers into different groups and subgroups.
Let's create an Inventory file on the Ansible Master Server with the below command:
                             Sudo vi inventory
Create a group name webservers and under it type the IP addresses of all the node servers as shown below:


                              

9. Create an Ansible Playbook:
                         An Ansible Playbook is a blueprint of automation tasks—which are complex IT actions executed with limited or no human involvement. Ansible Playbooks are executed on a set, group, or classification of hosts, which together make up an Ansible inventory.
Let’s create the Playbook on the Ansible Master Serve in the home directory where we have our inventory file as well:
                          Sudo vi website.yml
Once the file gets opened paste the code into the playbook file. You can find the link to the GitHub repo at the end of this blog.


[ec2-user@ip-172-31-92-75 ~]$ cat website.yml
---

- name: deploy bootstrap-website
  hosts: all
  become: yes
  become_user: root

  tasks:
    - name: update ec2 instance
      yum:
        name: "*"
        state: latest
        update_cache: yes

    - name: install apache server
      yum:
        name: httpd
        state: latest

    - name: change directory to the html directory
      shell: cd /var/www/html

    - name: download web files from github
      get_url:
        url: https://github.com/mudasirhaji/website/archive/refs/tags/zipfile.zip
        dest: /var/www/html/

    - name: unzip the zip folder
      ansible.builtin.unarchive:
        src: /var/www/html/website-zipfile.zip
        dest: /var/www/html
        remote_src: yes

    - name: copy webfiles from the website-zipfile directory to the html directory
      copy:
        src: /var/www/html/website-zipfile/
        dest: /var/www/html
        remote_src: yes

    - name: remove the website-zipfile directory
      file:
        path: /var/www/html/website-zipfile
        state: absent

    - name: remove the website-zipfile.zip folder
      file:
        path: /var/www/html/website-zipfile.zip
        state: absent

    - name: start apache server, if not started
      ansible.builtin.service:
        enabled: yes
        name: httpd
        state: started



Before running the playbook to install our website on the Apache server let’s first test the connectivity between the master and all the nodes using the Ansible command: 
                    ansible all --key-file ~/.ssh/id_rsa -i inventory -m ping -u ec2-user
The successful output of the above command should look like this:



The above output in green shows that we are able to connect with the nodes using Ansible.
One more thing to do before running our Ansible Playbook is to create an Ansible configuration file in which we will list the path to our key pair, inventory file, and the default username of ec2-user so that we don’t have to mention every time we run our playbook.
To create the Ansible config file type
                      Sudo vi ansible.cfg
Once the file is open enter the below details as follows:
                         [defaults]
                         remote_user = ec2-user
                         inventory = inventory
                         private_key_file = ~/.ssh/id_rsa

Save and exit the file.

10. Run the Playbook to Install Webserver on the Servers:
      To run the playbook type the below command on the Ansible Master Server:
                ansible-playbook website.yml


We can also confirm by accessing any one of the Node Server public IP from our browser. Go to the EC2 console and copy the Public IP of Ansible-node-1 server as follows:



Then paste the Pubic IP on your browser and you should see your website as I am able to see mine:


In the same, you can confirm the successful deployment of our website on other node servers as well.

Conclusion
In this blog, we learned how to write a playbook using Ansible on an EC2 instance and deploy a website on an Apache web server on multiple EC2 instances.

