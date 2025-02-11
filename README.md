# introduction

* This playbook makes setting up a multi node Kafka cluster with kraft protocol trivial. 
* It is deliberately in a single file to keep it easily readable.
* For the moment, only Debian/Ubuntu based and RedHat based targets are supported. 
* Sensible defaults were assumed in configuration files - all can be altered. 
* There is no authentication/encryption enabled.
* This playbook has been tested 
    * with Ansible 11.2.0 and Ansible-core 2.18.2
    * on Python 3.13.1 
    * against Ubuntu 24.04.1 and Alma Linux 9.5 targets
    * on x86_64 and aarch64 architectures

# important settings, defaults and variables

* By default:
    * Kafka will be set up to run under user `kafka`. This is defined under variable `kafka_user`.
    * Kafka will be installed in version 3.9.0. This is defined under variable `kafka_version`.
    * Kafka's application logs will be saved in `/var/log/kafka`. This is defined under variable `kafka_application_log_dir`.
        * No changes have been made to log4j configuration.
    * All kafka message logs will be saved in `/srv/kafka/messages`. This is defined in variable `kafka_message_log_dir`.

* `server.properties.j2` template file defaults:
    * EVERY broker is a broker and a controller and a quorum voter. This will be a problem for very large clusters if left as such after installation.
    * only plaintext communication gets set up. (To avoid questions about setting up client certificates and ACLs.)
    * Auto creation of topics is disabled. Change the value of `auto.create.topics.enable` if you want it.
    * Default number of partitions for new topics is equal to the number of brokers in the cluster. 
    * Default number of replicas for new topics is 1 for a single node "cluster" and 3 in multi-node clusters. 
    * Minimum in-sync replicas is set to 1 in a single node "cluster" and 2 in multi-node clusters. 

* `kafka.service.j2` template file defaults:
    * `User`, under which this service runs is specified in variable `kafka_user` and defaults to `kafka`.
    * Multiple environment variables are set:
        * `JMX_PORT` is set to 9001 for monitoring purposes.
        * Java heap memory settings:
            * initial heap size is set to total RAM minus 1024 MiB
            * maximum heap size is set to total RAM minus 1024 MiB
            * These are sensible defaults for a dedicated kafka brokers. If your brokers serve multiple purposes, alter these to leave more RAM for other tasks.
    * `LOG_DIR`, the directory where kafka messages are stored, is specified in variable `kafka_application_log_dir` and defaults to `/srv/kafka/messages`.
    * `LimitNOFILE` increases the low system default setting for maximum number of open file descriptors to 65535. This is needed on busy systems. 
    * `OOMScoreAdjust` gets set to -500 to decrease the chance of this being killed during out-of-memory events. 

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
