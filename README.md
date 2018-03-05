# Indexing
Indexing poc for iRODS

Ubuntu 16.04 LTS

Install and configure iRODS 4.2.2:
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

Install iRODS Audit (AMQP) Rule Engine Plugin:
```
sudo apt-get -y install irods-rule-engine-plugin-audit-amqp
```
