# introduction

* This playbook makes setting up a multi node Kafka cluster with kraft protocol trivial. 
* It is deliberately in a single file to keep it easily readable.
    * The variables are also in the playbook files for that reason.
* For the moment, only Debian/Ubuntu based and RedHat based targets are supported. 
* Sensible defaults were assumed in configuration files - all can be altered. 
* There is no authentication/encryption enabled.
* This playbook has been tested 
    * with Ansible 11.3.0 and Ansible-core 2.18.3
    * on Python 3.13.2 
    * against Ubuntu 24.04.1 and Rocky Linux 9.5 targets
    * on x86_64 and aarch64 architectures

# important settings, defaults and variables

General defaults and settables:

item | variable | default | where used
-----|-----------|--------|----------
kafka version | `kafka_version` | 4.0.0 | main playbook
kafka's message logs | `kafka_message_log_dir` | /srv/kafka/messages | main playbook
ports to open in firewalld (RedHat only) | `kafka_ports` | 9092/tcp 9093/tcp | main playbook
firewalld zone in which to open ports (RedHat only) | `kafka_firewalld_zone` | public | main playbook
each target host's role | no variable | broker AND controller | `server.properties.j2`
authentication | no variable | no authentication | `server.properties.j2`
encryption | no variable | PLAINTEXT only | `server.properties.j2`
topic auto creation | `auto.create.topics.enable` | false | `server.properties.j2`
number of partitions for new topics | no variable | equal to number of brokers | `server.properties.j2`
replication factor for new topics | no variable | 1 for single node; 3 for multi-node clusters | `server.properties.j2`
minimum in-sync replicas | no variable | 1 in single node mode; 2 for multi-node clusters | `server.properties.j2`
user under which kafka runs | `kafka_user` | kafka | `kafka.service.j2`
java management extension port for monitoring | `JMX_PORT` | 9001 | `kafka.service.j2`
java heap memory settings - initial heap size | no variable | total RAM minus 1024MB | `kafka.service.j2`
java heap memory settings - maximum heap size | no variable | total RAM minus 1024MB | `kafka.service.j2`
kafka's application logs (env LOG_DIR) | `kafka_application_log_dir` | /var/log/kafka | `kafka.service.j2`
maximum number of open file descriptors (env LimitNOFILE) | no variable | 65535 | `kafka.service.j2`
adjustment to OOM score (env OOMScoreAdjust) | no variable | -500 | `kafka.service.j2`

# how to use

## set up ansible

You only need to do this once.

```
apt install python3-venv build-essential
mkdir -p ~/bin/venvs
cd ~/bin/venvs
python3 -m venv ansible
source ansible/bin/activate
# Your prompt has changed to indicate that you are now in a virtual Python environment.
python3 -m pip install --upgrade pip
python3 -m pip install ansible
# Next line is only needed if you use Amazon AWS.
python3 -m pip install boto3 botocore
ansible --version
```

## run the playbook

* If you do not agree with any of the defaults described above, alter the variables in the playbook/templates OR (preferably) set them in the inventory file.
* Put your targets in the inventory file. (i.e.: `inventory.txt`)
* Activate the Python virtual environment you have set up in the **set up ansible** stanza
    * `source ~/bin/venvs/ansible/bin/activate`
    * Your prompt will change.
* Run the playbook:
    * `ansible-playbook -i inventory.txt setup_kafka_cluster.yml`
    * To confirm every single step of its execution and verify on the target machines, you can append `--step`
* Deactivate the Python virtual environment:
    * `deactivate`
    * Your prompt will return to normal.

# suggestions, tips, limitations

* For large clusters:
    * This playbook sets up all nodes as brokers and controllers. Ideally, no more than 5 controllers are ever needed and 3 are usually enough. (The number should be odd.)
    * If you have an extremely large cluster, keep 3 or 5 **dedicated** nodes as controllers-only and the rest as brokers-only.
    * You can still use this playbook to set up the nodes, just make sure that after the initial setup, you remove unneeded roles from certain nodes. This is mostly just `process.roles` and `controller.quorum.voters`.

# TODO
* Provision for extending an existing cluster. At the moment there is no provision for fetching cluster.id from meta.properties and reusing that on new nodes. 
