# OCI-AutoScale

Welcome to the Scheduled Auto Scaling Script for OCI (Oracle Cloud Infrastructure).

The **AutoScaleALL** script: A single Auto Scaling script for all OCI resources that support scaling up/down and power on/off operations.

# NEW 
- Support for changing the CPU and Memory Count for Compute Flex Shapes (WILL REBOOT THE INSTANCE!!)
- Support for Nth day of the Month Scheuld (Like 1st saturday or 3rd Saturday)
- Support for a Day of the Month Schedule (Like 1st of the month or 15th of the month)
- Support running on all regions 
- Added flags as parameters for execution:

```
   -t config  - Config file section to use (tenancy profile)
   -ip        - Use Instance Principals for Authentication
   -dt        - Use Instance Principals with delegation token for cloud shell
   -a         - Action - All,Up,Down (Default All)
   -tag       - Tag to use (Default Schedule)
   -rg        - Filter on Region
   -ic        - include compartment ocid
   -ec        - exclude compartment ocid
   -override  - Override schedule based on a tag
   -ignrtime  - ignore region time zone (Use host time)
   -printocid - print ocid of resource
   -topic     - topic OCID to sent summary (in home region)
   -h         - help
```

- Support for MySQL service added
- Support for GoldenGate service added
- Bug fix for DB RAC instances, now both nodes are shutdown / started

# Supported services
- Compute VMs: On/Off
- Compute Flex Shapes: Change CPU count and Memory Count (WILL REBOOT THE INSTANCE!!)
- Instance Pools: On/Off and Scaling (# of instances)
- Database VMs: On/Off
- Database Baremetal Servers: Scaling (# of CPUs)
- Database Exadata CS: Scaling (# of CPUs)*
- Autonomous Databases: On/Off and Scaling (# of CPUs)
- Oracle Digital Assistant: On/Off
- Oracle Analytics Cloud: On/Off and Scaling (between 2-8 oCPU and 10-12 oCPU)
- Oracle Integration Service: On/Off
- Load Balancer: Scaling (between 10, 100, 400, 8000 Mbps)**
- MySQL Service: On/Off***
- GoldenGate: On/Off
- Data Integration Workspaces: On/Off
- Visual Builder (v2 Native OCI version): On/Off

*Supports the original DB System resource model and the newer Cloud VM Cluster resource model (introduced in Nov 2020)

**For the loadbalancer service, specify the number 10,100,400 or 8000 for each hour to set the correct shape.
When changing shape, All existing connections to this load balancer will be reset during the update process and may take up to a minute, leading to potential connection loss. For non session persistent web based applications, I did not see any noticeable interruption or downtime in my own tests, but please test yourself!

***MySQL Instances are not found by the search function :-( So a special routine is run to query them. 
Also MySQL instances that are not running (Active state)) do not allow their tags to be changed/added/removed. 

# Features
- Support for using the script with Instance Principle. Meaning you can run this script inside OCI and when configured properly, you do not need to provide any details or credentials.
- Support for sending Notification after script is done. Thanks to Joel Nation for this! All you need to do is configure the Topic OCID in the script and make sure the user or instance principle has the correct permissions to publish Notifications.

# Install script into (free-tier) Autonomous Linux Instance
Youtube demonstration video: https://youtu.be/veHbyvDB74A

- Create a (free-tier) compute instance using the Autonomous Linux or Oracle Linux image
- Create a Dynamic Group called Autoscaling and add the OCID of your instance to the group, using this command:
  - ANY {instance.id = 'your_OCID_of_your_Compute_Instance'}
- Create a root level policy, giving your dynamic group permission to manage all resources in tenancy:
  - allow dynamic-group Autoscaling to manage all-resources in tenancy
- Login to your instance using an SSH connection
- run the following commands:
  - wget https://raw.githubusercontent.com/arjen-keesy/OCI-AutoScale/master/install.sh
  - bash install.sh
  - sudo su
- create a user in Oracle Cloud and run the command 'oci setup config'
    - and give user OCID, Tenant OCID and region; generate a private key and use the public key as API in user maintenance of the just created user.
    - give oracle cloud some time to populate the rights of the user ( maybe 5 minutes? )
- If this is the first time you are using the Autoscaling script, go to the OCI-Autoscale directory and run the following command:
  - python3 CreateNameSpaces.py -ip

The Install script will configure the time zone to European Central Time (CET). If you want to operate in a difference timezone, run the command:
- sudo timedatectl set-timezone Europe/Amsterdam

The instance is now all setup and will run 2 minutes before the hour all scaling down/power down operations 
and 1 minute after the hour scaling up/power on operations.

# How to use
To control what to scale up/down or power on/off, you need to create a predefined tag called **Schedule**. If you want to
localize this, that is possible in the script. For the predefined tag, you need entries for the days of the week, weekdays, weekends and anyday. The tags names are case sensitive! 

A single resource can contain multiple tags. The priority of tags is as followed (from low to high) 
- Anyday
- Weekday or Weekend
- Day of the week (Like Monday, Tuesday...)
- Day of the month (Example 1 = 1st or 15 = 15th of the month)

### Values for the AnyDay, Weekday, Weekend and Day of week tags:
The value of the tag needs to contain 24 numbers and/or wildcards (*) (else it is ignored), separated by commas. If the value is 0 it will power off the resource (if that is supported for that resource). Any number higher then 0 will re-scale the resource to that number. If the resource is powered off, it first will power-on the resource and then scale to the correct size.

When a wild card is used, the service will stay unmodified for that hour. For example, the below schedule will turn of a compute instance in the evening/night, but allows the user to manage the state during the day.

Schedule.AnyDay : 0,0,0,0,0,0,0,0,\*,\*,\*,\*,\*,\*,\*,\*,0,0,0,0,0,0,0,0

Comments can be added to the end of a schedule and start with `#`

Schedule.AnyDay : 0,0,0,0,0,0,1,1,1,1,1,1,1,1,1,1,1,1,0,\*,\*,\*,\*,\* #Ashburn

Schedule.AnyDay : 0,0,0,0,1,1,1,1,1,1,1,1,1,1,1,1,0,\*,\*,\*,\*,\*,0,0 #Phoenix

### Values for DayOfMonth tags
The DayOfMonth tag allows to set a resource on a specific day of the month. This can be specified by day:size

For each day <b>Only one value</b> can be specified! This value is valid for all hours of that day

The below example tag schedules the resource on the 1st of the month to 4, on the 3rd of the month to 2 and on the 28th of the month back to 5:

Schedule.DayOfMonth : 1:4,3:2,28:4

### Values for Nth day of the Month
If you want to have a schedule for the Nth day of the month, like 1st Saturday or 3rd Saturday, you need to add extra Tag Key definitions.

For example, if you want to have an schedule for the 2nd Saturday of the months, create an extra Tag Key definition called **Saturday2**

A specific Nth day of month schedule overwrites a normal day of the month schedule. So a **Saturday2** overwrites a **Saturday** schedule.

![Scaling Example Instance Pool](http://oc-blog.com/wp-content/uploads/2022/06/Screenshot-2022-06-13-at-11.10.31.png)

![Power Off Example DB VM](https://oc-blog.com/wp-content/uploads/2019/06/ScaleExampleDB.png)

### Override Tag

There may times when you want to force a group of servers to start or stop outside a predefined schedule. This could be a group of lab servers used for training that you want to control ad-hoc. You can add a defined tag `Schedule.Override` with any value to the instances and reference them with the `-override` parameter. 

For example, you have the tag `Schedule.Override: labservers` set on instances, you can start those instances using this command.

```bash
python3 AutoScaleALL.py -a Up -override labservers
```

Using the override feature, it will only do PowerOn (1) / PowerOff (0) actions, not rescale any resource.

### Changing the CPU and/or Memory Count of Compute Flex Shape
If a value in the schedule is written as: (4:8) it will modify the CPU and Memory Count. The format should be (cpu:memory). 

IMPORTANT: This will reboot the compute instance! 

![Changing a Flex Shape](https://oc-blog.com/wp-content/uploads/2022/11/flex.png)


### Running the Script

The script supports 3 running methods: All, Up, Down

- All: This will execute any scaling and power on/off operation
- Down: This will execute only power off and scaling down operations
- Up: This will execute only power on and scaling up operations

The thinking behind this is that most OCI resources are charged per hour. So you likely want to run scale down / power off operations 
just before the end of the hour and run power on and scale up operations just after the hour.

To ensure the script runs as fast as possible, all blocking operations (power on, wait to be available and then re-scale) are executed in seperate threads. I would recommend you run scaling down actions 2 minutes before the end of the hour and run scaling up actions just after the hour.

You can deploy this script anywhere you like as long as the location has internet access to the OCI API services. 

## Disclaimer
**Please test properly on test resources, before using it on production resources to prevent unwanted outages or very expensive bills.**
