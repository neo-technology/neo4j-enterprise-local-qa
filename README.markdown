Neo4j Local JVM Setup
=====================

This (naive) rakefile will set up a local cluster of Neo4j HA servers. By default it sets up 3 machines and 3 coordinators. For more machines (don't use less because the coordinators won't be quorate), simply add to the array: 

    machines = ["machineA", "machineB", "machineC"] # give each machine a unique name

Adjust your local machine ip if needed

    local_ip = '172.16.12.142'

Execute a full setup and test with

    rake qa

The servers should now be available at http://localhost:7474, http://localhost:7475 and http://localhost:7476

Contacts
--------

[Jim Webber](mailto:jim@neotechnology.com), [@jimwebber](http://twitter.com/jimwebber)




