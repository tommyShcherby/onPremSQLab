
### On-premise PostgreSQL laboratory envoironment (mostly) as code.
PostgreSQL Deployment on Premise 

Version: 1.0.0 


Project Description 
A single-node PostgreSQL deployment for experimental purposes. 


Environment Summary (project’s unique components in bold): 
Windows Host 
Hyper-V
pgAdmin

Linux Guest 1 
Ansible
Git

Linux Guest 2 
PostgreSQL (server) 
psql 


Platform Specification 
The deployment is hosted locally on the workstation using Hyper-V. 
CentOS Stream 8 is used in anticipation of CentOS Linux 8 end-of-life Dec 2021. 


Linux Guest 2 Virtual Machine Requirements 
The Local Standard Disk is the only storage device. 
As the database is not predicted to occupy more than 10GB of space, a total of 13GB would be the space requirement for the PostgreSQL component of the deployment. That is assuming DBsize * 0,1 for the transaction log, and DBsize * 0,2 for the backup needs. This will be used when sizing the initial deployment. 
The required 10GB of disk space for the operating system would bring the total to 23GB.  
The swap space will be sized using the RAM*0,5 ratio. 
Here a particular advantage of doing a Hyper-V deployment presents itself. Even though the disk size is very small the IOPS parameter is not artificially throttled. The IOPS throughput in public cloud deployments is always disk size dependent. 
2GiB of RAM as well as 1 vCPU should be enough to ensure smooth operation of the system under experimental load. 

Linux Guest 2 Initial Sizing: 
1 vCPU 
2 GiB RAM 
25GB vHDD (underlying SSD) 


Virtual Machine Creation Procedure 
A .vhdx format, fixed size virtual disk is created first, with the capacity stated in the VM requirements. 
Type 2 VM is selected. 
A Hyper-V VM gets created with 1 vCPU assigned. If required for a particular VM an additional vCPU will be added after creation to match the requirements. 
A vDVD containing the .iso is added under the SCSI Controller after the VM creation. 
A VHD prepared as described in Appendix A is added under the SCSI Controller after the VM creation. 
The Secure Boot template is changed to Microsoft UEFI Certificate Authority. 
Hyper-V Checkpoints are disabled as they would never be used. The whole idea behind this project is that the deployment can be recreated easily from scratch if necessary. 
 

Operating System Installation Settings (Kickstart) 
Automatic Partitioning would not be sufficient. 

The custom partitioning looks as follows (difference vs. Automatic Partitioning in bold): 
/boot 
/boot/efi 
An LVM physical volume containing the remaining space. 

The above volume will be assigned to a volume group, which will contain the following logical volumes: 
/
/tmp 
/var/tmp 
/var/log 
/home 
SWAP 

The VM’s disk is not separately encrypted. The underlying SSD is already encrypted and since I control both the hypervisor and the physical layer that is deemed sufficient for this deployment. 
Kdump is deactivated, as I do not currently possess sufficient expertise to properly analyze the vmcore that would be written and specialist assistance is not available for this project. 

The –noipv6 option sets the ipv6.method to “ignore” instead of “disabled”. That seems to be the way Kickstart is implemented and there is no way to specify the ipv6.method parameter directly using the native Kickstart syntax. 
As a result, the second network interface does receive an IPv6 address upon activation, the traffic is simply ignored. 

(...)


PostgreSQL Operating System Footprint 
Whenever using the recommended approach of installing PostgreSQL from a repository:
A new directory named pgsql-<version> is created in /usr/. 

Symbolic links with the pattern: 
/usr/bin/<command> -> 
/etc/alternatives/pgsql-<command> -> 
/usr/pgsql-<version>/bin/<command> 

are created for the different client applications (e.g., createdb, psql) included in the binary installation. 
The main server process is named “postmaster” on CentOS 8. 
The cluster data is located in: 
/var/lib/pgsql/<version>/data/ 


Security Considerations 
“Security through Obscurity” is not going to be used. PostgreSQL listens on a standard port, but the Firewalld rules only permit such access from a specific source IP address. 
As it currently stands, the “targeted” SELinux policy is enforced. 

If during the course of the experiments any non-standard PostgreSQL configurations get introduced, SELinux booleans will be used to adapt the behaviour of the “targeted” policy. Such modifications for example are required whenever files labelled with a PostgreSQL-specific type are relocated in the filesystem. 

Sepgsql allows labelling of tables and functions by SELinux and provides another layer of access control. The use of sepgsql is in scope of another project exploring high security configurations. Setting it up here would enforce it in all higher-level projects building on this one – which is not the point. As with parts of the Networking Configuration it is mentioned here for completeness. 


Security Footprint (“targeted” policy) 
FILESYSTEM 
/usr/pgsql-<version> has the context: 
system_u:object_r:usr_t:s0 
/var/lib/pgsql/ is owned by the “postgres” user and has the context: 
system_u:object_r:postgresql_db_t:s0 

SERVER PROCESS 
The “postmaster” runs with the context: 
system_u:system_r:unconfined_service_t:s0 
which might be a problem. To be determined. 

CLIENT APPLICATIONS 
psql is classified by SELinux “targeted” policy as: 
system_u:object_r:bin_t:s0 
in its actual filesystem location (/usr/pgsql-<version>/bin/). 
 

Logging Considerations 
The interoperation settings between journald and rsyslog are left with the defaults found in: 
/etc/systemd/journald.conf 

e.g., ForwardToSyslog=no by default on RHEL & derivatives. In general, it means that rsyslog uses the journald API directly whenever it needs something only journald has access to. 
As Storage=auto is the default, the easiest way to save the journald logs between reboots is to create a directory: 
/var/log/journal 
, which is done by the Ansible Playbook. 

The default logrotate configuration would only be changed if a particular experiment conducted in this environment would cause /var/log to fill up. 
When it comes to PostgreSQL-specific logging the default output method is the stderr of the main process - “postgres” or “postmaster” depending on the distribution. With additional configuration, the logs can be output using two other methods: csvlog, syslog. An example configuration would be: 

Inside /var/lib/pgsql/<version>/data/postgresql.conf 
log_destination=syslog 

There is also an option of using pgBadger – a tool designed specifically to work with PostgreSQL logs consisting of a Perl script and a JavaScript library. Could not find any information in its documentation regarding running it in a client-server mode on two separate machines, so for now it’s not deemed a fit for a system with no GUI. 
As was the case with parts of the Networking Configuration and Security Considerations, after the redesign, any particular PostgreSQL server configuration is in the scope of another project, and only mentioned here for completeness. 
Shipping logs to “Linux Guest 1” aka the “management machine” was considered, but the two machines are ON at the same time only under very specific circumstances. Log shipping would make more sense in a deployment where the target machine is constantly ON. 
 

Monitoring Considerations 
Using a real-time (e.g., Nagios) or time-series (e.g., Prometheus) monitoring database, although an informative experiment would be an extreme overhead for what this project is supposed to accomplish. 
Instead, a “toolbox” of shell tools will be used from time to time to check the pulse of the system. 
logwatch, sar, uptime, df, du, free, iostat, lsof, mpstat, vmstat 
 

Shortcomings 
While this project uses the “Infrastructure as Code” paradigm, it is not managed as “Immutable Infrastructure”. As such, any setting/package not explicitly declared in the automation files might introduce a configuration drift as even defaults might change. 


Automation (Infrastructure as Code) 
Kickstart file, Ansible Playbook 


Appendix A. Kickstart Installation 
1) preparing the “Kickstart Drive”: 
A separate VHD has to be created and attached to the manually installed VM (Linux Guest 1). 
It should then be formatted with xfs, labeled “OEMDRV” and mounted. 
Note that the Kickstart file available in the repository requires modification before use. Any <placeholders> should be replaced, and the file has to be renamed “ks.cfg”. 
The prepared ks.cfg file should be copied to the root of the OEMDRV-labelled filesystem. 
The drive should be unmounted. After shutting down the manually installed VM, the drive should be disconnected in Hyper-V. 
With that the "Kickstart Drive” is ready for use. 

2) automating the system installation on any experimental VM 
The “Kickstart Drive” should be attached to the VM as a secondary drive. 
A DVD Drive containing the .iso installer image should be attached as well. 
After starting the VM, the installation should complete unattended. 
