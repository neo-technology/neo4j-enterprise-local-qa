Neo4j Local JVM Setup
=====================

NOTE: THIS IS _NOT_ AUTOMATED QA!
It does not run out of the box at present (release 1.9RC1), and also it doesn't tell you that it didn't work - it just spams the console and you have to pick apart the failing bits yourself. Do _not_ rely on this for release QA.

This (naive) rakefile will set up a local cluster of Neo4j HA servers. By default it sets up 3 machines and 3 coordinators. For more machines (don't use less because the coordinators won't be quorate), simply add to the array: 

    machines = ["machineA", "machineB", "machineC"] # give each machine a unique name

Adjusting local machine ip is not needed. Empty 'local_ip' will default to local ip.

Execute a full setup and test with

    rake qa

The servers should now be available at http://localhost:7474, http://localhost:7475 and http://localhost:7476




