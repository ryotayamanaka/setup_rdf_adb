# Setup RDF Graph Server with Autonomous Database

Quick start:

- [Prerequisites](#Prerequisites)
- [Create Autonomous Database](#Create-Autonomous-Database)
- [Create Network for RDF Graph Server](#Create-Network-for-RDF-Graph-Server)
- [Create RDF Graph Server](#Create-RDF-Graph-Server)
- [Download Wallet](#Download-Wallet)
- [Modify Wallet](#Modify-Wallet)
- [Upload Wallet](#Upload-Wallet)
- [Create RDF Network](#Create-RDF-Network)
- [Run SPARQL Query](#Run-SPARQL-Query)

Appendix:

- [Change RDF Server Password](#Change-RDF-Server-Password)
- []()

# Quick Start

## Prerequisites

- Oracle Cloud account
- 1 SSH key pair (private / public key)
- 3 passwords
  - `<password_1>`: DB login (`ADMIN` user)
  - `<password_2>`: DB Connection Wallet
  - `<password_3>`: RDF Server login (`weblogic` user)

## Create Autonomous Database

Oracle Cloud console > Oracle Database > Autonomous Database > Create Autonomous Database

- Configure the database
  - Database name: `<db_name>` (e.g. `ADB1`)
  - Database version: `19c` or `21c` 
  - Password: `<password_1>`

## Create Network for RDF Graph Server

Oracle Cloud console > Networking > Virtual Cloud Networks > Start VCN Wizard > VCN with Internet Connectivity

- Configuration
  - VCN NAME: (e.g. `VCN1`)
  - The rest of itmes: Do not need to change

Public Subnet vcn01

- Add Ingress Rules
  - Source CIDR: `0.0.0.0/0`
  - Destination port range: `7002,8001`
  - Description: `For RDF Graph Server`

## Create RDF Graph Server

Oracle Cloud console > Marketplace > All Applications

- Image
  - Name: `Oracle RDF Graph Server and Query UI`
  - Version: `21.2.0`
- Configure Variables
  - Server available domain: Any domain
  - Server shape: Any shape
  - SSH public key: Paste the content of the public key (e.g. `ssh-rsa AAAAB3NzaC...`)
  - Existing virtual cloud network: The VCN created above
  - Existing subnet: The public subnet of the VCN

Please check the IP address of this compute VM via the Web console.

## Download Wallet

Oracle Cloud console > Oracle Database > Autonomous Database

Select the ADB created above > DB Connection

- Wallet Type: `Instance Wallet`

Download

- Password: `<password_2>`

## Modify Wallet

Upload the wallet to the RDF Server and run a script to add the user name and password to the wallet.
```
$ scp -i <key> ~/Downloads/Wallet_<db_name>.zip opc@<ip_address>:
$ ssh -i <key> opc@<ip_address>
$ mkdir wallet
$ cd wallet
$ unzip ../Wallet_<db_name>.zip
$ export JAVA_HOME=/usr/local/java/jdk1.8.0_221
$ /u01/app/oracle/middleware/wls12214/oracle_common/bin/mkstore
  -wrl /home/opc/wallet -createCredential <db_name>_high ADMIN <password_1>
Enter wallet password: <password_2>
```

Example:
```
$ scp -i .ssh/id_rsa ~/Downloads/Wallet_ADB1.zip opc@192.0.2.1:
$ ssh -i .ssh/id_rsa opc@192.0.2.1
$ mkdir wallet
$ cd wallet
$ unzip ../Wallet_ADB1.zip
$ export JAVA_HOME=/usr/local/java/jdk1.8.0_221
$ /u01/app/oracle/middleware/wls12214/oracle_common/bin/mkstore \
  -wrl /home/opc/wallet -createCredential adb1_high ADMIN <password_1>
Enter wallet password: <password_2>
```

Download the modified wallet.
```
$ zip ../wallet_with_cred.zip *
$ exit
$ scp -i <key> opc@<ip_address>:~/wallet_with_cred.zip ~/Downloads/
```

Example:
```
$ zip ../wallet_with_cred.zip *
$ exit
$ scp -i .ssh/id_rsa opc@192.0.2.1:~/wallet_with_cred.zip ~/Downloads/
```

## Upload Wallet

Sign in to the RDF Graph Server and Query UI

- URL: `https://<ip_address>:8001/orardf`
- User: `weblogic`
- Password: `welcome1` (This password should be changed in the latter step)

Data sources tab > Create > Wallet

- Zip file: Select `wallet_with_cred.zip`
- Name: Any name of this data source (e.g. `ADB1`)
- Wallet Service: The service modified above `<db_name>_high` (e.g. `adb1_high`)

## Create RDF Network

Data tab

- Data source: The data source created above (e.g. `ADB1`)

Data tab > RDF Network > Create network icon

- Network owner: `ADMIN`
- Network name: Any network name (e.g. `NETWORK1`)
- Tablespace: `DATA`

Data tab

- RDF network: The RDF network created above (e.g. `ADMIN.NETWORK1`)

Data tab > Import > Upload data icon

- Upload: Select `sample.nt`
- Staging table: Input any new table name (e.g. `SAMPLE_TABLE`)
- Overwrite: Off (unless the table name is used already)

Data tab > Import > Bulk load data icon

- RDF Data
  - Model: Input any model name (e.g. `SAMPLE_MODEL`)
  - Staging table owner: `ADMIN`
  - Staging table: Select the table created above (e.g. `SAMPLE_TABLE`)
- Options
  - All of the items: Do not need to change
- Event Trace
  - All of the items: Do not need to change

## Run SPARQL Query

Data tab > Directory Tree > RDF Objects > Regular models

Right click the model (e.g. `SAMPLE_MODEL`) > Open

Click **Execute** button to run the SPARQL query in the text box.

![rdf_query_ui](https://user-images.githubusercontent.com/4862919/122530814-d8a08f80-d059-11eb-9728-2d8bbbbd763b.jpg)

From a new browser tab, access the URL below to send a SPARQL query as a GET request.

```
https://<ip_address>:8001/orardf/api/v1/datasets/query?datasource=ADB1&datasetDef={"metadata":[{"networkOwner":"ADMIN","networkName":"NETWORK1","models":["SAMPLE_MODEL"]}]}&query=select ?s ?p ?o where { ?s ?p ?o} limit 10
```

# Appendix

## Change RDF Server Password

Access WebLogic Server console using Firefox.

- https://<ip_address>:7002/console

Login with the default password.

- Username: weblogic
- Password: welcome1

Go to `Security Realms` > `Users and Groups`, and select `weblogic`.

Change password from the `Passwords` tab.

Login to RDF Graph Server console with the new password.

## Load Triples with Graph Name

From RDF Server, upload data to a staging table as explained at [Create RDF Network](#Create-RDF-Network).

- Upload: Select `sample.nt`
- Staging table: Input any new table name (e.g. `SAMPLE_TABLE`)
- Overwrite: Off (unless the table name is used already)

Add a new column for a graph name to the staging table as follows.

Data tab > Import > Bulk load data icon

Oracle Cloud console > Oracle Database > Autonomous Database

Select the ADB created above > Tools > Open Database Actions

```
create view SAMPLE_VIEW as
select
  RDF$STC_SUB
, RDF$STC_PRED
, RDF$STC_OBJ
, 'http://togogenome.db.naro.affrc.go.jp/ontology/taxonomy' as RDF$STC_GRAPH
from SAMPLE_TABLE
```

Go back to RDF Server and import from the staging table (= a new view `V_STAGE`) to a model.

Data tab > Import > Bulk load data icon

- RDF Data
  - Model: Input any model name (e.g. `SAMPLE_MODEL`)
  - Staging table owner: `ADMIN`
  - Staging table: Select the table created above (e.g. `SAMPLE_VIEW`)
- Options
  - All of the items: Do not need to change
- Event Trace
  - All of the items: Do not need to change
