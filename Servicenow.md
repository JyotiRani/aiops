# ServiceNow Integration


## Contents

- [Introduction](#introduction)
- [Pre requisites](#pre-requisites)
    - [Procure a ServiceNow Developer Instance](#procure-a-servicenow-developer-instance)
    - [Install the WAIOPS App in your ServiceNow Developer Instance](#install-the-waiops-app-in-your-servicenow-developer-instance)
- [Configuring ServiceNow](#configuring-servicenow)
    - [Create Update Users in ServiceNow](#create-update-users-in-servicenow)
    - [ServiceNow Service Management configuration](#servicenow-service-management-configuration)
    - [Similar incident search configuration](#similar-incident-search-configuration)
- [Configuring Cloud Pak for Watson AIOps](#configuring-cloud-pak-for-watson-aiops)
    - [Creating ServiceNow integrations](#creating-servicenow-integrations)
- [Testing the ServiceNow Integration](#testing-the-servicenow-integration)
    - [Creating stories as incidents in ServiceNow](#creating-stories-as-incidents-in-servicenow)
    - [Historical similar incidents](#historical-similar-incidents)
    - [Pulling inventory and topology data from ServiceNow](#pulling-inventory-and-topology-data-from-servicenow)
    - [Change Risk Assessment](#change-risk-assessment)


---
## Introduction

The purpose of this guide is to provide a self-contained set of instructions to integrate with ServiceNow, summarizing instructions from [IBM Documentation](TBD) and other sources.

ServiceNow integrations provide historical and live data for change requests, incidents, and problems. ServiceNow integrations also provide inventory data for the ServiceNow observer.

**Note:** You can only have one ServiceNow integration per instance.


The instructions have been successfully tested against a ROKS cluster on the IBM Cloud.

---
## Pre requisites



### Procure a ServiceNow Developer Instance

Join the [ServiceNow Developer Program](https://developer.servicenow.com/dev.do), login with your credentials and request a Personal Developer Instance (PDI) with **version "Paris" and patch 4 or higher**. If you get a different version, you can release/return the instance you just got from the top right menu and request a new one. Then it will ask you to choose the version as shown below:

  ![image-version-select](./images/servicenow/image-version-select.png)

You can read about PDI [here](https://developer.servicenow.com/dev.do#!/guides/quebec/developer-program/pdi-guide/personal-developer-instance-guide-introduction). 

Note that your instance will go to sleep after a few hours if not used and if you are inactive for ten or more days the developer instance will be deleted!. You will have two sets of credentials: one for your ServiceNow developer account and another set for the developer instance itself. The ServiceNow developer instance comes loaded with some test data such as open incidents, change requests, etc. 


### Install the WAIOPS App in your ServiceNow Developer Instance

Customers will typically install the WAIOPS app (or plug-in) from the official [ServiceNow App Store](https://store.servicenow.com/sn_appstore_store.do#!/store/application/632a6d81db102010253148703996197e/1.1.0). In our case, because we have a developer instance, we will have to install it from a GitHub repo. 


#### How to import WAIOPS app into ServiceNow developer instance

Pre-req: Personal access token for Github. If you don't have one, follow the instructions https://docs.github.com/en/free-pro-team@latest/github/authenticating-to-github/creating-a-personal-access-token to create one (no elevated permissions needed for the token).

1. Obtain a developer instance for ServiceNow following the instructions from https://developer.servicenow.com/dev.do#!/guides/paris/now-platform/pdi-guide/obtaining-a-pdi (this requires a ServiceNow account which can be created as part of the steps - just sign up for a new account). Note: development instances go into hibernation when not used and are decommissined if not used for 10 days.
1. Log on to the ServiceNow developer instance.
1. Search for `Credentials`.
![Credentials](./images/servicenow/credentials1.png)
1. Select `Credentials` under `Connections & Credentials`. Click `New` button to create a new credential.
![Credentials](./images/servicenow/credentials2.png)
1. Select `Basic Auth Credentials`.
![Credentials](./images/servicenow/credentials3.png)
1. Enter any name, provide your login email address for Github and as password provide your personal access token. Then click `Submit`.
![Credentials](./images/servicenow/credentials4.png)
1. In the Search field, type in `Studio`. Select `Studio` under `System Applications`. This will open the Application Studio in a new tab.
![Studio](./images/servicenow/studio1.png)
1. In the `Select Application` dialog, click `Import from Source Control`.
![Studio](./images/servicenow/studio2.png)
1. The instructions from the integration repo state to use `https://github.ibm.com/watson-ai4it/servicenow-integration` as the repository value in the dialog. You can attempt this, but if you get an error message saying that you need read-write permission on the repo, then you will need to fork this repo and reference your forked copy instead (e.g.. `https://github.ibm.com/jorgego/servicenow-integration-fork`). Use `main` as the branch and in the `Credential` drop-down, select the entry with the credentials that were created in the previous steps. 
    * Again, make sure you use `main` as the branch.
![Studio](./images/servicenow/studio3.png)
1. Click `Import` to import the source code.

---
## Configuring ServiceNow

You can create stories in IBM Cloud Pak for Watson AIOps and issues in ServiceNow simultaneously. Both share information and can be tracked together. To ensure that this integration operates effectively, you must configure ServiceNow and ServiceNow integration in IBM Cloud Pak for Watson AIOps. After you configure ServiceNow and IBM Cloud Pak for Watson AIOps, the updates that you make in IBM Cloud Pak for Watson AIOps will flow into the Agent Workspace of ServiceNow. Updates to event data and the state of your story in IBM Cloud Pak for Watson AIOps appear in ServiceNow. Any updates that you make in Slack or Microsoft Teams to the description, short description, or severity after you create the story do not overwrite the existing values in ServiceNow. Also, edits that are made in ServiceNow (changing descriptions or incident names, for example) are not reflected in Slack, Microsoft Teams, or IBM Cloud Pak for Watson AIOps.

### Create Update Users in ServiceNow

Before you create your integration, you must assign IBM Cloud Pak for Watson AIOps roles to your users in ServiceNow.

1. Log in to your ServiceNow developer instance.
2. In the filter field, search for users as shown below:

    ![users](./images/servicenow/users.png)

3. Click User Administration > Users to see the list of users as shown below:

    ![users](./images/servicenow/user-list.png)

4. Assign at least two created or existing users to the following roles. Assign at least one user to each role.

      **Notes:** In both cases ensure that your IBM Cloud Pak for Watson AIOps users in ServiceNow are assigned the itil role in ServiceNow. The itil role enables them to have access to the Agent Workspace. If the user does not have access to the Agent Workspace, they cannot work on incidents in ServiceNow and receive a permissions error.

      **Important:** Ensure that the time zone of your Events user (the user value that is used to create your ServiceNow integration) matches your system time zone in ServiceNow. If the two values are not synchronized, the flow of your change request data from ServiceNow to IBM Cloud Pak for Watson AIOps is disrupted.

    x_ibm_waiops.watson_aiops_admin: An administrative user who can configure instances, URLs, usernames, and passwords in ServiceNow. The admin user is the only user who can see the menu options to configure IBM Cloud Pak for Watson AIOps in ServiceNow. 
    
    In the following two slides, select the existing user abraham.lincoln, set the password and assign the following roles (click Roles Edit):

    * x_ibm_waiops.watson_aiops_admin 
    * itil
    
     Make sure you Save the roles and click Update on the user page.

    ![events user](./images/servicenow/user-lincoln.png)

    ![events user](./images/servicenow/user-lincoln-roles.png)


    x_ibm_waiops.watson_aiops_events_user: A non-administrative user who is required to insert IBM Cloud Pak for Watson AIOps data into ServiceNow and is configured as the connector between IBM Cloud Pak for Watson AIOps and ServiceNow. **Important:** Ensure that you select Internal Integration User when you create or edit your Events user (the user that is used in your ServiceNow integration). Your integration user exists to exchange data between ServiceNow and IBM Cloud Pak for Watson AIOps and does not require access to the ServiceNow interface.

    In the following two slides, select the existing user abel.tuter, set the password, check Internal Integration User and assign the following roles (click Roles Edit) 

    * x_ibm_waiops.watson_aiops_events_user
    * itil
    * cmdb_read (for ServiceNow topology observer)
    * rest_api_explorer (for ServiceNow topology observer)
    * service_viewer (for ServiceNow topology observer)
    * web_service_admin (for ServiceNow topology observer)

    Make sure you Save the roles and click Update on the user page.

    ![events user](./images/servicenow/user-abel.png)

    ![events user](./images/servicenow/user-abel-roles.png)


Your two ServiceNow roles correlate to different functions within IBM Cloud Pak for Watson AIOps. The user assigned to the Events user role in ServiceNow is the user ID that you create the integration with in IBM Cloud Pak for Watson AIOps. That role is responsible for inserting data into ServiceNow. The Admin user maintains visibility into IBM Cloud Pak for Watson AIOps settings in ServiceNow (like configuring the information that is needed to integrate similar incidents).


### ServiceNow Service Management configuration

1. Log in to your ServiceNow developer instance with the user that has the `x_ibm_waiops.watson_aiops_admin` role (might need to log out after assigning the role).
2. In the filter field, search for Watson AIOps as shown in the following picture:

    ![search](./images/servicenow/search4waiops.png)

3. Click Configuration.
4. Enter values for the following fields as shown in the next picture.

    * URL for AIOps connection: The URL of your AI Manager instance in the format of protocol://hostname:port.

      **Note:** If you use a tunnel to expose the server for the ServiceNow connection, you must replace the URL with the forwarding address of the tunnel.

    * Name of the Watson AIOps instance: The name of your instance in IBM Cloud Pak for Watson AIOps. You can identify your instance name by logging in to the IBM Cloud Pak for Watson AIOps console and getting the value in the My instances pane.

    * User name for AIOps connection: The username for the IBM Cloud Pak for Watson AIOps instance.

    * Password for AIOps connection: The password for the IBM Cloud Pak for Watson AIOps instance.

    ![configuration](./images/servicenow/configuration.png)

5. Click on the Save button in the top right. 


### Similar incident search configuration

While not necessary, configuring similar incident search in ServiceNow facilitates problem resolution from within the ServiceNow interface. Configuring similar incident search enables Watson AIOps Similar Incidents in Agent Assist. You can use Agent Assist to look for other tickets that share details with the ticket that you're reviewing. Similar incident search uses the Similar incident model that you trained in IBM Cloud Pak for Watson AIOps.

**Important:** To complete this task, you must be assigned the admin role in ServiceNow, otherwise you cannot configure Search contexts.

1. Log in to your ServiceNow developer instance.
2. In the filter field, search for Search contexts as shown below:

    ![configuration](./images/servicenow/search-context.png)

3. Click Incident Deflection to open the Incident Deflection record.
4. Click the Additional Resource Configurations tab, then click `Edit` as shown below. 

    **Note:** You may see a this message: "This record is in the Global application, but IBM Watson AIOps is the current application. To edit this record click here". Click on the top-right `gear icon` settings to go to the System Settings window, select the Developer tab and select `Global` application to switch to the Global application. This will enable the `Edit` button option. 

    ![dev-app-to-global](./images/servicenow/dev-app-to-global.png)


    ![configuration](./images/servicenow/additional-resources.png)

5. Select Watson AIOps Similar Incidents from the Collection column and add it to the Additional Resource Configurations List, then click Save.

    ![config-list](./images/servicenow/config-list.png)

6. (Optional) You can adjust the order of how the search elements appear by adjusting the Order attribute.
7. Save your Incident Deflection record.

Watson AIOps Similar Incidents now appears as an option to search by in the Agent Assist.

---
## Configuring Cloud Pak for Watson AIOps

Now that we have created users and configured ServiceNow, we need to configure Cloud Pak for Watson AIOps to be able to integrate with ServiceNow.

### Creating ServiceNow integrations 

To create a ServiceNow integration, complete the following steps:

1. Log in to IBM Cloud Pak for Watson AIOps.
2. From the console, click the navigation menu (four horizontal bars), then click Define > Data and tool integrations as shown below:

    ![config-list](./images/servicenow/define-data-integration.png)

3. On the ServiceNow tile, click Add integration.

    **Note:** If you do not immediately see the integration that you want to create, you can filter the tiles by type of integration. Click the type of integration that you want in the Category section.

4. Enter the following integration information as shown below:
  ![servicenow-form](./images/servicenow/servicenow-form.png)

    * Name: The display name of your integration.
    * Description: An optional description for the integration.
    * URL: The URL of your ServiceNow developer instance.
    * User ID: the ServiceNow Events user username.

      **Important:** You must create a new, unique user in ServiceNow specifically for your ServiceNow integration before you create your integration. Make sure that you are using the correct user role, the Events user, for your integration. You must also make sure that the time zone set for this user matches your system time zone in ServiceNow. For more information about ServiceNow users, see Create users.

    * Password: the ServiceNow Events user password.

    * Encrypted password: the ServiceNow Events user encrypted password.

      **Important:** You must specify an IBM Cloud Pak for Watson AIOps encrypted version of your ServiceNow password to collect inventory and topology data from ServiceNow. To encrypt your ServiceNow password, complete the following steps:

      * Make sure that you have oc installed on your local system. For more information about these requirements, see Preparing to install IBM Cloud Pak for Watson AIOps. Log in to your Red Hat OpenShift Container Platform cluster by using the oc login command. You can identify your specific oc login by clicking the user name dropdown menu in the Red Hat OpenShift Container Platform console, then clicking Copy Login Command.

      * Switch your project to where the Cloud Pak for Watson AIOps is installed (e.g. cp4waiops) using the following command (use your own project name):
      ```
      oc project cp4waiops
      ```
      * Enter the following command to encrypt your password. The content of <your_password> is your unencrypted ServiceNow password.
      ```
      oc exec -ti $(oc get pods -l app.kubernetes.io/instance=evtmanager-topology -l app.kubernetes.io/name=topology -o jsonpath='{ .items[*].metadata.name }') -- java -jar /opt/ibm/topology-service/topology-service.jar encrypt_password -p '<your_password>'
      ```
      * Copy and paste the results of the preceding command (your encrypted ServiceNow password) in this field.

    * Connection timeout: The amount of time before a connection times out in milliseconds. The default value is 5000 milliseconds.

    * Read timeout: The amount of time before a read operation times out in milliseconds. The default value is 10000 milliseconds.
    * Proxy host: The proxy host for your ServiceNow instance.
    * Proxy port: The proxy port for your ServiceNow instance. The default value is 8080.

    You can test your integration connection by clicking Test connection.

5. You can improve search performance by mapping the fields from your implementation fields to IBM Cloud Pak for Watson AIOps's standard fields. Enter the values that you want to map, then click Format to ensure that you entered a valid JSON configuration.

6. Select how you want to manage collecting data for use in AI training and change risk scoring. Click the Data flow toggle to turn on data collection then select how you want to collect data:

    * **Live data for continuous AI training:** A continuous flow of data from your integration is used to both train AI models and analyze your data for anomalous behavior.

      **Note:** Before selecting this option, you must first set **Mode** to **Historical data for initial AI training** to collect a minimum amount of data, and then run AI training on that data. For more information, see Planning data loading and training.

    * **Live data for initial AI training**: A single set of training data used to define your AI model. Data collection takes place over a specified time period that starts when you create your integration.

      **Note:** If you select this option, you must disable your integration when you have collected enough data for training. If you do not disable the integration, it continues to collect data. For more information about AI model training, including minimum and ideal data quantities, see Configuring AI training. For more information about disabling and enabling integrations, see Enabling and disabling ServiceNow integrations.

    * **Historical data for initial AI training:** A single set of training data used to define your AI model. You must also specify the parallelism of your source data. Historical data is harvested from existing logs in your integration over a specified time period in the past.

    **Important:** Different types of AI models have different requirements to properly train a model. Make sure that your settings satisfy minimum data requirements. For more information about how much data you need to train different AI models, see Configuring AI training.

7. Select whether you want to collect inventory and topology data from your ServiceNow instance. Click the Collect inventory and topology data toggle to turn on inventory and data collection.

8. To schedule ServiceNow observer jobs, enter the following information:

    * Start date: The start date of when the job is to run.
    * Time: The time to run the observer job.
    * Time interval (period): How frequently to run the job (either by hour, or by minute).
    * Interval: The duration of time between runs based on the Time interval (period). For example, if you wanted the job to run every 2 hours, in Time interval (period), select hours, and in Interval, enter 2. Enter 0 to run the observer job one time (and manually through the interface otherwise).

9. Click Integrate.

---
## Testing the ServiceNow Integration

### Creating stories as incidents in ServiceNow

After the integration is finished, for every chatbot story created, an incident will be created in ServiceNow. For example, the creation of the following story number 9 in Slack:

  ![slack-story](./images/servicenow/slack-story.png)

This story has a corresponding incident created in ServiceNow. To access these incidents, from the ServiceNow home page search for agent, and click on Agent Workspace Home, as shown below

  ![agent-workspace](./images/servicenow/agent-workspace.png)

In the Agent Workspace, if you click on the burger menu and select Watson AIOps Incidents, you will see the list of incidents created by Watson AIOps as shown below

  ![waiops-incidents-in-sn](./images/servicenow/waiops-incidents-in-sn.png)

This is the incident in ServiceNow that corresponds to the story number 9 that was mentioned before

  ![story9-in-sn](./images/servicenow/story9-in-sn.png)

### Historical similar incidents

When an incident occurs, it can be helpful to review details for similar incidents to help determine a resolution. This AI model aggregates information about similar messages, anomalies, and events for a component or application. It can also extract the steps used to fix previous incidents, if documented. Training this AI model will help you discover historical incidents to aid in the remediation of current problems.

**Important:** Incidents in ServiceNow must contain meaningful entries in the Resolution notes (close_notes value in the logs) field to successfully populate the Search recommended actions feature in your ChatOps interface. If your incidents do not include resolutions, or include default entires like Closed by Caller, these values do not identify paths to remediation. As a result, empty or non-meaningful entries yield no results in your ChatOps interface because they provide no potential actions.

The process can be defined in the following steps:

1) Change at least five existing incidents in your ServiceNow developer instance in order to match the stories you already have in Slack. To see the existing sample incidents in your ServiceNow developer instance search for incidents and select Service Desk -> Incidents as shown below

  ![search-incidents](./images/servicenow/search-incidents.png)

Select an existing open incident and update the description and short description with some known log anomaly error from an existing story. For example the next picture shows the description as "Unknown Error web". Then click on the Resolution Information tab and Close the incident with resolution code Solved and fill the Resolution Notes with some resolution such as "web service was restarted". Repeat this process for at least five tickets.

  ![updated-incident](./images/servicenow/updated-incident.png)

2) Now we can load the historical incidents data. We will verify that there is no incident data loaded yet by running the following command

```
<<<login into the cluster>>>
oc exec -it  `oc get pods|grep ai-platform-api-server|awk '{print $1;}'` bash
```
Now run the following CURL command
```
curl --insecure -u $ES_USERNAME:$ES_PASSWORD -X GET https://$ES_ENDPOINT/_cat/indices -k|sort|grep -e incident -e snow
yellow open snowchangerequest                   tyEtirBzSh6sAgN6mVc51g 1 1      0  0    208b    208b
yellow open snowincident                        bEX48U3rT4GzpM0zt6Mgaw 1 1      0  0    208b    208b
```
As you can see there is no data yet. 

We will enable the incident data flow now. From the Home page click on Data and tool integrations. Click on the ServiceNow integration and select Edit. From the integration form, we will pull data by selecting "Data Flow" ON, check "Historical data for initial AI training" and select a start/end date to cover the existing sample incidents in the ServiceNow developer instance. In my instance this range was from 08/08/2015 until 05/13/2021. Select 2 for "Source parallelism". Keep topology Data off. Click on the Save button at the bottom of the page. 

   ![flow-on](./images/servicenow/flow-on.png)

Now incident data is being loaded into WAIOPS. Wait for an hour. 

**Note:** this is a limitation of the v3.1 product in the sense that there is no feedback in the UI regarding how the data is being loaded or what data has been loaded already.

After an hour wait, we will verify that now there is indeed incident data loaded by running the same command:

```
<<<login into the cluster>>>
oc exec -it  `oc get pods|grep ai-platform-api-server|awk '{print $1;}'` bash
```
Now run the following CURL command
```
curl --insecure -u $ES_USERNAME:$ES_PASSWORD -X GET https://$ES_ENDPOINT/_cat/indices -k|sort|grep -e incident -e snow
yellow open snowchangerequest                   tyEtirBzSh6sAgN6mVc51g 1 1    164  0 169.3kb 169.3kb
yellow open snowproblem                         xBjP5lvrRYWOlidFd0X_Xw 1 1      2  0 123.2kb 123.2kb
yellow open snowincident                        bEX48U3rT4GzpM0zt6Mgaw 1 1    138  0 280.8kb 280.8kb
```
As you can see there is incident data now loaded in elastic search. Now we need to disable the data flow. Click on the ServiceNow integration and select the 3dot menu on the far right and select "Disable Ticket Data Flow".

3) Now its time to use that data for historical similar incident training. Click AI Model Management from the Home page and select the Similar Incidents card as shown below.

  ![similar-incidents-button](./images/servicenow/similar-incidents-button.png)

  Create a training definition by clicking on the Create Training Definition button. Select "manually" for the schedule option and click save. The next picture shows a training definition (not that in this case it has already been used for training)

  ![training-definition](./images/servicenow/training-definition.png)

Now its time to run the training. From the top right Actions menu select Start Training. After some time, you will the training results and the confirmation that the model has been deployed as shown in the next picture:

  ![training-done](./images/servicenow/training-done.png)

Now, lets take a look at elastic search again and see if the new model is there. Run the following commands: 

```
<<<login into the cluster>>>
oc exec -it  `oc get pods|grep ai-platform-api-server|awk '{print $1;}'` bash
```
Now run the following CURL command
```
curl --insecure -u $ES_USERNAME:$ES_PASSWORD -X GET https://$ES_ENDPOINT/_cat/indices -k|sort|grep -e incident -e snow
yellow open 1000-1000-incident_models_latest    DEFV7jFXQcO9j16w9wg3Wg 1 1      1  0     4kb     4kb
yellow open incident-train-last-timestamp       qZBKSzkyR528ou48vR3O3g 1 1      1  0   3.5kb   3.5kb
yellow open incidenttrain                       _mI7aKfSSgi9EFPfPln-eQ 1 1     88  0 141.1kb 141.1kb
yellow open normalized-incidents-1000-1000      dknPqXemSjyfYbTXPeJ91Q 1 1     44  0  82.9kb  82.9kb
yellow open snowchangerequest                   tyEtirBzSh6sAgN6mVc51g 1 1    246  0   271kb   271kb
yellow open snowincident                        bEX48U3rT4GzpM0zt6Mgaw 1 1    207  0 456.7kb 456.7kb
yellow open snowproblem                         xBjP5lvrRYWOlidFd0X_Xw 1 1      3  0 241.2kb 241.2kb
```

As you can see, there is a new model created for incidents. 

4. Finally, lets use the new model by running the Search Similar Incidents functionality from an existing story in Slack. Select a story that has a reference to the updated incident you did previously and click on Search Similar Incident button 

  ![similar-incidents-slack](./images/servicenow/similar-incidents-slack.png)

As we can see in the previous picture, when we click search, a historical incident is found and a summary of the ticket resolution activities are shown. You can click on the text in blue to go to the actual ticket in your ServiceNow developer instance.  

### Pulling inventory and topology data from ServiceNow

Using the WAIOPS ServiceNow Observer job, you can retrieve the Configuration Management Database (CMDB) data from ServiceNow. The ServiceNow developer instance comes loaded with sample topology and inventory data. In this section, we will define an observer job to pull this information into WAIOPS, that can be later be seen from the Topology Viewer.

#### ServiceNow CMDB

The ServiceNow Configuration Management data base creates and maintains the logical configurations the network infrastructure needs to support a ServiceNow service. These logical service configurations are mapped to the physical layout data of the supporting network and application infrastructure in each of the respective domains. They track the physical and logical state of IT service elements and associate incidents to the state of service elements, which helps in analyzing trends and reducing problems and incidents. The configurations are stored in a configuration management database which consists of entities, called Configuration Items (CI), that are part of your environment. A CI may be:

* A physical entity, such as a computer or router
* A logical entity, such as an instance of a database
* Conceptual, such as a Requisition Service

A CI does not exist on its own. CIs have dependencies and relationship with other CIs. For example, the loss of disk drives may take a database instance down, which affects the requisition service that the HR department uses to order equipment for new employees.

In order to see existing CMDB data in ServiceNow, from the developer instance home page, from the top left you can search for CI, click on CI Class Manager and click the 'Hierarchy' button at the upper left of this page to start the CI Class Manager. 

  ![ci-class-manager](./images/servicenow/ci-class-manager.png)

For example, on the *CI Classes* column, click on *Hardware*, select the CI List to see the actual items and search for SAP. You will find a number of SAP servers own by the Lenovo Organization as shown below. We will see these same SAP servers later in the Topology Viewer in WAIOPS.

  ![sap-lenovo](./images/servicenow/sap-lenovo.png)

#### ServiceNow Observer Job

In WAIOPS v3.1 there are two ways to enable a ServiceNow observer job: the first way is inside the ServiceNow main integration form and the second way is as a regular observer job. In this cookbook, we will follow the second option. 

From the WAIOPS home page, click on *Data and tool integrations*, then click on the *Advanced* tab and finally select *Manage observer jobs*. You will see the list of available observers after you click on the *Add a new job* card as shown below (note that the number of observer cards will vary depending on what observers are enabled in your WAIOPS instance)

  ![observer-list](./images/servicenow/observer-list.png)

Click on the ServiceNow card and complete the observer job form as shown below: 

* Unique ID: Be sure to give your job a unique name so you can recognize it in the future.
* ServiceNow instance: Specify the URL of the ServiceNow developer instance.
* ServiceNow username: Specify the same ServiceNow *events* user username you configured before under the **Creating ServiceNow integrations** section.
* ServiceNow password: Specify the same encrypted password you configured before under the **Creating ServiceNow integrations** section.

Leave the optional **Additional parameters** section unchanged. Click on the **Job schedule** section and confirm that Run time is set to *now* and Time interval as *once only* 

The following screenshot shows the complete form: 

  ![edit-observer-job](./images/servicenow/edit-observer-job.png)

Finally click on **Save** to save the observer job configuration and run the job. This job will pull all the existing topology and inventory data from your ServiceNow developer instance into WAIOPS.

#### Observer Job logs

After you let the job run, lets check the logs of the observer pod by running the following command:

```
oc logs `oc get pods|grep servicenow-observer|awk '{print $1;}'` > servicenow-observer-logs.txt
```

Open the log output file *servicenow-observer-logs.txt* and verify that you see log lines like those shown below:

```
INFO   [2021-05-21 19:08:45,222] [dw-28 - POST /1.0/servicenow-observer/jobs/load] c.i.i.t.o.a.JobPostApi -  dw-28 - POST /1.0/servicenow-observer/jobs/load - submitJobRequest - Accepted job id: DcXec2deRKyC4GbcHmygew, name: ServiceNow Observer Job 1, for scheduling. 
INFO   [2021-05-21 19:08:45,298] [pool-12-thread-1] c.i.i.t.k.c.a.ConsumerWorker -  Consumer topic [itsm.nodes.json] Read 2 records from Kafka 
INFO   [2021-05-21 19:08:45,301] [pool-12-thread-1] c.i.i.t.o.p.ResourceUploadPluginManager -  Change of {"keyIndexName":"servicenow-observer-provider","name":"servicenow-observer-provider","entityTypes":["provider"],"vertexType":"mgmtArtifact","_id":"VD6JvpGcMrUUbsKfYyucXQ"} 
...
INFO   [2021-05-21 19:08:51,808] [Fetch:cmn_department] c.i.i.t.o.s.a.r.ServiceNowRestApi -  Fetched records 1 to 7 of /api/now/table/cmn_department from a total 7 records 
INFO   [2021-05-21 19:08:52,057] [Fetch:cmn_department] c.i.i.t.o.s.a.r.ServiceNowRestApi -  Process all 7 records for /api/now/table/cmn_department 
...
INFO   [2021-05-21 19:08:59,722] [Fetch:cmdb_ci] c.i.i.t.o.s.a.r.ServiceNowRestApi -  Fetched records 1 to 2784 of /api/now/table/cmdb_ci from a total 2784 records 
...
INFO   [2021-05-21 19:09:23,087] [pool-12-thread-1] c.i.i.t.o.p.ResourceUploadPluginManager -  Change of {"keyIndexName":"servicenow-observer:ServiceNow Observer Job 1","name":"ServiceNow Observer Job 1","entityTypes":["ASM_OBSERVER_JOB"],"vertexType":"mgmtArtifact","tags":["OBSERVATION_JOB","servicenow-observer"],"_id":"DcXec2deRKyC4GbcHmygew","_changeMap":{}} 
```

#### Verify Data in Topology Viewer

Finally, lets verify the ServiceNow inventory and topology data in the Topology Viewer. From the WAIOPS home page click on *Topology viewer* and search for Lenovo and click on the search result blue target. You will see that the topology for the ServiceNow Organization *Lenovo* is shown. If you zoom-in, you will see that this topology includes the SAP servers (SAP-SD-01, SAP-SD-02, etc) we saw before in ServiceNow.

  ![lenovo-topology](./images/servicenow/lenovo-topology.png)

### Change Risk Assessment

The ServiceNow integration for Change Risk Assessment is broken. A defect has been open here https://github.ibm.com/katamari/dev-issue-tracking/issues/9090
Additional info here: https://ibm-cloud.slack.com/archives/C01L36KNJ4U/p1622068401194600

