# This repository contains the files needed to set up a secure NiFi via docker at GCP and how to connect with Registry and Github

![dataStructure](https://github.com/tatianamara/nifi-configuration-docker/blob/main/dataStructure.PNG)

## Follow the steps below to make the necessary settings

- Create a GCP account and a new project
- For this project we will use a virtual machine, an instance group, a website domain, an SSL certificate, a DNS zone, a static address and a load balancer

**Let's start by configuring the virtual machine that we will use to configure the NiFi:**  
- Access the VM instance menu through the link below and go to "create instance"  
https://console.cloud.google.com/compute/instances
- Give your instance a name, make sure the region is marked as "us-central1" and in Zone select "us-central1-f"
- Select the E2 series and the machine type according to the amount of memory that will be needed, in my case I used the 8 GB machine
- Change the Boot Disk and in Operating System select "Container Optimized OS" and select the latest stable version available
- Remember to check the Firewall check-boxes: "Allow HTTP traffic" and "Allow HTTPS traffic"
- Click on "Create" and wait while your instance starts up

**Now we will install NiFi in our created instance**
- Click the SSH button to connect to the machine
- Clone your repository using the ```git``` command  
```git clone https://github.com/tatianamara/nifi-configuration-docker.git```  
```cd nifi-configuration-docker```  


- Download and run the Docker Compose image. Check the Docker Compose tags to use the latest version  
```docker run docker / compose: 1.27.4 version```  
- Make sure you have write permission on the directory  
```$ pwd```  
```/ home / username / nifi-configuration-docker```  
- Run the Docker Compose command to run the code.  
For the Docker Compose container to have access to the Docker daemon, mount the Docker socket with the -v /var/run/docker.sock:/var/run/docker.sock option.  
To make the current directory available to the container, use the option -v "$ PWD: $ PWD" to mount it as a volume and -w = "$ PWD" to change the working directory.  
```docker run --rm \```    
```-v /var/run/docker.sock:/var/run/docker.sock \```  
```-v "$PWD:$PWD" \```  
```-w="$PWD" \```  
```docker/compose:1.27.4 up```  

**Now we are almost ready to access NiFi via the external IP provided by GCP, but first we need to create a Firewall rule to release the ports that NiFi will use**  
- In the side menu go to Network -> VPC Network -> Firewall
- Click on "Create Firewall Rule", choose a name for your rule
- In destination tags, place "http-server", "https-server"
- In IP ranges, enter: "0.0.0.0/0"
- And finally in Protocols and ports add the ports that will be used in the project, in our case they will be: "8080", "443"
- Click create rule

**Done! We are now able to access NiFi via the external IP, just click on it and add in the path ": 8080 / nifi" to access**  
**Note: for now it is only accessible via HTTP connection**  


## Registry and NiFi connection
- Access the NiFi interface, click on the menu in the upper right corner and access the controller settings
- In Registry Clients add a connection to the Registry
- Give a name and in the URL paste the URL you use to access the Registry interface and click on ADD
- If you have not yet configured the nifi registry, follow this tutorial: https://github.com/tatianamara/registry-configuration

**Ok, now just create a new Process Group, right click and select the "version" option, select the Registry Bucket you want to use and place a commit message**  

## HTTPS connection configuration
- The first step is to register a domain, in this example I will use a free service called FreeNom, go to the website below and register a domain  
http://www.freenom.com/
- After having the domain, access the GCP page and in the side menu go to Network -> Network services -> Cloud DNS and select "Create Zone"
- Give a name for your zone and in "DNS Name" paste the name of the domain that was created
- Click on create and then click on the name of your zone to see more information, you will see that GCP has already created two records, we will use the NS type to configure our domain there on FreeNom
- Access your domains on FreeNom and on "Manage Domain", then on "Magement Tools" select "Nameservers"
- Select the option "Use custom Nameservers" and copy and paste the data provided by GCP in the fields available on FreeNom
- Click on "Change Nameservers"

## SSL certificate managed by google
- Visit the Certificates page  
https://console.cloud.google.com/loadbalancing/advanced/sslCertificates/list?_ga=2.170675872.188826785.1608555797-56108235.1603201106&_gac=1.238274996.1606138283.EAIaYQBQIYAQAQI 
- Click on "Create SSL certificate", give the certificate a name
- Select "Create Google managed certificate"
- Add the domain you created, click "Create"

## Instance group
- We need to create a group of VM instances to be able to use them when creating the load balancer
- Visit the Instance Group page  
https://console.cloud.google.com/compute/instanceGroups/list
- Click on "Create Instance Group, give it a name
- On the side select "New instance group unmanaged"
- Select the same region and zone that you selected for your VM
- Select the network your VM is on
- And under "VM instances select the one you created earlier and click create

## Reserve an external IP address
- Go to the External IP addresses page  
https://console.cloud.google.com/addresses/list?_ga=2.132522510.188826785.1608555797-56108235.1603201106&_gac=1.18490125.1606138283.EAIaIQobChMIYAQIQQ
- Click on reserve static address and give a name
- Set the "Network service level" to Premium
- Set the IP version to IPv4
- Set the type to global
- Click create

## Load Balancer
- Access the Load Balancing page  
https://console.cloud.google.com/networking/loadbalancing/?_ga=2.61277740.188826785.1608555797-56108235.1603201106&_gac=1.140830214.160613828.EAIaIQobIYEYAIYYYYAIQIYII  
- Click on "Create load balancer"
- Under HTTP (S) load balancing, click Start configuration
- Select From the Internet to my VMs and click Continue
- Give a name to the balancer
- Click Backend Configuration
    - Under Create or select back-end services and buckets, select Back-end services> Create a back-end service
    - Add a name for the backend service
    - In Protocol, select HTTP
    - In Named port, enter http
    - In Back-ends> New back-end> Instance group, select the instance group that was created earlier
    - In Port number, enter 8080
    - Keep the other default settings
    - Under Health check, select Create health check and add a name for it, such as nifi-health-check
    - Set the protocol to HTTP, in path put "/ nifi-api / system-diagnostics" click Save and continue
    - Keep the other default settings
    - Click Create
- In Host and path rules, keep the default settings
- In Front-end configuration, use the following values:
    - Set Protocol to HTTPS
    - Set the IP address as the static address created in the previous step
    - Verify that Port is set to 443 to allow HTTPS traffic
    - Click the Certificate drop-down list and select your primary SSL certificate
    - Click Done
- Click on Analyze and finish
- When you are finished configuring the load balancer, click Create
- Wait for the load balancer to be created
- Click on the load balancer name
- On the Load Balancer Details screen, note the IP: Load balancer port
- Access the domains page   
https://console.cloud.google.com/net-services/dns/zones
- Click on your zone and click "Add recordset"
- Select "A" as "Resource record type" and paste the load balancer IP address in the "IPv4 address" field
- Click save

## Done! We can now access NiFi through our domain and using an HTTPS connection.
