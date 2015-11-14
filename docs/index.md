# Elasticsearch, Logstash, Kibana (ELK) Docker image documentation

This web page documents how to use the [sebp/elk](https://hub.docker.com/r/sebp/elk/) Docker image, which provides a convenient centralised log server and log management web interface, by packaging [Elasticsearch](http://www.elasticsearch.org/), [Logstash](http://logstash.net/), and [Kibana](http://www.elasticsearch.org/overview/kibana/), collectively known as ELK.

### Contents ###

- [Installation](#installation)
- [Usage](#usage)
	- [Running the container using Docker Compose](#running-with-docker-compose)
	- [Creating a dummy log entry](#creating-dummy-log-entry)
- [Forwarding logs](#forwarding-logs)
	- [Linking a Docker container to the ELK container](#linking-containers)
- [Building the image](#building-image)
- [Extending the image](#extending-image)
	- [Installing Elasticsearch plugins](#installing-elasticsearch-plugins)
	- [Installing Logstash plugins](#installing-logstash-plugins)
- [Storing log data](#storing-log-data)
- [Security considerations](#security-considerations)
- [References](#references)
- [About](#about)

## Installation <a name="installation"></a>

Install [Docker](https://docker.com/), either using a native package (Linux) or wrapped in a virtual machine (Windows, OS X – e.g. using [Boot2Docker](http://boot2docker.io/) or [Vagrant](https://www.vagrantup.com/)).

To pull this image from the [Docker registry](https://hub.docker.com/r/sebp/elk/), open a shell prompt and enter:

	$ sudo docker pull sebp/elk

**Note** – This image has been built automatically from the source files in the [source Git repository on GitHub](https://github.com/spujadas/elk-docker). If you want to build the image yourself, see the *[Building the image](#building-image)* section below.

**Note** – The last image using the Elasticsearch 1.x and Logstash 1.x branches can be pulled using `sudo docker pull sebp/elk:E1L1K4`. The versions of the ELK components in that image are: Elasticsearch 1.7.3, Logstash 1.5.5, and Kibana 4.1.2.

## Usage <a name="usage"></a>

Run the container from the image with the following command:

	$ sudo docker run -p 5601:5601 -p 9200:9200 -p 5000:5000 -it --name elk sebp/elk

This command publishes the following ports, which are needed for proper operation of the ELK stack:

- 5601 (Kibana web interface).
- 9200 (Elasticsearch JSON interface).
- 5000 (Logstash server, receives logs from Logstash forwarders – see the *[Forwarding logs](#forwarding-logs)* section below).

**Note** – The image also exposes Elasticsearch's transport interface on port 9300. Use the `-p 5300:5300` option with the `docker` command above to publish it.

The figure below shows how the pieces fit together.

![](http://i.imgur.com/RVW12Md.png)

Access Kibana's web interface by browsing to `http://<your-host>:5601`, where `<your-host>` is the hostname or IP address of the host Docker is running on (see note), e.g. `localhost` if running a local native version of Docker, or the IP address of the virtual machine if running a VM-hosted version of Docker (see note).

**Note** – To configure and/or find out the IP address of a VM-hosted Docker installation, see [https://docs.docker.com/installation/windows/](https://docs.docker.com/installation/windows/) (Windows) and [https://docs.docker.com/installation/mac/](https://docs.docker.com/installation/mac/) (OS X) for guidance if using Boot2Docker. If you're using [Vagrant](https://www.vagrantup.com/), you'll need to set up port forwarding (see [https://docs.vagrantup.com/v2/networking/forwarded_ports.html](https://docs.vagrantup.com/v2/networking/forwarded_ports.html).

You can stop the container with `^C`, and start it again with `sudo docker start elk`.

As from Kibana version 4.0.0, you won't be able to see anything (not even an empty dashboard) until something has been logged (see the *[Creating a dummy log entry](#creating-dummy-log-entry)* sub-section below on how to test your set-up, and the *[Forwarding logs](#forwarding-logs)* section on how to forward logs from regular applications).

### Running the container using Docker Compose <a name="running-with-docker-compose"></a>

If you're using [Docker Compose](https://docs.docker.com/compose/) to manage your Docker services (and if not you really should as it will make your life much easier!), then you can create an entry for the ELK Docker image by adding the following lines to your `docker-compose.yml` file:

	elk:
	  image: sebp/elk
	  ports:
	    - "5601:5601"
	    - "9200:9200"
	    - "5000:5000"

You can then start the ELK container like this:

	$ sudo docker-compose up elk

### Creating a dummy log entry <a name="creating-dummy-log-entry"></a>

If you haven't got any logs yet and want to manually create a dummy log entry for test purposes (for instance to see the dashboard), first start the container as usual (`sudo docker run ...` or `docker-compose up ...`).

In another terminal window, find out the name of the container running ELK, which is displayed in the last column of the output of the `sudo docker ps` command.

	$ sudo docker ps
	CONTAINER ID        IMAGE                  ...   NAMES
	86aea21cab85        elkdocker_elk:latest   ...   elkdocker_elk_1

Open a shell prompt in the container and type (replacing `<container-name>` with the name of the container, e.g. `elkdocker_elk_1` in the example above):

	$ sudo docker exec -it <container-name> /bin/bash

At the prompt, enter:

_(before Logstash 2.0.0)_

	# /opt/logstash/bin/logstash -e 'input { stdin { } } output { elasticsearch { host => localhost } }'

_(since Logstash 2.0.0)_

	# /opt/logstash/bin/logstash -e 'input { stdin { } } output { elasticsearch { hosts => ["localhost"] } }'

Wait for Logstash to start (as indicated by the message `Logstash startup completed`), then type some dummy text followed by Enter to create a log entry:

	this is a dummy entry

**Note** - You can create as many entries as you want. Use `^C` to go back to the bash prompt.

If you browse to `http://<your-host>:9200/_search?pretty` (e.g. [http://localhost:9200/_search?pretty](http://localhost:9200/_search?pretty) for a local native instance of Docker) you'll see that Elasticsearch has indexed the entry:

	{
	  ...
	  "hits": {
	    ...
	    "hits": [ {
	      "_index": "logstash-...",
	      "_type": "logs",
	      ...
	      "_source": { "message": "this is a dummy entry", "@version": "1", "@timestamp": ... }
	    } ]
	  }
	}

You can now browse to Kibana's web interface at `http://<your-host>:5601` (e.g. [http://localhost:5601](http://localhost:5601) for a local native instance of Docker).

Make sure that the drop-down "Time-field name" field is pre-populated with the value `@timestamp`, then click on "Create", and you're good to go.

## Forwarding logs <a name="forwarding-logs"></a>

Forwarding logs from a host relies on a Logstash forwarder agent that collects logs (e.g. from log files, from the syslog daemon) and sends them to our instance of Logstash.

Install [Logstash forwarder](https://github.com/elasticsearch/logstash-forwarder) on the host you want to collect and forward logs from (see the *[References](#references)* section below for links to detailed instructions).

Here is a sample configuration file for Logstash forwarder, that forwards syslog and authentication logs, as well as [nginx](http://nginx.org/) logs.

	{
	  "network": {
	    "servers": [ "elk:5000" ],
	    "timeout": 15,
	    "ssl ca": "/etc/pki/tls/certs/logstash-forwarder.crt"
	  },
	  "files": [
	    {
	      "paths": [
	        "/var/log/syslog",
	        "/var/log/auth.log"
	       ],
	      "fields": { "type": "syslog" }
	    },
	    {
	      "paths": [
	        "/var/log/nginx/access.log"
	       ],
	      "fields": { "type": "nginx-access" }
	    }
	   ]
	}

By default (see `/etc/init.d/logstash-forwarder` if you need to tweak anything):

- The Logstash forwarder configuration file must be located in `/etc/logstash-forwarder`.
- The Logstash forwarder needs a syslog daemon (e.g. rsyslogd, syslog-ng) to be running.

In the sample configuration file, make sure that you:

- Replace `elk` in `elk:5000` with the hostname or IP address of the ELK-serving host.
- Copy the `logstash-forwarder.crt` file (which contains the Logstash server's certificate) from the ELK image to `/etc/pki/tls/certs/logstash-forwarder.crt`.

**Note** – The ELK image includes configuration items (`/etc/logstash/conf.d/11-nginx.conf` and `/opt/logstash/patterns/nginx`) to parse nginx access logs, as forwarded by the Logstash forwarder instance above.

### Linking a Docker container to the ELK container <a name="linking-containers"></a>

If you want to forward logs from a Docker container to the ELK container, then you need to link the two containers.

**Note** – The log-emitting Docker container must have a Logstash forwarder agent running in it for this to work.

First of all, give the ELK container a name (e.g. `elk`) using the `--name` option:

	$ sudo docker run -p 5601:5601 -p 9200:9200 -p 5000:5000 -it --name elk sebp/elk

Then start the log-emitting container with the `--link` option (replacing `your/image` with the name of the Logstash-forwarder-enabled image you're forwarding logs from):

	$ sudo docker run -p 80:80 -it --link elk:elk your/image

From the perspective of the log emitting container, the ELK container is now known as `elk`, which is the hostname to be used in the `logstash-forwarder` configuration file.

With Compose here's what example entries for a (locally built log-generating) container and an ELK container might look like in the `docker-compose.yml` file.

	yourapp:
	  image: your/image
	  ports:
	    - "80:80"
	  links:
	    - elk

	elk:
	  image: sebp/elk
	  ports:
	    - "5601:5601"
	    - "9200:9200"
	    - "5000:5000"

## Building the image <a name="building-image"></a>

To build the Docker image from the source files, first clone the [Git repository](https://github.com/spujadas/elk-docker), go to the root of the cloned directory (i.e. the directory that contains `Dockerfile`), and:

- If you're using the vanilla `docker` command then run `sudo docker build -t <repository-name> .`, where `<repository-name>` is the repository name to be applied to the image, which you can then use to run the image with the `docker run` command.

- If you're using Compose then run `sudo docker-compose build elk`, which uses the `docker-compose.yml` file from the source repository to build the image. You can then run the built image with `sudo docker-compose up`.


## Extending the image <a name="extending-image"></a>

To extend the image, you can either fork the source Git repository and hack away, or – more in the spirit of the Docker philosophy – use the image as a base image and build on it, adding files (e.g. configuration files to process logs sent by log-producing applications, plugins for Elasticsearch) and overwriting files (e.g. configuration files, certificate and private key files) as required.

To create a new image based on this base image, you want your `Dockerfile` to include:

	FROM sebp/elk

followed by instructions to extend the image (see Docker's [Dockerfile Reference page](https://docs.docker.com/reference/builder/) for more information).

The next few subsections present some typical use cases.

### Installing Elasticsearch plugins <a name="installing-elasticsearch-plugins"></a>

Elasticsearch's home directory in the image is `/usr/share/elasticsearch`, its [plugin management script](https://www.elastic.co/guide/en/elasticsearch/reference/current/modules-plugins.html) (`plugin`) resides in the `bin` subdirectory, and plugins are installed in `plugins`.

A `Dockerfile` like the following will extend the base image and install Elastic HQ, a management and monitoring plugin for Elasticsearch, using `plugin`.

	FROM sebp/elk

	ENV ES_HOME /usr/share/elasticsearch
	WORKDIR ${ES_HOME}

	RUN bin/plugin -i royrusso/elasticsearch-HQ

You can now build the new image (see the *[Building the image](#building-image)* section above) and run the container in the same way as you did with the base image. The Elastic HQ interface will be accessible at `http://<your-host>:9200/_plugin/HQ/` (e.g. [http://localhost:9200/_plugin/HQ/](http://localhost:9200/_plugin/HQ/) for a local native instance of Docker).

### Installing Logstash plugins <a name="installing-logstash-plugins"></a>

The name of Logstash's home directory in the image is stored in the `LOGSTASH_HOME` environment variable (which is set to `/opt/logstash` in the base image). Logstash's plugin management script (`plugin`) is located in the `bin` subdirectory.

The following `Dockerfile` can be used to extend the base image and install the [RSS input plugin](https://www.elastic.co/guide/en/logstash/current/plugins-inputs-rss.html):

	FROM sebp/elk

	WORKDIR ${LOGSTASH_HOME}
	RUN bin/plugin install logstash-input-rss

See the *[Building the image](#building-image)* section above for instructions on building the new image. You can then run a container based on this image using the same command line as the one in the *[Usage](#usage)* section.

## Storing log data <a name="storing-log-data"></a>

In order to keep log data across container restarts, this image mounts `/var/lib/elasticsearch` — which is the directory that Elasticsearch stores its data in — as a volume.

You may however want to use a dedicated data volume to store this log data, for instance to facilitate back-up and restore operations.

One way to do this with the `docker` command-line tool is to first create a named container called `elk_data` with a bound Docker volume by using the `-v` option:

	$ sudo docker run -p 5601:5601 -p 9200:9200 -5000:5000 -v /var/lib/elasticsearch --name elk_data sebp/elk

You can now reuse the persistent volume from that container using the `--volumes-from` option:

	$ sudo docker run -p 5601:5601 -p 9200:9200 -p 5000:5000 --volumes-from elk_data --name elk sebp/elk

**Note** – By design, Docker never deletes a volume automatically (e.g. when no longer used by any container). Whilst this avoids accidental data loss, it also means that things can become messy if you're not managing your volumes properly (i.e. using the `-v` option when removing containers with `docker rm` to also delete the volumes... bearing in mind that the actual volume won't be deleted as long as at least one container is still referencing it, even if it's not running). As of this writing, managing Docker volumes can be a bit of a headache, so you might want to have a look at [docker-cleanup-volumes](https://github.com/chadoe/docker-cleanup-volumes), a shell script that deletes unused Docker volumes.

See Docker's page on [Managing Data in Containers](https://docs.docker.com/userguide/dockervolumes/) and Container42's [Docker In-depth: Volumes](http://container42.com/2014/11/03/docker-indepth-volumes/) page for more information on managing data volumes.

## Security considerations <a name="security-considerations"></a>

As it stands this image is meant for local test use, and as such hasn't been secured: access to the ELK services is not restricted, and a default authentication server certificate (`logstash-forwarder.crt`) and private key (`logstash-forwarder.key`) are bundled with the image.

To harden this image, at the very least you would want to:

- Restrict the access to the ELK services to authorised hosts/networks only, as described in e.g. [Elasticsearch Scripting and Security](http://www.elasticsearch.org/blog/scripting-security/) and [Elastic Security: Deploying Logstash, ElasticSearch, Kibana "securely" on the Internet](http://blog.eslimasec.com/2014/05/elastic-security-deploying-logstash.html).
- Password-protect the access to Kibana and Elasticsearch (see [SSL And Password Protection for Kibana](http://technosophos.com/2014/03/19/ssl-password-protection-for-kibana.html)).
- Generate a new self-signed authentication certificate for the Logstash server (`cd /etc/pki/tls; sudo openssl req -x509 -batch -nodes -days 3650 -newkey rsa:2048 -keyout private/logstash-forwarder.key -out certs/logstash-forwarder.crt` for a 10-year certificate) or (better) get a proper certificate from a commercial provider (known as a certificate authority), and keep the private key private.

## References <a name="references"></a>

- [How To Install Elasticsearch, Logstash, and Kibana 4 on Ubuntu 14.04](https://www.digitalocean.com/community/tutorials/how-to-install-elasticsearch-logstash-and-kibana-4-on-ubuntu-14-04)
- [The Docker Book](http://www.dockerbook.com/)
- [The Logstash Book](http://www.logstashbook.com/)
- [Elastic's reference documentation](https://www.elastic.co/guide/index.html)

## About <a name="about"></a>

Written by [Sébastien Pujadas](http://pujadas.net), released under the [Apache 2 license](http://www.apache.org/licenses/LICENSE-2.0).
