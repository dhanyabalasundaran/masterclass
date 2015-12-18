# Environment notes

## Note to Hortonworkers

Find our Active Directory instance details in Google Drive. Title "infrastructure" in the Training folder.

## Accessing your Cluster

Credentials will be provided for these services:

* SSH
* Ambari

## Use your Cluster

### Configure name resolution & certificate to Active Directory

1. Add your Active Directory to /etc/hosts (if not in DNS)

   ```
echo "172.31.0.175 ad01.lab.hortonworks.net ad01" | sudo tee -a /etc/hosts
   ```

2. Add your CA certificate (if using self-signed & not already configured)

   ```
sudo yum -y install openldap-clients ca-certificates
sudo curl -sSL https://raw.githubusercontent.com/seanorama/masterclass/master/security-advanced/extras/ca.crt \
    -o /etc/pki/ca-trust/source/anchors/hortonworks-net.crt

sudo update-ca-trust force-enable
sudo update-ca-trust extract
sudo update-ca-trust check
   ```

3. Test certificate & name resolution with `ldapsearch`

   ```
## Update ldap.conf with our defaults
sudo tee -a /etc/openldap/ldap.conf > /dev/null << EOF
TLS_CACERT /etc/pki/tls/cert.pem
URI ldaps://ad01.lab.hortonworks.net ldap://ad01.lab.hortonworks.net
BASE dc=lab,dc=hortonworks,dc=net
EOF

## test with
ldapsearch -W -D hadoopadmin@lab.hortonworks.net

openssl s_client -connect ad01:636 </dev/null
   ```
4. (Optional) Install Logsearch 
- To deploy the Logsearch stack, run below
```
VERSION=`hdp-select status hadoop-client | sed 's/hadoop-client - \([0-9]\.[0-9]\).*/\1/'`
sudo git clone https://github.com/abajwa-hw/logsearch-service.git /var/lib/ambari-server/resources/stacks/HDP/$VERSION/services/LOGSEARCH
```

- Edit the `/var/lib/ambari-server/resources/stacks/HDP/$VERSION/role_command_order.json` file...
```
sudo vi /var/lib/ambari-server/resources/stacks/HDP/$VERSION/role_command_order.json
```
- ...by adding the below entries to the middle of the file
```
    "LOGSEARCH_SOLR-START" : ["ZOOKEEPER_SERVER-START"],
    "LOGSEARCH_MASTER-START": ["LOGSEARCH_SOLR-START"],
    "LOGSEARCH_LOGFEEDER-START": ["LOGSEARCH_SOLR-START", "LOGSEARCH_MASTER-START"],
```

- Restart Ambari
```
sudo service ambari-server restart
```
- Then you can click on 'Add Service' from the 'Actions' dropdown menu in the bottom left of the Ambari dashboard:
  - Note: on multinode clusters, on the screen where you configure which nodes services should go to, install Solr on all nodes by clicking the + icon
On bottom left -> Actions -> Add service -> check Logsearch service -> Next -> Next -> Next -> Deploy

- The SolrCloud console should be available at http://(yourhost):8886. Check that the hadoop_logs and history collections got created
- Launch the Logsearch webapp via navigating to http://(yourhost):8888/


## Active Directory environment
Enable kerberos using Ambari security wizard 

- KDC:
    - KDC host: ad01.lab.hortonworks.net
    - Realm name: LAB.HORTONWORKS.NET
    - LDAP url: ldaps://ad01.lab.hortonworks.net
    - Container DN: ou=HadoopServices,dc=lab,dc=hortonworks,dc=net
    - Domains: us-west-2.compute.internal,.us-west-2.compute.internal
- Kadmin:
    - Kadmin host: ad01.lab.hortonworks.net
    - Admin principal: hadoopadmin@lab.hortonworks.net
    - Admin password:

## Setup AD/OS integration via SSSD
- Run below on each node
```
# Pre-req: give registersssd user permissions to add the workstation to OU=HadoopNodes (needed to run 'adcli join' successfully)

ad_user="registersssd"
ad_domain="lab.hortonworks.net"
ad_dc="ad01.lab.hortonworks.net"
ad_root="dc=lab,dc=hortonworks,dc=net"
ad_ou="ou=HadoopNodes,${ad_root}"
ad_realm=${ad_domain^^}

sudo kinit ${ad_user}

sudo yum makecache fast
sudo yum -y -q install epel-release ## epel is required for adcli
sudo yum -y -q install sssd oddjob-mkhomedir authconfig sssd-krb5 sssd-ad sssd-tools libnss-sss libpam-sss 
sudo yum -y -q install adcli



sudo adcli join -v \
  --domain-controller=${ad_dc} \
  --domain-ou="${ad_ou}" \
  --login-ccache="/tmp/krb5cc_0" \
  --login-user="${ad_user}" \
  -v \
  --show-details

## todo:
##   pam, ssh, autofs should be disabled on master & data nodes
##   - we only need nss on those nodes
##   - edge nodes need the ability to login
sudo tee /etc/sssd/sssd.conf > /dev/null <<EOF
[sssd]
## master & data nodes only require nss. Edge nodes require pam.
services = nss, pam, ssh, autofs, pac
config_file_version = 2
domains = ${ad_realm}
override_space = _
[domain/${ad_realm}]
id_provider = ad
auth_provider = ad
chpass_provider = ad
#access_provider = ad
ad_server = ${ad_dc}
ldap_id_mapping = true
debug_level = 9
enumerate = true
override_homedir = /home/%d/%u
#ldap_schema = ad
#cache_credentials = true
#ldap_group_nesting_level = 5
ldap_tls_cacertdir = /etc/pki/tls/certs
ldap_tls_cacert = /etc/pki/tls/certs/ca-bundle.crt
ldap_tls_reqcert = never
[nss]
override_shell = /bin/bash
EOF
sudo chmod 0600 /etc/sssd/sssd.conf

sudo authconfig --enablesssd --enablesssdauth --enablemkhomedir --enablelocauthorize --update

sudo chkconfig oddjobd on
sudo service oddjobd restart
sudo chkconfig sssd on
sudo service sssd restart

sudo kdestroy
```
- Test your nodes can recognize AD users
```
id sales1
groups sales1
```
## Setup Ambari/AD sync

Run below on only Ambari node
1. Add your AD properties as defaults for Ambari LDAP sync  
  ```
ad_dc="ad01.lab.hortonworks.net"
ad_root="ou=CorpUsers,dc=lab,dc=hortonworks,dc=net"
ad_user="cn=ldapconnect,ou=ServiceUsers,dc=lab,dc=hortonworks,dc=net"

sudo tee -a /etc/ambari-server/conf/ambari.properties > /dev/null << EOF
authentication.ldap.baseDn=${ad_root}
authentication.ldap.managerDn=${ad_user}
authentication.ldap.primaryUrl=${ad_dc}:389
authentication.ldap.bindAnonymously=false
authentication.ldap.dnAttribute=distinguishedName
authentication.ldap.groupMembershipAttr=member
authentication.ldap.groupNamingAttr=cn
authentication.ldap.groupObjectClass=group
authentication.ldap.useSSL=false
authentication.ldap.userObjectClass=user
authentication.ldap.usernameAttribute=sAMAccountName
EOF

  ```
  
1. Run Ambari LDAP sync. Press enter to accept all defaults and enter password at the end
  ```
  sudo ambari-server setup-ldap
  ```

2. Reestart Ambari server and agents
  ```
   sudo ambari-server restart
   sudo ambari-agent restart
  ```
3. Run LDAP sync. When prompted for username/password enter admin/admin
  ```
  sudo ambari-server sync-ldap --all  
  ```

4. Now you should be able to login as AD users. Login as admin/BadPass#1 and give ambari user Admin priviledge via 'Manage Ambari'


## Day two

Agenda:

  - LDAP tool demo
  - Ranger pre-reqs
    - Install any missing services i.e. Kafka
    - Setup MySQL
    - Setup Solr for Ranger audits
  - Ranger install
    - Configure MySQL/Solr/HDFS audits
    - Configure user/group sync with AD
    - Configure plugins
    - Auth via AD
  - Ambari views setup on secure cluster
    - Kerberos for Ambari 
    - Files
    - Hive
    - Others (Jobs/Capacity Scheduler...)
  - Using Hadoop components in secured mode. Audit excercises for:
    - HDFS
    - Hive
    - Hbase
    - YARN
    - Storm
    - Kafka
    - Knox
      - AD integration
      - WebHDFS
      - Hive
  - Access webUIs via SPNEGO 
  - Manually setup Solr Ranger plugin(?)

## Ranger prereqs

###### Manually install missing components

- Use the 'Add Service' Wizard to install Kafka 


###### Create & confirm MySQL user 'root'

Prepare MySQL DB for Ranger use. Run these steps on MySQL 
- `sudo mysql`
- Execute following in the MySQL shell. Change the password to your preference. 

    ```sql
CREATE USER 'root'@'%';
GRANT ALL PRIVILEGES ON *.* to 'root'@'%' WITH GRANT OPTION;
SET PASSWORD FOR 'root'@'%' = PASSWORD('BadPass#1');
SET PASSWORD = PASSWORD('BadPass#1');
FLUSH PRIVILEGES;
exit
```

- Confirm MySQL user: `mysql -u root -h $(hostname -f) -p -e "select count(user) from mysql.user;"`
  - Output should be a simple count. Check the last step if there are errors.

###### Prepare Ambari for MySQL *(or the database you want to use)*
- Run this on Ambari node
- Add MySQL JAR to Ambari:
  - `sudo ambari-server setup --jdbc-db=mysql --jdbc-driver=/usr/share/java/mysql-connector-java.jar`
    - If the file is not present, it is available on RHEL/CentOS with: `sudo yum -y install mysql-connector-java`

###### install SolrCloud from HDPSearch for Audits

- install Solr from HDPSearch for Audits (steps are based on http://docs.hortonworks.com/HDPDocuments/HDP2/HDP-2.3.2/bk_Ranger_Install_Guide/content/solr_ranger_configure_standalone.html)

- Install Solr Cloud *on each node*. Note that Zookeeper must be running on nodes where this is setup
```
# change JAVA_HOME, SOLR_ZK and SOLR_RANGER_HOME as needed
export JAVA_HOME=/usr/java/default   
export host=$(curl -4 icanhazip.com)
sudo yum -y install lucidworks-hdpsearch
sudo wget https://issues.apache.org/jira/secure/attachment/12761323/solr_for_audit_setup_v3.tgz -O /usr/local/solr_for_audit_setup_v3.tgz
cd /usr/local
sudo tar xvf solr_for_audit_setup_v3.tgz
cd /usr/local/solr_for_audit_setup
sudo mv install.properties install.properties.org

sudo tee install.properties > /dev/null <<EOF
#!/bin/bash
JAVA_HOME=$JAVA_HOME
SOLR_USER=solr
SOLR_INSTALL=false
SOLR_INSTALL_FOLDER=/opt/lucidworks-hdpsearch/solr
SOLR_RANGER_HOME=/opt/ranger_audit_server
SOLR_RANGER_PORT=6083
SOLR_DEPLOYMENT=solrcloud
SOLR_ZK=localhost:2181/ranger_audits
SOLR_HOST_URL=http://$host:\${SOLR_RANGER_PORT}
SOLR_SHARDS=1
SOLR_REPLICATION=1
SOLR_LOG_FOLDER=/var/log/solr/ranger_audits
SOLR_MAX_MEM=1g
EOF
sudo ./setup.sh
sudo /opt/ranger_audit_server/scripts/add_ranger_audits_conf_to_zk.sh
sudo /opt/ranger_audit_server/scripts/start_solr.sh

sudo sed -i 's,^SOLR_HOST_URL=.*,SOLR_HOST_URL=http://localhost:6083,' \
   /opt/ranger_audit_server/scripts/create_ranger_audits_collection.sh
sudo /opt/ranger_audit_server/scripts/create_ranger_audits_collection.sh 
# access Solr webui at http://hostname:6083/solr
```

- optional - install banana dashboard
```
sudo wget https://raw.githubusercontent.com/abajwa-hw/security-workshops/master/scripts/default.json -O /opt/lucidworks-hdpsearch/solr/server/solr-webapp/webapp/banana/app/dashboards/default.json
export host=$(curl -4 icanhazip.com)
# replace host/port in this line::: "server": "http://sandbox.hortonworks.com:6083/solr/",
sudo sed -i "s,sandbox.hortonworks.com,$host," \
   /opt/lucidworks-hdpsearch/solr/server/solr-webapp/webapp/banana/app/dashboards/default.json
sudo chown solr:solr /opt/lucidworks-hdpsearch/solr/server/solr-webapp/webapp/banana/app/dashboards/default.json
# access banana dashboard at http://hostname:6083/solr/banana/index.html
```
- At this point you should be able to: 
  - access Solr webui at http://hostname:6083/solr
  - access banana dashboard at http://hostname:6083/solr/banana/index.html (if installed)


## Ranger install

###### Install Ranger via Ambari 2.1.3

- Install Ranger using Amabris 'Add Service' wizard on the same node as Mysql. Set the below configs for below tabs:

1. Ranger Admin tab
![Image](https://raw.githubusercontent.com/abajwa-hw/security-workshops/master/screenshots/ranger-213-setup/ranger-213-1.png)
![Image](https://raw.githubusercontent.com/abajwa-hw/security-workshops/master/screenshots/ranger-213-setup/ranger-213-2.png)

2. Ranger User info tab - Common configs subtab
![Image](https://raw.githubusercontent.com/abajwa-hw/security-workshops/master/screenshots/ranger-213-setup/ranger-213-3.png)
![Image](https://raw.githubusercontent.com/abajwa-hw/security-workshops/master/screenshots/ranger-213-setup/ranger-213-3.5.png)

3. Ranger User info tab - User configs subtab
![Image](https://raw.githubusercontent.com/abajwa-hw/security-workshops/master/screenshots/ranger-213-setup/ranger-213-4.png)
![Image](https://raw.githubusercontent.com/abajwa-hw/security-workshops/master/screenshots/ranger-213-setup/ranger-213-5.png)

4. Ranger User info tab - Group configs subtab
![Image](https://raw.githubusercontent.com/abajwa-hw/security-workshops/master/screenshots/ranger-213-setup/ranger-213-6.png)

5. Ranger plugins tab
![Image](https://raw.githubusercontent.com/abajwa-hw/security-workshops/master/screenshots/ranger-213-setup/ranger-213-7.png)

6. Ranger Audits tab 
- add a prefix "/ranger" to the HDFS audits. 
- For solr don't use the cloud option for now due to Ranger bug. Use Solr standlone and use internal IP of host where cloud running
![Image](https://raw.githubusercontent.com/abajwa-hw/security-workshops/master/screenshots/ranger-213-setup/ranger-213-8.png)
![Image](https://raw.githubusercontent.com/abajwa-hw/security-workshops/master/screenshots/ranger-213-setup/ranger-213-9.png)

7.Advanced tab
![Image](https://raw.githubusercontent.com/abajwa-hw/security-workshops/master/screenshots/ranger-213-setup/ranger-213-10.png)


## Secured Ambari
- (Optional) SPNEGO: http://docs.hortonworks.com/HDPDocuments/Ambari-2.1.2.0/bk_Ambari_Security_Guide/content/_configuring_http_authentication_for_HDFS_YARN_MapReduce2_HBase_Oozie_Falcon_and_Storm.html
- Setup Ambari as non root http://docs.hortonworks.com/HDPDocuments/Ambari-2.1.1.0/bk_Ambari_Security_Guide/content/_configuring_ambari_for_non-root.html
- Setup kerberos for Ambari: http://docs.hortonworks.com/HDPDocuments/Ambari-2.1.2.0/bk_Ambari_Security_Guide/content/_optional_set_up_kerberos_for_ambari_server.html
```
cd /etc/security/keytabs/
sudo wget https://github.com/seanorama/masterclass/raw/master/security-advanced/extras/ambari.keytab
sudo chown ambari:hadoop ambari.keytab
sudo chmod 400 ambari.keytab

sudo ambari-server stop
centos@ip-172-31-4-234 keytabs]$ sudo ambari-server setup-security
Using python  /usr/bin/python2.7
Security setup options...
===========================================================================
Choose one of the following options:
  [1] Enable HTTPS for Ambari server.
  [2] Encrypt passwords stored in ambari.properties file.
  [3] Setup Ambari kerberos JAAS configuration.
  [4] Setup truststore.
  [5] Import certificate to truststore.
===========================================================================
Enter choice, (1-5): 3
Setting up Ambari kerberos JAAS configuration to access secured Hadoop daemons...
Enter ambari server's kerberos principal name (ambari@EXAMPLE.COM): ambari@LAB.HORTONWORKS.NET
Enter keytab path for ambari server's kerberos principal: /etc/security/keytabs/ambari.keytab
Ambari Server 'setup-security' completed successfully.
sudo ambari-server restart

sudo ambari-server restart
sudo ambari-agent restart
```
- Setup http://docs.hortonworks.com/HDPDocuments/Ambari-2.1.2.0/bk_ambari_views_guide/content/ch_configuring_views_for_kerberos.html

- Automation to install views
```
git clone https://github.com/seanorama/ambari-bootstrap
cd ambari-bootstrap/extras/
grep pass ambari_functions.sh
export ambari_pass=BadPass#1
source ambari_functions.sh
./ambari-views/create-views.sh
   
```
#### HDFS Exercise

- Import data
```
cd /tmp
wget https://raw.githubusercontent.com/abajwa-hw/security-workshops/master/data/sample_07.csv
wget https://raw.githubusercontent.com/abajwa-hw/security-workshops/master/data/sample_08.csv
```
  - Create user dir for admin
  ```
   sudo -u hdfs hadoop fs  -mkdir /user/admin
   sudo -u hdfs hadoop fs  -chown admin:hadoop /user/admin

   sudo -u hdfs hadoop fs  -mkdir /user/sales1
   sudo -u hdfs hadoop fs  -chown sales1:hadoop /user/sales1
   
sudo -u hdfs hadoop fs  -mkdir /user/sales2
sudo -u hdfs hadoop fs  -chown sales2:hadoop /user/sales2
  ```
  
  - Now login to ambari as admin and run this via Hive view
```
CREATE TABLE `sample_07` (
`code` string ,
`description` string ,  
`total_emp` int ,  
`salary` int )
ROW FORMAT DELIMITED FIELDS TERMINATED BY '\t' STORED AS TextFile;

load data local inpath '/tmp/sample_07.csv' into table sample_07;


CREATE TABLE `sample_08` (
`code` string ,
`description` string ,  
`total_emp` int ,  
`salary` int )
ROW FORMAT DELIMITED FIELDS TERMINATED BY '\t' STORED AS TextFile;

load data local inpath '/tmp/sample_08.csv' into table sample_08;

```


- Create a file as sales1 
```
su - sales1
kinit
hdfs dfs -put /tmp/sample_07.csv /user/sales1
exit
```
- Access it as sales2
```
su - sales2
kinit
hdfs dfs -cat /user/sales1/sample_07.csv
```
- Notice that it works 

- Login to Ranger as admin/admin > Audit > check the audit for the event above

```
#still  as sales2
hdfs dfs -ls /user/sales1
```
- Notice that everyone has read permissions on this file
- Login to Ambari as admin. Under HDFS > Configs > Set below and restart HDFS
  - fs.permissions.umask-mode = 077
  
- Add a second file into the same dir after changing the umask
```
su - sales1
hdfs dfs -put /tmp/sample_08.csv /user/sales1
```

- Try to access it as sales2: this should fail now with `Permission denied` and permissions should reflect it
```
su - sales2
hdfs dfs -cat /user/sales1/sample_08.csv
hdfs dfs -ls /user/sales1
```
- Recommendation for application files in HDFS. e.g.
```
hdfs dfs -chown -R hdfs /apps/hive/warehouse
hdfs dfs -chmod -R 000 /apps/hive/warehouse
```

## Day 3

Agenda:
1. Architechture
2. Wire encryption overview
  - http://docs.hortonworks.com/HDPDocuments/HDP2/HDP-2.2.0/Wire_Encryption_v22/index.html#Item1.1
3. Knox/AD
4. Ranger KMS
5. Security training flow review
6. Roadmap/next steps
7. SME rules of engagement
  - Enabling regions
  - Contributing to training team
8. Ranger Logo


## Knox

- Create keystore alias for the ldap manager user (which you set in 'systemUsername' in the topology)
   - Read password for use in following command (this will prompt you for a password):
   ```
read -s -p "Password: " knoxpass
   ```
   - Create password alias
   ```
sudo sudo -u knox /usr/hdp/current/knox-server/bin/knoxcli.sh create-alias knoxLdapSystemPassword --cluster default --value ${knoxpass}
unset knoxpass
   ```
- Ambari > HDFS > Config > Custom core-site > hadoop.proxyuser.knox.groups=hadoop,sales,hr,legal
- Enter below in Ambari > Knox > Config > Advanced topology. Then restart Knox
```
        <topology>

            <gateway>

                <provider>
                    <role>authentication</role>
                    <name>ShiroProvider</name>
                    <enabled>true</enabled>
                    <param>
                        <name>sessionTimeout</name>
                        <value>30</value>
                    </param>
                    <param>
                        <name>main.ldapRealm</name>
                        <!-- <value>org.apache.shiro.realm.ldap.JndiLdapRealm</value> -->
                        <value>org.apache.hadoop.gateway.shirorealm.KnoxLdapRealm</value> 
                    </param>

<!-- changes for AD/user sync -->

<param>
    <name>main.ldapContextFactory</name>
    <value>org.apache.hadoop.gateway.shirorealm.KnoxLdapContextFactory</value>
</param>

<!-- main.ldapRealm.contextFactory needs to be placed before other main.ldapRealm.contextFactory* entries  -->
<param>
    <name>main.ldapRealm.contextFactory</name>
    <value>$ldapContextFactory</value>
</param>

                    <param>
                        <name>main.ldapRealm.contextFactory.url</name>
                        <value>ldap://ad01.lab.hortonworks.net:389</value> 
                    </param>

<param>
    <name>main.ldapRealm.contextFactory.systemUsername</name>
    <value>cn=ldapconnect,ou=ServiceUsers,dc=lab,dc=hortonworks,dc=net</value>
</param>

<param>
    <name>main.ldapRealm.contextFactory.systemPassword</name>
    <value>${ALIAS=knoxLdapSystemPassword}</value>
</param>

                    <param>
                        <name>main.ldapRealm.contextFactory.authenticationMechanism</name>
                        <value>simple</value>
                    </param>
                    <param>
                        <name>urls./**</name>
                        <value>authcBasic</value> 
                    </param>

<param>
    <name>main.ldapRealm.searchBase</name>
    <value>ou=CorpUsers,dc=lab,dc=hortonworks,dc=net</value>
</param>
<param>
    <name>main.ldapRealm.userObjectClass</name>
    <value>person</value>
</param>
<param>
    <name>main.ldapRealm.userSearchAttributeName</name>
    <value>sAMAccountName</value>
</param>

<!-- changes needed for group sync-->
<param>
    <name>main.ldapRealm.authorizationEnabled</name>
    <value>true</value>
</param>
<param>
    <name>main.ldapRealm.groupSearchBase</name>
    <value>ou=CorpUsers,dc=lab,dc=hortonworks,dc=net</value>
</param>
<param>
    <name>main.ldapRealm.groupObjectClass</name>
    <value>group</value>
</param>
<param>
    <name>main.ldapRealm.groupIdAttribute</name>
    <value>cn</value>
</param>


                </provider>

                <provider>
                    <role>identity-assertion</role>
                    <name>Default</name>
                    <enabled>true</enabled>
                </provider>

                <provider>
                    <role>authorization</role>
                    <name>XASecurePDPKnox</name>
                    <enabled>true</enabled>
                </provider>

            </gateway>

            <service>
                <role>NAMENODE</role>
                <url>hdfs://{{namenode_host}}:{{namenode_rpc_port}}</url>
            </service>

            <service>
                <role>JOBTRACKER</role>
                <url>rpc://{{rm_host}}:{{jt_rpc_port}}</url>
            </service>

            <service>
                <role>WEBHDFS</role>
                <url>http://{{namenode_host}}:{{namenode_http_port}}/webhdfs</url>
            </service>

            <service>
                <role>WEBHCAT</role>
                <url>http://{{webhcat_server_host}}:{{templeton_port}}/templeton</url>
            </service>

            <service>
                <role>OOZIE</role>
                <url>http://{{oozie_server_host}}:{{oozie_server_port}}/oozie</url>
            </service>

            <service>
                <role>WEBHBASE</role>
                <url>http://{{hbase_master_host}}:{{hbase_master_port}}</url>
            </service>

            <service>
                <role>HIVE</role>
                <url>http://{{hive_server_host}}:{{hive_http_port}}/{{hive_http_path}}</url>
            </service>

            <service>
                <role>RESOURCEMANAGER</role>
                <url>http://{{rm_host}}:{{rm_port}}/ws</url>
            </service>
        </topology>
```
- Login to Ranger and setup a Knox policy for sales group for WEBHDFS

- Now ensure WebHDFS working by opening terminal to host where Knox is running by sending curl reuqest to 8443 port where Knox is running:
```
curl -ik -u sales1:BadPass#1 https://localhost:8443/gateway/default/webhdfs/v1/?op=LISTSTATUS
```

## Ranger KMS/Data encryption setup

- Follow the docs: http://docs.hortonworks.com/HDPDocuments/HDP2/HDP-2.3.2/bk_Ranger_KMS_Admin_Guide/content/ch_ranger_kms_overview.html
- Note that when the Ranger repo is created you need to ensure that it uses principal name (e.g. keyadmin@LAB.HORTONWORKS.NET) instead of username


## Appendix

###### Install Ranger via Ambari 2.1.2 (current GA version)

1. Install Ranger using Amabris 'Add Service' wizard on the same node as MySQL. 
  - Ranger Admin
    - Ranger DB Host: mysqlnodeinternalhostname.us-west-2.compute.internal 
    - passwords


  - External URL: http://mysqlinternalhostname.compute.internal:6080
  - ranger-admin-site: 
    - ranger.audit.source.type solr
    - ranger.audit.solr.urls http://localhost:6083/solr/ranger_audits

**TODO** Need to fix focs for getting ranger.audit.solr.zookeepers working. For now don't change this property

###### Setup Ranger/AD user/group sync

1. Once Ranger is up, under Ambari > Ranger > Config, set the below and restart Ranger to sync AD users/groups
```
ranger.usersync.source.impl.class ldap
ranger.usersync.ldap.searchBase dc=lab,dc=hortonworks,dc=net
ranger.usersync.ldap.user.searchbase dc=lab,dc=hortonworks,dc=net
ranger.usersync.group.searchbase dc=lab,dc=hortonworks,dc=net
ranger.usersync.ldap.binddn cn=ldapconnect,ou=ServiceUsers,ou=lab,dc=hortonworks,dc=net
ranger.usersync.ldap.ldapbindpassword BadPass#1
ranger.usersync.ldap.url ldap://ad01.lab.hortonworks.net
ranger.usersync.ldap.user.nameattribute sAMAccountName
ranger.usersync.ldap.user.searchfilter (objectcategory=person)
ranger.usersync.ldap.user.groupnameattribute memberof, ismemberof, msSFU30PosixMemberOf
ranger.usersync.group.memberattributename member
ranger.usersync.group.nameattribute cn
ranger.usersync.group.objectclass group
```
2. Check the usersyc log and Ranger UI if users/groups got synced
```
tail -f /var/log/ranger/usersync/usersync.log
```

###### Setup Ranger/AD auth

1. Enable AD users to login to Ranger by making below changes in Ambari > Ranger > Config > ranger-admin-site
```
ranger.authentication.method ACTIVE_DIRECTORY
ranger.ldap.ad.domain lab.hortonworks.net
ranger.ldap.ad.url "ldap://ad01.lab.hortonworks.net:389"
ranger.ldap.ad.base.dn "dc=lab,dc=hortonworks,dc=net"
ranger.ldap.ad.bind.dn "cn=ldapconnect,ou=ServiceUsers,ou=lab,dc=hortonworks,dc=net"
ranger.ldap.ad.referral follow
ranger.ldap.ad.bind.password "BadPass#1"
```

###### Setup Ranger HDFS plugin

In Ambari > HDFS > Config > ranger-hdfs-audit:
```
xasecure.audit.provider.summary.enabled true
xasecure.audit.destination.hdfs.dir hdfs://yournamenodehostname:8020/ranger/audit
xasecure.audit.destination.db true
xasecure.audit.destination.hdfs true
xasecure.audit.destination.solr true
xasecure.audit.is.enabled true
```
**TODO** Need to update docs on xasecure.audit.destination.solr.zookeepers. For now don't change this property

In Ambari > HDFS > Config > ranger-hdfs-plugin-properties:
```
ranger-hdfs-plugin-enabled Yes
REPOSITORY_CONFIG_USERNAME "rangeradmin@lab.hortonworks.net"
REPOSITORY_CONFIG_PASSWORD "BadPass#1"
policy_user "rangeradmin"
common.name.for.certificate " "
hadoop.rpc.protection " "
```

###### Setup Ranger Hive plugin

- In Ambari > HIVE > Config > Settings
  - Under Security > 'Choose authorization' > Ranger
- In Ambari > HIVE > Config > Advanced > ranger-hdfs-audit
```
xasecure.audit.provider.summary.enabled true
xasecure.audit.destination.hdfs.dir hdfs://yournamenodehostname:8020/ranger/audit
xasecure.audit.destination.db true
xasecure.audit.destination.hdfs true
xasecure.audit.destination.solr true
xasecure.audit.is.enabled true
```
- In Ambari > Hive > Config > ranger-hive-plugin-properties:
```
ranger-hdfs-plugin-enabled Yes
REPOSITORY_CONFIG_USERNAME "rangeradmin@lab.hortonworks.net"
REPOSITORY_CONFIG_PASSWORD "BadPass#1"
policy_user "rangeradmin"
common.name.for.certificate " "
hadoop.rpc.protection " "
```