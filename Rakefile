require 'net/http'
require 'uri'

#the ip of your local machine
#local_ip = '172.16.12.142'
local_ip = 'localhost'

# This is where you configure the product version
filename = "neo4j-enterprise-1.8.M05"

# You probably don't need to touch this
tarfile = filename + "-unix.tar.gz"

# Three machines by default. You can add more, but be sure to give each a unique name
machines = ["machineA", "machineB", "machineC"]

cluster_name="local.jvm.only.cluster"

zk_client_port = 2181
web_server_port = 7474
https_port = 8484
ha_server_id = 1
ha_server = 6001
jmx_port = 3637
backup_port = 1234


task :default => 'qa'

task :download_neo4j do
  
  if !File::exists? tarfile

    uri = "dist.neo4j.org"

    Net::HTTP.start(uri) do |http|
      begin
       file = open(tarfile, 'wb')
       http.request_get("/" + tarfile) do |response|
        response.read_body do |segment|
         file.write(segment)
        end
       end
      ensure
       file.close
      end
    end
  end
end

task :untar => tarfile do 
  puts "untarring: " + tarfile
  command = "tar -xzf " + tarfile
  system(command)
end

task :clone do
 
  machines.each do |machine| 
    FileUtils::copy_entry filename, machine, preserve=true, remove_destination=true
  end

end

def replace_in_file(regex, replacement, file)
  text = File.read(file) 
  str = text.gsub(regex, replacement)
  File.open(file, "w") { |file| file << str }
end

def coordinators_list(machines, local_ip, start_port)
  str = "ha.coordinators="

  (1..machines.length).each do |i|
    str << local_ip << ":" << start_port.to_s << ","
    start_port += 1 
  end
 
  str.chomp(',')
end

def server_list(machines, local_ip)
  str = ""
  (1..machines.length).each do |i|
    str << "server." << i.to_s << "=" << local_ip << ":" << (2888 +i-1).to_s << ":" << (3888 +i-1).to_s << "\n"
  end
 
  str
 
end


task :change_config do
  machine_list = server_list(machines, local_ip)
  i = 0
  machines.each do |machine| 

    replace_in_file('ha.pull_interval = 10', 'ha.pull_interval = 1ms', machine + "/conf/neo4j.properties")
    replace_in_file('#ha.coordinators=localhost:2181', coordinators_list(machines, local_ip, zk_client_port), machine + "/conf/neo4j.properties")  
    replace_in_file('#ha.cluster_name =', "ha.cluster_name="+cluster_name, machine + "/conf/neo4j.properties")  
    replace_in_file('online_backup_enabled=true', "online_backup_enabled=true\nonline_backup_port="+(backup_port+i).to_s, machine + "/conf/neo4j.properties")
    replace_in_file('#ha.server_id=', "ha.server_id=" +(ha_server_id+i).to_s, machine + "/conf/neo4j.properties")
    replace_in_file("#ha.server = localhost:6001", "ha.server = "+local_ip+":" + (ha_server+i).to_s, machine + "/conf/neo4j.properties")
    
    replace_in_file('server.1=localhost:2888:3888', machine_list, machine + "/conf/coord.cfg")
    replace_in_file('#server.2=my_second_server:2889:3889', "", machine + "/conf/coord.cfg")
    replace_in_file('#server.3=192.168.1.1:2890:3890', "", machine + "/conf/coord.cfg")
  
    replace_in_file('clientPort=2181', "clientPort=" + (zk_client_port+i).to_s, machine + "/conf/coord.cfg")


    replace_in_file('org.neo4j.server.webserver.port=7474', "org.neo4j.server.webserver.port=" + (web_server_port+i).to_s, machine + "/conf/neo4j-server.properties")
    
    replace_in_file('org.neo4j.server.webserver.https.port=7473', "org.neo4j.server.webserver.https.port=" + (https_port+i).to_s, machine + "/conf/neo4j-server.properties")
    
    replace_in_file('#org.neo4j.server.database.mode=HA', "org.neo4j.server.database.mode=HA", machine + "/conf/neo4j-server.properties")
    
    replace_in_file('#wrapper.java.additional.3=-Dcom.sun.management.jmxremote.port=3637', "wrapper.java.additional.3=-Dcom.sun.management.jmxremote.port=" + (jmx_port+i).to_s, machine + "/conf/neo4j-wrapper.conf")
    replace_in_file('#wrapper.java.additional.4=-Dcom.sun.management.jmxremote.authenticate=true', "wrapper.java.additional.4=-Dcom.sun.management.jmxremote.authenticate=false" + (jmx_port+i).to_s, machine + "/conf/neo4j-wrapper.conf")
    i += 1
  end

end

task :start_coordinators do
  count = 1
  machines.each do |machine|
  File.open(machine + "/data/coordinator/myid", "w") { |file| file << count }
  count += 1
  end

  machines.each do |machine|
    system(machine + "/bin/neo4j-coordinator start")
  end
  puts "waiting a sec for the coordinators to be ready ...."
  sleep 5

end

task :start_cluster do
  machines.each do |machine|
    system(machine + "/bin/neo4j start")
  end
end

task :setup_cluster => [:download_neo4j, :untar, :clone, :change_config, :start_coordinators, :start_cluster] do

end

task :qa => [:setup_cluster, :test] do

end




def execute(cmd)
  puts "executing '"<< cmd << "'"
  system(cmd)
end

task :test do

  # Create a node on machineA
  execute(machines[0] + "/bin/neo4j-shell -c 'mknode'")

  # Make sure it's propagated to the slaves
  execute(machines[1] + "/bin/neo4j-shell -c 'cd -a 1 && set name prop1'")

  #do an onine backup on machine 3
  execute(machines[2].to_s + "/bin/neo4j-backup -full -from ha://"+local_ip+":"+zk_client_port.to_s + " -to " + machines[2].to_s+ "/backup -ha.cluster_name "+cluster_name)
  # Create a node on machineC
  execute(machines[2] + "/bin/neo4j-shell -c 'mknode'")

  #stop master (machineA)
  execute(machines[0].to_s << "/bin/neo4j stop")

  #execute incremental backup
  execute(machines[2].to_s + "/bin/neo4j-backup -incremental -from ha://"+local_ip+":"+(zk_client_port+1).to_s + " -to " + machines[2].to_s+ "/backup -ha.cluster_name "+cluster_name)

end

task :clean do
  if File::exists? filename
    FileUtils::rm_rf filename
  end

  machines.each do |m|
    FileUtils::rm_rf m
  end
end

task :nuke => [:clean] do

  if File::exists? tarfile
    File::delete tarfile
  end

end
