# quay-server-3.10
Private Image Registry Server using RedHat Quay 3.10

### RED HAT QUAY FEATURES    

Red Hat Quay is regularly released with new features and software updates. The following features are
available for Red Hat Quay deployments, however the list is not exhaustive:

- High availability
- Geo-replication
- Repository mirroring
- Docker v2, schema 2 (multi-arch) support
- Continuous integration
- Security scanning with Clair
- Custom log rotation
- Zero downtime garbage collection
- 24/7 support

### ARCHITECTURE
Red Hat Quay includes several core components, both internal and external.
For a fuller architectural breakdown, see the Red Hat Quay architecture guide.

### Internal components

Red Hat Quay includes the following internal components:

- **Quay (container registry)**. Runs the **Quay** container as a service, consisting of several
components in the pod.

- **Clair**. Scans container images for vulnerabilities and suggests fixes.

### External components

Red Hat Quay includes the following external components:

- **Database**. Used by Red Hat Quay as its primary metadata storage. Note that this is not for
image storage.

- **Redis (key-value store)**. Stores live builder logs and the Red Hat Quay tutorial. Also includes
the locking mechanism that is required for garbage collection.

# SETTING UP THE HAPROXY LOAD BALANCER AND THE POSTGRESQL DATABASE

### Prerequisites

You have installed the Podman or Docker CLI.

### Procedure

On the first system q01 then install the HAProxy load balancer and the PostgreSQL database. This configures HAProxy as the access point and load balancer for the following services running on other systems:

- Red Hat Quay (ports 80 and 443)
- Redis (port 6379 )

Open all HAProxy ports in SELinux and selected HAProxy ports in the firewall:

    # setsebool -P haproxy_connect_any=on
    # firewall-cmd --permanent --zone=public --add-port=6379/tcp --add-port=7480/tcp
    success
    # firewall-cmd --reload
    success
    
Configure the **/etc/haproxy/haproxy.cfg** to point to the systems and ports providing the Red Hat Quay, Redis and Ceph RADOS services. The following are examples of defaults and added frontend and backend settings:
    
    # Global settings
    #---------------------------------------------------------------------
    global
        maxconn     20000
        log         /dev/log local0 info
        chroot      /var/lib/haproxy
        pidfile     /var/run/haproxy.pid
        user        haproxy
        group       haproxy
        daemon
    
        # turn on stats unix socket
        stats socket /var/lib/haproxy/stats
    
    #---------------------------------------------------------------------
    # common defaults that all the 'listen' and 'backend' sections will
    # use if not designated in their block
    #---------------------------------------------------------------------
    defaults
        log                     global
        mode                    http
        option                  httplog
        option                  dontlognull
        option http-server-close
        option redispatch
        option forwardfor       except 127.0.0.0/8
        retries                 3
        maxconn                 20000
        timeout http-request    10000ms
        timeout http-keep-alive 10000ms
        timeout check           10000ms
        timeout connect         40000ms
        timeout client          300000ms
        timeout server          300000ms
        timeout queue           50000ms
    
    # Enable HAProxy stats
    listen stats
        bind :9000
        stats uri /stats
        stats refresh 10000ms
    
    # OCP Ingress - layer 4 tcp mode for each. Ingress Controller will handle layer 7.
    frontend ocp_http_ingress_frontend
        bind :80
        default_backend ocp_http_ingress_backend
        mode tcp
    
    backend ocp_http_ingress_backend
        balance source
        mode tcp
        server      quay01 192.168.0.137:80 check
        server      quay02 192.168.0.138:80 check
        server      quay03 192.168.0.139:80 check
    
    frontend ocp_https_ingress_frontend
        bind *:443
        default_backend ocp_https_ingress_backend
        mode tcp
    
    backend ocp_https_ingress_backend
        mode tcp
        balance source
        server      quay01 192.168.0.137:443 check
        server      quay02 192.168.0.138:443 check
        server      quay03 192.168.0.139:443 check
    
    frontend ocp_redis_frontend 
        bind *:6379
        default_backend ocp_redis_backend
        mode tcp
    backend ocp_redis_backend
        server quay01 192.168.0.137:6379 check inter 1s
        server quay02 192.168.0.138:6379 check inter 1s
        server quay03 192.168.0.139:6379 check inter 1s
  
After the new haproxy.cfg file is in place, restart the HAProxy service by entering the following command:
    
    [root@quay01 ~]# systemctl restart haproxy  

Create a folder for the PostgreSQL database by entering the following command:

    [root@quay01 ~]# mkdir -p /var/lib/pgsql/data

Set the following permissions for the /var/lib/pgsql/data folder:

    [root@quay01 ~]# chmod 777 /var/lib/pgsql/data

Enter the following command to start the PostgreSQL database:

    [root@quay01 ~]# vim ./postgresql.sh
    sudo podman run -d --rm --name postgresql-quay \
     -e POSTGRESQL_USER=quayuser \
     -e POSTGRESQL_PASSWORD=quaypass \
     -e POSTGRESQL_DATABASE=quay \
     -e POSTGRESQL_ADMIN_PASSWORD=adminpass \
     -p 5432:5432 \
     -v /mnt/quay/postgres-quay:/var/lib/pgsql/data:Z \
     registry.redhat.io/rhel8/postgresql-10:1  
    
> **[!WARNING]**
> Data from the container will be stored on the host system in the /var/lib/pgsql/data directory.

    [root@quay01 ~]# sh ./postgresl.sh 

    [root@quay01 ~]# podman ps
    CONTAINER ID  IMAGE                                       COMMAND         CREATED      STATUS      PORTS                                        NAMES
    6a35beff18c4  registry.redhat.io/rhel8/postgresql-10:1    run-postgresql  3 hours ago  Up 3 hours  0.0.0.0:5432->5432/tcp                       postgresql-quay

List the available extensions by entering the following command:
        
    [root@quay01 ~]# sudo podman exec -it postgresql_database /bin/bash -c 'echo "SELECT * FROM pg_available_extensions" | psql'
            name        | default_version | installed_version |                                comment                                 
    --------------------+-----------------+-------------------+------------------------------------------------------------------------
     adminpack          | 2.1             |                   | administrative functions for PostgreSQL
     amcheck            | 1.2             |                   | functions for verifying relation integrity
     autoinc            | 1.0             |                   | functions for autoincrementing fields
     bloom              | 1.0             |                   | bloom access method - signature file based index
     btree_gin          | 1.3             |                   | support for indexing common datatypes in GIN
     btree_gist         | 1.5             |                   | support for indexing common datatypes in GiST
     citext             | 1.6             |                   | data type for case-insensitive character strings
     cube               | 1.4             |                   | data type for multidimensional cubes
     dblink             | 1.2             |                   | connect to other PostgreSQL databases from within a database
     dict_int           | 1.0             |                   | text search dictionary template for integers
     dict_xsyn          | 1.0             |                   | text search dictionary template for extended synonym processing
     earthdistance      | 1.1             |                   | calculate great-circle distances on the surface of the Earth
     file_fdw           | 1.0             |                   | foreign-data wrapper for flat file access
     fuzzystrmatch      | 1.1             |                   | determine similarities and distance between strings
     hstore             | 1.7             |                   | data type for storing sets of (key, value) pairs
     hstore_plperl      | 1.0             |                   | transform between hstore and plperl
     hstore_plperlu     | 1.0             |                   | transform between hstore and plperlu
     hstore_plpython2u  | 1.0             |                   | transform between hstore and plpython2u
     hstore_plpython3u  | 1.0             |                   | transform between hstore and plpython3u
     hstore_plpythonu   | 1.0             |                   | transform between hstore and plpythonu
     insert_username    | 1.0             |                   | functions for tracking who changed a table
     intagg             | 1.1             |                   | integer aggregator and enumerator (obsolete)
     intarray           | 1.3             |                   | functions, operators, and index support for 1-D arrays of integers
     isn                | 1.2             |                   | data types for international product numbering standards
     jsonb_plperl       | 1.0             |                   | transform between jsonb and plperl
     jsonb_plperlu      | 1.0             |                   | transform between jsonb and plperlu
     jsonb_plpython2u   | 1.0             |                   | transform between jsonb and plpython2u
     jsonb_plpython3u   | 1.0             |                   | transform between jsonb and plpython3u
     jsonb_plpythonu    | 1.0             |                   | transform between jsonb and plpythonu
     lo                 | 1.1             |                   | Large Object maintenance
     ltree              | 1.2             |                   | data type for hierarchical tree-like structures
     ltree_plpython2u   | 1.0             |                   | transform between ltree and plpython2u
     ltree_plpython3u   | 1.0             |                   | transform between ltree and plpython3u
     ltree_plpythonu    | 1.0             |                   | transform between ltree and plpythonu
     moddatetime        | 1.0             |                   | functions for tracking last modification time
     pageinspect        | 1.8             |                   | inspect the contents of database pages at a low level
     pg_buffercache     | 1.3             |                   | examine the shared buffer cache
     pg_freespacemap    | 1.2             |                   | examine the free space map (FSM)
     pg_prewarm         | 1.2             |                   | prewarm relation data
     pg_stat_statements | 1.8             |                   | track planning and execution statistics of all SQL statements executed
     pg_trgm            | 1.5             |                   | text similarity measurement and index searching based on trigrams
     pg_visibility      | 1.2             |                   | examine the visibility map (VM) and page-level visibility info
     pgaudit            | 1.5             |                   | provides auditing functionality
     pgcrypto           | 1.3             |                   | cryptographic functions
     pgrowlocks         | 1.2             |                   | show row-level locking information
     pgstattuple        | 1.5             |                   | show tuple-level statistics
     plpgsql            | 1.0             | 1.0               | PL/pgSQL procedural language
     postgres_fdw       | 1.0             |                   | foreign-data wrapper for remote PostgreSQL servers
     refint             | 1.0             |                   | functions for implementing referential integrity (obsolete)
     seg                | 1.3             |                   | data type for representing line segments or floating-point intervals
     sslinfo            | 1.2             |                   | information about SSL certificates
     tablefunc          | 1.0             |                   | functions that manipulate whole tables, including crosstab
     tcn                | 1.0             |                   | Triggered change notifications
     tsm_system_rows    | 1.0             |                   | TABLESAMPLE method which accepts number of rows as a limit
     tsm_system_time    | 1.0             |                   | TABLESAMPLE method which accepts time in milliseconds as a limit
     unaccent           | 1.1             |                   | text search dictionary that removes accents
     uuid-ossp          | 1.1             |                   | generate universally unique identifiers (UUIDs)
     xml2               | 1.1             |                   | XPath querying and XSLT
    (58 rows)
    
Create the pg_trgm extension by entering the following command:   
    
    [root@quay01 ~]# podman exec -it postgresql_database /bin/bash -c 'echo "CREATE EXTENSION IF NOT EXISTS pg_trgm;" | psql -d quaydb'
    CREATE EXTENSION
  
Confirm that the **pg_trgm** has been created by entering the following command:

    [root@quay01 ~]# sudo podman exec -it postgresql_database /bin/bash -c 'echo "SELECT * FROM pg_extension" |psql'
      oid  | extname | extowner | extnamespace | extrelocatable | extversion | extconfig | extcondition 
    -------+---------+----------+--------------+----------------+------------+-----------+--------------
     13422 | plpgsql |       10 |           11 | f              | 1.0        |           | 
    (1 row)

Alter the privileges of the Postgres user quayuser and grant them the superuser role to give the user unrestricted access to the database:
    
    [root@quay01 ~]# sudo podman exec -it postgresql_database /bin/bash -c 'echo "ALTER USER quayuser WITH SUPERUSER;" | psql'
    ALTER ROLE
        
If you have a firewalld service active on your system, run the following commands to make the PostgreSQL port available through the firewall:
    
    [root@quay01 ~]# firewall-cmd --permanent --zone=trusted --add-port=5432/tcp
    success
    [root@quay01 ~]# firewall-cmd --reload
    success
    
Optional. If you do not have the postgres CLI package installed, install it by entering the following command:
    
    [root@quay01 ~]# yum install postgresql -y

Use the psql command to test connectivity to the PostgreSQL database.

    [root@quay01 ~]# psql -h localhost quaydb quayuser
    Password for user quayuser: 
    psql (13.14, server 13.7)
    Type "help" for help.
    quaydb=# \q


## SET UP REDIS

With Red Hat Enterprise Linux 8 server installed on each of the three Red Hat Quay systems (quay01, quay02, and quay03), install and start the Redis service as follows:

Install / Deploy Redis: Run Redis as a container on each of the three quay0* systems:

    [root@quay01 ~]# mkdir -p /var/lib/redis
    [root@quay01 ~]# chmod 777 /var/lib/redis
    
    [root@quay01 ~]# cat redis.sh
    podman run -d --rm --name redis   -p 6379:6379   -e REDIS_PASSWORD=redhXXXX   registry.redhat.io/rhel8/redis-5:1
    
    [root@quay01 ~]# sh redis.sh

    [root@quay01 ~]# podman ps
    CONTAINER ID  IMAGE                                       COMMAND         CREATED      STATUS      PORTS                                        NAMES
    6a35beff18c4  registry.redhat.io/rhel8/postgresql-10:1    run-postgresql  3 hours ago  Up 3 hours  0.0.0.0:5432->5432/tcp                       postgresql-quay
    59d942b4eead  registry.redhat.io/rhel8/redis-5:1          run-redis       3 hours ago  Up 3 hours  0.0.0.0:6379->6379/tcp                       redis
    
Check redis connectivity: You can use the telnet command to test connectivity to the redis service. Type MONITOR (to begin monitoring the service) and QUIT to exit:

    [root@quay01 ~]# telnet `hostname` 6379
    Trying 192.168.0.139...
    Connected to quay03.lab.example.com.
    Escape character is '^]'.
    MONITOR
    +OK
    PING
    +1712078306.306533 [0 192.168.0.139:47148] "PING"
    +PONG
    QUIT
    +OK
    Connection closed by foreign host.

## CONFIGURING RED HAT QUAY

Before running the Red Hat Quay service as a container, you need to use that same Quay container to create the configuration file (config.yaml) needed to deploy Red Hat Quay. To do that, you pass a config argument and a password (replace my-secret-password here) to the Quay container. Later, you use that password to log into the configuration tool as the user quayconfig.

Start quay in setup mode : On the first quay node, run the following:
    
    [root@quay01 ~]# cat quay.sh
    podman run --rm -it --name quay -p 80:8080 -p 443:8443 registry.redhat.io/quay/quay-rhel8:v3.10.3 config redhat
    
    [root@quay01 ~]# sh quay.sh
    
    [root@quay01 ~]# podman ps
    CONTAINER ID  IMAGE                                       COMMAND         CREATED      STATUS      PORTS                                        NAMES
    6a35beff18c4  registry.redhat.io/rhel8/postgresql-10:1    run-postgresql  3 hours ago  Up 3 hours  0.0.0.0:5432->5432/tcp                       postgresql-quay
    59d942b4eead  registry.redhat.io/rhel8/redis-5:1          run-redis       3 hours ago  Up 3 hours  0.0.0.0:6379->6379/tcp                       redis
    917eb41c81ea  registry.redhat.io/quay/quay-rhel8:v3.10.3  registry        3 hours ago  Up 3 hours  0.0.0.0:80->8080/tcp, 0.0.0.0:443->8443/tcp  quay

Open browser: When the quay configuration tool starts up, open a browser to the URL and port 8080 of the system you are running the configuration tool on (for example http://quay01.lab.example.com:8080). You are prompted for a username and password.
    

**Create directories:** Create two directories to store configuration information and data on the host. 

    [root@quay01 ~]# mkdir /mnt/quay/config -p

**Copy config files:** Copy the tarball (quay-config.tar.gz) to the configuration directory and unpack it.  

    [root@quay01 ~]# cp quay-config.tar.gz /mnt/quay/config/
    [root@quay01 ~]# cd /mnt/quay/config/
    [root@quay01 ~]# tar xvf quay-config.tar.gz   
    extra_ca_certs/
    config.yaml

Deploy Red Hat Quay : Having already authenticated to Quay.io (see Accessing Red Hat Quay ) run Red Hat Quay as a container

    [root@quay01 ~]# cat quay-final.sh
    podman run --restart=always --name quay -p 443:8443 -p 80:8080 --sysctl net.core.somaxconn=4096 --privileged=true -v /mnt/quay/config:/conf/stack:Z -v /mnt/quay/storage:/datastorage:Z -d registry.redhat.io/quay/quay-rhel8:v3.10.3 
    
    [root@quay01 ~]# sh quay-final.sh

    [root@quay01 ~]# podman ps
    CONTAINER ID  IMAGE                                       COMMAND         CREATED      STATUS      PORTS                                        NAMES
    6a35beff18c4  registry.redhat.io/rhel8/postgresql-10:1    run-postgresql  3 hours ago  Up 3 hours  0.0.0.0:5432->5432/tcp                       postgresql-quay
    59d942b4eead  registry.redhat.io/rhel8/redis-5:1          run-redis       3 hours ago  Up 3 hours  0.0.0.0:6379->6379/tcp                       redis
    917eb41c81ea  registry.redhat.io/quay/quay-rhel8:v3.10.3  registry        3 hours ago  Up 3 hours  0.0.0.0:80->8080/tcp, 0.0.0.0:443->8443/tcp  quay
    
**Open browser to UI:** Once the **Quay** container has started, go to your web browser and open the URL, to the node running the **Quay** container.

**Log into Red Hat Quay :** Using the **superuser** account you created during configuration, log in and make sure Red Hat Quay is working properly.
    
    [root@quay01 ~]# podman login http://quay01.lab.example.com --tls-verify=false --username quayadmin 
    Password: Password
    Login Succeeded!

Pull busybox container image.
    
    [root@quay01 ~]# podman pull busybox
    Resolved "busybox" as an alias (/etc/containers/registries.conf.d/000-shortnames.conf)
    Trying to pull docker.io/library/busybox:latest...
    Getting image source signatures
    Copying blob 7b2699543f22 done  
    Copying config ba5dc23f65 done  
    Writing manifest to image destination
    ba5dc23f65d4cc4a4535bce55cf9e63b068eb02946e3422d3587e8ce803b6aab

Tagging busybox images to quayadmin/busybox. 
    
    [root@quay01 ~]# podman tag docker.io/library/busybox:latest quay01.lab.example.com/quayadmin/busybox:test

Push to local quay server.
    
    [root@quay01 ~]# podman push --tls-verify=false quay01.lab.example.com/quayadmin/busybox:test
    Getting image source signatures
    Copying blob 95c4a60383f7 done  
    Copying config ba5dc23f65 done  
    Writing manifest to image destination

Removing old image of busybox.
     
    [root@quay01 ~]# podman rmi quay01.lab.example.com/quayadmin/busybox:test
    Untagged: quay01.lab.example.com/quayadmin/busybox:test

Pulling busybox image from local quay.
    
    [root@quay01 ~]# podman pull --tls-verifby=false quay01.lab.example.com/quayadmin/busybox:test
    Trying to pull quay01.lab.example.com/quayadmin/busybox:test...
    Getting image source signatures
    Copying blob 39532de96124 skipped: already exists  
    Copying config ba5dc23f65 done  
    Writing manifest to image destination
    ba5dc23f65d4cc4a4535bce55cf9e63b068eb02946e3422d3587e8ce803b6aab

## Installing the oc-mirror OpenShift CLI plugin

### Prerequisites

- You have installed the OpenShift CLI (oc).

### Procedure

Download the oc-mirror CLI plugin.

- Navigate to the [**Downloads**](https://mirror.openshift.com/pub/openshift-v4/clients/ocp/4.15.0/oc-mirror.tar.gz) page of the [**OpenShift Cluster Manager**](https://mirror.openshift.com/pub/openshift-v4/clients/ocp/4.15.0/).
- Under the OpenShift disconnected installation tools section, click Download for OpenShift Client (oc) mirror plugin and save the file.

Extract the archive:
    
    [root@quay01 ~]# tar xvzf oc-mirror.tar.gz

If necessary, update the plugin file to be executable:

    [root@quay01 ~]# chmod +x oc-mirror

Install the oc-mirror CLI plugin by placing the file in your **PATH**, for example, **/usr/bin**:

    [root@quay01 ~]# sudo mv oc-mirror /usr/bin/.

Verification

Run **oc mirror help** to verify that the plugin was successfully installed:

    [root@quay01 ~]#  oc mirror help

## Configuring credentials that allow images to be mirrored

Complete the following steps on the installation host:

Download your [registry.redhat.io](https://console.redhat.com/openshift/downloads) pull secret from the Red Hat OpenShift Cluster Manager.

Make a copy of your pull secret in JSON format:

    [root@quay01 ~]# cat redhat-secret.json 
    {
            "auths": {
                    "cloud.openshift.com": {
                            "auth": "b3BlbT05XQg=="
                    },
                    "quay.io": {
                            "auth": "b3B="
                    },
                    "registry.connect.redhat.com": {
                            "auth": "fHVoYy"
                    },
                    "registry.redhat.io": {
                            "auth": "fHVoY"
                    }
            }
    }
    
Add local secret credential in redhat-secret.json file. 

    [root@quay01 ~]# podman login quay01.lab.example.com -u quayadmin -p  password  --tls-verify=false 
    Login Succeeded!
    
    [root@quay01 ~]# podman login quay01.lab.example.com -u quayadmin -p  password  --tls-verify=false --authfile redhat-secret.json
    
    [root@quay01 ~]# cat redhat-secret.json 
    {
            "auths": {
                    "cloud.openshift.com": {
                            "auth": "b3BlbT05XQg=="
                    },
                    "quay.io": {
                            "auth": "b3B="
                    },
                    "quay01.lab.example.com": {    <=== local quay
                            "auth": "cXVheb3Jk"    
                    },
                    "registry.connect.redhat.com": {
                            "auth": "fHVoYy"
                    },
                    "registry.redhat.io": {
                            "auth": "fHVoY"
                    }
            }
    }
    

Specify the path to the folder to store the pull secret in and a name for the JSON file that you create.
Save the file either as ~/.docker/config.json or $XDG_RUNTIME_DIR/containers/auth.json.

    [root@quay01 ~]# mkdir .docker 
    [root@quay01 ~]# cp redhat-secret.json > .docker/config.json
    [root@quay01 ~]# bash 

### Mirroring images for a disconnected installation using the oc-mirror plugin   
 
Create an image set configuration file.
   
    [root@quay01 ~]# cat imageset-config.yaml 
    kind: ImageSetConfiguration
    apiVersion: mirror.openshift.io/v1alpha2
    storageConfig:
      #registry:
      #  imageURL: quay01.lab.example.com/redhat:latest
      #  skipTLS: true
      local:
        path: web-terminal
    mirror:
      platform:
        channels:
        - name: stable-4.15
          type: ocp
      operators:
      - catalog: registry.redhat.io/redhat/redhat-operator-index:v4.15
        packages:
        - name: web-terminal
      additionalImages: []
      helm: {}

Mirror the image set to the mirror registry.
    
    [root@quay01 ~]# oc-mirror --from=web-terminal docker://quay01.lab.example.com/quayadmin/web-terminal  --dest-use-http 
    Checking push permissions for quay01.lab.example.com
    Publishing image set from archive "web-terminal" to registry "quay01.lab.example.com"
    quay01.lab.example.com/
      quayadmin/web-terminal/openshift/release
        blobs:
          file://openshift/release sha256:db7e80b18fa86515551aee66b3b70b4ffd6fa7487619a981abd37442f797a5da 23.09KiB
          file://openshift/release sha256:0bb5b0a96fc5b04a832b39611e0df013af3028093559a6ee5873909a693d52d2 3.643MiB
          file://openshift/release sha256:df57bd50942f2e10f1682dc353206a946076dc120d2f166ed8b9e1fd2632a7ac 9.822MiB
    ...
    Rendering catalog image "quay01.lab.example.com/quayadmin/web-terminal/redhat/redhat-operator-index:v4.15" with file-based catalog 
    Writing image mapping to oc-mirror-workspace/results-1712164792/mapping.txt
    Writing CatalogSource manifests to oc-mirror-workspace/results-1712164792
    Writing ICSP manifests to oc-mirror-workspace/results-1712164792                            
    
Configure your cluster to use the resources generated by the oc-mirror plugin.

    [root@quay01 ~]# cd oc-mirror-workspace/results-1712164792/
    
    [root@quay01 results-1712164792]# cat catalogSource-cs-redhat-operator-index.yaml 
    apiVersion: operators.coreos.com/v1alpha1
    kind: CatalogSource
    metadata:
      name: cs-redhat-operator-index
      namespace: openshift-marketplace
    spec:
      image: quay01.lab.example.com/quayadmin/web-terminal/redhat/redhat-operator-index:v4.15
      sourceType: grpc
    
    [root@quay01 results-1712164792]# cat imageContentSourcePolicy.yaml 
    ---
    apiVersion: operator.openshift.io/v1alpha1
    kind: ImageContentSourcePolicy
    metadata:
      labels:
        operators.openshift.org/catalog: "true"
      name: operator-0
    spec:
      repositoryDigestMirrors:
      - mirrors:
        - quay01.lab.example.com/quayadmin/web-terminal/devworkspace
        source: registry.redhat.io/devworkspace
      - mirrors:
        - quay01.lab.example.com/quayadmin/web-terminal/openshift4
        source: registry.redhat.io/openshift4
      - mirrors:
        - quay01.lab.example.com/quayadmin/web-terminal/web-terminal
        source: registry.redhat.io/web-terminal
      - mirrors:
        - quay01.lab.example.com/quayadmin/web-terminal/ubi8-minimal
        source: registry.redhat.io/ubi8-minimal
    ---
    apiVersion: operator.openshift.io/v1alpha1
    kind: ImageContentSourcePolicy
    metadata:
      name: release-0
    spec:
      repositoryDigestMirrors:
      - mirrors:
        - quay01.lab.example.com/quayadmin/web-terminal/openshift/release
        source: quay.io/openshift-release-dev/ocp-v4.0-art-dev
      - mirrors:
        - quay01.lab.example.com/quayadmin/web-terminal/openshift/release-images
        source: quay.io/openshift-release-dev/ocp-release
