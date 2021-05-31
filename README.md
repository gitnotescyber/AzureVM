
Setting Up Azure VMs


- Deploying containers using Ansible and Docker.
- Deploying Filebeat using Ansible.
- Deploying the ELK stack on a server.
- Diagramming networks and creating a README.




Resources
- [Ansible Documentation](https://docs.ansible.com/ansible/latest/modules/modules_by_category.html)
- [`elk-docker` Documentation](https://elk-docker.readthedocs.io/#Elasticsearch-logstash-kibana-elk-docker-image-documentation)
- [Virtual Memory Documentation](https://www.elastic.co/guide/en/elasticsearch/reference/current/vm-max-map-count.html)
- ELK Server URL: http://your-IP:5601/app/kibana#/home?_g=()
- [Docker Commands Cheatsheet](https://phoenixnap.com/kb/list-of-docker-commands-cheat-sheet)



Configuring an ELK Server

Involves the following: 

  - Create a new vNet in Azure in a different region, within the same resource group.
  - Create a peer-to-peer network connection between your vNets.
  - Create a new VM in the new vNet that has 2vCPUs and a minimum of 4GiB of memory.
  - Add the new VM to Ansible’s `hosts` file in your provisioner VM.
  - Create an Ansible playbook that installs Docker and configures an ELK container.
  - Run the playbook to launch the container.
  - Restrict access to the ELK VM.

---

Before deploying an ELK Stack, let's cover what the stack can do and how it work. You should be familiar with ELK from previous units:

- ELK is an acronym. Each letter stands for the name of a different open-source technology:

  - **Elasticsearch**: Search and analytics engine.

  - **Logstash**: Server‑side data processing pipeline that sends data to Elasticsearch.

  - **Kibana**: Tool for visualizing Elasticsearch data with charts and graphs.

- ELK started with Elasticsearch. Elasticsearch is a powerful tool for security teams because it was initially designed to handle any kind of information. This means that logs and arbitrary file formats, such as PCAPs, can be easily stored and saved.

- After Elasticsearch became popular for logging, Logstash was added to make it easier to save logs from different machines into the Elasticsearch database. It also processes logs before saving them, to ensure that data from multiple sources has the same format before it is added to the database.

- Since Elasticsearch can store so much data, analysts often use visualizations to better understand the data at a glance. Kibana is designed easily visualize massive amounts of data in Elasticsearch. It is also well known for its complex dashboards.

In summary:

- Elasticsearch is a special database for storing log data.

- Logstash is a tool that makes it easy to collect logs from any machine.

- Kibana allows analysts to easily visualize your data in complex ways.

Together, these three tools provide security specialists with everything they need to monitor traffic in any network.

The Beats Family

The ELK stack works by storing log data in Elasticsearch with the help of Logstash.

Traditionally, administrators would configure servers to collect logs using a built-in tool, like `auditd` or `syslog`. They would then configure Logstash to send these logs to Elasticsearch.

- While functional, this approach is not ideal because it requires administrators to collect all of the data reported by tools like `syslog`, even if they only need a small portion of it.

- For example, administrators often need to monitor changes to specific files, such as `/etc/passwd`, or track specific information, such as a machine's uptime. In cases like this, it is wasteful to collect all of the machine's log data in order to only inspect a fraction of it.

ELK addressed this issue by adding an additional tool to its data collection suite called **Beats**.

- Beats are special-purpose data collection modules. Rather than collecting all of a machine's log data, Beats allow you to collect only the very specific pieces you are interested in.

ELK officially supports eight Beats. You will use two of them in this project:

- **Filebeat** collects data about the file system.

- **Metricbeat** collects machine metrics, such as uptime.

  - A **metric** is simply a measurement about an aspect of a system that tells analysts how "healthy" it is. 
  
  - Common metrics include:
    
    - **CPU usage**: The heavier the load on a machine's CPU, the more likely it is to fail. Analysts often receive alerts when CPU usage gets too high.

    - **Uptime**: Uptime is a measure of how long a machine has been on. Servers are generally expected to be available for a certain percentage of the time, so analysts typically track uptime to ensure your deployments meet service-level agreements (SLAs).

- In other words, Metricbeat makes it easy to collect specific information about the machines in the network. Filebeat enables analysts to monitor files for suspicious changes.

You can find documentation about the other Beats at the official Elastic.co site: [Getting Started with Beats](https://www.elastic.co/guide/en/beats/libbeat/current/getting-started.html).


The goal of this project is to add an instance of the ELK stack to a new virtual network in another region in Azure and configure your 3 Web VMs to send logs to it.

Make sure you are logged into your personal Azure accounts and not the cyberxsecurity accountr. You will be using the VMs you created during the week on cloud security.

Since you will be building off of that week, let's take a moment to review the network architecture built in that unit.

This network contains:


- A gateway. This is the jump box configured during the cloud security week.

- Three additional virtual machines, one of which is used to configure the others, and two of which function as load-balanced web servers.

Due to Azure Free account limitations, you can only utilize 4vCPUs per region in Azure. Therefore, we will need to create a new vNet in another region in Azure for our ELK server.

- By the end of the project, we will have an ELK server deployed and receiving logs from web machines in the first vNet.

:warning: **Important:** Azure may run out of available VMs for you to create a particular region. If this happens, you will need to do one of two things:

1. You can open a support ticket with Azure support using [these instructions](https://docs.microsoft.com/en-us/azure/azure-portal/supportability/how-to-create-azure-support-request). Azure support is generally very quick to resolve issues.

2. You can create another vNet in another region and attempt to create the ELK sever in that region.

    - In order to set this up, you will perform the following steps:

      1. Create a new vNet in a new region (but same resource group).
      2. Create a peer-to-peer network connection between your two vNets.
      3. Create a new VM within the new network that has a minimum of 4GiB of memory, 8GiB is preferred.
      4. Download and configure an ELK stack Docker container on the new VM.
      5. Install Metricbeat and Filebeat on the web-DVWA-VMs in your first vNet.

You will use Ansible to automate each configuration step.

At the end of the project, you will be able to use Kibana to view dashboards visualizing the activity of your first vNet.

![Cloud Network with ELK](Images/finished-elk-diagram.png)

Note: You will install an ELK container on the new VM, rather than setting up each individual application separately.

:warning: **Important:** The VM for the ELK server **must** have at least 4GiB of memory for the ELK container to run properly. Azure has VM options that have `3.5 GiB` of memory, but **do not use them**. They will not properly run the ELK container because they do not have enough memory.

- If a VM that has 4GiB of memory is not available, the ELK VM will need to be deployed in a different region that has a VM with 4GiB available.

- Before containers, we would not have been able to do this. We would have had to separately configure an Elasticsearch database, a Logstash server, and a Kibana server, wire them together, and then integrate them into the existing network. This would require at least three VMs, and definitely many more in a production deployment.

- Instead, now we can leverage Docker to install and configure everything all at once.

Remember that you took a similar approach when creating an Ansible control node within the network. You installed an Ansible container rather than installing Ansible directly. This project uses the same simplifying principle, but to even greater effect.

Troubleshooting Theory


Before we dive into the work today, let's briefly enforce the importance of independent troubleshooting as a means to not only solve the problem at hand, but also learn more about the technology that you are working with.  

Specifically, we will review the [Split-Half Search](https://www.peachpit.com/articles/article.aspx?p=420908&seqNum=3), an effective troubleshooting methodology that can be applied to _any_ technical issue to find a solution quickly.

The general procedure states that you should remove half of the variables that could be causing a problem and then re-test. 

  - If the issue is resolved, you know that your problem resides in the variables that you removed. 
  
  - If the problem is still present you know your problem resides in the variables that you did not remove. 
  
  - Next, take the set of variables where you know the problem resides. Remove half of them again and retest. Repeat this process until you find the problem.

In the context of this project, removing half of your variables could mean:

- Logging into the ELK server and running the commands from your Ansible script manually.

	- This removes your Ansible script from the equation and you can determine if the problem is with your Ansible Script, or the problem is on the ELK Server.

	- You can manually launch the ELK container with: `sudo docker start elk` or (if the container doesn't exist yet); `sudo docker run -p 5601:5601 -p 9200:9200 -p 5044:5044 -it --name elk sebp/elk:761`

- Downloading and running a different container on the ELK server.

	- This removes the ELK container from the equation and you can determine if the issue may be with Docker or it may be with the ELK container.

- Removing half of the commands of your Ansible script (or just comment them out).

	- This removes half of the commands you are trying to run and you can see which part of the script is failing.

Another effective strategy is to change only one thing before your retest. This is especially helpful when troubleshooting code. If you change several things before you re-test, you will not know if any one of those things has helped the situation or made it worse.


References

- For more information about ELK, visit [Elastic: The Elastic Stack](https://www.elastic.co/elastic-stack).

- For more information about Filebeat, visit [Elastic: Filebeat](https://www.elastic.co/beats/filebeat).

- To set up the ELK stack, we will be using a Docker container. Documentation can be found at [elk-docker.io](https://elk-docker.readthedocs.io/).

- For more information about peer networking in Azure, visit [Global vNet Peering](https://azure.microsoft.com/en-ca/blog/global-vnet-peering-now-generally-available/)

- If Microsoft Support is needed, visist [How to open a support ticket](https://docs.microsoft.com/en-us/azure/azure-portal/supportability/how-to-create-azure-support-request)

- [Split-Half Search](https://www.peachpit.com/articles/article.aspx?p=420908&seqNum=3)

ELK Installation


---


- Deployed a new VM on your virtual network.
- Created an Ansible play to install and configure an ELK instance.
- Restricted access to the new server.

Completing these steps required you to leverage your systems administration, virtualization, cloud, and automation skills. This is an impressive set of tools to have in your toolkit!

</details>

---
Filebeat

Lectures cover the following

- Provide a brief overview of Filebeat.

Activities involve the following: 

- Navigate to the ELK server’s GUI to view Filebeat installation instructions.
- Create a Filebeat configuration file.
- Create an Ansible playbook that copies this configuration file to the DVWA VMs and then installs Filebeat.
- Run the playbook to install Filebeat.
- Confirm that the ELK Stack is receiving logs.
- Install Metricbeat as a bonus activity.

<details>
<summary> <b> Click here to view the 13.2 Student Guide. </b> </summary>

---

### 01. Instructor Do: Filebeat Overview

In the previous class, we installed the ELK server. Now it's time to install data collection tools called **Beats**. 

Before  working on the activity, let's review the following about Filebeat: 

- Filebeat helps generate and organize log files to send to Logstash and Elasticsearch. Specifically, it logs information about the file system, including which files have changed and when. 

- Filebeat is often used to collect log files from very specific files, such as those generated by Apache, Microsoft Azure tools, the Nginx web server, and MySQL databases.

- Since Filebeat is built to collect data about specific files on remote machines, it must be installed on the VMs you want to monitor.

If you have not completed all Day 1 activities, you should finish them before continuing onto the Filebeat installation. 

Filebeat Installation



 **Note:** The Resources folder includes an `ansible.cfg` file. You will not need to do anything with this file. It's included in case a you accidentally edit or delete your configuration file.


If your ELK server is receiving logs, congratulations! You've successfully deployed a live, functional ELK stack and now have plays that can:

- Install and launch Docker containers on a host machine.
- Configure and deploy an ELK server.
- Install Filebeat on any Debian-flavored Linux server.

Even more significant is that you've done all of this through automation with Ansible. Now you can recreate exactly the same setup in minutes.

</details>

---

Exploration, Diagramming and Documentation

Lectures will cover:

  -  Kibana's features and demonstration of how to navigate datasets.
  -  Supplemental assignment in which you answer interview-style questions about your project. 

Activities involve the following: 

  - Finalize the network diagram you began during the cloud security week.
  - Draft a README explaining what you've built.
  - Craft interview response questions.




---

Overview

Today's lesson will proceed as follows:

- An instructor demo on navigating logs using Kibana, followed by a Kibana activity and review.
- A brief overview of the supplemental interview questions that you can answer. 

- You can use the rest of the day to complete your project.

  - If you need more time installing Filebeat on your DVWA machines, you can continue this work.

  - If you finished the Filebeat installation, you can work on a network diagram and project README.

  - You will also have the opportunity to answer questions about the project as it relates to different cybersecurity domains.


Exploring Kibana

Now that we have a working instance of Kibana, we will learn a bit about how to use it.

- Companies use tools like Kibana to research events that have happened on your network.

- Any attack leaves a trace that can be followed and investigated using logs. As well,
registrars sometimes don't take down clever malicious domains, leaving businesses to index and defend against them themselves.

Kibana is an interface to such data, and allows cyber professionals to gain insight from a lots of data that otherwise would be un manageable.

The following walkthrough provides a quick overview of using Kibana.

Kibana Walkthrough

1. - Start by importing Kibana's Sample Web Logs data.

      - You can import it by clicking **Try our sample data**.

     ![](Images/kibana/Welcome.png)

    - You can also import it from the homepage by clicking on **Load a data set and a Kibana dashboard** under **Add sample data**.

     ![](Images/kibana/add-data.png)

    - Click **Add Data** under the **Sample Web Logs** data pane.

      ![](Images/kibana/sampledata.png)


2. A brief overview of the interface, starting with the time dropdown in the top right of the screen:

   - Kibana categorizes everything based on timestamps.

    - Click on the drop down and show that there are several predefined options that can be chosen: Today, Last 7 days, Last 24 hours, etc.
        
   ![](Images/kibana/Change-time.png)
      
   - Overview of data panes:

        - **Unique Visitors**: Unique visitors to the website for the time frame specified.
      - **Source Country**: Web traffic by country.
      - **Visitors by OS**: The kind of OS visitors are using. 
      - **Response Codes Over Time**: HTTP response codes 200, 404 and 503.
      - **Unique Visitors vs. Average Bytes**: The number of visitors and the amount of data they use.
      - **File Type Scatter Plot**: A graph showing the types of files that were accessed.
      - **Host, Visits and Bytes Table**: A table showing the kinds of files that were accessed.
      - **Heatmap**: Hours of the day that are most active by country.
      - **Source and Destination Sankey Chart**: Connections that have been made by country. The thicker the line, the more data was transferred between machines.
      - **Unique Visitors by Country**: Countries the traffic is originating from.

  - These panes are interactive and can help filter data.

  -  Click on the United States inside Unique Visitors by Country to see how the panes change to reflect only the data that originated from the United States.

  ![](Images/kibana/us.png)

3. We can further dive into this data using Kibana's Discover page.

    - Locate the hamburger dropdown menu at the top-left of the page and choose the **Discover** option under the **Kibana** heading.

      ![](Images/kibana/Discover.png)

    - We can now look at interactions between clients and the server with more detail.

    - Each item listed is not a single packet, but represents the entire interaction between a specific client and the server (i.e. it represents _many_ network packets that were exchanged).

    - Click the expansion arrow next to the first interaction and show the resulting table of information.

      ![](Images/kibana/discover2.png)

    - We can see things like source and destination IPs, the amount of bytes exchanged, the geo coordinates of the source traffic, and much more.

      ![](Images/kibana/discover3.png)

   - This data is still filtered by traffic originating from the United States.

Click the hamburger dropdown menu again and return to the **Dashboard** option listed under **Kibana**.

    ![](Images/kibana/Discover.png)

   - Remove the `geo.src: US` filter that is applied to the data by clicking on the small **x** near the filter tab.

     ![](Images/kibana/filter.png)

   - In the next activity, we will have the opportunity to explore these logs further and gain more insight from the traffic.














