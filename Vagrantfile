Vagrant.configure(2) do |config|
  config.vm.box = "chef/centos-7.0"
  config.vm.network "private_network", ip: "192.168.33.10"
  config.vm.provision "shell", inline: <<-SHELL
    sudo yum -y install epel-release
    sudo yum -y update
    sudo yum -y install nodejs npm collectd tcpdump nc
    sudo npm install -g statsd statsd-influxdb-backend
    sudo yum -y install https://s3.amazonaws.com/influxdb/influxdb-latest-1.x86_64.rpm
    sudo yum -y install https://grafanarel.s3.amazonaws.com/builds/grafana-2.0.2-1.x86_64.rpm
    sudo yum -y clean all
    sudo systemctl enable influxdb
    sudo systemctl enable grafana-server
    sudo systemctl enable collectd
    sudo systemctl start influxdb
    sudo systemctl start grafana-server
    sudo systemctl start collectd
  SHELL
end
