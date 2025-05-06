# Hands-On Workshop: GCP MIG Autoscaling with HTTP Load Balancer
*Last updated: May 6, 2025*

*Event: Cloud Roadshow x Build With AI 2025*

**Important Disclaimer**

* This lab is provided solely for **educational purposes** and utilizes simplified configurations suitable for a demonstration environment.
* Real-world implementations may vary significantly based on specific requirements, security policies, and production best practices.
* **Use caution:** Google Cloud Platform services incur costs based on usage. It is crucial to follow the "Clean up" instructions carefully to **avoid unexpected charges** to your account.
* The author provides this guide as-is and assumes **no responsibility** for any expenses, damages, misconfigurations, budget overruns, or other issues that may arise from following these instructions.


## 1. Introduction

In this hands-on lab, you will learn how to configure a resilient and scalable web application infrastructure on Google Cloud Platform (GCP). We will set up a Managed Instance Group (MIG) that automatically adjusts the number of virtual machine instances based on CPU load. We will then place an HTTP Load Balancer in front of the MIG to distribute incoming traffic efficiently and provide a single access point for users.

By the end of this lab, you will have a functioning autoscaling group of web servers accessible through a global load balancer.

## 2. Before you begin

* **Google Cloud Project:**
    * This codelab assumes that you already have a Google Cloud project with billing enabled.
    * If you do not have it yet: In the **Google Cloud Console**, on the **project selector page**, **select or create** a Google Cloud project. ([Go to project selector](https://console.cloud.google.com/projectselector2/home/dashboard))
    * Ensure that billing is enabled for your Google Cloud project. [Learn how to check if billing is enabled on a project](https://cloud.google.com/billing/docs/how-to/verify-billing-enabled).

* **Cloud Shell Environment:**
    * You'll use **Cloud Shell**, a command-line environment running in Google Cloud.
    * Activate **Cloud Shell** by clicking the **Activate Cloud Shell** button at the top of the Google Cloud console.
    * Once connected to Cloud Shell, check that you're already authenticated and that the project is set to your project ID using the following command:
        ```bash
        gcloud auth list
        ```
    * Run the following command in Cloud Shell to confirm that the `gcloud` command knows about your project:
        ```bash
        gcloud config list project
        ```
    * If your project is not set, use the following command to set it, replacing `<YOUR_PROJECT_ID>` with your actual Project ID:
        ```bash
        gcloud config set project <YOUR_PROJECT_ID>
        ```
    * *Tip:* You can find your `<YOUR_PROJECT_ID>` in the Google Cloud Console. Click the project name dropdown menu at the top of the console; the Project ID is usually listed in the selection dialog on the right side next to the project name.

* **Basic GCP Console Familiarity:** This lab assumes you are comfortable navigating the Google Cloud Console.

* **Permissions:** Ensure you have sufficient IAM permissions to create Compute Engine resources (Instance Templates, MIGs, Firewall Rules, Health Checks) and Load Balancers within your project.

## 3. Prepare Instance Template

First, we need a template that defines the configuration for the virtual machines (VMs) in our instance group. This ensures all instances are identical.

1.  Navigate to **Compute Engine** -> **Instance Templates** in the GCP Console.
2.  Create a new **Instance Template**.
3.  Configure the template:
    * **Name:** Give it a descriptive name (e.g., `webserver-template`).
    * **Machine type:** Choose an appropriate size (e.g., `e2-micro`).
    * **Boot disk:** Select a standard OS image (e.g., `Debian 12`).
    * **Identity and API access:**
        * **Service account:** Use the Compute Engine default service account or a specific one if required.
        * **Access scopes:** **Allow default access** is usually sufficient.
    * **Firewall:** Check **Allow HTTP traffic**. This automatically creates or uses a firewall rule allowing TCP traffic on port 80 tagged with `http-server`.
    * **Advanced options** -> **Management** -> **Automation** -> **Startup script:** Add a script to install a web server and serve a basic test page.
        ```bash
        #!/bin/bash
        # Update package list and install Apache
        apt-get update
        apt-get install -y apache2

        # Get the instance name and zone for display purposes
        INSTANCE_NAME=$(curl -H "Metadata-Flavor: Google" [http://metadata.google.internal/computeMetadata/v1/instance/name](http://metadata.google.internal/computeMetadata/v1/instance/name))
        INSTANCE_ZONE=$(curl -H "Metadata-Flavor: Google" [http://metadata.google.internal/computeMetadata/v1/instance/zone](http://metadata.google.internal/computeMetadata/v1/instance/zone) | sed 's:.*/::')

        # Create a simple index page displaying instance info
        cat <<EOF > /var/www/html/index.html
        <!doctype html>
        <html>
        <head><title>MIG Demo</title></head>
        <body>
        <h1>Hello from MIG!</h1>
        <p>Served by: <b>$INSTANCE_NAME</b></p>
        <p>Zone: <b>$INSTANCE_ZONE</b></p>
        </body>
        </html>
        EOF

        # Ensure Apache is running and enabled on boot
        systemctl start apache2
        systemctl enable apache2
        ```
4.  **Create** the instance template.

## 4. Create Managed Instance Group (MIG)

Now, use the instance template to create a group of identical instances that can be managed together and will autoscale.

1.  Navigate to **Compute Engine** -> **Instance Groups**.
2.  **Create Instance Group**.
3.  Configure the MIG:
    * **Name:** Give your MIG a descriptive name (e.g., `webserver-mig`).
    * **Instance template:** Select the template you created in the previous step.
    * **Location:** Choose **Multiple zones** (Regional MIG) for higher availability. Select your desired region.
    * **Autoscaling:**
        * **Autoscaling mode:** Select **On: Add and remove instances to the group**.
        * **Minimum number of instances:** Set to `1`.
        * **Maximum number of instances:** Set to `5` (or your desired limit).
        * **Autoscaling signals:**
            * **Signal type:** Choose **CPU utilization**.
            * **Target CPU utilization:** Set to `60` (%). This means the autoscaler will add instances if the average CPU exceeds 60% and remove them if it drops significantly below this target.
        * **Cooldown period:** Leave the default (e.g., `60` seconds) unless you have specific needs.
    * **Autohealing** -> **Health check:**
        * Click **Create a health check**.
        * **Name:** Give it a name (e.g., `http-basic-check`).
        * **Protocol:** `HTTP`.
        * **Port:** `80`.
        * **Request path:** `/` (the root path served by Apache).
        * Configure **Check interval**, **Timeout**, **Healthy threshold**, and **Unhealthy threshold** (defaults are usually suitable for a demo).
        * **Save and continue**.
4.  **Create** the MIG. Wait for the initial instance(s) to be created, run the startup script, and become healthy according to the health check. This might take a few minutes.

## 5. Create HTTP Load Balancer

Set up an external HTTP Load Balancer to distribute traffic evenly across the healthy instances in your MIG.

1.  Navigate to **Network Services** -> **Load Balancing**.
2.  Click **Create load balancer**.
3.  Under **HTTP(S) Load Balancing**, click **Start configuration**.
4.  **Internet facing or internal only:** Select **From Internet to my VMs or serverless services**.
5.  **Global or Regional:** Select **Global HTTP(S) Load Balancer (classic)** for this demo. (The newer Global External HTTP(S) Load Balancer offers more features but has a slightly different setup flow).
6.  **Continue**.
7.  **Name:** Give your load balancer a descriptive name (e.g., `web-lb`).
8.  **Backend configuration:**
    * Click **Backend configuration** -> **Create a backend service**.
    * **Name:** Give the backend service a name (e.g., `webserver-backend`).
    * **Backend type:** `Instance group`.
    * **Protocol:** `HTTP`.
    * **Named Port:** `http` (this corresponds to port 80 defined in the instance group).
    * **Backends** -> **New backend:**
        * **Instance group:** Select the MIG you created (`webserver-mig`).
        * **Port numbers:** `80`.
        * **Balancing mode:** Choose **Utilization** (appropriate for CPU-based scaling).
        * **Capacity settings:** Leave defaults for this demo.
    * Click **Done**.
    * **Health check:** Select the health check you created earlier (`http-basic-check`).
    * **Enable Cloud CDN** (optional): Leave unchecked for this basic demo.
    * Click **Create**.
9.  **Frontend configuration:**
    * **Name:** Give the frontend a name (e.g., `http-frontend`).
    * **Protocol:** `HTTP`.
    * **IP version:** `IPv4`.
    * **IP address:** Select **Ephemeral** for the demo. For production, you'd typically create and assign a Static IP. Note down the assigned Ephemeral IP address.
    * **Port:** `80`.
    * Click **Done**.
10. **Review and finalize:** Check the configuration details on the summary page.
11. **Create** the Load Balancer. It can take several minutes for the Load Balancer to be fully provisioned and the configuration to propagate globally.
12. **Test:** Once the GCP console indicates the Load Balancer is ready, open a web browser and navigate to `http://<LB-EXTERNAL-IP>` (using the IP address you noted). You should see the "Hello from MIG!" test page. Refreshing the page might show different instance names/zones if you manually scaled the MIG up or if traffic is distributed.

## 6. Simulate Load & Observe Autoscaling (Optional / Instructor-led)

Now, let's generate sufficient traffic to trigger the autoscaling policy based on CPU utilization.

1.  **Identify the Load Balancer's External IP:** Use the IP address noted during the frontend configuration.
2.  **Use a load testing tool:** You can run these commands from cloudshell

    * **Using `hey`:** (Install via `go install go.uber.org/hey` or package manager). This command sends traffic for 2 minutes (`-z 2m`) with 50 concurrent connections (`-c 50`). Adjust duration/concurrency as needed.
        ```bash
        hey -z 2m -c 50 http://<LB-EXTERNAL-IP>
        ```
    * **Using `curl` in a loop:** This generates continuous requests. Use `Ctrl+C` to stop. The `grep` filters for the server name, and `sleep` adds a small delay.
        ```bash
        while true; do curl -s http://<LB-EXTERNAL-IP> | grep Served; sleep 0.1; done
        ```
    * **Using ApacheBench (`ab`):** (Often pre-installed or available via `apache2-utils`). Example: 1000 total requests (`-n`), 50 concurrent (`-c`).
        ```bash
        ab -n 1000 -c 50 http://<LB-EXTERNAL-IP>/
        ```
4.  **Monitor Scaling Activity:**
    * **Instance Groups Page:** Go back to **Compute Engine** -> **Instance Groups**. Select your MIG (`webserver-mig`). Observe the **Instances** tab or the monitoring graphs. As the average CPU utilization across the group exceeds the 60% target, you should see the number of instances increase towards the maximum (5).
    * **Monitoring Page:** Navigate to **Monitoring** -> **Metrics Explorer**.
        * Query the metric `compute.googleapis.com/instance/cpu/utilization`. Filter by `resource.instance_group_manager_name` matching your MIG name. Observe the average CPU usage.
        * Query the metric `loadbalancing.googleapis.com/https/request_count`. Filter by `resource.forwarding_rule_name` matching your load balancer's frontend name. Observe the incoming request rate.
5.  **Observe Scale-Down:** Stop the load generation tool. After the load decreases and the average CPU utilization drops significantly below the target for a sustained period (longer than the cooldown period), the autoscaler will begin terminating instances until it reaches the minimum count (1). This scale-down process is typically slower than scaling up to prevent thrashing.

## 7. What you learned

* How to create a reusable Instance Template with a startup script to configure VMs.
* How to set up a Regional Managed Instance Group (MIG) for high availability.
* How to configure Autoscaling based on CPU utilization.
* How to implement Autohealing using HTTP Health Checks.
* How to create an External HTTP Load Balancer to distribute traffic to a MIG.
* How to test load balancing and observe autoscaling behavior.

## 8. Clean up

To avoid incurring charges to your Google Cloud account for the resources used in this lab, delete the resources you created:

1.  **Delete the HTTP Load Balancer:**
    * Navigate to **Network Services** -> **Load Balancing**.
    * Select the load balancer (`web-lb`).
    * Click **Delete**. Confirm the deletion (this also deletes the associated frontend and backend service).
  
2.  **Delete the Managed Instance Group (MIG):**
    * Navigate to **Compute Engine** -> **Instance Groups**.
    * Select the MIG (`webserver-mig`).
    * Click **Delete**. Confirm the deletion.
  
3.  **Delete the Instance Template:**
    * Navigate to **Compute Engine** -> **Instance Templates**.
    * Select the template (`webserver-template`).
    * Click **Delete**. Confirm the deletion.
      
4.  **Delete the Health Check:**
    * Navigate to **Compute Engine** -> **Health Checks**.
    * Select the health check (`http-basic-check`).
    * Click **Delete**. Confirm the deletion.
      
5.  **Delete the Firewall Rule (Optional):**
    * If the `http-server` firewall rule was created specifically for this lab and is no longer needed, navigate to **VPC network** -> **Firewall**.
    * Find the rule allowing HTTP traffic (often tagged `http-server` or named like `default-allow-http`).
    * Select it and click **Delete**. Be careful not to delete firewall rules needed by other applications.
      
6.  Alternatively, if you created a dedicated project for this lab, you can delete the entire project from the **Manage resources** page.


**Congratulations on completing the demo!** 🎉

Any questions? Go ask **Gemini Cloud Assist**!
