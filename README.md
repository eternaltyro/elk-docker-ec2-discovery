# Elasticsearch, Logstash, Kibana (ELK) Docker image documentation

This web page documents how to use the [sebp/elk](https://hub.docker.com/r/sebp/elk/) Docker image, which provides a convenient centralised log server and log management web interface, by packaging [Elasticsearch](http://www.elasticsearch.org/), [Logstash](http://logstash.net/), and [Kibana](http://www.elasticsearch.org/overview/kibana/), collectively known as ELK.

### Contents ###

- [Prerequisites](#prerequisites)
- [Installation](#installation)
- [Usage](#usage)
	- [Starting services selectively](#selective-services)
	- [Overriding start-up variables](#overriding-variables)
- [Forwarding logs](#forwarding-logs)
	- [Forwarding logs with Filebeat](#forwarding-logs-filebeat)
- [Building the image](#building-image)
- [Tweaking the image](#tweaking-image)
	- [Updating Logstash's configuration](#updating-logstash-configuration)
	- [Installing Elasticsearch plugins](#installing-elasticsearch-plugins)
	- [Installing Logstash plugins](#installing-logstash-plugins)
	- [Installing Kibana plugins](#installing-kibana-plugins)
- [Persisting log data](#persisting-log-data)
- [Setting up an Elasticsearch cluster](#elasticsearch-cluster)
	- [Running Elasticsearch nodes on different hosts](#elasticsearch-cluster-different-hosts)
	- [Running Elasticsearch nodes on a single host](#elasticsearch-cluster-single-host)
	- [Optimising your Elasticsearch cluster](#optimising-elasticsearch-cluster)
- [Security considerations](#security-considerations)
	- [Notes on certificates](#certificates)
	- [Disabling SSL/TLS](#disabling-ssl-tls)
- [Reporting issues](#reporting-issues)
- [References](#references)
- [About](#about)

## Prerequisites<a name="prerequisites"></a>

You need:
1. An amazon AWS account with web-console and/or API keys
2. IAM user with permissions to EC2, ECS.

- A minimum of 3GB RAM assigned to Docker - Elasticsearch alone needs at least 2GB of RAM to run.

- A limit on mmap counts equal to 262,144 or more

	On Linux, use `sysctl vm.max_map_count` on the host to view the current value. Note that the limits **must be changed on the host**; they cannot be changed from within a container.

## Port configuration - Security Groups, nACL, etc.

## Installation <a name="installation"></a>

To pull this image from the [Docker registry](https://hub.docker.com/r/eternaltyro/elk-aws-discovery/), open a shell prompt and enter:

	$ sudo docker pull sebp/elk

**Note** – This image has been built automatically from the source files in the [source Git repository on GitHub](https://github.com/eternaltyro/elk-docker-ec2-discovery). If you want to build the image yourself, see the [Building the image](#building-image) section.

This image has only been tested on ELK version 5.4.0.

## Usage <a name="usage"></a>

**Note** – The whole ELK stack will be started. See the *[Starting services selectively](#selective-services)* section to selectively start part of the stack.

This command publishes the following ports, which are needed for proper operation of the ELK stack:

- 5601 (Kibana web interface).
- 9200 (Elasticsearch JSON interface).
- 5044 (Logstash Beats interface, receives logs from Beats such as Filebeat – see the *[Forwarding logs with Filebeat](#forwarding-logs-filebeat)* section).

**Note** – The image also exposes Elasticsearch's transport interface on port 9300. Use the `-p 9300:9300` option with the `docker` command above to publish it. This transport interface is notably used by [Elasticsearch's Java client API](https://www.elastic.co/guide/en/elasticsearch/client/java-api/current/index.html), and to run Elasticsearch in a cluster.

### Starting services selectively <a name="selective-services"></a>

By default, when starting a container, all three of the ELK services (Elasticsearch, Logstash, Kibana) are started.

The following environment variables may be used to selectively start a subset of the services:

- `ELASTICSEARCH_START`: if set and set to anything other than `1`, then Elasticsearch will not be started.
 
- `LOGSTASH_START`: if set and set to anything other than `1`, then Logstash will not be started.

- `KIBANA_START`: if set and set to anything other than `1`, then Kibana will not be started.

For example, the following command starts Elasticsearch only:

	$ sudo docker run -p 5601:5601 -p 9200:9200 -p 5044:5044 -it \
		-e LOGSTASH_START=0 -e KIBANA_START=0 --name elk sebp/elk

Note that if the container is to be started with Elasticsearch _disabled_, then:

- If Logstash is enabled, then you need to make sure that the configuration file for Logstash's Elasticsearch output plugin (`/etc/logstash/conf.d/30-output.conf`) points to a host belonging to the Elasticsearch cluster rather than `localhost` (which is the default in the ELK image, since by default Elasticsearch and Logstash run together), e.g.:
	
		output {
		  elasticsearch { hosts => ["elk-master.example.com"] }
		}

- Similarly, if Kibana is enabled, then Kibana's `kibana.yml` configuration file must first be updated to make the `elasticsearch.url` setting (default value: `"http://localhost:9200"`) point to a running instance of Elasticsearch.

### Overriding start-up variables <a name="overriding-variables"></a>

The following environment variables can be used to override the defaults used to start up the services:

- `TZ`: the container's time zone (see [list of valid time zones](https://en.wikipedia.org/wiki/List_of_tz_database_time_zones)), e.g. `America/Los_Angeles` (default is `Etc/UTC`, i.e. UTC).

- `ES_HEAP_SIZE`: Elasticsearch heap size (default is 256MB min, 1G max)

	Specifying a heap size – e.g. `2g` – will set both the min and max to the provided value. To set the min and max values separately, see the `ES_JAVA_OPTS` below. 

- `ES_JAVA_OPTS`: additional Java options for Elasticsearch (default: `""`)
 
	For instance, to set the min and max heap size to 512MB and 2G, set this environment variable to `-Xms512m -Xmx2g`.

- `ES_CONNECT_RETRY`: number of seconds to wait for Elasticsearch to be up before starting Logstash and/or Kibana (default: `30`) 

- `CLUSTER_NAME`: the name of the Elasticsearch cluster (default: automatically resolved when the container starts if Elasticsearch requires no user authentication).

	The name of the Elasticsearch cluster is used to set the name of the Elasticsearch log file that the container displays when running. By default the name of the cluster is resolved automatically at start-up time (and populates `CLUSTER_NAME`) by querying Elasticsearch's REST API anonymously. However, when Elasticsearch requires user authentication (as is the case by default when running X-Pack for instance), this query fails and the container stops as it assumes that Elasticsearch is not running properly. Therefore, the `CLUSTER_NAME` environment variable can be used to specify the name of the cluster and bypass the (failing) automatic resolution.

- `LS_HEAP_SIZE`: Logstash heap size (default: `"500m"`)

- `LS_OPTS`: Logstash options (default: `"--auto-reload"` in images with tags `es231_l231_k450` and `es232_l232_k450`, `""` in `latest`; see [Breaking changes](#breaking-changes))

- `NODE_OPTIONS`: Node options for Kibana (default: `"--max-old-space-size=250"`)

- `MAX_MAP_COUNT`: limit on mmap counts (default: system default)

	**Warning** – This setting is system-dependent: not all systems allow this limit to be set from within the container, you may need to set this from the host before starting the container (see [Prerequisites](#prerequisites)). 

- `MAX_OPEN_FILES`: maximum number of open files (default: system default; Elasticsearch needs this amount to be equal to at least 65536)

As an illustration, the following command starts the stack, running Elasticsarch with a 2GB heap size, Logstash with a 1GB heap size and Logstash's configuration auto-reload disabled:

	$ sudo docker run -p 5601:5601 -p 9200:9200 -p 5044:5044 -it \
		-e ES_HEAP_SIZE="2g" -e LS_HEAP_SIZE="1g" -e LS_OPTS="--no-auto-reload" \
		--name elk sebp/elk

## Forwarding logs <a name="forwarding-logs"></a>

Forwarding logs from a host relies on a forwarding agent that collects logs (e.g. from log files, from the syslog daemon) and sends them to our instance of Logstash.

### Forwarding logs with Filebeat <a name="forwarding-logs-filebeat"></a>

Install [Filebeat](https://www.elastic.co/products/beats/filebeat) on the host you want to collect and forward logs from (see the *[References](#references)* section for links to detailed instructions).

**Note** – Make sure that the version of Filebeat is the same as the version of the ELK image.

#### Example Filebeat set-up and configuration

**Note** – The `nginx-filebeat` subdirectory of the [source Git repository on GitHub](https://github.com/spujadas/elk-docker) contains a sample `Dockerfile` which enables you to create a Docker image that implements the steps below.

Here is a sample `/etc/filebeat/filebeat.yml` configuration file for Filebeat, that forwards syslog and authentication logs, as well as [nginx](http://nginx.org/) logs.

	output:
	  logstash:
	    enabled: true
	    hosts:
	      - elk:5044
	    ssl:
		  certificate_authorities:
      	    - /etc/pki/tls/certs/logstash-beats.crt
	    timeout: 15
	
	filebeat:
	  prospectors:
	    -
	      paths:
	        - /var/log/syslog
	        - /var/log/auth.log
	      document_type: syslog
	    -
	      paths:
	        - "/var/log/nginx/*.log"
	      document_type: nginx-access

In the sample configuration file, make sure that you replace `elk` in `elk:5044` with the hostname or IP address of the ELK-serving host.

You'll also need to copy the `logstash-beats.crt` file (which contains the certificate authority's certificate – or server certificate as the certificate is self-signed – for Logstash's Beats input plugin; see [Security considerations](#security-considerations) for more information on certificates) from the [source repository of the ELK image](https://github.com/spujadas/elk-docker) to `/etc/pki/tls/certs/logstash-beats.crt`.

**Note** – Alternatively, when using Filebeat on a Windows machine, instead of using the `certificate_authorities` configuration option, the certificate from `logstash-beats.crt` can be installed in Windows' Trusted Root Certificate Authorities store. 

**Note** – The ELK image includes configuration items (`/etc/logstash/conf.d/11-nginx.conf` and `/opt/logstash/patterns/nginx`) to parse nginx access logs, as forwarded by the Filebeat instance above.

Before starting Filebeat for the first time, run this command (replace `elk` with the appropriate hostname) to load the default index template in Elasticsearch:

		curl -XPUT 'http://elk:9200/_template/filebeat?pretty' -d@/etc/filebeat/filebeat.template.json

Start Filebeat:

		sudo /etc/init.d/filebeat start

## Building the image <a name="building-image"></a>

To build the Docker image from the source files, first clone the [Git repository](https://github.com/eternaltyro/elk-docker-ec2-discovery), go to the root of the cloned directory (i.e. the directory that contains `Dockerfile`), and:

- If you're using the vanilla `docker` command then run `sudo docker build -t <repository-name> .`, where `<repository-name>` is the repository name to be applied to the image, which you can then use to run the image with the `docker run` command.

- If you're using Compose then run `sudo docker-compose build elk`, which uses the `docker-compose.yml` file from the source repository to build the image. You can then run the built image with `sudo docker-compose up`.

## Tweaking the image <a name="tweaking-image"></a>

There are several approaches to tweaking the image:

- Use the image as a base image and extend it, adding files (e.g. configuration files to process logs sent by log-producing applications, plugins for Elasticsearch) and overwriting files (e.g. configuration files, certificate and private key files) as required. See Docker's [Dockerfile Reference page](https://docs.docker.com/reference/builder/) for more information on writing a `Dockerfile`.

- Replace existing files by bind-mounting local files to files in the container. See Docker's [Manage data in containers](https://docs.docker.com/engine/userguide/containers/dockervolumes/) page for more information on volumes in general and bind-mounting in particular.

- Fork the source Git repository and hack away.

The next few subsections present some typical use cases.

### Updating Logstash's configuration <a name="updating-logstash-configuration"></a>

The image contains several configuration files for Logstash (e.g. `01-lumberjack-input.conf`, `02-beats-input.conf`), all located in `/etc/logstash/conf.d`.

To modify an existing configuration file, you can bind-mount a local configuration file to a configuration file within the container at runtime. For instance, if you want to replace the image's `30-output.conf` Logstash configuration file with your local file `/path/to/your-30-output.conf`, then you would add the following `-v` option to your `docker` command line:

	$ sudo docker run ... \
		-v /path/to/your-30-output.conf:/etc/logstash/conf.d/30-output.conf \
		...

To create your own image with updated or additional configuration files, you can create a `Dockerfile` that extends the original image, with contents such as the following:

	FROM sebp/elk
	
	# overwrite existing file
	ADD /path/to/your-30-output.conf /etc/logstash/conf.d/30-output.conf
	
	# add new file
	ADD /path/to/new-12-some-filter.conf /etc/logstash/conf.d/12-some-filter.conf

Then build the extended image using the `docker build` syntax. 

### Installing Elasticsearch plugins <a name="installing-elasticsearch-plugins"></a>

Elasticsearch's home directory in the image is `/opt/elasticsearch`, its [plugin management script](https://www.elastic.co/guide/en/elasticsearch/reference/current/modules-plugins.html) (`elasticsearch-plugin`) resides in the `bin` subdirectory, and plugins are installed in `plugins`.

Elasticsearch runs as the user `elasticsearch`. To avoid issues with permissions, it is therefore recommended to install Elasticsearch plugins as `elasticsearch`, using the `gosu` command (see below for an example, and references for further details).  

A `Dockerfile` like the following will extend the base image and install the [GeoIP processor plugin](https://www.elastic.co/guide/en/elasticsearch/plugins/master/ingest-geoip.html) (which adds information about the geographical location of IP addresses):

	FROM sebp/elk

	ENV ES_HOME /opt/elasticsearch
	WORKDIR ${ES_HOME}

	RUN gosu elasticsearch bin/elasticsearch-plugin install \
	    -Edefault.path.conf=/etc/elasticsearch ingest-geoip

You can now build the new image (see the *[Building the image](#building-image)* section above) and run the container in the same way as you did with the base image.

### Installing Logstash plugins <a name="installing-logstash-plugins"></a>

The name of Logstash's home directory in the image is stored in the `LOGSTASH_HOME` environment variable (which is set to `/opt/logstash` in the base image). Logstash's plugin management script (`logstash-plugin`) is located in the `bin` subdirectory.

Logstash runs as the user `logstash`. To avoid issues with permissions, it is therefore recommended to install Logstash plugins as `logstash`, using the `gosu` command (see below for an example, and references for further details).  

The following `Dockerfile` can be used to extend the base image and install the [RSS input plugin](https://www.elastic.co/guide/en/logstash/current/plugins-inputs-rss.html):

	FROM sebp/elk

	WORKDIR ${LOGSTASH_HOME}
	RUN gosu logstash bin/logstash-plugin install logstash-input-rss

See the *[Building the image](#building-image)* section above for instructions on building the new image. You can then run a container based on this image using the same command line as the one in the *[Usage](#usage)* section.

### Installing Kibana plugins <a name="installing-kibana-plugins"></a>

The name of Kibana's home directory in the image is stored in the `KIBANA_HOME` environment variable (which is set to `/opt/kibana` in the base image). Kibana's plugin management script (`kibana-plugin`) is located in the `bin` subdirectory, and plugins are installed in `installedPlugins`.

Kibana runs as the user `kibana`. To avoid issues with permissions, it is therefore recommended to install Kibana plugins as `kibana`, using the `gosu` command (see below for an example, and references for further details).  

The following `Dockerfile` can be used to extend the base image and install the latest version of the [Sense plugin](https://www.elastic.co/guide/en/sense/current/index.html), a handy console for interacting with the REST API of Elasticsearch:

	FROM sebp/elk

	WORKDIR ${KIBANA_HOME}
	RUN gosu kibana bin/kibana-plugin install elastic/sense

See the *[Building the image](#building-image)* section above for instructions on building the new image. You can then run a container based on this image using the same command line as the one in the *[Usage](#usage)* section. The Sense interface will be accessible at `http://<your-host>:5601/apss/sense` (e.g. [http://localhost:5601/app/sense](http://localhost:5601/app/sense) for a local native instance of Docker).

## Persisting log data <a name="persisting-log-data"></a>

In order to keep log data across container restarts, this image mounts `/var/lib/elasticsearch` — which is the directory that Elasticsearch stores its data in — as a volume.

You may however want to use a dedicated data volume to persist this log data, for instance to facilitate back-up and restore operations.

One way to do this is to mount a Docker named volume using `docker`'s `-v` option, as in:

	$ sudo docker run -p 5601:5601 -p 9200:9200  -p 5044:5044 \
		-v elk-data:/var/lib/elasticsearch --name elk sebp/elk

This command mounts the named volume `elk-data` to `/var/lib/elasticsearch` (and automatically creates the volume if it doesn't exist; you could also pre-create it manually using `docker volume create elk-data`).

**Note** – By design, Docker never deletes a volume automatically (e.g. when no longer used by any container). Whilst this avoids accidental data loss, it also means that things can become messy if you're not managing your volumes properly (e.g. using the `-v` option when removing containers with `docker rm` to also delete the volumes... bearing in mind that the actual volume won't be deleted as long as at least one container is still referencing it, even if it's not running). You can keep track of existing volumes using `docker volume ls`.

See Docker's page on [Managing Data in Containers](https://docs.docker.com/engine/userguide/containers/dockervolumes/) and Container42's [Docker In-depth: Volumes](http://container42.com/2014/11/03/docker-indepth-volumes/) page for more information on managing data volumes.

In terms of permissions, Elasticsearch data is created by the image's `elasticsearch` user, with UID 991 and GID 991.

There is a [known situation](https://github.com/spujadas/elk-docker/issues/69) where SELinux denies access to the mounted volume when running in _enforcing_ mode. The workaround is to use the `setenforce 0` command to run SELinux in _permissive_ mode.

## Setting up an Elasticsearch cluster <a name="elasticsearch-cluster"></a>

The ELK image can be used to run an Elasticsearch cluster, either on [separate hosts](#elasticsearch-cluster-different-hosts) or (mainly for test purposes) on a [single host](#elasticsearch-cluster-single-host), as described below.

For more (non-Docker-specific) information on setting up an Elasticsearch cluster, see the [Life Inside a Cluster section](https://www.elastic.co/guide/en/elasticsearch/guide/current/distributed-cluster.html) section of the Elasticsearch definitive guide.

### Running Elasticsearch nodes on different hosts <a name="elasticsearch-cluster-different-hosts"></a>

To run cluster nodes on different hosts, you'll need to update Elasticsearch's `/etc/elasticsearch/elasticsearch.yml` file in the Docker image so that the nodes can find each other:

- Configure the [zen discovery module](http://www.elastic.co/guide/en/elasticsearch/reference/current/modules-discovery.html), by adding a `discovery.zen.ping.unicast.hosts` directive to point to the IP addresses or hostnames of hosts that should be polled to perform discovery when Elasticsearch is started on each node.

- Set up the `network.*` directives as follows:

		network.host: 0.0.0.0
		network.publish_host: <reachable IP address or FQDN>

	where `reachable IP address` refers to an IP address that other nodes can reach (e.g. a public IP address, or a routed private IP address, but *not* the Docker-assigned internal 172.x.x.x address).

- Publish port 9300

As an example, start an ELK container as usual on one host, which will act as the first master. Let's assume that the host is called *elk-master.example.com*.

Have a look at the cluster's health:

	$ curl http://elk-master.example.com:9200/_cluster/health?pretty
	{
	  "cluster_name" : "elasticsearch",
	  "status" : "yellow",
	  "timed_out" : false,
	  "number_of_nodes" : 1,
	  "number_of_data_nodes" : 1,
	  "active_primary_shards" : 6,
	  "active_shards" : 6,
	  "relocating_shards" : 0,
	  "initializing_shards" : 0,
	  "unassigned_shards" : 6,
	  "delayed_unassigned_shards" : 6,
	  "number_of_pending_tasks" : 0,
	  "number_of_in_flight_fetch" : 0,
	  "task_max_waiting_in_queue_millis" : 0,
	  "active_shards_percent_as_number" : 50.0
	}

This shows that only one node is up at the moment, and the `yellow` status indicates that all primary shards are active, but not all replica shards are active.

Then, on another host, create a file named `elasticsearch-slave.yml` (let's say it's in `/home/elk`), with the following contents:

	network.host: 0.0.0.0
	network.publish_host: <reachable IP address or FQDN>
	discovery.zen.ping.unicast.hosts: ["elk-master.example.com"]

You can now start an ELK container that uses this configuration file, using the following command (which mounts the configuration files on the host into the container):

	$ sudo docker run -it --rm=true -p 9200:9200 -p 9300:9300 \
	  -v /home/elk/elasticsearch-slave.yml:/etc/elasticsearch/elasticsearch.yml \
	  sebp/elk

Once Elasticsearch is up, displaying the cluster's health on the original host now shows:

	$ curl http://elk-master.example.com:9200/_cluster/health?pretty
	{
	  "cluster_name" : "elasticsearch",
	  "status" : "green",
	  "timed_out" : false,
	  "number_of_nodes" : 2,
	  "number_of_data_nodes" : 2,
	  "active_primary_shards" : 6,
	  "active_shards" : 12,
	  "relocating_shards" : 0,
	  "initializing_shards" : 0,
	  "unassigned_shards" : 0,
	  "delayed_unassigned_shards" : 0,
	  "number_of_pending_tasks" : 0,
	  "number_of_in_flight_fetch" : 0,
	  "task_max_waiting_in_queue_millis" : 0,
	  "active_shards_percent_as_number" : 100.0
	}

### Running Elasticsearch nodes on a single host <a name="elasticsearch-cluster-single-host"></a>

Setting up Elasticsearch nodes to run on a single host is similar to running the nodes on different hosts, but the containers need to be linked in order for the nodes to discover each other.

Start the first node using the usual `docker` command on the host:

	$ sudo docker run -p 5601:5601 -p 9200:9200 -p 5044:5044 -it --name elk sebp/elk

Now, create a basic `elasticsearch-slave.yml` file containing the following lines:

	network.host: 0.0.0.0
	discovery.zen.ping.unicast.hosts: ["elk"]

Start a node using the following command:

	$ sudo docker run -it --rm=true \
	  -v /var/sandbox/elk-docker/elasticsearch-slave.yml:/etc/elasticsearch/elasticsearch.yml \
	  --link elk:elk --name elk-slave sebp/elk

Note that Elasticsearch's port is not published to the host's port 9200, as it was already published by the initial ELK container.

### Optimising your Elasticsearch cluster <a name="optimising-elasticsearch-cluster"></a>

You can use the ELK image as is to run an Elasticsearch cluster, especially if you're just testing, but to optimise your set-up, you may want to have:

- One node running the complete ELK stack, using the ELK image as is.

- Several nodes running _only_ Elasticsearch (see *[Starting services selectively](#selective-services)*).

An even more optimal way to distribute Elasticsearch, Logstash and Kibana across several nodes or hosts would be to run only the required services on the appropriate nodes or hosts (e.g. Elasticsearch on several hosts, Logstash on a dedicated host, and Kibana on another dedicated host).

## Security considerations <a name="security-considerations"></a>

As it stands this image is meant for local test use, and as such hasn't been secured: access to the ELK services is unrestricted, and default authentication server certificates and private keys for the Logstash input plugins are bundled with the image.

To harden this image, at the very least you would want to:

- Restrict the access to the ELK services to authorised hosts/networks only, as described in e.g. [Elasticsearch Scripting and Security](http://www.elasticsearch.org/blog/scripting-security/) and [Elastic Security: Deploying Logstash, ElasticSearch, Kibana "securely" on the Internet](http://blog.eslimasec.com/2014/05/elastic-security-deploying-logstash.html).
- Password-protect the access to Kibana and Elasticsearch (see [SSL And Password Protection for Kibana](http://technosophos.com/2014/03/19/ssl-password-protection-for-kibana.html)).
- Generate a new self-signed authentication certificate for the Logstash input plugins (see [Notes on certificates](#certificates)) or (better) get a proper certificate from a commercial provider (known as a certificate authority), and keep the private key private.

The [sebp/elkx](https://hub.docker.com/r/sebp/elkx/) image, which extends the ELK image with X-Pack, may be a useful starting point to improve the security of the ELK services.

If on the other hand you want to disable certificate-based server authentication (e.g. in a demo environment), see [Disabling SSL/TLS](#disabling-ssl-tls).

### Notes on certificates <a name="certificates"></a>

Dummy server authentication certificates (`/etc/pki/tls/certs/logstash-*.crt`) and private keys (`/etc/pki/tls/private/logstash-*.key`) are included in the image.

**Note** – For Logstash 2.4.0 a PKCS#8-formatted private key must be used (see *[Breaking changes](#breaking-changes)* for guidance).

The certificates are assigned to hostname `*`, which means that they will work if you are using a single-part (i.e. no dots) domain name to reference the server from your client.

**Example** – In your client (e.g. Filebeat), sending logs to hostname `elk` will work, `elk.mydomain.com` will not (will produce an error along the lines of `x509: certificate is valid for *, not elk.mydomain.com`), neither will an IP address such as 192.168.0.1 (expect `x509: cannot validate certificate for 192.168.0.1 because it doesn't contain any IP SANs`).

If you cannot use a single-part domain name, then you could consider:

- Issuing a self-signed certificate with the right hostname using a variant of the commands given below.

- Issuing a certificate with the [IP address of the ELK stack in the subject alternative name field](https://github.com/elastic/logstash-forwarder/issues/221#issuecomment-48390920), even though this is bad practice in general as IP addresses are likely to change.

- Adding a single-part hostname (e.g. `elk`) to your client's `/etc/hosts` file.    

The following commands will generate a private key and a 10-year self-signed certificate issued to a server with hostname `elk` for the Beats input plugin:

	$ cd /etc/pki/tls
	$ sudo openssl req -x509 -batch -nodes -subj "/CN=elk/" \
		-days 3650 -newkey rsa:2048 \
		-keyout private/logstash-beats.key -out certs/logstash-beats.crt

As another example, when running a non-predefined number of containers concurrently in a cluster with hostnames _directly_ under the `.mydomain.com` domain (e.g. `elk1.mydomain.com`, `elk2.mydomain.com`, etc.; _not_ `elk1.subdomain.mydomain.com`, `elk2.othersubdomain.mydomain.com` etc.), you could create a certificate assigned to the wildcard hostname `*.example.com` by using the following command (all other parameters are identical to the ones in the previous example).   

	$ cd /etc/pki/tls
	$ sudo openssl req -x509 -batch -nodes -subj "/CN=*.example.com/" \
		-days 3650 -newkey rsa:2048 \
		-keyout private/logstash-beats.key -out certs/logstash-beats.crt

To make Logstash use the generated certificate to authenticate to a Beats client, extend the ELK image to overwrite (e.g. using the `Dockerfile` directive `ADD`):

- the certificate file (`logstash-beats.crt`) with `/etc/pki/tls/certs/logstash-beats.crt`.
- the private key file (`logstash-beats.key`) with `/etc/pki/tls/private/logstash-beats.key`,

Additionally, remember to configure your Beats client to trust the newly created certificate using the `certificate_authorities` directive, as presented in [Forwarding logs with Filebeat](#forwarding-logs-filebeat).

### Disabling SSL/TLS <a name="disabling-ssl-tls"></a>

Certificate-based server authentication requires log-producing clients to trust the server's root certificate authority's certificate, which can be an unnecessary hassle in zero-criticality environments (e.g. demo environments, sandboxes).

To disable certificate-based server authentication, remove all `ssl` and `ssl`-prefixed directives (e.g. `ssl_certificate`, `ssl_key`) in Logstash's input plugin configuration files.

For instance, with the default configuration files in the image, replace the contents of `02-beats-input.conf` (for Beats emitters) with:

	input {
	  beats {
	    port => 5044
	  }
	}

## Frequently encountered issues <a name="frequent-issues"></a>

### Elasticsearch is not starting (1): `max virtual memory areas vm.max_map_count [65530] likely too low, increase to at least [262144]` <a name="es-not-starting-max-map-count"></a>

If the container stops and its logs include the message `max virtual memory areas vm.max_map_count [65530] likely too low, increase to at least [262144]`, then the limits on mmap counts are too low, see [Prerequisites](#prerequisites).

### Elasticsearch is not starting (2): `cat: /var/log/elasticsearch/elasticsearch.log: No such file or directory` <a name="es-not-starting-not-enough-memory"></a>

If Elasticsearch's logs are *not* dumped (i.e. you get the following message: `cat: /var/log/elasticsearch/elasticsearch.log: No such file or directory`), then Elasticsearch did not have enough memory to start, see [Prerequisites](#prerequisites). 

### Elasticsearch is not starting (3): bootstrap tests <a name="es-not-starting-bootstrap-tests"></a>

**As from version 5**, if Elasticsearch is no longer starting, i.e. the `waiting for Elasticsearch to be up (xx/30)` counter goes up to 30 and the container exits with `Couln't start Elasticsearch. Exiting.` _and_ Elasticsearch's logs are dumped, then read the recommendations in the logs and consider that they *must* be applied.

In particular, in case (1) above, the message `max virtual memory areas vm.max_map_count [65530] likely too low, increase to at least [262144]` means that the host's limits on mmap counts **must** be set to at least 262144.

### Elasticsearch is suddenly stopping after having started properly <a name="es-suddenly-stopping"></a>

With the default image, this is usually due to Elasticsearch running out of memory after the other services are started, and the corresponding process being (silently) killed.

As a reminder (see [Prerequisites](#prerequisites)), you should use no less than 3GB of memory to run the container... and possibly much more.

## Troubleshooting <a name="troubleshooting"></a>

**Important** – If you need help to troubleshoot the configuration of Elasticsearch, Logstash, or Kibana, regardless of where the services are running (in a Docker container or not), please head over to the [Elastic forums](https://discuss.elastic.co/). The troubleshooting guidelines below only apply to running a container using the ELK Docker image.

Here are a few pointers to help you troubleshoot your containerised ELK.

### AWS ECS Logging (drivers)


## Reporting issues <a name="reporting-issues"></a>

**Important** – For _non-Docker-related_ issues with Elasticsearch, Kibana, and Elasticsearch, report the issues on the appropriate [Elasticsearch](https://github.com/elastic/elasticsearch), [Logstash](https://github.com/elastic/logstash), or [Kibana](https://github.com/elastic/kibana) GitHub repository. 

You can report issues with this image using [GitHub's issue tracker](https://github.com/eternaltyro/elk-docker-ec2-discovery/issues).

Bearing in mind that the first thing I'll need to do is reproduce your issue, please provide as much relevant information (e.g. logs, configuration files, what you were expecting and what you got instead, any troubleshooting steps that you took, what _is_ working) as possible for me to do that.

[Pull requests](https://github.com/spujadas/elk-docker/pulls) are also welcome if you have found an issue and can solve it.

## References <a name="references"></a>

- [How To Install Elasticsearch, Logstash, and Kibana 4 on Ubuntu 14.04](https://www.digitalocean.com/community/tutorials/how-to-install-elasticsearch-logstash-and-kibana-4-on-ubuntu-14-04)
- [The Docker Book](http://www.dockerbook.com/)
- [The Logstash Book](http://www.logstashbook.com/)
- [Elastic's reference documentation](https://www.elastic.co/guide/index.html):
	- [Elasticsearch Reference](https://www.elastic.co/guide/en/elasticsearch/reference/current/index.html)
	- [Logstash Reference](https://www.elastic.co/guide/en/logstash/current/index.html)
	- [Kibana Reference](https://www.elastic.co/guide/en/kibana/current/index.html)
	- [Filebeat Reference](https://www.elastic.co/guide/en/beats/filebeat/current/index.html)
- [gosu, simple Go-based setuid+setgid+setgroups+exec](https://github.com/tianon/gosu), a convenient alternative to `USER` in Dockerfiles (see [Best practices for writing Dockerfiles](https://docs.docker.com/engine/userguide/eng-image/dockerfile_best-practices/#user))

## About <a name="about"></a>

Written by [Sébastien Pujadas](http://pujadas.net), released under the [Apache 2 license](http://www.apache.org/licenses/LICENSE-2.0).

Modified by eternaltyro.
