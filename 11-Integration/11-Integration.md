# Integrations

## OP5 - Naemon logs ##

### Logstash ###

1. In ITRS Log Analytics `naemon_beat.conf` set up `ELASTICSEARCH_HOST`, `ES_PORT`, `FILEBEAT_PORT`

2. Copy ITRS Log Analytics `naemon_beat.conf` to `/etc/logstash/conf.d`

3. Based on "FILEBEAT_PORT" if firewall is running:

   ```bash
   sudo firewall-cmd --zone=public --permanent --add-port=FILEBEAT_PORT/tcp
   sudo firewall-cmd --reload
   ```

   

4. Based on amount of data that elasticsearch will receive you can also choose whether you want index creation to be based on moths or days:

   ```bash
   index => "ITRS Log Analytics-naemon-%{+YYYY.MM}"
   or
   index => "ITRS Log Analytics-naemon-%{+YYYY.MM.dd}"
   ```

   

5. Copy `naemon` file to `/etc/logstash/patterns` and make sure it is readable by logstash process
5. Restart *logstash* configuration e.g.:

   ```bash
   sudo systemct restart logstash
   ```

   

   ### Elasticsearch
1. Connect to Elasticsearch node via SSH and Install index pattern for naemon logs. Note that if you have a default pattern covering *settings* section you should delete/modify that in naemon_template.sh:

   ```bash
   "settings": {
       "number_of_shards": 5,
       "auto_expand_replicas": "0-1"
     },
   ```

   

2. Install template by running:
   `./naemon_template.sh`
### ITRS Log Analytics Monitor ###

1. On ITRS Log Analytics Monitor host install filebeat (for instance via rpm `https://www.elastic.co/downloads/beats/filebeat`)
1. In `/etc/filebeat/filebeat.yml` add:

		#=========================== Filebeat inputs =============================
		filebeat.config.inputs:
		  enabled: true
		  path: configs/*.yml

1. You also will have to configure the output section in `filebeat.yml`. You should have one logstash output:

		#----------------------------- Logstash output --------------------------------
		output.logstash:
		  # The Logstash hosts
		  hosts: ["LOGSTASH_IP:FILEBEAT_PORT"]

	If you have few logstash instances - `Logstash` section has to be repeated on every node and `hosts:` should point to all of them:

		hosts: ["LOGSTASH_IP:FILEBEAT_PORT", "LOGSTASH_IP:FILEBEAT_PORT", "LOGSTASH_IP:FILEBEAT_PORT" ]

1. Create `/etc/filebeat/configs` catalog.
1. Copy `naemon_logs.yml` to a newly created catalog.
1. Check the newly added configuration and connection to logstash. Location of executable might vary based on os:

		/usr/share/filebeat/bin/filebeat --path.config /etc/filebeat/ test config
		/usr/share/filebeat/bin/filebeat --path.config /etc/filebeat/ test output

1. Restart filebeat:

		sudo systemctl restart filebeat # RHEL/CentOS 7
		sudo service filebeat restart # RHEL/CentOS 6

### Elasticsearch ###

At this moment there should be a new index on the Elasticsearch node:

	curl -XGET '127.0.0.1:9200/_cat/indices?v'

Example output:

		health status index                 uuid                   pri rep docs.count docs.deleted store.size pri.store.size
		green  open   ITRS Log Analytics-naemon-2018.11    gO8XRsHiTNm63nI_RVCy8w   1   0      23176            0      8.3mb          8.3mb

If the index has been created, in order to browse and visualise the data, "index pattern" needs to be added in Kibana.

## OP5 - Performance data ##

Below instruction requires that between ITRS Log Analytics node and Elasticsearch node is working Logstash instance.

### Elasticsearch ###
1.	First, settings section in *ITRS Log Analyticstemplate.sh* should be adjusted, either:
	- there is a default template present on Elasticsearch that already covers shards and replicas then settings sections should be removed from the *ITRS Log Analyticstemplate.sh* before executing
	- there is no default template - shards and replicas should be adjusted for you environment (keep in mind replicas can be added later, while changing shards count on existing index requires 
		reindexing it)

			"settings": {
			  "number_of_shards": 5,
			  "number_of_replicas": 0
			}

1. In URL *ITRS Log Analyticsperfdata* is a name for the template - later it can be search for or modify with it.

1. The "*template*" is an index pattern. New indices matching it will have the settings and mapping applied automatically (change it if you index name for *ITRS Log Analytics perfdata* is different).

1. Mapping name should match documents type:

    ```
    "mappings": {
    	  "ITRS Log Analyticsperflogs"
    ```

    Running ITRS Log Analyticstemplate.sh will create a template (not index) for ITRS Log Analytics perf data documents.

### Logstash ###

1.	The *ITRS Log Analyticsperflogs.conf* contains example of *input/filter/output* configuration. It has to be copied to */etc/logstash/conf.d/*. Make sure that the *logstash* has permissions to read the configuration files:
	
	```bash
	chmod 664 /etc/logstash/conf.d/ITRS Log Analyticsperflogs.conf
	```
	
	
	
2. In the input section comment/uncomment *“beats”* or *“tcp”* depending on preference (beats if *Filebeat* will be used and tcp if *NetCat*). The port and the type has to be adjusted as well:

   ```bash
   port => PORT_NUMBER
   type => "ITRS Log Analyticsperflogs"
   ```

   

3. In a filter section type has to be changed if needed to match the input section and Elasticsearch mapping.

4. In an output section type should match with the rest of a *config*. host should point to your elasticsearch node. index name should correspond with what has been set in elasticsearch template to allow mapping application. The date for index rotation in its name is recommended and depending on the amount of data expecting to be transferred should be set to daily (+YYYY.MM.dd) or monthly (+YYYY.MM) rotation:

   ```bash
   hosts => ["127.0.0.1:9200"]
   index => "ITRS Log Analytics-perflogs-%{+YYYY.MM.dd}"
   ```

5. Port has to be opened on a firewall:

   ```bash
   sudo firewall-cmd --zone=public --permanent --add-port=PORT_NUMBER/tcp
   sudo firewall-cmd --reload
   ```

6. Logstash has to be reloaded:

   ```bash
   sudo systemctl restart logstash
   ```

   or

   ```bash
   sudo kill -1 LOGSTASH_PID
   ```

### ITRS Log Analytics Monitor ###

1. You have to decide wether FileBeat or NetCat will be used. In case of Filebeat - skip to the second step. Otherwise:

   - Comment line:

     ```bash
     54    open(my $logFileHandler, '>>', $hostPerfLogs) or die "Could not open $hostPerfLogs"; #FileBeat
     •	Uncomment lines:
     55 #    open(my $logFileHandler, '>', $hostPerfLogs) or die "Could not open $hostPerfLogs"; #NetCat
     ...
     88 #    my $logstashIP = "LOGSTASH_IP";
     89 #    my $logstashPORT = "LOGSTASH_PORT";
     90 #    if (-e $hostPerfLogs) {
     91 #        my $pid1 = fork();
     92 #        if ($pid1 == 0) {
     93 #            exec("/bin/cat $hostPerfLogs | /usr/bin/nc -w 30 $logstashIP $logstashPORT");
     94 #        }
     95 #    }
     ```

     

   - In process-service-perfdata-log.pl and process-host-perfdata-log.pl: change logstash IP and port:

     ```bash
     92 my $logstashIP = "LOGSTASH_IP";
     93 my $logstashPORT = "LOGSTASH_PORT";
     ```
     
     

2. In case of running single ITRS Log Analytics node, there is no problem with the setup. In case of a peered environment *$do_on_host* variable has to be set up and the script *process-service-perfdata-log.pl/process-host-perfdata-log.pl* has to be propagated on all of ITRS Log Analytics nodes:

   ```bash
   16 $do_on_host = "EXAMPLE_HOSTNAME"; # ITRS Log Analytics node name to run the script on
   17 $hostName = hostname; # will read hostname of a node running the script
   ```

   

3. Example of command definition (*/opt/monitor/etc/checkcommands.cfg*) if scripts have been copied to */opt/plugins/custom/*:

   ```bash
   # command 'process-service-perfdata-log'
   define command{
       command_name                   process-service-perfdata-log
       command_line                   /opt/plugins/custom/process-service-perfdata-log.pl $TIMET$
       }
   # command 'process-host-perfdata-log'
   define command{
       command_name                   process-host-perfdata-log
       command_line                   /opt/plugins/custom/process-host-perfdata-log.pl $TIMET$
       }
   ```

   

4. In */opt/monitor/etc/naemon.cfg service_perfdata_file_processing_command* and *host_perfdata_file_processing_command* has to be changed to run those custom scripts:

   ```bash
   service_perfdata_file_processing_command=process-service-perfdata-log
   host_perfdata_file_processing_command=process-host-perfdata-log
   ```

   

5. In addition *service_perfdata_file_template* and *host_perfdata_file_template* can be changed to support sending more data to Elasticsearch. For instance, by adding *$HOSTGROUPNAMES$* and *$SERVICEGROUPNAMES$* macros logs can be separated better (it requires changes to Logstash filter config as well)

5. Restart naemon service:

   ```bash
   sudo systemctl restart naemon # CentOS/RHEL 7.x
   sudo service naemon restart # CentOS/RHEL 7.x
   ```

   

6. If *FileBeat* has been chosen, append below to *filebeat.conf* (adjust IP and PORT):

   ```bash
   filebeat.inputs:
   type: log
   enabled: true
   paths:
     - /opt/monitor/var/service_performance.log
     - /opt/monitor/var/host_performance.log
   tags: ["ITRS Log Analyticsperflogs"]
     output.logstash:
   # The Logstash hosts
     hosts: ["LOGSTASH_IP:LOGSTASH_PORT"]
   ```

   

7. Restart FileBeat service:

   ```bash
   sudo systemctl restart filebeat # CentOS/RHEL 7.x
   sudo service filebeat restart # CentOS/RHEL 7.x
   ```

   

   ### Kibana

At this moment there should be new index on the Elasticsearch node with performance data documents from ITRS Log Analytics Monitor. 
Login to an Elasticsearch node and run: `curl -XGET '127.0.0.1:9200/_cat/indices?v'` Example output:

	health status index                      pri rep docs.count docs.deleted store.size pri.store.size
	green  open   auth                       5   0          7         6230      1.8mb          1.8mb
	green  open   ITRS Log Analytics-perflogs-2018.09.14    5   0      72109            0     24.7mb         24.7mb

After a while, if there is no new index make sure that: 

- Naemon is runnig on ITRS Log Analytics node
- Logstash service is running and there are no errors in: */var/log/logstash/logstash-plain.log* 
- Elasticsearch service is running an there are no errors in: */var/log/elasticsearch/elasticsearch.log*

If the index has been created, in order to browse and visualize the data “*index pattern*” needs to be added to Kibana. 

1. After logging in to Kibana GUI go to *Settings* tab and add *ITRS Log Analytics-perflogs-** pattern. Chose *@timestamp* time field and click *Create*. 
2. Performance data logs should be now accessible from Kibana GUI Discovery tab ready to be visualize.

## OP5 Beat

The op5beat is small agent for collecting metrics from op5 Monitor.

The op5beat  is located in the installation directory: `utils/op5integration/op5beat`

### Installation for Centos7 and newer

1. Copy the necessary files to the appropriate directories:

   ```bash
   cp -rf etc/* /etc/
   cp -rf usr/* /usr/
   cp -rf var/* /var/
   ```

   

2. Configure and start  op5beat service (systemd):

   ```bash
   cp -rf op5beat.service /usr/lib/systemd/system/
   systemctl daemon-reload
   systemctl enable op5beat
   systemctl start op5beat
   ```

   

### Installation for Centos6 and older

1. Copy the necessary files to the appropriate directories:

   ```bash
   cp -rf etc/* /etc/
   cp -rf usr/* /usr/
   cp -rf var/* /var/
   ```

2. Configure and start  op5beat service:

   - sysV init:

     ```bash
     cp -rf op5beat.service /etc/rc.d/init.d/op5beat
     chkconfig op5beat on
     service op5beat start
     ```

   - supervisord (optional):

     ```bash
     yum install supervisor
     cp -rf supervisord.conf /etc/supervisord.conf
     ```

    
## The Grafana instalation ##

1. To install the Grafana application you should:

   - add necessary repository to operating system:

     ```bash
     [root@localhost ~]# cat /etc/yum.repos.d/grafan.repo
       [grafana]
       name=grafana
       baseurl=https://packagecloud.io/grafana/stable/el/7/$basearch
       repo_gpgcheck=1
       enabled=1
       gpgcheck=1
       gpgkey=https://packagecloud.io/gpg.key https://grafanarel.s3.amazonaws.com/RPM-GPG-KEY-grafana
       sslverify=1
       sslcacert=/etc/pki/tls/certs/ca-bundle.crt
       [root@localhost ~]#
     ```

     

   - install the Grafana with following commands: 

     ```bash
      [root@localhost ~]# yum search grafana
       Loaded plugins: fastestmirror
       Loading mirror speeds from cached hostfile
        * base: ftp.man.szczecin.pl
        * extras: centos.slaskdatacenter.com
        * updates: centos.slaskdatacenter.com
       =========================================================================================================== N/S matched: grafana ===========================================================================================================
       grafana.x86_64 : Grafana
       pcp-webapp-grafana.noarch : Grafana web application for Performance Co-Pilot (PCP)
     
         Name and summary matches only, use "search all" for everything.
     
       [root@localhost ~]# yum install grafana
     ```

     

   - to run application use following commands:

     ```bash
      [root@localhost ~]# systemctl enable grafana-server
       Created symlink from /etc/systemd/system/multi-user.target.wants/grafana-server.service to /usr/lib/systemd/system/grafana-server.service.
       [root@localhost ~]#
       [root@localhost ~]# systemctl start grafana-server
       [root@localhost ~]# systemctl status grafana-server
       ● grafana-server.service - Grafana instance
          Loaded: loaded (/usr/lib/systemd/system/grafana-server.service; enabled; vendor preset: disabled)
          Active: active (running) since Thu 2018-10-18 10:41:48 CEST; 5s ago
            Docs: http://docs.grafana.org
        Main PID: 1757 (grafana-server)
          CGroup: /system.slice/grafana-server.service
                  └─1757 /usr/sbin/grafana-server --config=/etc/grafana/grafana.ini --pidfile=/var/run/grafana/grafana-server.pid cfg:default.paths.logs=/var/log/grafana cfg:default.paths.data=/var/lib/grafana cfg:default.paths.plugins=/var...
     
        [root@localhost ~]#
     ```

     

2. To connect the Grafana application you should:

   - define the default login/password (line 151;154 in config file):

     ```bash
     [root@localhost ~]# cat /etc/grafana/grafana.ini
     148 #################################### Security ####################################
     149 [security]
     150 # default admin user, created on startup
     151 admin_user = admin
     152
     153 # default admin password, can be changed before first start of grafana,  or in profile settings
     154 admin_password = admin
     155
     ```

     

   - restart *grafana-server* service:

     ```bash
     systemctl restart grafana-server
     ```

     

   - Login to Grafana user interface using web browser: *http://ip:3000*

     ![](/media/media/image112.png)

   - use login and password that you set in the config file.

   - Use below example to set conection to Elasticsearch server:

     ![](/media/media/image113.png)

## The Beats configuration ##

### Kibana API ###

Reference link: [https://www.elastic.co/guide/en/kibana/master/api.html](https://www.elastic.co/guide/en/kibana/master/api.html "https://www.elastic.co/guide/en/kibana/master/api.html")

After installing any of beats package you can use ready to use dashboard related to this beat package. For instance dashboard and index pattern are available in */usr/share/filebeat/kibana/6/* directory on Linux.

Before uploading index-pattern or dashboard you have to authorize yourself: 

1. Set up *login/password/kibana_ip* variables, e.g.:

		login=my_user
		password=my_password
		kibana_ip=10.4.11.243

1. Execute command which will save authorization cookie:

		curl -c authorization.txt -XPOST -k "https://${kibana_ip}:5601/login" -d "username=${username}&password=${password}&version=6.2.3&location=https%3A%2F%2F${kibana_ip}%3A5601%2Flogin"

1.	Upload index-pattern and dashboard to *Kibana*, e.g.:

		curl -b authorization.txt -XPOST -k "https://${kibana_ip}:5601/api/kibana/dashboards/import" -H 'kbn-xsrf: true' -H 'Content-Type: application/json' -d@/usr/share/filebeat/kibana/6/index-pattern/filebeat.json
		curl -b authorization.txt -XPOST -k "https://${kibana_ip}:5601/api/kibana/dashboards/import" -H 'kbn-xsrf: true' -H 'Content-Type: application/json' -d@/usr/share/filebeat/kibana/6/dashboard/Filebeat-mysql.json

1.	When you want to upload beats index template to Ealsticsearch you have to recover it first (usually you do not send logs directly to Es rather than to Logstash first):

		/usr/bin/filebeat export template --es.version 6.2.3 >> /path/to/beats_template.json

1.	After that you can upload it as any other template (Access Es node with SSH):

		curl -XPUT "localhost:9200/_template/ITRS Log Analyticsperfdata" -H'Content-Type: application/json' -d@beats_template.json

## Wazuh integration ##

ITRS Log Analytics can integrate with the Wazuh, which is lightweight agent is designed to perform a number of tasks with the objective of detecting threats and, when necessary, trigger automatic responses. The agent core capabilities are:

- Log and events data collection
- File and registry keys integrity monitoring
- Inventory of running processes and installed applications
- Monitoring of open ports and network configuration
- Detection of rootkits or malware artifacts
- Configuration assessment and policy monitoring
- Execution of active responses

The Wazuh agents run on many different platforms, including Windows, Linux, Mac OS X, AIX, Solaris and HP-UX. They can be configured and managed from the Wazuh server.
#### Deploying Wazuh Server ####

https://documentation.wazuh.com/3.13/installation-guide/installing-wazuh-manager/linux/centos/index.html

#### Deploing Wazuh Agent ####

https://documentation.wazuh.com/3.13/installation-guide/installing-wazuh-agent/index.html

#### Filebeat configuration ####

## 2FA authorization with Google Auth Provider (example)

### Software used (tested versions):

- NGiNX (1.16.1 - from CentOS base reposiory)
- oauth2_proxy ([https://github.com/pusher/oauth2_proxy/releases](https://github.com/pusher/oauth2_proxy/releases) - 4.0.0)

### The NGiNX configuration:
1. Copy the [ng_oauth2_proxy.conf](/files/ng_oauth2_proxy.conf) to `/etc/nginx/conf.d/`;

   ```bash
   server {
       listen 443 default ssl;
       server_name logserver.local;
       ssl_certificate /etc/kibana/ssl/logserver.org.crt;
       ssl_certificate_key /etc/kibana/ssl/logserver.org.key;
       ssl_session_cache   builtin:1000  shared:SSL:10m;
       add_header Strict-Transport-Security max-age=2592000;
   
     location /oauth2/ {
       proxy_pass       http://127.0.0.1:4180;
       proxy_set_header Host                    $host;
       proxy_set_header X-Real-IP               $remote_addr;
       proxy_set_header X-Scheme                $scheme;
       proxy_set_header X-Auth-Request-Redirect $request_uri;
       # or, if you are handling multiple domains:
       # proxy_set_header X-Auth-Request-Redirect $scheme://$host$request_uri;
     }
     location = /oauth2/auth {
       proxy_pass       http://127.0.0.1:4180;
       proxy_set_header Host             $host;
       proxy_set_header X-Real-IP        $remote_addr;
       proxy_set_header X-Scheme         $scheme;
       # nginx auth_request includes headers but not body
       proxy_set_header Content-Length   "";
       proxy_pass_request_body           off;
     }
   
     location / {
       auth_request /oauth2/auth;
       error_page 401 = /oauth2/sign_in;
   
       # pass information via X-User and X-Email headers to backend,
       # requires running with --set-xauthrequest flag
       auth_request_set $user   $upstream_http_x_auth_request_user;
       auth_request_set $email  $upstream_http_x_auth_request_email;
       proxy_set_header X-User  $user;
       proxy_set_header X-Email $email;
   
       # if you enabled --pass-access-token, this will pass the token to the backend
       auth_request_set $token  $upstream_http_x_auth_request_access_token;
       proxy_set_header X-Access-Token $token;
   
       # if you enabled --cookie-refresh, this is needed for it to work with auth_request
       auth_request_set $auth_cookie $upstream_http_set_cookie;
       add_header Set-Cookie $auth_cookie;
   
       # When using the --set-authorization-header flag, some provider's cookies can exceed the 4kb
       # limit and so the OAuth2 Proxy splits these into multiple parts.
       # Nginx normally only copies the first `Set-Cookie` header from the auth_request to the response,
       # so if your cookies are larger than 4kb, you will need to extract additional cookies manually.
       auth_request_set $auth_cookie_name_upstream_1 $upstream_cookie_auth_cookie_name_1;
   
       # Extract the Cookie attributes from the first Set-Cookie header and append them
       # to the second part ($upstream_cookie_* variables only contain the raw cookie content)
       if ($auth_cookie ~* "(; .*)") {
           set $auth_cookie_name_0 $auth_cookie;
           set $auth_cookie_name_1 "auth_cookie__oauth2_proxy_1=$auth_cookie_name_upstream_1$1";
       }
   
       # Send both Set-Cookie headers now if there was a second part
       if ($auth_cookie_name_upstream_1) {
           add_header Set-Cookie $auth_cookie_name_0;
           add_header Set-Cookie $auth_cookie_name_1;
       }
   
       proxy_pass https://127.0.0.1:5601;
       # or "root /path/to/site;" or "fastcgi_pass ..." etc
     }
   }
   ```


1. Set `ssl_certificate` and `ssl_certificate_key` path in ng_oauth2_proxy.conf

When SSL is set using nginx proxy, Kibana can be started with http. 
However, if it is to be run with encryption, you also need to change `proxy_pass` to the appropriate one.

### The [oauth2_proxy](/files/oauth2_proxy.cfg) configuration:

1. Create a directory in which the program will be located and its configuration:

    ```bash
    mkdir -p /usr/share/oauth2_proxy/
    mkdir -p /etc/oauth2_proxy/
    ```

2. Copy files to directories:

    ```bash
    cp oauth2_proxy /usr/share/oauth2_proxy/
    cp oauth2_proxy.cfg /etc/oauth2_proxy/
    ```

3. Set directives according to OAuth configuration in Google Cloud project

    ```bash
            cfg
            client_id =
            client_secret =
            # the following limits domains for authorization (* - all)
        ​	email_domains = [
        ​	  "*"
        ​	]
    ```

4. Set the following according to the public hostname:

    ```bash
    cookie_domain = "kibana-host.org"
    ```

5. In case 	og-in restrictions for a specific group defined on the Google side:
	- Create administrative account: https://developers.google.com/identity/protocols/OAuth2ServiceAccount ; 
	- Get configuration to JSON file and copy Client ID;
	- On the dashboard of the Google Cloud select "APIs & Auth" -> "APIs";
	- Click on "Admin SDK" and "Enable API";
	- Follow the instruction at [https://developers.google.com/admin-sdk/directory/v1/guides/delegation#delegate_domain-wide_authority_to_your_service_account](https://developers.google.com/admin-sdk/directory/v1/guides/delegation#delegate_domain-wide_authority_to_your_service_account) and give the service account the following permissions:

			https://www.googleapis.com/auth/admin.directory.group.readonly
			https://www.googleapis.com/auth/admin.directory.user.readonly

	- Follow the instructions to grant access to the Admin API [https://support.google.com/a/answer/60757](https://support.google.com/a/answer/60757)
	- Create or select an existing administrative email in the Gmail domain to flag it `google-admin-email`
	- Create or select an existing group to flag it `google-group`
	- Copy the previously downloaded JSON file to `/etc/oauth2_proxy/`.
	- In file [oauth2_proxy](/files/oauth2_proxy.cfg) set the appropriate path:

    ```bash
    google_service_account_json =
    ```

### Service start up

- Start the NGiNX service 

- Start the oauth2_proxy service

```bash
/usr/share/oauth2_proxy/oauth2_proxy -config="/etc/oauth2_proxy/oauth2_proxy.cfg"
```

In the browser enter the address pointing to the server with the ITRS Log Analytics installation

 --type=alias
```

#### Import aliases into ES

```bash
elasticdump \
  --input=./alias.json \
  --output=http://es.com:9200 \
  --type=alias
```

#### Backup templates to a file

```bash
elasticdump \
  --input=http://es.com:9200/template-filter \
  --output=templates.json \
  --type=template
```

#### Import templates into ES

```bash
elasticdump \
  --input=./templates.json \
  --output=http://es.com:9200 \
  --type=template
```

#### Split files into multiple parts

```bash
elasticdump \
  --input=http://production.es.com:9200/my_index \
  --output=/data/my_index.json \
  --fileSize=10mb
```

#### Import data from S3 into ES (using s3urls)

```bash
elasticdump \
  --s3AccessKeyId "${access_key_id}" \
  --s3SecretAccessKey "${access_key_secret}" \
  --input "s3://${bucket_name}/${file_name}.json" \
  --output=http://production.es.com:9200/my_index
```

#### Export ES data to S3 (using s3urls)

```bash
elasticdump \
  --s3AccessKeyId "${access_key_id}" \
  --s3SecretAccessKey "${access_key_secret}" \
  --input=http://production.es.com:9200/my_index \
  --output "s3://${bucket_name}/${file_name}.json"
```

#### Import data from MINIO (s3 compatible) into ES (using s3urls)

```bash
elasticdump \
  --s3AccessKeyId "${access_key_id}" \
  --s3SecretAccessKey "${access_key_secret}" \
  --input "s3://${bucket_name}/${file_name}.json" \
  --output=http://production.es.com:9200/my_index
  --s3ForcePathStyle true
  --s3Endpoint https://production.minio.co
```

#### Export ES data to MINIO (s3 compatible) (using s3urls)

```bash
elasticdump \
  --s3AccessKeyId "${access_key_id}" \
  --s3SecretAccessKey "${access_key_secret}" \
  --input=http://production.es.com:9200/my_index \
  --output "s3://${bucket_name}/${file_name}.json"
  --s3ForcePathStyle true
  --s3Endpoint https://production.minio.co
```

#### Import data from CSV file into ES (using csvurls)

```bash
elasticdump \

  # csv:// prefix must be included to allow parsing of csv files

  # --input "csv://${file_path}.csv" \

  --input "csv:///data/cars.csv"
  --output=http://production.es.com:9200/my_index \
  --csvSkipRows 1    # used to skip parsed rows (this does not include the headers row)
  --csvDelimiter ";" # default csvDelimiter is ','

```

#### Copy a single index from a elasticsearch:

```bash
elasticdump \
  --input=http://es.com:9200/api/search \
  --input-index=my_index \
  --output=http://es.com:9200/api/search \
  --output-index=my_index \
  --type=mapping
```


## Embedding dashboard in iframe

It is possible to send alerts containing HTML *iframe* as notification content. For example:

```html
<a href="https://siem-vip:5601/app/kibana#/discover/72503360-1b25-11ea-bbe4-d7be84731d2c?_g=%28refreshInterval%3A%28display%3AOff%2Csection%3A0%2Cvalue%3A0%29%2Ctime%3A%28from%3A%272021-03-03T08%3A36%3A50Z%27%2Cmode%3Aabsolute%2Cto%3A%272021-03-04T08%3A36%3A50Z%27%29%29" target="_blank" rel="noreferrer">https://siem-vip:5601/app/kibana#/discover/72503360-1b25-11ea-bbe4-d7be84731d2c?_g=%28refreshInterval%3A%28display%3AOff%2Csection%3A0%2Cvalue%3A0%29%2Ctime%3A%28from%3A%272021-03-03T08%3A36%3A50Z%27%2Cmode%3Aabsolute%2Cto%3A%272021-03-04T08%3A36%3A50Z%27%29%29</a>
```

If you want an existing HTTP session to be used to display the iframe content, you need to set the following parameters in the `/etc/kibana/kibana.yml` file:

```yaml
login.isSameSite: "Lax"
login.isSecure: true
```

Possible values for *isSameSite* are: **"None", "Lax", "Strict", false**

For *isSecure*: **false or true**

## Integration with AWS service

### The scope of integration

The integration of ITRS Log Analytics with the AWS cloud environment was prepared based on the following requirements:

1. General information of the EC2 area, i.e .:
   - number of machines
   - number of CPUs
   - amount of RAM
2. General information of the RDS area, i.e.:
   - Number of RDS instances
   - The number of RDS CPUs
   - Amount of RDS RAM
3. EC2 area information containing information for each machine i.e .:
   - list of tags;
   - cloudwatch alarms configured;
   - basic information (e.g. imageID, reservtionid, accountid, launch date, private and public address, last backup, etc.);
   - list of available metrics in cloudwatch;
   - list of snapshots;
   - AMI list;
   - cloudtrail (all records, with detailed details).
4. Information on Backups of EC2 and RDS instances
5. Search for S3 objects, shoes, AMI images
6. Downloading additional information about other resources, ie IG, NAT Gateway, Transit Gateway.
7. Monitoring changes in the infrastructure based on Cloudtrail logs;
8. Monitoring costs based on billing and usage reports.
9. Monitoring the Security Group and resources connected to them and resources not connected to the Security Group
10. Monitoring user activity and inactivity.
11. Integration supports service for multiple member accounts in AWS organization

The integration uses a Data Collector, i.e. the ITRS Log Analytics host, which is responsible for receiving data from external sources.

### Data download mechanism

The integration was prepared based on AWS (CLI), a unified tool for managing AWS services, with which it is possible to download and monitor many AWS services from the command line. The AWS (CLI) tool is controlled by the ITRS Log Analytics data collector, which execute commands at specified intervals and captures the results of data received from the AWS service. The obtained data is processed and enriched and, as a result, saved to the ITRS Log Analytics indexes.

### AWS Cost & Usage Report

The integration of ITRS Log Analytics with the AWS billing environment requires access to AWS Cost & Usage reports, which generated in accordance with the agreed schedule constitute the basic source of data for cost analysis in ITRS Log Analytics. The generated report is stored on S3 in the bucket defined for this purpose and cyclically downloaded from it by the ITRS Log Analytics collector. After the report is downloaded, it is processed and saved to a dedicated Elasticsearch index. The configuration of generating and saving a report to S3 is described in the AWS documentation:  https://aws.amazon.com/aws-cost-management/aws-cost-and-usage-reporting/.

### Cloud Trail

The integration of the ITRS Log Analytics with the AWS environment in order to receive events from the AWS environment requires access to the S3 bucket, on which the so-called AWS Trails. The operation of the ITRS Log Analytics collector is based on periodical checking of the "cloudtraillogs" bucket and downloading new events from it. After the events are retrieved, they are processed so that the date the event occurred matches the date the document was indexed.
The AWS Trail creation configuration is described in the AWS documentation:  https://docs.aws.amazon.com/awscloudtrail/latest/userguide/cloudtrail-create-a-trail-using-the-console-first-time.html#creating-a-trail-in-the-console. 

### Configuration

#### Configuration of access to the AWS account

Configuration of access to AWS is in the configuration file of the AWS service (CLI), which was placed in the home directory of the Logstash user:

```
/home/logstash/.aws/config
 [default]
aws_access_key_id=A************************4
aws_secret_access_key=*******************************************u

```
The "default" section contains aws_access_key_id and aws_secret_access_key.
Configuration file containing the list of AWS accounts that are included in the integration:

`/etc/logstash/lists/account.txt`

#### Configuration of AWS profiles

AWS profiles allow you to navigate to different AWS accounts using the defined AWS role  for example : "LogserverReadOnly". Profiles are defined in the configuration file:

`/home/logstash/.aws/config`

```
Profile configuration example:
[profile 111111111222]
role_arn = arn: aws: iam :: 111111111222: role / LogserverReadOnly
source_profile = default
region = eu-west-1
output = json
```

The above section includes
- profile name;
- role_arn - definition of the account and the role assigned to the account;
- source_profile - definition of the source profile;
- region - AWS region;
- output - the default format of the output data.

#### Configure S3 buckets scanning

The configuration of scanning buckets and S3 objects for the "s3" dashboard was placed in the following configuration files:
- /etc/logstash/lists/bucket_s3.txt - configuration of buckets that are included in the scan;

- /etc/logstash/lists/account_s3.txt - configuration of accounts that are included in the scan;

#### Configuration of AWS Cost & Usage reports

Downloading AWS Cost & Usage reports is done using the script:
"/etc/logstash/lists/bin/aws_get_billing.sh"

In which the following parameters should be set:
- BUCKET = bucket_bame - bucket containing packed rarpotes;
- PROFILE = profile_name - a profile authorized to download reports from the bucket.

#### Logstash Pipelines

Integration mechanisms are managed by the Logstash process, which is responsible for executing scripts, querying AWS, receiving data, reading data from files, processing the received data and enriching it and, as a result, submitting it to the ITRS Log Analytics index. These processes were set up under the following Logstash pipelines:

```
- pipeline.id: aws
  path.config: "/etc/logstash/aws/conf.d/*.conf"
  pipeline.workers: 1
 
- pipeline.id: awstrails
  path.config: "/etc/logstash/awstrails/conf.d/*.conf"
  pipeline.workers: 1
 
- pipeline.id: awss3
  path.config: "/etc/logstash/awss3/conf.d/*.conf"
  pipeline.workers: 1
 
- pipeline.id: awsbilling
  path.config: "/etc/logstash/awsbilling/conf.d/*.conf"
  pipeline.workers: 1 
```

#### Configuration of AWS permissions and access

To enable the correct implementation of the integration assumptions in the configuration of the IAM area, an Logserver-ReadOnly account was created with programming access with the following policies assigned:

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "backup:Describe*",
                "backup:Get*",
                "backup:List*",
                "cloudwatch:Describe*",
                "cloudwatch:Get*",
                "cloudwatch:List*",
                "ec2:Describe*",
                "iam:GenerateCredentialReport",
                "iam:GetCredentialReport",
                "logs:Describe*",
                "logs:Get*",
                "rds:Describe*",
                "rds:List*",
                "tag:Get*"
            ],
            "Resource": "*"
        },
        {
            "Sid": "AllowSpecificS3ForLogServer",
            "Effect": "Allow",
            "Action": [
                "s3:Get*",
                "s3:List*"
            ],
            "Resource": [
                "arn:aws:s3:::veoliaplcloudtraillogs",
                "arn:aws:s3:::veoliaplcloudtraillogs/*"
            ]
        }
    ]
}
```

#### Data indexing

The data in the indexes has been divided into the following types:
   - awscli-* - storing volumetric data about AWS infrastructure;
   - awsbilling-* - storing billing data from billing reports;
   - awscli-trail-* - storing AWS environment events / logs from CloudTrail;
   - awsusers-000001 - storing data about users and administrators of the AWS service.

#### Dashboards

The data collected in the integration process has been visualized and divided into the following sections (dashboards):
   - Overview - The section provides an overview of the quantitative state of the environment
   - EC2 - the section contains details about the EC2 instance;
   - RDS - the section contains details about RDS instances;
   - AMI - the section contains details about Images;
   - S3 - section for searching for objects and buckets S3;
   - Snapshots - section for reviewing snapshots taken;
   - Backups - section to review the backups made;
   - CloudTrail - a section for analyzing logs downloaded from CloudTrail;
   - IAM - a section containing user and administrator activity and configuration of AWS environment access accounts;
   - Billing - AWS service billing section;
   - Gateways - section containing details and configuration of AWS Gateways.

##### Overview
The following views are included in the "Overview" section:
   - [AWS] Navigation - navigation between sections;
   - [AWS] Overview Selector - active selector used to filter sections;
   - [AWS] Total Instances - metric indicator of the number of EC2 instances;
   - [AWS] Total CPU Running Instances - metric indicator of the number of CPUs running EC2 instances;
   - [AWS] Total Memory Running Instances - metric indicator of RAM [MB] amount of running EC2 instances;
   - [AWS] Total RDS Instances - metric indicator of the number of RDS instances;
   - [AWS] Total CPU Running RDS - metric indicator of the number of CPUs running RDS instances;
   - [AWS] Total Memory Running RDS - metric indicator of the amount of RAM [GB] of running RDS instances;
   - [AWS] Instance List - an array containing aggregated details about an EC2 instance;
   - [AWS] RDS Instance List - an array containing aggregated details about an EC2 instance;
   - [AWS] Alarm List - table containing the list of AWS environment alarms;
   - [AWS] Tags List - an array containing a list of AWS tags;
   - [AWS] CloudWatch Metrics - table containing a list of AWS metrics;

##### EC2
The following views have been placed in the "EC2" section:
   - [AWS] Navigation - navigation between sections;
   - [AWS] State Selector - active selector used to filter sections;
   - [AWS] Total Instances - metric indicator of the number of EC2 instances;
   - [AWS] Total CPU Running Instances - metric indicator of the number of CPUs running EC2 instances;
   - [AWS] Running histogram - graphical interpretation of the instance status in the timeline;
   - [AWS] Total Memory Running Instances - metric indicator of RAM [MB] amount of running EC2 instances;
   - [AWS] OP5 Monitored Count - metric indicator of monitored instances in the OP5 Monitor system;
   - [AWS] OP5 NOT Monitored Count - metric indicator of unmonitored instances in the OP5 Monitor system;
   - [AWS] OP5 Monitored Details - a table containing a list of instances with monitoring details in the OP5 Monitoring system;
   - [AWS] Instance Details List - table containing details of the EC2 instance;
   - [AWS] CloudWatch Metrics - table containing details of EC2 metrics downloaded from AWS service;

##### RDS
The following views have been placed in the "RDS" section:
   - [AWS] Navigation - navigation between sections;
   - [AWS] RDS State Selector - active selector used for section filtering;
   - [AWS] Total RDS Instances - metric indicator of the number of RDS instances;
   - [AWS] Total CPU Running RDS - metric indicator of the number of CPUs running RDS instances;
   - [AWS] RDS Running histogram - graphical interpretation of the instance status in the timeline;
   - [AWS] RDS Instance Details - a table containing aggregated details of a RDS instance;
   - [AWS] RDS Details - table containing full details of the RDS instance;
   - [AWS] CloudWatch Metrics - table containing details of EC2 metrics downloaded from AWS service;

##### AMI
The following views have been placed in the "AMI" section:
   - [AWS] Navigation - navigation between sections;
   - [AWS] Image Selector - active selector used to filter sections;
   - [AWS] Image Details - a table containing full details of the images taken;
   - [AWS] Image by Admin Details - a table containing full details of images made by the administrator;
   - [AWS] AMI type by time - graphical interpretation of image creation presented in time;

##### Security
The following views have been placed in the "Security" section:
   - [AWS] Navigation - navigation between sections;
   - [AWS] Security Selector - active selector used to filter sections;
   - [AWS] Security Group ID by InstanceID - a table containing Security Groups with assigned Instances;
   - [AWS] Instance by Security Group - a table containing Instances with assigned Security Groups and details;
   - [AWS] Security Group connect state - table containing the status of connecting the Security Groups to the EC2 and RDS instances.

##### Snapshots
The following views have been placed in the "Snapshots" section:
   - [AWS] Navigation - navigation between sections;
   - [AWS] Snapshot Selector - active selector used to filter sections;
   - [AWS] Snapshots List - a view containing a list of snapshots made with details;
   - [AWS] Snapshots by time - graphical interpretation of creating snapshots over time;

##### Backups
The following views have been placed in the "Backup" section:
   - [AWS] Navigation - navigation between sections;
   - [AWS] Backup Selector - active selector used to filter sections;
   - [AWS] Backup List - view containing the list of completed Backup with details;
   - [AWS] Backup by time - graphical interpretation of backups presented in time;

##### CloudTrail
The following views have been placed in the "CloudTrail" section:
   - [AWS] Navigation - navigation between sections;
   - [AWS] Event Selector - active selector used to filter sections;
   - [AWS] Events Name Activity - event activity table with event details;
   - [AWS] CloudTrail - graphical interpretation of generating events in the AWS service presented over time;

##### IAM
The following views have been placed in the "IAM" section:
   - [AWS] Navigation - navigation between sections;
   - [AWS] IAM Selector - active selector used to filter sections;
   - [AWS] IAM Details - the table contains AWS service users, configured login methods, account creation time and account assignment;
   - [AWS] User last login - user activity table containing the period from the last login depending on the login method;

##### Gateways
The following views have been placed in the Gateways section:
   - [AWS] Navigation - navigation between sections;
   - [AWS] Gateways Selector - active selector used to filter sections;
   - [AWS] Internet Gateway - details table of configured AWS Internet Gateways;
   - [AWS] Transit Gateways - details table of configured AWS Transit Gateways;
   - [AWS] Nat Gateway - details table of configured AWS Nat Gateways;

## Integration with Azure / o365

### Introduction
The goal of the integration is to create a single repository with aggregated information from multiple Azure / o365 accounts or subscriptions and presented in a readable way with the ability to search, analyze and generate reports.

### Scope of Integration

The scope of integration include:

1. User activity:

   - Event category,
   - Login status,
   - Client application,
   - Location,
   - Type of activity,
   - Login problems and their reasons.

2. Infrastructure Metrics:

   - Azure Monitor Metrics (or Metrics) is a platform service that provides a single source for monitoring Azure resources.
   - Application Insights is an extensible Application Performance Management (APM) service for web developers on multiple platforms and can be used for live web application monitoring - it automatically detects performance anomalies.


### System components
#### Logstash
Logstash is an event collector and executor of queries which, upon receipt, are initially processed and sent to the event buffer.

#### Kafka
Component that enables buffering of events before they are saved on ITRS Log Analytics Data servers. Kafka also has the task of storing data when the ITRS Log Analytics Data nodes are unavailable.

#### ITRS Log Analytics Data
The ITRS Log Analytics cluster is responsible for storing and sharing data.

#### ITRS Log Analytics GUI
ITRS Log Analytics GUI is a graphical tool for searching, analyzing and visualizing data. It has an alert module that can monitor the collected metrics and take action in the event of a breach of the permitted thresholds.

### Data sources
ITRS Log Analytics can access metrics from the Azure services via API. Service access can be configured with the same credentials if the account was configured with Azure AD.
Configuration procedures:

 - https://docs.microsoft.com/en-us/azure/active-directory/develop/howto-create-service-principal-portal

 - https://dev.loganalytics.io/documentation/Authorization/AAD-Setup

 - https://dev.applicationinsights.io/quickstart/


#### Azure Monitor datasource configuration
To enable an Azure Monitor data source, the following information from the Azure portal is required:
   -  Tenant Id (Azure Active Directory -> Properties -> Directory ID)
   -  Client Id (Azure Active Directory -> App Registrations -> Choose your app -> Application ID)
   -  Client Secret (Azure Active Directory -> App Registrations -> Choose your app -> Keys)
   -  Default Subscription Id (Subscriptions -> Choose subscription -> Overview -> Subscription ID)


#### Azure Insights datasource configuration
To enable an Azure Insights data source, the following information is required from the Azure portal: 
   -  Application ID
   -  API Key

### Azure Command-Line Interface
To verify the configuration and connect ITRS Log Analytics to the Azure cloud, it is recommended to use the Azure command line interface:

   -  https://docs.microsoft.com/en-us/cli/azure/?view=azure-cli-latest

This tool deliver a set of commands for creating and managing Azure resources. Azure CLI is available in Azure services and is designed to allow you to work quickly with Azure with an emphasis on automation.
Example command:
   -  Login to the Azure platform using azure-cli:

```basg
az login --service-principal -u $ (client_id) -p $ (client_secret) --tenant $ (tenant_id)
```


#### Permission
The following permissions are required to access the metrics:
   -  Logon,
   -  Geting a resource list with an ID (az resource list),
   -  Geting a list of metrics for a given resource (az monitor metrics list-definitions),
   -  Listing of metric values for a given resource and metric (az monitor metrics list).

### Service selection
The service is selected by launching the appropriate pipeline in Logstash collectors:
   -  Azure Meters
   -  Azure Application Insights
The collector's queries will then be properly adapted to the chosen service.

#### Azure Monitor metrics
Sample metrics:
   -  Microsoft.Compute / virtualMachines - Percentage CPU
   -  Microsoft.Network/networkInterfaces - Bytes sent
   -  Microsoft.Storage/storageAccounts - Used Capacity

The Logstash collector gets the metrics through the following commands:
   -  downloading a list of resources for a given account:
/usr/bin/az resource list
   -  downloading a list of resource-specific metrics:
/usr/bin/az monitor metrics list-definitions --resource $ (resource_id)
   -  for a given resource, downloading the metric value in the 1-minute interval
/usr/bin/az monitor metrics list --resource "$ (resource_id)" --metric "$ (metric_name)"

Azure Monitor metric list:

   -  https://docs.microsoft.com/en-us/azure/azure-monitor/essentials/metrics-supported 

The downloaded data is decoded by the filter logstash:

```conf
filter {
        ruby {
            code => "
              e = event.to_hash
              data = e['value'][0]['timeseries'][0]['data']
                for d in Array(data) do
                  new_event = LogStash::Event.new()
                  new_event.set('@timestamp', e['@timestamp'])
                  new_event.set('data', d)
                  new_event.set('namespace', e['namespace'])
                  new_event.set('resourceregion', e['resourceregion'])
                  new_event.set('resourceGroup', e['value'][0]['resourceGroup'])
                  new_event.set('valueUnit', e['value'][0]['unit'])
                  new_event.set('valueType', e['value'][0]['type'])
                  new_event.set('id', e['value'][0]['id'])
                  new_event.set('errorCode', e['value'][0]['errorCode'])
                  new_event.set('displayDescription', e['value'][0]['displayDescription'])
                  new_event.set('localizedValue', e['value'][0]['name']['localizedValue'])
                  new_event.set('valueName', e['value'][0]['name']['@value'])
                  new_event_block.call(new_event)
                end
              event.cancel()
           "
        }
        if "_rubyexception" in [tags] {
                drop {}
        }
        date {
                match => [ "[data][timeStamp]", "yyyy-MM-dd'T'HH:mm:ssZZ" ]
        }
        mutate {
                convert => {
                        "[data][count]" => "integer"
                        "[data][minimum]" => "integer"
                        "[data][total]" => "integer"
                        "[data][maximum]" => "integer"
                        "[data][average]" => "integer"
                }
        }
}
```

After processing, the obtained documents are saved to the Kafka topic using Logstash output:

```conf
output {
        kafka {
                bootstrap_servers => "localhost:9092"
                client_id => "gk-eslapp01v"
                topic_id => "azurelogs"
                codec => json
        }
}
```

#### Azure Application Insights metrics

Sample metrics:
   -  performanceCounters / exceptionsPerSecond
   -  performanceCounters / memoryAvailableBytes
   -  performanceCounters / processCpuPercentage
   -  performanceCounters / processIOBytesPerSecond
   -  performanceCounters / processPrivateBytes

Sample query:

   ```bash
   GET https://api.applicationinsights.io/v1/apps/[appIdarówka/metrics/ nutsmetricId}
   ```

Metrics List:

   - https://docs.microsoft.com/en-us/rest/api/application-insights/metrics/get 


### ITRS Log Analytics GUI

#### Metrics
Metric data is recorded in the monthly indexes:

   `azure-metrics -% {YYYY.MM}`

The pattern index in ITRS Log Analytics GUI is:

  `azure-metrics *`

ITRS Log Analytics Discover data is available using the saved search: "[Azure Metrics] Metrics Details"

![](/media/media/image232.png)

The analysis of the collected metrics is possible using the provided dashboard:

![](/media/media/image233.png)

On which the following views have been placed:

   -  **[Azure Metrics] Main Selector** - a selector that allows you to search by name and select a resource group, metric or namespace for a filter.
   -  **[Azure Metrics] Main Average** - a numeric field that calculates the average value of a selected metric;
   -  **[Azure Metrics] Main Median** - numeric field that calculates the median of the selected metric;
   -  **[Azure Metrics] Average Line** - a line chart of the value of the selected metric over time;
   -  **[Azure Metrics] Top Resource Group** - horizontal bar chart of resource groups with the most metrics
   -  **[Azure Metrics] Top Metrics** - horizontal bar chart, metrics with the largest amount of data
   -  **[Azure Metrics] Top Namespace** - horizontal bar chart, namespace with the most metrics
   -  **[Azure Metrics] Metrics Details** - table containing details / raw data;

Dashboard with an active filter:

![](/media/media/image234.png)

#### Events

Events are stored in the monthly indexes:

`azure_events -% {YYYY.MM}`

The index pattern in ITRS Log Analytics GUI is:

`azure_events *`

![](/media/media/image235.png)

Examples of fields decoded in the event:

The analysis of the collected events is possible using the provided dashboard:

![](/media/media/image236.png)

Componens:

   -  **[AZURE] Event category** - pie chart, division into event categories,
   -  **[AZURE] Login Status** - pie chart, login status breakdown,
   -  **[AZURE] User localtion** - map, location of logging in users,
   -  **[AZURE] Client App Type** - pie chart, division into client application type,
   -  **[AZURE] Client APP** - bar chart, the most used client application,
   -  **[AZURE] Top activity type** - pie chart, division into user activity type,
   -  **[AZURE] Client Top App** - table, the most frequently used client application,
   -  **[AZURE] Failed login reason** - save search, user access problems, raw data.

## F5 
The ITRS Log Analytics accepts data from the F5 system using the SYSLOG protocol. The F5 configuration procedure is as follows:  https://support.f5.com/csp/article/K13080 

To identify events from a specific source, add the following condition to the Logstash configuration file:

   ```conf
   filter {
        if "syslog" in [tags] {
                if [host] == "$IP" {
                    mutate {
                        add_tag => ["F5"]
                    }
                }
        }
   }
   ```

Where $IP is IP address of source system and each document coming from the address will be tagged with 'F5'
Using the assigned tag, the documents is send to the appropriate index:

   ```conf
   output {
     if "F5" in [tags] {
       elasticsearch {
         hosts => "https://localhost:9200"
         ssl => true
         ssl_certificate_verification => false
         index => "F5-%{+YYYY.MM.dd}"
         user => "logstash"
         password => "logstash"
       }
     }
   }
   ```

## Aruba Devices

The ITRS Log Analytics accepts data from the Aruba Devices system using the SYSLOG protocol. The Aruba Switches configuration procedure is as follows:  https://community.arubanetworks.com/browse/articles/blogviewer?blogkey=80765a47-fe42-4d69-b500-277217f5312e

To identify events from a specific source, add the following condition to the Logstash configuration file:

   ```conf
   filter {
        if "syslog" in [tags] {
                if [host] == "$IP" {
                    mutate {
                        add_tag => ["ArubaSW"]
                    }
                }
        }
   }
   ```

Where $IP is IP address of source system and each document coming from the address will be tagged with 'ArubaSW'
Using the assigned tag, the documents is send to the appropriate index:

   ```conf
   output {
     if "ArubaSW" in [tags] {
       elasticsearch {
         hosts => "https://localhost:9200"
         ssl => true
         ssl_certificate_verification => false
         index => "ArubaSW-%{+YYYY.MM.dd}"
         user => "logstash"
         password => "logstash"
       }
     }
   }
   ```

## Sophos Central

The ITRS Log Analytics accepts data from the Sophos Central system using the API interface. The Sophos Central configuration procedure is as follows: https://github.com/sophos/Sophos-Central-SIEM-Integration

Pipeline configuration in Logstash collector:

```conf
   input {
      exec {
          command => "/etc/lists/bin/Sophos-Central/siem.py -c /usr/local/Sophos-Central/config.ini -q"
          interval => 60
          codec => "json_lines"
      }
   }
   filter {
      date {
              match => [ "[data][created_at]", "UNIX_MS" ]
      }
   }
   output {
      elasticsearch {
          hosts => "http://localhost:9200"
          index => "sophos-central-%{+YYYY.MM}"
          user => "logstash"
          password => "logstash"
      }
   }
```
Example of `config.ini` file:

```conf
   /usr/local/Sophos-Central/config.ini
   [login]
   token_info = 'url: https://api4.central.sophos.com/gateway, x-api-key: dcaz, Authorization: Basic abdc'
   client_id = UUID
   client_secret = client-secrter
   tenant_id =
   auth_url = https://id.sophos.com/api/v2/oauth2/token
   api_host = api.central.sophos.com
   format = json
   filename = stdout
   endpoint = all
   address = /var/run/syslog
   facility = daemon
   socktype = udp
   state_file_path = siem_sophos.json
```

## FreeRadius
The ITRS Log Analytics accepts data from the FreeRadius system using the SYSLOG protocol. The FreeRadius configuration procedure is as follows:  https://wiki.freeradius.org/config/Logging

To identify events from a specific source, add the following condition to the Logstash configuration file:

   ```conf
   filter {
        if "syslog" in [tags] {
                if [host] == "$IP" {
                    mutate {
                        add_tag => ["FreeRadius"]
                    }
                }
        }
   }
   ```

Where $IP is IP address of source system and each document coming from the address will be tagged with 'FreeRadius'
Using the assigned tag, the documents is send to the appropriate index:

   ```conf
   output {
     if "FreeRadius" in [tags] {
       elasticsearch {
         hosts => "http://localhost:9200"
         index => "FreeRadius-%{+YYYY.MM.dd}"
         user => "logstash"
         password => "logstash"
       }
     }
   }
   ```

## Microsoft Advanced Threat Analytics
The ITRS Log Analytics accepts data from the Advanced Threat Analytics  system using the SYSLOG protocol with message in CEF format. The Advanced Threat Analytics  configuration procedure is as follows: https://docs.microsoft.com/pl-pl/advanced-threat-analytics/cef-format-sa

To identify events from a specific source, add the following condition to the Logstash configuration file:

   ```conf
   filter {
        if "syslog" in [tags] {
                if [host] == "$IP" {
                    mutate {
                        add_tag => ["ATA"]
                    }
                }
        }
   }
   ```


Where $IP is IP address of source system and each document coming from the address will be tagged with 'ATA'

The event is recognized and decoded:

```conf
filter {
   if [msg] =~ /CEF:/ {
       grok {
         keep_empty_captures => true
         named_captures_only => true
         remove_field => [
           "msg",
           "[cef][version]"
         ]
         match => {
           "msg" => [
             "^%{DATA} CEF:%{NUMBER:[cef][version]}\|%{DATA:[cef][device][vendor]}\|%{DATA:[cef][device][product]}\|%{DATA:[cef] [device][version]}\|%{DATA:[cef][sig][id]}\|%{DATA:[cef][sig][name]}\|%{DATA:[cef][sig][severity]}\|%{GREEDYDATA:[cef]   [extensions]}"
           ]
         }
       }
     }
   if "ATA" in [tags] {
       if [cef][extensions] {

         kv {
           source => "[cef][extensions]"
           remove_field => [
             "[cef][extensions]",
             "device_time"
           ]
           field_split_pattern => "\s(?=\w+=[^\s])"
           include_brackets => true
           transform_key => "lowercase"
           trim_value => "\s"
           allow_duplicate_values => true
         }
        if [json] {

           mutate {
             gsub => [
               "json", "null", '""',
               "json", ":,", ':"",'
             ]
           }

           json {
             skip_on_invalid_json => true
             source => "json"
             remove_field => [
               "json"
             ]
           }

         }
         mutate {
           rename => { "device_ip" => "[device][ip]" }
           rename => { "device_uid" => "[device][uid]" }
           rename => { "internalhost" => "[internal][host]" }
           rename => { "external_ip" => "[external][ip]" }
           rename => { "internalip" => "[internal][ip]" }
         }
       }
     }
   }
}
```

Using the assigned tag, the documents is send to the appropriate index:
   ```conf
   output {
     if "ATA" in [tags] {
       elasticsearch {
         hosts => "http://localhost:9200"
         index => "ATA-%{+YYYY.MM.dd}"
         user => "logstash"
         password => "logstash"
       }
     }
   }
   ```

## CheckPoint Firewalls
The ITRS Log Analytics accepts data from the CheckPoint Firewalls system using the SYSLOG protocol. The CheckPoint Firewalls configuration procedure is as follows: https://sc1.checkpoint.com/documents/SMB_R80.20/AdminGuides/Locally_Managed/EN/Content/Topics/Configuring-External-Log-Servers.htm?TocPath=Appliance%20Configuration%7CLogs%20and%20Monitoring%7C_____3


To identify events from a specific source, add the following condition to the Logstash configuration file:

   ```conf
   filter {
        if "syslog" in [tags] {
                if [host] == "$IP" {
                    mutate {
                        add_tag => ["CheckPoint"]
                    }
                }
        }
   }
   ```

Where $IP is IP address of source system and each document coming from the address will be tagged with 'CheckPoint'
Using the assigned tag, the documents is send to the appropriate index:

   ```conf
   output {
     if "F5BIGIP" in [tags] {
       elasticsearch {
         hosts => "http://localhost:9200"
         index => "CheckPoint-%{+YYYY.MM.dd}"
         user => "logstash"
         password => "logstash"
       }
     }
   }
   ```

## WAF F5 Networks Big-IP
The ITRS Log Analytics accepts data from the F5 system using the SYSLOG protocol. The F5 configuration procedure is as follows:  https://support.f5.com/csp/article/K13080 

To identify events from a specific source, add the following condition to the Logstash configuration file:

   ```conf
   filter {
        if "syslog" in [tags] {
                if [host] == "$IP" {
                    mutate {
                        add_tag => ["F5BIGIP"]
                    }
                }
        }
   }
   ```

Where $IP is IP address of source system and each document coming from the address will be tagged with 'F5'
Using the assigned tag, the documents is send to the appropriate index:

   ```conf
   output {
     if "F5BIGIP" in [tags] {
       elasticsearch {
         hosts => "https://localhost:9200"
         ssl => true
         ssl_certificate_verification => false
         index => "F5BIGIP-%{+YYYY.MM.dd}"
         user => "logstash"
         password => "logstash"
       }
     }
   }
   ```

## Infoblox DNS Firewall

The ITRS Log Analytics accepts data from the Infoblox system using the SYSLOG protocol. The Infoblox configuration procedure is as follows:  https://docs.infoblox.com/space/NAG8/22252249/Using+a+Syslog+Server#Specifying-Syslog-Servers

To identify and collect events from a Infoblox, is nessery to use Filebeat with `infoblox` module.
To run Filebeat with infoblox moduel run following commnds:

```bash
filebeat modules enable infoblox
```

Configure output section in `/etc/filebat/filebeat.yml` file:

```yml
output.logstash:
  hosts: ["127.0.0.1:5044"]
```

Test the configuration:

```bash
filebeat test config
```

and:

```bash
filebeat test output
```

## CISCO Devices
The ITRS Log Analytics accepts data from the Cisco devices - router, switch, firewall and access point using the SYSLOG protocol. The Cisco devices configuration procedure is as follows: https://www.ciscopress.com/articles/article.asp?p=426638&seqNum=3

To identify events from a specific source, add the following condition to the Logstash configuration file:

   ```conf
   filter {
        if "syslog" in [tags] {
                if [host] == "$IP" {
                    mutate {
                        add_tag => ["CISCO"]
                    }
                }
        }
   }
   ```

Where $IP is IP address of source system and each document coming from the address will be tagged with 'CISCO'.
Using the assigned tag, the documents is send to the appropriate index:

   ```conf
   output {
     if "CISCO" in [tags] {
       elasticsearch {
         hosts => "http://localhost:9200"
         index => "CISCO-%{+YYYY.MM.dd}"
         user => "logstash"
         password => "logstash"
       }
     }
   }
   ```

## Microsoft Windows Systems

The ITRS Log Analytics getting events from Microsoft Systems using the Winlogbeat agent. 

To identify and collect events from a Windows eventchannel, it is nessery to setup following parameters in `winlobeat.yml` configuration file.

```yml
   winlogbeat.event_logs:
   - name: Application
      ignore_older: 72h
   - name: Security
   - name: System

   #output.elasticsearch:
   # Array of hosts to connect to.
   #hosts: ["localhost:9200"]

   output.logstash:
   # The Logstash hosts
  hosts: ["$IP:5044"]
```
Where $IP is IP address of ITRS Log Analytics datanode.

## Linux Systems
The ITRS Log Analytics accepts data from the Linux systems using the SYSLOG protocol.

To identify events from a specific source, add the following condition to the Logstash configuration file:

   ```conf
   filter {
        if "syslog" in [tags] {
                if [host] == "$IP" {
                    mutate {
                        add_tag => ["LINUX"]
                    }
                }
        }
   }
   ```

Where $IP is IP address of source system and each document coming from the address will be tagged with 'LINUX'.
Using the assigned tag, the documents is send to the appropriate index:

   ```conf
   output {
     if "LINUX" in [tags] {
       elasticsearch {
         hosts => "http://localhost:9200"
         index => "LINUX-%{+YYYY.MM.dd}"
         user => "logstash"
         password => "logstash"
       }
     }
   }
   ```

## AIX Systems
The ITRS Log Analytics accepts data from the AIX systems using the SYSLOG protocol.

To identify events from a specific source, add the following condition to the Logstash configuration file:

   ```conf
   filter {
        if "syslog" in [tags] {
                if [host] == "$IP" {
                    mutate {
                        add_tag => ["AIX"]
                    }
                }
        }
   }
   ```

Where $IP is IP address of source system and each document coming from the address will be tagged with 'AIX'.
Using the assigned tag, the documents is send to the appropriate index:

   ```conf
   output {
     if "AIX" in [tags] {
       elasticsearch {
         hosts => "http://localhost:9200"
         index => "AIX-%{+YYYY.MM.dd}"
         user => "logstash"
         password => "logstash"
       }
     }
   }
   ```

## Microsoft Windows DNS, DHCP Service

The ITRS Log Analytics accepts data from the Microsoft DNS and DHCP services using the Filebeat agent. Servers

To identify and collect events from Microsoft DNS and DHCP services, is nessery to set correct path do log files.

Configure output section in `/etc/filebat/filebeat.yml` file:

```yml
filebeat.inputs:
- type: log
  paths:
    - c:\\Path_to_DNS_logs\*.log
```

```yml
output.logstash:
  hosts: ["127.0.0.1:5044"]
```

Test the configuration:

```bash
filebeat test config
```

and:

```bash
filebeat test output
```