# Indexing
Indexing poc for iRODS

Ubuntu 16.04 LTS

1.) Install and configure iRODS 4.2.2:
```
wget -qO - https://packages.irods.org/irods-signing-key.asc | sudo apt-key add -
echo "deb [arch=amd64] https://packages.irods.org/apt/ $(lsb_release -sc) main" | sudo tee /etc/apt/sources.list.d/renci-irods.list
sudo apt-get update
sudo apt-get -y install irods-server irods-database-plugin-postgres postgresql
sudo -u postgres bash -c "psql -c \"create user irods with password 'testpassword';\""
sudo -u postgres bash -c "psql -c 'create database \"ICAT\";'"
sudo -u postgres bash -c "psql -c 'grant all privileges on database \"ICAT\" to irods;'"
sudo python /var/lib/irods/scripts/setup_irods.py < /var/lib/irods/packaging/localhost_setup_postgres.input
```

2.) Install and configure iRODS Audit (AMQP) Rule Engine Plugin:
```
sudo apt-get -y install irods-rule-engine-plugin-audit-amqp
```
After installing the plugin, `/etc/irods/server_config.json` needs to be configured to use the plugin.
Add a new stanza to the "rule_engines" array within `server_config.json`:
```json
            {
                "instance_name": "irods_rule_engine_plugin-audit_amqp-instance",
                "plugin_name": "irods_rule_engine_plugin-audit_amqp",
                "plugin_specific_configuration" : {
                     "amqp_location" : "ANONYMOUS@localhost:5672",
                     "amqp_options" : "",
                     "amqp_topic" : "audit_messages",
                     "pep_regex_to_match" : "audit_.*"
                 }
            },
```
Add the new `audit_` namespace to the "rule_engine_namespaces" array within `server_config.json`:
```
    "rule_engine_namespaces": [
        "", 
        "audit_"
    ], 
```
Restart the irods server.

3.) Verify Java8 is installed:
```
which java
java -version
```

4.) Install RabbitMQ:
```
curl -s https://packagecloud.io/install/repositories/rabbitmq/rabbitmq-server/script.deb.sh | sudo bash
wget https://packages.erlang-solutions.com/erlang-solutions_1.0_all.deb
sudo apt-get update
sudo apt-get -y install erlang
sudo apt-get -y install rabbitmq-server
sudo rabbitmq-plugins enable rabbitmq_amqp1_0
sudo rabbitmq-plugins enable rabbitmq_management
sudo service rabbitmq-server start
sudo rabbitmqctl add_user test test
sudo rabbitmqctl set_user_tags test administrator
sudo rabbitmqctl set_permissions -p / test ".*" ".*" ".*"

```
5.) Install, enable and start Elasticsearch:
```
wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo apt-key add -
sudo apt-get -y install apt-transport-https
echo "deb https://artifacts.elastic.co/packages/6.x/apt stable main" | sudo tee -a /etc/apt/sources.list.d/elastic-6.x.list
sudo apt-get update && sudo apt-get -y install elasticsearch
sudo systemctl enable elasticsearch.service
sudo service elasticsearch start
sudo service elasticsearch restart
curl http://localhost:9200
```

6.) Install and configure Logstash:
```
wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo apt-key add -
echo "deb https://artifacts.elastic.co/packages/6.x/apt stable main" | sudo tee -a /etc/apt/sources.list.d/elastic-6.x.list
sudo apt-get update && sudo apt-get -y install logstash
```
Create a conf file /etc/logstash/conf.d/irods_audit.conf 
```
sudo vi /etc/logstash/conf.d/irods_audit.conf
```
Insert the following text in the conf file that you created above
```
input {
    # Read the audit_messages queue messages using the amqp protocol
    rabbitmq {
      host => "localhost"
      queue => "audit_messages"
    }
}

filter {

    if "_jsonparsefailure" in [tags] {
        mutate {
                  gsub => [ "message", "[\\\\]","" ]
                  gsub => [ "message", ".*__BEGIN_JSON__", ""]
                  gsub => [ "message", "__END_JSON__", ""]

        }
        mutate { remove_tag => [ "tags", "_jsonparsefailure" ] }
        json { source => "message" }

    }


    # Parse the JSON message
    json {
        source       => "message"
        remove_field => ["message"]
    }

    # Replace @timestamp with the timestamp stored in time_stamp
    date {
        match => [ "time_stamp", "UNIX_MS" ]
    }

    # Convert select fields to integer
    mutate {
        convert => { "int" => "integer" }
        convert => { "int__2" => "integer" }
        convert => { "int__3" => "integer" }
        convert => { "file_size" => "integer" }
    }

}

output {
    # Write the output to elastic search under the irods_audit index.
    elasticsearch {
        hosts => ["localhost:9200"]
        index => "irods_audit"
    }
    stdout {
        codec => rubydebug {}
    }
}
```
Start Logstash
```
sudo service logstash start
```
To view all the PEPs now stored in Elasticsearch:
```
curl -XGET 'localhost:9200/irods_audit/_search?pretty' -H 'Content-Type: application/json' -d'
{
    "sort" : [
        {"@timestamp":{"order": "asc"}}
    ],
    "size" :10000,
    "query": {
        "match_all": {}
    }
}
'
```

To delete all Elasticsearch data and start over:
```
curl -X DELETE 'http://localhost:9200/_all'
```


