# solidifire: ID-to-Name Mapping for SolidFire Objects

An approach for ID-to-Name mapping between varios Object IDs and Names in NetApp SolidFire clusters.

## Why?

NetApp SolidFire clusters use unique integer ID's for Clusters, Volumes, Accounts and so on. 

Object Names (Volume Name, IQN Name, Account Name) have a 1:1 mapping to these objects and are required (Cluster, Volume, Account). An Object Name does not, however, have to be unique and as of now (Element v12.0 and earlier supported versions) cannot be modified either.

Humans aren't great with numeric IDs (which in the case of SolidFire are unique). They're better with Names, which sadly are allowed to repeat (but can be avoided if automated, for example by Kubernetes).

### Wait, why do I need this again?

You probably don't. SolidFire and NetApp HCI have a great UI and API and this mapping is done for you.

## Use cases

- Resolve various IDs to Names and display them in a human-friendly fashion (Names, or both Names and IDs)
- Track (in a non-compliant manner) objects of interest before *and after* they're gone
- If additional Objects were tracked, that could be useful in certain storage automation scenarios, including Kubernetes environments

For enterprise auditing and legal compliance one should use established methods (redirect and capture SolidFire log to Elastic, Splunk or other system that provides required features.)

## Approaches

- Pull: periodically query the system and populate a database (KISS principle; may miss to capture short-lived objects (volumes, for example), but simple)
- Push: listen to cluster event log; use events to add and delete data (more resource intensive, but doesn't require the use of a Cluster Admin account)
- Both: a combination of both (impressive but complex)

Due to its small size, database records are unlikely to require deletion. Quite possibly `INSERT` and `SELECT` are the only SQL operations required for casual use. Use `UPSERT` to update volume ownership changes.

### Push approach: syslog redirection

I captured two lines that contain typical records we'd want to capture and reference (`CreateVolume` and `DeleteVolume`):

```
Jun  3 16:41:35 192.168.1.29 master-1[20395]: [APP-5] [API] 24016 DBCallback httpserver/RestAPIServer.cpp:292:LogAndDispatch|RestAPI::CreateVolume CALL: requestID=36 loggedParams={"accountID":1,"enable512e":true,"name":"sean","qosPolicyID":2,"requestAPIVersion":"12.0","totalSize":15000000000} user=[admin] authMethod=[Cluster] ip=[192.168.1.12]
...
Jun  3 16:41:40 192.168.1.29 master-1[20395]: [APP-5] [API] 24014 DBCallback httpserver/RestAPIServer.cpp:304:LogAndDispatch|RestAPI::DeleteVolume SUCCESS result={"volume":{"access":"readWrite","accountID":1,"attributes":{},"blockSize":4096,"createTime":"2020-06-03T16:41:35Z","currentProtectionScheme":"singleHelix","deleteTime":"2020-06-03T16:41:40Z","enable512e":true,"enableSnapMirrorReplication":false,"iqn":"iqn.2010-01.com.solidfire:ozv4.sean.7","lastAccessTime":null,"lastAccessTimeIO":null,"name":"sean","previousProtectionScheme":null,"purgeTime":"2020-06-04T00:41:40Z","qos":{"burstIOPS":800,"burstTime":60,"curve":{"1048576":15000,"131072":1950,"16384":270,"262144":3900,"32768":500,"4096":100,"524288":7600,"65536":1000,"8192":160},"maxIOPS":400,"minIOPS":200},"qosPolicyID":2,"scsiEUIDeviceID":"6f7a763400000007f47acc0100000000","scsiNAADeviceID":"6f47acc1000000006f7a763400000007","sliceCount":1,"status":"deleted","totalSize":15000928256,"virtualVolumeID":null,"volumeAccessGroups":[],"volumeConsistencyGroupUUID":"95380b91-626d-492f-aa77-93fa8a030063","volumeID":7,"volumePairs":[],"volumeUUID":"c8f2c87e-db79-43b8-b658-a57502c96400"}}

```

Compared to the Pull approach, syslog generates huge amounts of data (as expected), but it has the advantage of having many ready-made database back-ends *as well as* APIs to query them (say, ELK and Grafana). It is also able to provide Name-to-ID aliasing and many more features directly from Grafana as well as better reliability as the likelihood of missing any volume or account actions is very small.


### Pull approach: Database and Queries

- Element API methods:
  - GetClusterInfo (mvip, name, uniqueID, uuid)
  - ListAccounts (accountID, username, status (active, locked, removed), and volumes owned by the account)
  - ListVolumes and ListDeletedVolumes (accountID, volumeID, volumeUuid, virtualVolumeId, name, status (init, active, deleted - note that a volume may be Restored, or Purged (by default 8 hours after deletion) after it's been Deleted. No method can provide details on Purged volumes)

#### Database (SQLite example)

```sql
# Clusters Table 
CREATE TABLE clusters (
    uniqueId STRING (4, 0) UNIQUE ON CONFLICT FAIL
                           NOT NULL ON CONFLICT FAIL
                           PRIMARY KEY,
    name     STRING (32),
    uuid     STRING (64)   UNIQUE ON CONFLICT FAIL
                           NOT NULL,
    fqdn     STRING (46) 
);

# Accounts Table 
CREATE TABLE accounts (
    clusterUniqueId STRING (64) REFERENCES clusters (uniqueId) ON DELETE NO ACTION
                                                               MATCH [FULL]
                                NOT NULL,
    accountId       INTEGER     NOT NULL,
    username        STRING (64) NOT NULL,
    status          STRING (7),
    id              INTEGER     PRIMARY KEY AUTOINCREMENT
);

# Volumes Table 
CREATE TABLE volumes (
    accountId       INTEGER     REFERENCES accounts (accountId) ON DELETE SET DEFAULT
                                                                ON UPDATE CASCADE
                                                                MATCH [FULL]
                                NOT NULL,
    name            STRING (64) NOT NULL,
    virtualVolumeId STRING (64),
    volumeId        INTEGER     NOT NULL,
    volumeUuid      STRING (64) NOT NULL
                                UNIQUE,
    clusterUniqueId STRING (4)  REFERENCES clusters (uniqueId) 
                                NOT NULL,
    status          STRING (8),
    id              INTEGER     PRIMARY KEY ON CONFLICT FAIL AUTOINCREMENT
);

```

#### Sample content

```sql
# Create two clusters PROD and DR
INSERT INTO clusters (uniqueId,name,uuid,fqdn) VALUES ('ozv4','PROD','2b09ae6b-ba07-4ce2-a0f6-4ae211e8d930','192.168.1.30');
INSERT INTO clusters (uniqueId,name,uuid,fqdn) VALUES ('abc1','DR', '3339ae6b-ba07-4ce2-a0f6-4ae211e8d000','192.168.1.34');
SELECT * from clusters;
ozv4|PROD|2b09ae6b-ba07-4ce2-a0f6-4ae211e8d930|192.168.1.30
abc1|DR|3339ae6b-ba07-4ce2-a0f6-4ae211e8d000|192.168.1.34

# Create three accounts in each cluster 
INSERT INTO accounts(clusterUniqueId,accountId,username) VALUES ('ozv4','1','vcenter');
INSERT INTO accounts(clusterUniqueId,accountId,username) VALUES ('ozv4','2','test');
INSERT INTO accounts(clusterUniqueId,accountId,username) VALUES ('ozv4','3','empty');
INSERT INTO accounts(clusterUniqueId,accountId,username) VALUES ('abc1','1','vcenter');
INSERT INTO accounts(clusterUniqueId,accountId,username) VALUES ('abc1','2','docker');

# Create five volumes (not VVOLs!), three in PROD and two in DR site 
INSERT INTO volumes(clusterUniqueId,accountid,name,virtualVolumeID,volumeid,volumeUuid) VALUES ('ozv4','1','hcibench','','1','27ba7507-75e8-41bd-a01e-8e0689397ccf');
INSERT INTO volumes(clusterUniqueId,accountid,name,virtualVolumeID,volumeid,volumeUuid) VALUES ('ozv4','1','test','','2','fda0ec6a-8e6c-4efc-845c-06f6af75b82c');
INSERT INTO volumes(clusterUniqueId,accountid,name,virtualVolumeID,volumeid,volumeUuid) VALUES ('ozv4','2','empty','','3','f54c137b-bb37-4b21-afed-fce84ac8a29b');
INSERT INTO volumes(clusterUniqueId,accountid,name,virtualVolumeID,volumeid,volumeUuid) VALUES ('abc1','1','hcibench','','1','333ba7507-75e8-41bd-a01e-8e0689397333');
INSERT INTO volumes(clusterUniqueId,accountid,name,virtualVolumeID,volumeId,volumeUuid) VALUES ('abc1','1','hcibench-clone','','2','444ba7507-75e8-41bd-a01e-8e0689397444');
```

#### Queries

- Cluster 
  - Get a cluster's `UniqueID` from `MVIP` (actually `name`, FQDN, IPv4, (IPv6?) from its `clusterUniqueId`
- Accounts
  - Get all account `name`s for a cluster identified by `MVIP`
- Volumes
  - Get volume `name`s for one or more `volumeId`s from the cluster identifed by `MVIP` (or volume names by `volumeId`'s for `MVIP`, if list or wildcard)
  - Get all volume names from `MVIP` owned by `accountId`

```sql
# Get Cluster UniqueID for the cluster with MVIP (FQDN) 192.168.1.30
SELECT name, uniqueId FROM clusters WHERE fqdn = '192.168.1.30';
PROD|ozv4

# Get username for storage account ID "1" on cluster with Unique ID 'ozv4' (named PROD)
SELECT accounts.username FROM accounts LEFT JOIN clusters ON accounts.clusterUniqueId=clusters.uniqueid WHERE clusters.uniqueid='ozv4' AND accounts.accountId='1';
vcenter

# Get all account IDs for MVIP 192.168.1.30 
SELECT accountId FROM accounts LEFT JOIN clusters ON accounts.clusterUniqueId=clusters.uniqueid WHERE clusters.fqdn='192.168.1.30'
1
2
3
4

# Get all active accounts from the cluster with UniqueID 'ozv4' 
SELECT accounts.username FROM accounts LEFT JOIN clusters ON accounts.clusterUniqueId=clusters.uniqueid WHERE clusters.uniqueid='ozv4' AND accounts.status='active' ORDER BY accounts.username ASC;
empty
test
vcenter

# Get storage account names from cluster 'abc1' (DR)
SELECT accounts.username FROM accounts WHERE accounts.clusterUniqueId = 'abc1';
vcenter   
docker

# GET volume names owned by storage account ID "1" on cluster 'ozv4'
SELECT name FROM volumes WHERE accountId='1' AND clusterUniqueId='ozv4';
hcibench  
test

```

### Conclusion

- Possible approach to Pull:
  - Container
  - SolidFire Python CLI to periodically fetch and insert or update Cluster, Account and Volume information
  - JSON-RPC API to map various IDs to Names
- Possible approach to Push:
  - SolidFire syslog forwarding to syslog-ng
  - Preprocessing and forwarding to Elastic, Splunk, etc.

Although Pull is much simpler, log redirection is much easier and has many other advantages in this context, so I'll explore log capture and analysis rather than write an app that wouldn't add much value. 

## Frequently Asked Questions

Q: Is this what NetApp recommends for something?

A: No. It's not meant for production use either.

## License and Trademarks

- NetApp, SolidFire and other marks are owned by NetApp, Inc. Other marks may be owned by their respective owners.
- See [LICENSE](LICENSE).
