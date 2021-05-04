# Setup RDF Graph Server with Autonomous Database

## Prerequisites

- Oracle Cloud account
- 1 SSH key pair (private / public key)
- 3 passwords (DB admin, Wallet, RDF Server)

## Create Autonomous Database

Oracle Cloud console > Oracle Database > Autonomous Database > Create Autonomous Database

- Configure the database
  - Database name: `<db_name>` (e.g. adw01)
  - Database version: 21c 
  - Password: `<password_1>`

## Create Network for RDF Graph Server

Oracle Cloud console > Networking > Virtual Cloud Networks > Start VCN Wizard > VCN with Internet Connectivity

- Configuration
  - VCN NAME: (e.g. vcn01)
  - The rest of itmes: Do not need to change

Public Subnet vcn01

- Add Ingress Rules
  - Source CIDR: `0.0.0.0/0`
  - Destination port range: `7002,8001`
  - Description: `For RDF Server`

## Create RDF Graph Server

Oracle Cloud console > Marketplace > All Applications

- Image
  - Name: Oracle RDF Graph Server and Query UI
  - Version: 21.1.0.1

- Configure Variables
  - Server available domain: Any domain
  - Server shape: Any shape
  - SSH public key: Paste the content of the public key (e.g. `ssh-rsa AAAAB3NzaC...`)
  - Existing virtual cloud network: The VCN created above
  - Existing subnet: The public subnet of the VCN

## Download Wallet

Oracle Cloud console > Oracle Database > Autonomous Database

Select the ADW created above > DB Connection

- Wallet Type: `Instance Wallet`

Download

- Password: `<password_2>`

## Modify Wallet

Upload the wallet to the RDF Server and run a script to add the user name and password to the wallet.
```
scp -i <key> ~/Downloads/Wallet_<db_name>.zip opc@<ip_address>:
ssh -i <key> opc@<ip_address>
mkdir wallet
cd wallet
unzip ../Wallet_<db_name>.zip
export JAVA_HOME=/usr/local/java/jdk1.8.0_221
/u01/app/oracle/middleware/wls12214/oracle_common/bin/mkstore -wrl /home/opc/wallet -createCredential <db_name>_high ADMIN <password_1>
Enter wallet password: <password_2>
```

Example:
```
scp -i ssh/key/oci ~/Downloads/Wallet_ADW1.zip opc@111.222.333.444:
ssh -i ssh/key/oci opc@111.222.333.444
mkdir wallet
cd wallet
unzip ../Wallet_ADW1.zip
export JAVA_HOME=/usr/local/java/jdk1.8.0_221
/u01/app/oracle/middleware/wls12214/oracle_common/bin/mkstore -wrl /home/opc/wallet -createCredential RDF_high ADMIN WELcome123##
Enter wallet password: [WELcome123##]
```

Download the modified wallet.
```
zip ../wallet_with_cred.zip *
exit
scp -i <key> opc@<ip_address>:~/wallet_with_cred.zip ~/Downloads/
```

Example:
```
zip ../wallet_with_cred.zip *
exit
scp -i ssh/key/oci opc@111.222.333.444:~/wallet_with_cred.zip ~/Downloads/
```

## Upload Wallet

Sign in to the RDF Graph Server and Query UI

- URL: https://<ip_address>:8001/orardf
- User: `weblogic`
- Password: `welcome1` (This password should be changed in the latter step)

Data sources tab > Create > Wallet

- Zip file: Select `wallet_with_cred.zip`
- Name: Any name of this data source (e.g. `ADW1`)
- Wallet Service: The service modified above `<db_name>_high` (e.g. `ADB1_high`)

## Create RDF Network

Data tab

- Data source: The data source created above (e.g. `ADW1`)

Data tab > RDF Network > + icon (= Create network)

- Network owner: ADMIN
- Network name: network1
- Tablespace: DATA

Data tab

- RDF network: The RDF network created above (e.g. `ADMIN.NETWORK1`)

Data tab > Import > Upload data icon

- Upload: Select `sample.nt`
- Staging table: Input any new table name (e.g. `sample_table`)
- Overwrite:

Data tab > Import > Bulk load data icon

- RDF Data
  - Model: Input any model name (e.g. `sample_model`)
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

From a new browser tab, access the URL below to send a SPARQL query as a GET request.

```
https://<ip_address>:8001/orardf/api/v1/datasets/query?datasource=ADW1&datasetDef={"metadata":[{"networkOwner":"ADMIN","networkName":"NETWORK1","models":["SAMPLE_MODEL"]}]}&query=select ?s ?p ?o where { ?s ?p ?o} limit 10
```