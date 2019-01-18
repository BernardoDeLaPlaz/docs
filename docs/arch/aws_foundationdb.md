## Metadata
```
  Title: A Tutorial to Spin Up Urbit in AWS (with FoundationDB)
  Author: ~rigdyn-sondur
  Status: 
  Created: ~2018.12.13
  Last Modified: ~2018.12.14
```

## Overview

AWS EC2 is a market-dominant cloud provider with good prices and
geographically distributed data centers.  Note that AWS (Amazon Web
Services) provides many services. EC2 (Elastic Compute Cloud) is the
one that provides servers.

FoundationDB is a newly supported persistence engine for urbit that
makes use of a database distributed across multiple machines.

Together, they provide a solution to let Tlon host Urbit-as-a-service
for customers and make guarantees about persistence and reliability
(backed by Amazon).

This document is a list of steps that, mechanically followed, will
result in a post-condition of one urbit process using a "cluster"
(FoundationDB terminology) of three database servers to store events,
configured in such a way that if even two of the three database
servers fail, the urbit process loses no data.

## Replication Modes

FoundationDB offers multiple replication modes: "single", "double",
"triple", "three_data_hall", and "three_datacenter" mode.

Naively, "single" mode sounds like what we want: the docs tell us that
only one machine need be up in order to make progress.  The fdbcli
admin tool bolsters this impression, as typing "status" when
interacting with a cluster of three machines in "single" mode results
in "fault tolerance: 2 machines".

However, both testing and re-reading the FoundationDB docs make it
clear that single mode is appropriate only for single developer
testing, and not even for the lightest of production purposes.  In
practice, data is not replicated, and shutting down even one server in
a cluster of three disables the cluster.

"Double" mode allows the database to stay up and continue providing
services if at least two servers in a cluster are alive.

This is the replication mode proposed for use, and tested with, at the
time that this document was written.

## Performance

Using these steps I've demonstrated a configuration that can:

* inject events from a desktop-located urbit across the wire into an AWS data
  center and have the events persisted in a mean of 40ms.

* inject events from an AWS-located urbit on instance U into a colocated
  foundationDB cluster of instances A, B, C in "double ssd" mode and have them persisted
  in a mean time of 7 ms

* inject events from an AWS-located urbit on instance A into a colocated
  foundationDB cluster of instances A, B, C in "double ssd" mode (where the urbit is on the
  same machine as one of the foundationDB servers) and have them
  persisted in a mean time of 7ms.

* inject events from an AWS-located urbit into a colocated
  foundationDB cluster in "single memory" mode and have them persisted
  in a mean time of 4ms. N.B. As single mode provides no replication
  guarantees, this is not useful.

Caveat: this is in "single" mode, which means that the database
cluster is not bound to distribute the event across multiple servers
before reporting it as stored.  More performance characterization is
required.


## Topics for Future Elaboration

This document does not touch on several things that are worthy of interest:

* more details of FoundationDB configuration files
* FoundationDB datacenter identifier keys / locality codes
* FoundationDB satellite redundancy modes
* AWS "Elastic IP", which creates persistent IP addresses and allows
	  them to be reused by various instances.  This service
	  particularly interesting to us as the FoundationDB requires that
	  clusters be specified by IP address, not by DNS address, and AWS
	  EC2 instances can and do change IP addresses when stopped and
	  started.
* other AWS options and services
* security concerns (this recipe opens up all ports in both directions)

## Recipe

### Overview

This recipe will walk you through:

1. spinning up four AWS EC2 "instances" (servers): one for urbit to run on, four for FoundationDB to run on
1. creating a "security group" (policy) that allows them to communicate with each other, and with the outside world
1. associating all the instances with that security group
1. installing necessary software on the instances
1. configuring foundationDB on the instances
1. verifying correctness and getting speed statistics

This recipe also contains a variety of debugging hints, mapping errors to remedial steps.

### Details

1. get to EC2 console
    1. go to https://console.aws.amazon.com/ec2/v2/
	1. select region via pull down in upper-right-hand corner.  This region will persist for all EC2
1. create a security group
    1. click left sidebar "network & security " -> security groups
    1. click "create security group"
    1. in "security group name" field, type "foundationDB"
    1. select "inbound" tab
        1. click "add rule"; type = all / protocol = all / port = 0-65535 / source = "anywhere"
        1. click "add rule"; type = all / protocol = all / port = 0-65535 / source = "my IP"
    1. select "outbound" tab
        1. verify "all traffic" allowed
    1. click "create"
1. create an instance for urbit to run on
    1. click left sidebar "instances"
    1. click "launch instance"
    1. pick "Ubuntu Server 18.04 (x86, 64 bit)"
    1. pick size. N.B.  t2.micro (1 GB RAM) works, but is too small to compile urbit on ; t2.small (2 GB RAM) allows compiles in situ
    1. review and launch"
	1. on the console web page, mouse over the "name" field of the new instance. Click the pencil icon. Type "U" and hit return.
    1. save the PEM file to your local machine
    1. on your local machine `chmod 600` the PEM file
1. create three new instances for FoundationDB to run on
    1. as above, except ...
	    1. do not bother to generate a PEM file or save it; re-use the existing PEM file
        1. name the four new servers A, B, C
1. associate all four instance with security group ( repeat this for each instance )
    1. click left sidebar "instances"
    1. click chicklet next to server
    1. click actions -> networking -> change security groups
    1. click check box next to "foundationDB"
    1. click "assign security groups"
1. configure the three FoundationDB servers A, B, and C (repeat this for each instance )
    1. click a single radio button next to 1 server (just one, not multiple), then click "connect" at top of screen
    1. copy the 'ssh' text from pop-up, paste it into a terminal
    1. once ssh-ed in:
>	   		# install foundationDB
>			#
>	        wget https://www.foundationdb.org/downloads/6.0.15/ubuntu/installers/foundationdb-clients_6.0.15-1_amd64.deb
>			wget https://www.foundationdb.org/downloads/6.0.15/ubuntu/installers/foundationdb-server_6.0.15-1_amd64.deb
>			sudo apt-get --fix-broken install
>			sudo apt-get -y install python
>			sudo dpkg --install foundationdb-clients*
>			sudo dpkg --install  foundationdb-server*
>
>			# allow the configuration file to be edited by the admin tool
>			#
>			sudo usermod -G foundationdb ubuntu
>			sudo chmod 666 /etc/foundationdb/fdb.cluster
>
>
>			# allow the FoundationDB server to receive external connections
>			#
>			sudo /usr/lib/foundationdb/make_public.py
>
>			# N.B. error message "ERROR: '/etc/foundationdb/fdb.cluster' is not a valid cluster file"
>			# may be caused by wrong unix file permissions on the file /etc/foundation/fdb.cluster
>
>			# On the first server, look at  file
>			#     /etc/foundationdb/fdb.cluster.
>			# Note the cluster ID, to the left of the '@' sign, and copy it.
>			# Do nothing else.
>			
>			# On all the other servers in the cluster use 'nano' to edit the file
>			#     /etc/foundationdb/fdb.cluster
>			# replace everything to the left of
>			# '@' with the two part cluster identifier from the first server
>
>	  		sudo /etc/init.d/foundationdb restart
>
>			# start the FoundationDB admin tool
>			fdbcli
>
>			    # expect to see "using cluster file /etc/foundationdb/fdb.cluster"
>				# if you see "using fdb.cluster", you forgot to delete the file in your home directory
>				
>			# Create a new database.  
>			#    Note that 'double' specifies replication mode. 
>           #    Note that 'ssd' specifies storage mode.
>			# 
>			configure new double ssd
>
>			 	# expect to see "Database created"
>
>			# verify success of above step
>			# 
>			status
>
>			# expect to see
>			#            Using cluster file `/etc/foundationdb/fdb.cluster'.
>			#            Configuration:
>			#              Redundancy mode        - double
>			#              Storage engine         - ssd-2
>			# 
>
>			exit   
1. install urbit on AWS machine U.
    * There are two approaches, A and B.
    * approach A (compile in AWS)
        1. ssh to server U
        1. on server U
>  		# on server
>       #   (these steps are branched from the instructions found at https://github.com/urbit/docs/blob/master/docs/using/install.md and contain additional steps and apt-get packages ).  
>       #
>		sudo apt-get update
>		sudo apt-get install --yes autoconf automake cmake exuberant-ctags g++ git libcurl4-gnutls-dev libgmp3-dev libncurses5-dev libsigsegv-dev libssl-dev libtool make openssl pkg-config python python3 python3-pip ragel re2c zlib1g-dev liblmdb-dev sqlite3 libsqlite3-dev  libsnappy-dev libbz2-dev
>
>		sudo -H pip3 install --upgrade pip
>		sudo -H pip3 install meson==0.47
>
>		wget https://www.foundationdb.org/downloads/6.0.15/ubuntu/installers/foundationdb-clients_6.0.15-1_amd64.deb
>		wget https://www.foundationdb.org/downloads/6.0.15/ubuntu/installers/foundationdb-server_6.0.15-1_amd64.deb
>		sudo dpkg --install foundationdb-clients*
>		sudo dpkg --install  foundationdb-server*
>
>		git clone git://github.com/ninja-build/ninja.git
>		pushd ninja
>			git checkout release
>			./configure.py --bootstrap
>			sudo cp ./ninja /usr/local/bin
>		popd
>
>		git clone https://github.com/BernardoDeLaPlaz/urbit
>		cd urbit
>		git remote rename origin bdlp
>		git fetch bdlp cc-release
>		git checkout cc-release
>		scripts/bootstrap
>		scripts/build
>  
>       # verify that bin/urbit and bin/perstest (persistence test tool) exist
    * approach B (compile on desktop, scp binary)
        1. compile urbit on your desktop
        1. all of the above steps in approach A, up to the second "sudo dpkg..."
        1. scp the urbit binaries bin/urbit and bin/perstest from a desktop to machine U
1. on machine U, configure things so that clients (such as urbit) speak to a cluster of databases on machines A, B, C
    1. go to the AWS EC2 console, click to the instances page.
    1. look at the column "IPv4 Public IP".  Note the IP addresses listed there.
    1. ssh to machine U, then execute
>	  fdbcli
>	  coordinators <IP-A>:4500 <IP-B>:4500 <IP-B>:4500 
>	  coordinators  # verify above list
>     status
>	  exit
1. verify the cluster exists, and is reachable from machine A.  In case of any errors, see "Debugging" below.
    1.	   ssh to machine U, then execute
>	   fdcli
>	   status
>
>	   writemode on
>	   set 1 10
>	   set 2 20
>
>	   get 1    # expect to see "10"
>	   get 2    # expect to see "20"
>
>	   exit
1. verify the cluster is truly a cluster and replication is working
>	   ssh to machine A
>
>	   fdcli
>	   writemode on
>	   set 3 30
>	   exit
> 
>      sudo /etc/init.d/foundationdb stop
>
>	   fdcli
>	   get 3  # expect 30
>	   exit
>
1. verify the cluster can handle 3-of-4 outrages
>	   ssh to machine B
> 
>      sudo /etc/init.d/foundationdb stop
>
>	   fdcli
>	   get 3 # expect 30
>	   exit
>
1. generating speed specs
    1. build urbit on machine A, as above
	1. as a side effect, bin/perstest is generated
	1. run
>	bin perstest -n 10 -p f

## Debugging "write on A, shutdown A, can't read on A" errors
1. cluster set up incorrect?  on each machine:
    1. sudo /etc/init.d/foundationdb stop
    1. sudo rm -rf /var/lib/foundationdb/data/*
    1. copy /etc/foundationdb/fdb.cluster file from A to B & C
	1. sudo /etc/init.d/foundationdb start
    1. 
> fdbcli
> 
>
    1. redo test

## Debugging "write on A, can't read on B" errors

as above


## Debugging "can't connect" errors
1. forgot to run 'make_public.py' on server ?
>	  		on server, investigate via:
>			    cat /etc/foundationdb/fdb.cluster
>				# expect to see the local IP address (e.g. 172.31.39.25) , NOT localhost address (127.0.0.1)
>	  		on server, fix via:
>			    sudo /usr/lib/foundationdb/make_public.py
1. one of the coordinator servers has its own database ?
>	  	 	 on server, investigate via:
>			 	fdbci
>				status
>				# look for error message : 1 client(s) reported: Cluster file contents do not match current cluster connection string. Verify the cluster file and its parent directory are writable and that the cluster file has not been overwritten externally.
>
>	  	 	 on server, fix via:
>				 sudo /etc/init.d/foundationdb stop
>				 sudo rm -rf /var/lib/foundationdb/data/
>				 sudo /etc/init.d/foundationdb start
1. you are using the local IP address of server instead of external IP address in the "coordinators" command ?
>	  	 	   on each server: 
>			   	  ifconfig -a
>			   on client (machine A):
>			      fdbcli
>			   	  re-run "coordinators" command, using the correct IP

## Debugging "things are really fubared, and I want to start over" issues

Sometimes FoundationDB gets really fubared. (e.g. shutting down one server in a cluster in 'single' mode seems to contaminate everything, and the only solution is to burn it with fire and start over.)

The good news is that it's very easy to wipe FoundationDB from a machine and start over.

1. purge
>	   		ssh to a machine
>	   		sudo dpkg --purge  foundationdb-server
>	   		sudo dpkg --purge  foundationdb-clients
>	   		sudo rm -rf /variable/lib/foundationdb /var/log/foundationdb /etc/foundationdb/fdb.cluster
1. reinstall
>	   		ssh to a machine
>	   		# no need to do 'wget' because the .deb files are already here
>	   		sudo dpkg --install foundationdb-clients*
>	   		sudo dpkg --install  foundationdb-server*
> 			sudo chmod 666 /etc/foundationdb/fdb.cluster   



## References

* https://apple.github.io/foundationdb/administration.html
* https://apple.github.io/foundationdb/configuration.html
