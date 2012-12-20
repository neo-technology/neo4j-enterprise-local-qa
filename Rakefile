require 'net/http'
require 'uri'

#the ip of your local machine
#local_ip = '172.16.12.142'
local_ip = ''

# This is where you configure the product version
filename = "neo4j-enterprise-1.9.M03"

# You probably don't need to touch this
tarfile = filename + "-unix.tar.gz"

# Three machines by default. You can add more, but be sure to give each a unique name
machines = ["machineA", "machineB", "machineC"]

cluster_name="local.jvm.only.cluster"
cluster_port = 5000
shell_port = 2337
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

def server_list(machines, local_ip, port, prefix)
  str = ""
  (1..machines.length).each do |i|
    str << prefix << local_ip << ":" << (port +i-1).to_s+","
  end
 
  str.chop
 
end


task :change_config do
  machine_list = server_list(machines, local_ip,cluster_port,"")
  i = 0
  machines.each do |machine| 

    replace_in_file('ha.pull_interval=10', 'ha.pull_interval = 1ms', machine + "/conf/neo4j.properties")
    replace_in_file('#remote_shell_enabled=true', 'remote_shell_enabled=true', machine + "/conf/neo4j.properties")  
    replace_in_file('#remote_shell_port=1234', 'remote_shell_port = '+(shell_port+i).to_s, machine + "/conf/neo4j.properties")  
    replace_in_file('#ha.cluster_server=:5001-5099', "ha.cluster_server="+local_ip+":"+(cluster_port+i).to_s, machine + "/conf/neo4j.properties")  
    replace_in_file('online_backup_port=6362', "online_backup_port="+(backup_port+i).to_s, machine + "/conf/neo4j.properties")
    replace_in_file('#ha.server_id=', "ha.server_id=" +(ha_server_id+i).to_s, machine + "/conf/neo4j.properties")
    replace_in_file("#ha.server=:6001", "ha.server = "+local_ip+":" + (ha_server+i).to_s, machine + "/conf/neo4j.properties")
    replace_in_file('#ha.initial_hosts=:5001,:5002,:5003', "ha.initial_hosts="+machine_list, machine + "/conf/neo4j.properties")

    #replace_in_file('#server.2=my_second_server:2889:3889', "", machine + "/conf/coord.cfg")
    #replace_in_file('#server.3=192.168.1.1:2890:3890', "", machine + "/conf/coord.cfg")
    #replace_in_file('clientPort=2181', "clientPort=" + (zk_client_port+i).to_s, machine + "/conf/coord.cfg")

    replace_in_file('org.neo4j.server.webserver.port=7474', "org.neo4j.server.webserver.port=" + (web_server_port+i).to_s, machine + "/conf/neo4j-server.properties")
    replace_in_file('org.neo4j.server.webserver.https.port=7473', "org.neo4j.server.webserver.https.port=" + (https_port+i).to_s, machine + "/conf/neo4j-server.properties")
    replace_in_file('#org.neo4j.server.database.mode=HA', "org.neo4j.server.database.mode=HA", machine + "/conf/neo4j-server.properties")
    
    replace_in_file('#wrapper.java.additional.3=-Dcom.sun.management.jmxremote.port=3637', "wrapper.java.additional.3=-Dcom.sun.management.jmxremote.port=" + (jmx_port+i).to_s+"\n", machine + "/conf/neo4j-wrapper.conf")
    replace_in_file('#wrapper.java.additional.4=-Dcom.sun.management.jmxremote.authenticate=true', "wrapper.java.additional.4=-Dcom.sun.management.jmxremote.authenticate=false", machine + "/conf/neo4j-wrapper.conf")
    i += 1
  end

end


task :start_cluster do
  machines.each do |machine|
    system(machine + "/bin/neo4j start")
  end
end


task :stop_cluster do
  machines.each do |machine|
    system(machine + "/bin/neo4j stop")
  end
end


task :setup_cluster => [:download_neo4j, :untar, :clone, :change_config] do

end

task :qa => [:setup_cluster, :start_cluster, :test] do

end




def execute(cmd)
  puts "executing '"<< cmd << "'"
  system(cmd)
end

task :test do

  # Create a node on machineA
  execute(machines[0] + "/bin/neo4j-shell -port "+ (shell_port + 0).to_s + " -c 'mknode'")

  # Make sure it's propagated to the slaves
  execute(machines[1] + "/bin/neo4j-shell -port "+ (shell_port + 1).to_s + " -c 'cd -a 1 && set name prop1'")

  #do an onine backup on machine 3
  execute(machines[2].to_s + "/bin/neo4j-backup -full -from "+"ha://"+local_ip + ":"+ (cluster_port+2).to_s + " -to " + machines[2].to_s+ "/backup")
  # Create a node on machineC
  execute(machines[2] + "/bin/neo4j-shell -port "+ (shell_port + 2).to_s + "  -c 'mknode'")

  #stop master (machineA)
  execute(machines[0].to_s << "/bin/neo4j stop")

  #execute incremental backup to empty location
#  execute(machines[2].to_s + "/bin/neo4j-backup -incremental -from ha://"+local_ip+":"+(cluster_port+1).to_s + " -to " + machines[1].to_s+ "/backup")

  #incremental backup
  execute(machines[2].to_s + "/bin/neo4j-backup -incremental -from ha://"+local_ip+":"+(cluster_port+2).to_s + " -to " + machines[2].to_s+ "/backup")

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
