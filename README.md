# solidifire: ID-to-Name Mapping for SolidFire Objects

An approach for ID to Name mapping between various IDs and Names in NetApp SolidFire clusters.

## Why?

NetApp SolidFire clusters use unique ID's for Clusters, Volumes, Accounts and so on. 

Object Names (Volume Name, IQN Name, etc) have 1:1 mapping to these objects and are required for objects of interest (Cluster, Volume, Account). Object Names do not have to be unique and as of now (Element v12.0) cannot be modified either.

Humans aren't great with numeric IDs (which in the case of SolidFire are unique). They're better with names, which sadly are not (and need not be) unique.

### Wait, why do I need this again?

You probably don't. SolidFire and NetApp HCI have a great UI and API and this mapping is done for you.

## Use cases

- "Resolve" various IDs to Names and display them in a human-friendly fashion (Names, or both Names and IDs)
- Track (in a non-compliant manner) objects of interest before and after they're gone
- If additional Objects were tracked, that could be useful in certain storage automation, including Kubernetes

## Approaches

- Pull: periodically query the system and populate a database (KISS)
- Push: listen to cluster event log; use events to add and delete data (more resource intensive, but doesn't require the use of Cluster Admin account)
- Both: combination of both (impressive but complex)

Due to its small size, the database is unlikely to require deletion. Quite possibly `INSERT` and `SELECT` are the only two SQL operations required for casual use.

### Pull approach: Database and Queries

- Database (SQLite example)

```sql
# Clusters Table 
CREATE TABLE clusters (uniqueId STRING (4, 0) UNIQUE ON CONFLICT FAIL NOT NULL ON CONFLICT FAIL PRIMARY KEY, name STRING (32), uuid STRING (64) UNIQUE ON CONFLICT FAIL NOT NULL);

# Accounts Table 
CREATE TABLE accounts (clusterUniqueId STRING (64) REFERENCES clusters (uniqueid) ON DELETE CASCADE MATCH FULL NOT NULL, accountId INTEGER NOT NULL, username STRING (64) NOT NULL);
CREATE INDEX idx_username ON accounts (username);
CREATE INDEX idx_accountid ON accounts (accountId);

# Volumes Table 
CREATE TABLE volumes (accountId INTEGER REFERENCES accounts (accountId) ON DELETE SET DEFAULT ON UPDATE CASCADE MATCH FULL NOT NULL, name STRING (64) NOT NULL, virtualVolumeID STRING (64), volumeId INTEGER, volumeUuid STRING (64) NOT NULL UNIQUE, clusterUniqueId  REFERENCES clusters (uniqueId) NOT NULL);
CREATE INDEX idx_volumeUuid ON volumes (volumeUuid);
CREATE INDEX idx_volumeName ON volumes (name);
```

- Sample content

```sql
# Create two clusters PROD and DR
INSERT INTO clusters (uniqueId,name,uuid) VALUES ('ozv4','PROD','2b09ae6b-ba07-4ce2-a0f6-4ae211e8d930');
INSERT INTO clusters (uniqueId,name,uuid) VALUES ('abc1','DR', '3339ae6b-ba07-4ce2-a0f6-4ae211e8d000');
SELECT * from clusters;
ozv4|PROD|2b09ae6b-ba07-4ce2-a0f6-4ae211e8d930
abc1|DR|3339ae6b-ba07-4ce2-a0f6-4ae211e8d000

# Create three accounts in each cluster 
INSERT INTO accounts(clusterUniqueId,accountId,username) VALUES ('ozv4','1','vcenter');
INSERT INTO accounts(clusterUniqueId,accountId,username) VALUES ('ozv4','2','test');
INSERT INTO accounts(clusterUniqueId,accountId,username) VALUES ('ozv4','3','empty');
INSERT INTO accounts(clusterUniqueId,accountId,username) VALUES ('abc1','1','vcenter');
INSERT INTO accounts(clusterUniqueId,accountId,username) VALUES ('abc1','2','docker');

# Create five volumes (not VVOLs!), three in PROD and two in DR site 
INSERT INTO volumes(clusterUniqueId,accountid,name,virtualVolumeID,volumeid,volumeUuid) VALUES ('ozv4','1','hcibench','1', '','27ba7507-75e8-41bd-a01e-8e0689397ccf');
INSERT INTO volumes(clusterUniqueId,accountid,name,virtualVolumeID,volumeid,volumeUuid) VALUES ('ozv4','1','test','2', '','fda0ec6a-8e6c-4efc-845c-06f6af75b82c');
INSERT INTO volumes(clusterUniqueId,accountid,name,virtualVolumeID,volumeid,volumeUuid) VALUES ('ozv4','2','empty','3', '','f54c137b-bb37-4b21-afed-fce84ac8a29b');
INSERT INTO volumes(clusterUniqueId,accountid,name,virtualVolumeID,volumeid,volumeUuid) VALUES ('abc1','1','hcibench','1', '','333ba7507-75e8-41bd-a01e-8e0689397333');
INSERT INTO volumes(clusterUniqueId,accountid,name,virtualVolumeID,volumeId,volumeUuid) VALUES ('abc1','1','hcibench-clone','2', '','444ba7507-75e8-41bd-a01e-8e0689397444');

```
- Queries

```sql
# GET username for storage account ID 1 on cluster with Unique ID 'abc1' (named PROD)
SELECT username FROM accounts WHERE accountId=1 AND clusterUniqueId='abc1';
vcenter

# GET username for storage account ID "1" on cluster with Unique ID "ozv4" (named DR)
SELECT accounts.username FROM accounts LEFT JOIN clusters ON accounts.clusterUniqueId=clusters.uniqueid WHERE clusters.uniqueid='ozv4' AND accounts.accountId='1';
vcenter

# GET storage volume names owned by storage account ID "1" on cluster 'ozv4'
SELECT name FROM volumes WHERE accountId='1' AND clusterUniqueId='ozv4';
hcibench  
test

# Get storage account names from cluster 'abc1' (DR)
SELECT accounts.username FROM accounts WHERE accounts.clusterUniqueId = 'abc1';
vcenter   
docker
```

### Implementation

- TODO
- Possible approach
  - Container format
  - SolidFire Python CLI to periodically fetch and insert Cluster, Account and Volume information
  - JSON-RPC API to provide a way to map various IDs to Names

## Frequently Asked Questions

Q: Is this what NetApp recommends?

A: No. It's not meant for production use either.

## License and Trademarks

- NetApp, SolidFire and other marks are owned by NetApp, Inc. Other marks may be owned by their respective owners.
- See [LICENSE](LICENSE).
