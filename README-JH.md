NSO Local Install: https://developer.cisco.com/docs/nso/guides/#!nso-local-install/nso-local-install
NSO System Install : https://developer.cisco.com/docs/nso/guides/#!nso-system-install/nso-system-install

NEDs List: https://community.cisco.com/t5/nso-developer-hub-knowledge-articles/device-types-supported-by-cisco-nso-neds/ta-p/3641664

================
Pre-reqs Installation
================

> apt-get update
> apt-get upgrade

> apt install default-jdk		# Java
> apt install ant			# Apache Ant
> apt install libxml2-utils		# Development tools


==============
Local Installation
==============

> scp /Users/jinhoe/Downloads/nso-5.7.1.linux.x86_64.signed.bin administrator@10.16.0.136:/tmp

> apt install python-is-python3				# Because Ubuntu do not install python 2, only python 3
> sh nso-5.7.1.linux.x86_64.signed.bin

> sh nso-5.7.1.linux.x86_64.installer.bin --help	# Notice the two options for --local-install or --system-install

> sh nso-5.7.1.linux.x86_64.installer.bin --local-install ~/nso-5.7
> cd ~/nso-5.3/doc							# NSO installs a full set of documentation

*** How to read documents? https://10.16.0.136/index.html ***


================================
Installing NEDs (Network Element Drivers)
================================

Copy from local to vm:
scp /Users/jinhoe/Downloads/ncs-5.7-cisco-iosxr-7.38.3.signed.bin administrator@10.16.0.136:/tmp
scp /Users/jinhoe/Downloads/ncs-5.7-cisco-ios-6.77.10.signed.bin administrator@10.16.0.136:/tmp
scp /Users/jinhoe/Downloads/ncs-5.7-cisco-asa-6.13.12.signed.bin administrator@10.16.0.136:/tmp
scp /Users/jinhoe/Downloads/ncs-5.7-cisco-nx-5.22.8.signed.bin administrator@10.16.0.136:/tmp

Verify and extract it to tar.gz:
sh ncs-5.7-cisco-iosxr-7.38.3.signed.bin
sh ncs-5.7-cisco-ios-6.77.10.signed.bin
sh ncs-5.7-cisco-asa-6.13.12.signed.bin
sh ncs-5.7-cisco-nx-5.22.8.signed.bin

> cd ~/nso-5.7/packages/neds						# Go to the packages/neds directory

Update path accordingly:
tar -zxvf /tmp/ncs-5.7-cisco-iosxr-7.38.3.tar.gz
tar -zxvf /tmp/ncs-5.7-cisco-ios-6.77.10.tar.gz
tar -zxvf /tmp/ncs-5.7-cisco-asa-6.13.12.tar.gz
tar -zxvf /tmp/ncs-5.7-cisco-nx-5.22.8.tar.gz


=============================
ncsrc
!! Source this file before starting NSO
=============================

Once it has been "sourced" you have access to all the NSO executable commands - which start with ncs
> source ~/nso-5.7/ncsrc							# "echo $NCS_DIR" command to check 

Add "source ~/nso-5.7/ncsrc" to .bashrc so you don't have to enter this command every time you login.
> vi ~/.bashrc

# Access to all the NSO executable commands
source ~/nso-5.7/ncsrc


==================================================
Creating an Instance of NSO
Is like creating a Python virtual environment after installing Python
==================================================

* --dest defines the directory where you want to setup NSO (if the directory does not exist, it will be created)
* --package defines the NEDs you want to this NSO instance to have installed. You can specify this option multiple times.

> ncs-setup --package ~/nso-5.7/packages/neds/cisco-ios-cli-6.77 \
   --package ~/nso-5.7/packages/neds/cisco-nx-cli-5.22 \
   --package ~/nso-5.7/packages/neds/cisco-iosxr-cli-7.38 \
   --package ~/nso-5.7/packages/neds/cisco-asa-cli-6.13 \
   --dest nso-instance

Inside nso-instance:
	- ncs.conf is NSO application configuration file
	- packages/ is the directory that has symlinks to the NEDs that we referenced in.
	- logs

To start NSO instance
!! You need to be in the nso-instance directory each time you want to start or stop NSO
> cd ~/nso-5.7/nso-instance		# Go to the nso-instance directory
> ncs						# ncs command to start or "ncs --stop" to stop

> ncs --status | grep status		# Verify that NSO is running


=========================
Setting up device authentication
With NSO running, you add your devices into its inventory. This inventory enables NSO to read and write data. NSO uses authgroups to set up credentials for device access.
=========================

> ncs_cli -C -u admin			# Run this command from anywhere once NSO is running, not just in the nso-instance directory

> config						# Enter config mode, to exit config mode use "exit" command

Set up a new authgroup called labadmin. This group uses a default username / password combination of cisco / cisco for devices, with a secondary password of cisco. Use the following commands:

> devices authgroups group labadmin
> default-map remote-name cisco
> default-map remote-password cisco
> default-map remote-secondary-password cisco

> top						# Return back to the top of config mode
> show configuration

> commit						# Commit the configuration to the system


=======================
Adding devices to Cisco NSO
With your authgroup set up, add your devices to the inventory.
=======================

To add a device, you need:
	- The device's IP address or FQDN
	- The protocol (SSH, Telnet, HTTP, REST, and so on) and port (if nonstandard) to connect to the device
	- The authgroup to use for the device (which must be committed before adding devices)
	- The NED (device driver) to use to connect to the device

> devices device edge-sw01
> address 10.10.20.172						# IP address
> authgroup labadmin						# authgroup
> device-type cli ned-id cisco-ios-cli-6.67		# NED
> device-type cli protocol telnet				# Protocol
> ssh host-key-verification none				# You disable SSH host-key-verification (in production systems you can configure the host-key or just "fetch" to learn)
> commit

> pwd									# Verify that you are still in the CLI context for the device
> connect								# Determine if you can connect to the device with NSO (default mode is a locked state)

> state admin-state unlocked					# Unlock the device
> commit

3 options to add the remaining devices:
	a) Manually type the above commands for each device
	b) Use an API to add devices to the device list
	c) Query the production system install NSO server that has the full device list already and save that output to a file. You can then use the load merge command from NSO to read the file into NSO and stage it for committing.


======================================
Adding multiple devices to Cisco NSO (Option C)
======================================

SSH from same developer@devbox node
> ssh 10.10.20.49									# SSH into Production system-install NSO server
> ncs_cli -C -u admin

Get all of the device configuration stored in NSO, excluding the configuration it has stored.
Save it into a file called nso-device-config.txt in the user's home directory on the DevBox. (Use vim or nano, etc)
> show running-config devices device | de-select config	# Copy device configuration in a notepad

> exit											# Exit the management session
> exit											# Exit ssh session (10.10.20.49)

Back to developer@devbox node
> cd /home/developer
> vi nso-device-config.txt							# Paste device configuration from the notepad

> ncs_cli -C -u admin
> config
> load merge /home/developer/nso-device-config.txt		# Load the file
> commit											# Add these devices to NSO
> exit											# Exit config mode

> show devices list									# Verify
> devices connect									# Make sure that NSO can communicate with these devices


==================================
"Learning" Current network state (sync-from)
Now NSO can successfully connect to the network, but it hasn't "learned" the current network configuration deployed to the devices.
This exercise "teaches" the network configuration.
==================================

If not at admin@ncs (top level) enter "ncs_cli -C -u admin" command
> show running-config devices device edge-sw01 config	# Verify that there is no real device level configuration within NSO yet

> devices sync-from								# Enables the network devices to "learn" the current configuration
> show running-config devices device edge-sw01 config	# Check the NSO running-configuration of the devices again


=====================
Grouping devices together
You create device groups to organize your devices into logical groups.
Then you can apply similar configuration or take similar actions on a group
=====================

Create Network Operating System Based Groups:		# Groups can contain other groups for nesting
	- IOS-DEVICES
	- XR-DEVICES
	- NXOS-DEVICES
	- ASA-DEVICES

If not at admin@ncs (top level) enter "ncs_cli -C -u admin" command
> config
> devices device-group IOS-DEVICES
> device-name internet-rtr01
> device-name dist-rtr02

> devices device-group XR-DEVICES
> device-name core-rtr01
> device-name core-rtr02

> devices device-group NXOS-DEVICES
> device-name dist-sw01
> device-name dist-sw02

> devices device-group ASA-DEVICES
> device-name edge-firewall01

> devices device-group ALL							# One group that includes ALL your devices
> device-group ASA-DEVICES
> device-group IOS-DEVICES
> device-group NXOS-DEVICES
> device-group XR-DEVICES

> top
> show configuration
> commit
> exit

> devices device-group IOS-DEVICES check-sync		# Check the sync status of all internal devices in a group


======================
Exploring NSO CLI
Show network configuration
======================

> ncs_cli -C -u admin													# Start out by accessing the CLI

> show running-config devices device internet-rtr01 config						# Reviewing the full running configuration for a single device

> show running-config devices device internet-rtr01 | de-select config				# Use de-select keyword to filter out elements

> show running-config devices device internet-rtr01 config interface				# Retrieve the list of interfaces configured on the device

> show running-config devices device internet-rtr01 config interface | display json	# Retrieve the data in a format (JSON or XML)

> show running-config devices device dist-sw01 config interface Vlan				# VLAN interfaces (case-sensitive, use Vlan, not vlan)

> show running-config devices device dist-sw01 config interface Vlan * ip address	# Retrieve the IP addresses for all SVIs using the wildcard character *


========================
Updating device configuration
Single device
========================

> ncs_cli -C -u admin		# Start out at the ncs_cli and go into config mode
> config

> devices device dist-sw01	# Updating a single device configuration
> config

> pwd					# Use pwd to see where you are

Adding a new VLAN and interface (you can copy and paste the whole command):
> vlan 42
> name TheAnswer
> exit
> interface Vlan 42
> description "Answer to the Ultimate Question of Life, the Universe, and Everything"
> ip address 10.42.42.42/24
> exit

> top							# Return to the top of config mode
> show configuration				# Review the current configuration that is waiting to be committed
> show configuration | display xml		# Preview the XML version of this new configuration (or "display curly-braces")

> commit dry-run outformat native		# The native format shows you exactly what NSO sends
> commit
> show configuration				# Check all the staged changes have been completed


======================
Rolling back configurations
======================

> show configuration rollback changes		# In config mode

> rollback configuration					# Set up the rollback
> show configuration

> commit dry-run outformat native
> commit


====================
Templates and live status
Multiple devices
====================

> ncs_cli -C -u admin
> config

Make a template in the NSO CDB:
> devices template SET-DNS-SERVER
> ned-id cisco-nx-cli-5.20
> config
> ip name-server servers 208.67.222.222
> ip name-server servers 208.67.220.220

> top
> show configuration					# NSO takes the input of two different name servers and combines them into a list object within the CDB

> commit								# Even though the template is saved into the NSO CDB, you must commit it before applying to any devices


================
Testing the template
================

> do show devices list					# The do command enables you to issue nonconfiguration level commands in config mode

> devices device dist-sw01 apply-template template-name SET-DNS-SERVER		# Apply template to dist-sw01
> show configuration													# Review the staged configuration

> revert																# Unstage these changes


==================
Updating the template
==================

Update the template to support IOS and ASA devices as well:

devices template SET-DNS-SERVER
! IOS TEMPLATE
ned-id cisco-ios-cli-6.67
config
ip name-server name-server-list 208.67.222.222
ip name-server name-server-list 208.67.220.220
exit
exit
exit

! ASA TEMPLATE
! 3 exits to return back to the 'config-template-SET-DNS-SERVER' context
ned-id cisco-asa-cli-6.12
config
dns domain-lookup mgmt
dns server-group DefaultDNS
name-server 208.67.220.220
name-server 208.67.222.222

! IOS-XR TEMPLATE
ned-id cisco-iosxr-cli-7.32
config
domain name-server 208.67.222.222
exit
domain name-server 208.67.220.220

> commit
> top

Now your template knows how to configure the DNS Servers on all four of your network types. Since NSO is aware of each device type, it automatically applies the correct template syntax to the device.


=============================
Applying the template to the network
=============================

Use the device-groups that you set up initially to push this change out to the entire network.
With one simple command, you've updated every device all at once (in the CDB locally).

> devices device-group ALL apply-template template-name SET-DNS-SERVER
> commit dry-run outformat native
> commit


====================
Device operational state
basic platform information stored in the NSO CDB
====================

> ncs_cli -C -u admin									# Don't go into config mode

> show devices device dist-sw01 platform					# Review the information

> show devices device dist-sw01 platform serial-number		# View only serial number

> show devices device * platform serial-number				# Use a wildcard to retrieve ALL of the serial numbers of all your devices

> show devices device * platform version					# Retrieve the platform version for all devices


=========
Live status
=========

Devices have a live-status part of the model that indicates details that NSO reads from the device at the time of execution, rather than pulling from the CDB.
! You can't use live-status with device-groups because a group of devices could leverage different NEDs.

> show devices device dist-sw01 live-status port-channel		# Check on the current status of the port-channels on a device

> show devices device dist-sw01 live-status ip route			# View the routing table


============================
Live status and arbitrary commands
============================

> devices device dist-sw01 live-status exec show license usage		# Check the status on a single device

> devices device dist-sw01 live-status exec any dir				# Using the "exec any" command to run any command

> devices device dist* live-status exec show license usage			# The wildcard here uses dist* to issue the command to all dist devices

Often, you need to gather data like this for later processing, or to share with someone. NSO enables you to save command output to a file.
> devices device dist* live-status exec show license usage | save /home/developer/nexus-license-usage.txt
> exit
> cat /home/developer/nexus-license-usage.txt


===========================
Network configuration compliance
===========================

In this exercise, you ensure that across the network, all device configurations match for these network features.
You can extend the technique used in this example to any aspect of the configuration:
	- DNS server configuration
	- Syslog server configuration
	- NTP configuration

You use the following NSO features in this section
	- Device templates
	- Compliance reports

Understanding the compliance use case. This use case has two parts:
	1. Building a device template that contains the desired configuration to test.
	2. Building a compliance report to check the configuration of a group of devices against a template.


==================================================
Network configuration compliance - Building the device template
==================================================

> ncs_cli -C -u admin
> config

Create the device template with the desired configuration for features:

devices template COMPLIANCE-CHECK
ned-id cisco-ios-cli-6.67
config

ip name-server name-server-list 208.67.222.222
ip name-server name-server-list 208.67.220.220
exit

service timestamps log datetime localtime show-timezone year
logging host ipv4 10.225.1.11
exit

ntp server peer-list 10.225.1.11
exit

exit
exit

ned-id cisco-nx-cli-5.20
config

ip name-server servers 208.67.222.222
ip name-server servers 208.67.220.220

logging timestamp milliseconds
logging server 10.225.1.11 level 5
exit

ntp server 10.225.1.11
exit

exit
exit

ned-id cisco-asa-cli-6.12
config
dns domain-lookup mgmt
dns server-group DefaultDNS
name-server 208.67.222.222
name-server 208.67.220.220
exit

logging timestamp
logging host mgmt 10.225.1.11
exit

ntp server 10.225.1.11
exit

exit
exit

> top
> show configuration
> commit


===================================================
Network configuration compliance - Building the compliance report
Compliance reports simply compare a given template to a group of devices. So the configuration is simple.
===================================================

Be aware of the following features of compliance reports:
	- Results can come in in HTML, Bash, or XML format. XML is the default.
	- Because the results are likely very "verbose" and not something the engineer wants to see only at the CLI, NSO saves the results to a file located in the folder state/compliance-reports.
	- NSO runs a web server so these reports are available by navigating to the given URL, or you can find them by navigating the folders on your server.

> compliance reports report COMPLIANCE-CHECK
> compare-template COMPLIANCE-CHECK ALL

> commit
> end


=======================================================
Network configuration compliance - Executing the compliance use case
=======================================================

> compliance reports report COMPLIANCE-CHECK run					# Run the compliance report

> compliance reports report COMPLIANCE-CHECK run outformat text		# Generate the text and HTML versions of the report
> compliance reports report COMPLIANCE-CHECK run outformat html

> exit
> cat nso-instance/state/compliance-reports/report_*.text					# View report in diff format


=====================================================
Network configuration compliance - Resolving compliance problems
After you run a compliance report, you want to resolve any problems. NSO helps resolve compliance problems.
=====================================================

> ncs_cli -C -u admin
> config

> devices device-group ALL apply-template template-name COMPLIANCE-CHECK		# Apply the template to your devices

> show configuration
> commit dry-run outformat native
> commit

> end																	# Exit config mode and re-run the compliance report
> compliance reports report COMPLIANCE-CHECK run outformat text
> cat nso-instance/state/compliance-reports/report_*.text							# Should get "No discrepancies found"


===================================
NSO services - Creating the service package
===================================

> cd nso-instance/packages
> ls
> ncs-make-package -h											# Look at the available options
> ncs-make-package --service-skeleton template loopback-service		# Create a service skeleton

> tree loopback-service											# Use "tree" command to view the folder structure

The key files that you are editing are:
loopback-service/src/yang/loopback-service.yang and
loopback-service/templates/loopback-service-template.xml.
Services can also coding logic, but in this simple example, you only use the most basic features.

> cat loopback-service/src/yang/loopback-service.yang
> cat loopback-service/templates/loopback-service-template.xml


===============================
NSO services - Service package details
===============================

The YANG file (loopback-service.yang) - There are a few leaf variable names, including name, device, and dummy. A leaf is simple a single key:value input.
	- The "name" variable is the service instance unique key lookup (more on that topic later too).
	- The "device" variable is a dynamic reference in the NSO CDB to the device list.
	- The "dummy" variable is a sample input to use in a device template, with a variable type of IP address.

The XML file (loopback-service-template.xml) - The configuration NSO uses to apply to your network devices. You plug in the variable names from your YANG file into here to have dynamic configuration. You can also use multiple YANG files (more often with extra Python or Java code logic).
	- The important part of the XML file which you use is between the tags <config> and </config>, which right now has an XML comment.


======================================
NSO services - Developing your service package
======================================

Now you get some XML to plug into that template file.

> ncs_cli -C -u admin
> config

devices device dist-rtr01 config
interface Loopback 100
ip address 10.10.30.0 255.255.255.0

> show configuration
> commit dry-run outformat xml			# Copy between the config tags (only the interface part) and paste into a notepad

> exit								# ! Do not commit, discarding the pending NSO device change.
> exit

> cd nso-instance/packages
> vi loopback-service/templates/loopback-service-template.xml
Paste <interface> from notepad to in between the config tags. Add the IP Address variable {/dummy} into the configuration template replacing the existing address of 10.10.30.0.

> cd loopback-service/src				# Go to loopback-service/src/ directory
> make								# Compile the YANG data model

> ncs_cli -C -u admin					# Go back into the NSO CLI
> packages reload						# Load in the new service package


======================
Creating a service instance
With the service package loaded in NSO, you can start creating service instances.
These instances take custom inputs based on the YANG model and the config template you defined in XML.
======================

> ncs_cli -C -u admin
> config

> loop?					# Use the question mark to discover the new CLI option from the newly loaded package

> loopback-service test		# Use the new "loopback-service" command to create a service instance with a name called test
> device dist-rtr01			# populate the variables
> dummy 192.168.1.1

> top
> commit dry-run outformat native
> show configuration
> commit


==================
Redeploying a service
Let's assume that someone goes into your network device and makes a change. That change is an out-of-band change that NSO is unaware of, and conflicts with your service. What do you do?
==================

1. Exit NSO and go back to the bash terminal.

2. Verify the interface loopback is present
> telnet 10.10.20.175				# Username: cisco Password: cisco
> show running-config | s interface

3. While in that telnet session (dist-rtr01#), issue the following commands to remove the loopback that NSO added.
conf t
no interface Loopback 100
end
show running-config | s interface
exit

4. Reenter the NSO CLI and view NSO's understanding of the device
> ncs_cli -C -u admin
> show running-config devices device dist-rtr01 config interface Loopback		# It is not aware of the change yet; the Loopback is still present in the CDB snapshot of the device.

5. Preview and execute "sync-from" so that NSO can relearn the updated device config
> devices sync-from dry-run
> devices sync-from

6. Enter the config mode and view a dry run of the service being re-deployed to see that the Loopback is added again. Then re-deploy it.
> config
> loopback-service test re-deploy dry-run
> loopback-service test re-deploy

NSO tracks what the config should be on the device, so once it learns that the device is out of sync, it can easily redeploy the service configuration.

7. Exit the NSO CLI and telnet back to dist-rtr01 and verify that the loopback is now present.
> end
> exit
> telnet 10.10.20.175
> show running-config | s interface		# Run this command in telnet session (dist-rtr01#)


=======================================
========== NSO Simple Python ==========
=======================================

=============================
Set Up Your Lab Environment
=============================

Spin up your NSO Instance and a test device:
ncs-netsim create-device ~/nso-5.4.1/packages/neds/cisco-ios-cli-6.67 netsim-ios
cp ~/src/netsim-ios.xml ~/netsim/netsim-ios/netsim-ios/cdb/
rm ~/netsim/netsim-ios/netsim-ios/cdb/ios.xml
ncs-setup --dest . --netsim-dir netsim
ncs-netsim start
ncs
ncs_cli -C -u admin

> devices sync-from		# Have NSO learn the device configuration
> exit

> ncs-netsim cli-c netsim-ios	# Navigate and view the IOS device configuration

> paginate false		# Within the netsim prompt you can view the full config
> show running-config 
> exit


================
NSO Python API - CDP Run Audit
!! You must be on the same machine as NSO to use NSO Python library
================

Check to see if CDP is enabled on our netsim-ios device:

Create the following python script in src/nso-test.py:

import ncs

with ncs.maapi.single_read_trans("admin", "python", groups=["ncsadmin"]) as t:
    root = ncs.maagic.get_root(t)
    device_object = root.devices.device["netsim-ios"]
    cdp_result = device_object.config.ios__cdp.run
    print(
        "For Device {}, CDP being enabled is {}".format(device_object.name, cdp_result)
    )

> python src/nso-test.py	# Run the script


================
NSO Python API - Interface Audit
================
Generalizing the script a bit more, using a variable for the device name. We are also adding a loop to iterate over the interface list object, just for GigabitEthernet interface types. The script looks up the value for the interface number (which is called the name in the NED model), IP address and mask.

import ncs
with ncs.maapi.single_read_trans("admin", "python", groups=["ncsadmin"]) as t:
    device_name = "netsim-ios"
    root = ncs.maagic.get_root(t)
    device = root.devices.device[device_name]
    for interface in device.config.ios__interface["GigabitEthernet"]:
        int_name = interface.name
        int_ip = interface.ip.address.primary.address
        int_mask = interface.ip.address.primary.mask
        print(
            "For Device {}, Interface GigabitEthernet {} has an IP address of {} and a mask of {}".format(
                device_name, int_name, int_ip, int_mask
            )
        )








