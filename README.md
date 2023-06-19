usegalaxy-it.postgresql_replication
=========
The role is supposed to configigure master and standby machines for a basic replication process. It's created as a supplement to existing [galaxyproject.postgresql](https://github.com/galaxyproject/ansible-postgresql) role. 

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
This options may be configure by galaxyproject.postgresql role.

Check that IP addresses for 2 machines are defined in hostsvars or other vars files

Role Variables
--------------
`master_ip` (required) - IP address of a main database machine  
`replica_ip` (required) - IP address of a standby machine
`replication_role` (required) - defines which role to configure. Specify for each VM  
`postgresql_version` - default to 12  
`replica_user_name` - default to replica  
`replica_user_password` (required) - set the password and store it in a vault encripted file  
`replica_data_path` - the path to store the replicated data. Default to postgresql_conf_dir  


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


License
-------

BSD

Author Information
------------------

Polina Khmelevskaia