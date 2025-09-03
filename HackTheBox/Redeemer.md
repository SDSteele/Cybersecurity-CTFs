## Redeemer:

#### Challenge: 
- lab focuses on enumerating a Redis server remotely and then dumping its database in order to retrieve the flag

#### Skills:  
- learn about the usage of the redis-cli command line utility
-  basic redis-cli commands
-  using ip=x.x.x.x command to now have to type the ip over and over

#### Notes:

- There are different types of databases
- one is Redis, which is an 'in-memory' database
  - Inmemory databases are the ones that rely essentially on the primary memory for data storage (meaning that the database is managed in the RAM of the system)
  - contrast to databases that store data on the disk or SSD
  - data retrieval time in the case of 'in-memory' databases is very small, thus offering very efficient & minimal response times
  - databases like Redis are typically used to cache data that is frequently requested for quick
retrieval
  -  example: a website that returns some prices on the front page
- start by using ping again on ip
- Oddly, with zenmap and nmap I had issues finding the open ports, but with Rustscan i was able to find it easily.
- open port 6379/tcp redis
- loggedin with $ redis-cli -h 10.129.209.100
- googled redis-cli commands
- using the "info" command we see stuff like OS, ram usage, sha on/off, cpu, etc.
- look at the end, we see Keyspace, this provides  statistics on the main dictionary of each database
  - statistics include the number of keys, and the number of keys with an expiration
- using the select command with select 0 (the database (db0)) we saw, we can then run keys * to find all keys.
- The flag is listed there.
- we then use the old standby of get: get flag
