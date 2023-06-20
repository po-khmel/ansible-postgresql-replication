usegalaxy-it.postgresql_replication
=========
The role is supposed to configure master and standby servers for a basic replication process. It's created as a supplement to the existing [galaxyproject.postgresql](https://github.com/galaxyproject/ansible-postgresql) role. 

Dependencies
-------
galaxyproject.postgresql

Requirements
------------

Ansible >= 2.7  
PostgreSQL >= 12  
 
2 VM's are required. Make sure they have:  
- installed PostgreSQL
- postgres user
These options may be configured by galaxyproject.postgresql role.

Check that IP addresses for 2 machines are defined in hostsvars or other vars files

Role Variables
--------------
`master_ip` (required) - IP address of a main database machine. May be set to `groups['database'][0]` where 'database' is the group host name of database VM  
`replica_ip` (required) - IP address of a standby machine. May be set to `groups['replica'][0]` where 'replica' is the group host name of database VM  
`replication_role` (required) - defines which role to configure. Specify for each VM  
`postgresql_version` - default to 12  
`replica_user_name` - default to replica  
`replica_user_password` (required) - set the password and store it in a vault encrypted file  
`replica_data_path` - the path to store the replicated data. Default to postgresql_conf_dir   
`promote_trigger_file` - if this file exists on the standby server, standby is promoted to master  

To enable synchronous replication you can uncomment last lines in <templates/ansible_replica.conf.j2>:  
```
synchronous_commit = remote_apply       # required for a synchronous replication  
synchronous_standby_names = '*'         # list of standby servers that can support  
``` 
You may need to add additional settings as the behaviour of standby in case of failover is different under synchronous replication. (Standby server will take this settings when becomes master and it'll not complete any write operations until data is replicated to some other standby)  

Example Playbook
----------------

How to use the role:

    - hosts: database
      vars_files:
        - path/to/secret/replica/password.yml
      vars: 
        replication_role: master
      roles:
         - usegalaxy-it.postgresql_replication

    - hosts: replica
      vars_files:
        - path/to/secret/replica/password.yml
      vars: 
        replication_role: standby
      roles:
         - usegalaxy-it.postgresql_replication

Checks
----------------

To check if the replication process is successful you can execute:

### on Master VM

```
$ sudo su postgres

$ psql

postgres=# \x
Expanded display is on.

postgres=# select usename, application_name, client_addr, state, sync_priority, sync_state from pg_stat_replication;
usename | application_name | client_addr     | state      | sync_priority | sync_state
---------+------------------+-----------------+-----------+---------------+------------
replica | walreceiver      | 192.168.208.186 | streaming  | 1             | async
(1 row)

postgres=# SELECT \* FROM pg_stat_wal_receiver; 
-[ RECORD 1 ]---------+------------------------------
pid                   | 14667
status                | streaming
receive_start_lsn     | 0/C000000
receive_start_tli     | 1
received_lsn          | 0/C0012F8
received_tli          | 1
last_msg_send_time    | 2023-06-20 13:30:10.208088+00
last_msg_receipt_time | 2023-06-20 13:30:10.212497+00
latest_end_lsn        | 0/C0012F8
latest_end_time       | 2023-06-20 13:27:09.643938+00
slot_name             |
sender_host           | 192.168.208.62
sender_port           | 5432
conninfo              | user=replica passfile=/var/lib/pgsql/.pgpass dbname=replication host=192.168.208.62 port=5432 fallback_application_name=walreceiver sslmode=prefer sslcompression=0 gssencmode=prefer krbsrvname=postgres target_session_attrs=any
```
### on Standby VM

When the replication is ongoing, you can find `stanby.signal` in <postgresql_conf_dir> on a stanby server. And:

```
$ sudo su postgres

$ psql

postgres=# \x
Expanded display is on.

postgres=# create database test;
ERROR: cannot execute CREATE DATABASE in a read-only transaction

postgres=# SELECT * FROM pg_stat_replication;
-[ RECORD 1 ]----+------------------------------
pid          	   | 36365
usesysid     	   | 16386
usename      	   | replica
application_name | walreceiver
client_addr  	   | 192.168.208.186
client_hostname  |
client_port  	   | 53760
backend_start	   | 2023-06-20 13:05:12.90578+00
backend_xmin 	   |
state        	   | streaming
sent_lsn     	   | 0/C0001C0
write_lsn    	   | 0/C0001C0
flush_lsn    	   | 0/C0001C0
replay_lsn   	   | 0/C0001C0
write_lag    	   |
flush_lag    	   |
replay_lag   	   |
sync_priority	   | 1
sync_state   	   | async
reply_time   	   | 2023-06-20 13:19:13.230384+00

```

## Manual Failover

In case of failure of the master server, you have to **manually promote** stanby to become master. Since the <promote_trigger_file> is configured, the easiest way is:

1. connect to standby server
2. `$ sudo su postgres`
3. `touch <promote_trigger_file>`
4. check absence of `stanby.signal` file in <postgresql_conf_dir>
5. check if the replica can perform read-write operations:
  ```
  postgres=# create table t(n int);
  CREATE TABLE
  ```

Now the replica is the master server. You will need to change IP of the postgres in you configuration or configure Load Balancer to redirect traffic to the new instance.

License
-------

BSD

Author Information
------------------

Polina Khmelevskaia
