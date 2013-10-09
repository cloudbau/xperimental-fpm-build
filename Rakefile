# Needs fpm
# Usage: 
#   building packages:
#     $ rake build_package[/path/to/openstack/project]
#     Or:
#     $ WORKSPACE=/path/to/openstack/project rake
#   upload to s3:
#     $ rake upload_packages_for_<tempest|staging|production>
#

Version = "#{Time.now.to_i}"
# Dependencies that are in /etc/<component>/rootwrap.d/*.filters
SysDependencies = {
  "nova" => %w{
    e2fsprogs
    qemu-utils
    kpartx
    parted
    kvm
    libvirt-bin
    genisoimage
    iputils-arping
    iptables
    ebtables
    dnsmasq-utils
    bridge-utils
  },
  "glance" => %w{},
  "keystone" => %w{},
  "quantum" => %w{
    vlan
    iputils-arping
    iptables
    ebtables
    bridge-utils
    iputils-ping
    dnsmasq-base
    dnsmasq-utils
    psmisc
  },
  "cinder" => %w{},
  "all" => %w{
    libxslt1.1
    libxml2
  }
}
SysDependencies["neutron"] = SysDependencies["quantum"]

PyDependencies = {
  "nova" => %w{
  },
  "glance" => %w{},
  "keystone" => %w{
    keyring
    python-memcached
  },
  "quantum" => %w{
  },
  "cinder" => %w{},
  "all" => %w{
    distribute
    MySQL-python
    git+git://github.com/mouadino/logstasher.git
  }
}
PyDependencies["neutron"] = PyDependencies["quantum"]


# Mapping component => Service name
Services = {
  "keystone" => %w{keystone},
  "glance" => %w{glance-api glance-registry},
  "cinder" => %w{cinder-api cinder-scheduler cinder-volume},
  "nova" => %w{
    nova-api
    nova-api-ec2
    nova-api-os-compute
    nova-conductor
    nova-scheduler
    nova-network
    nova-api-metadata
    nova-api-os-volume
    nova-volume nova-compute
  },
  "quantum" => %w{
    quantum-server
    quantum-dhcp-agent
    quantum-l3-agent
    quantum-metadata-agent
    quantum-openvswitch-agent
  }
}

task :default => :build_package

[:glance, :nova, :keystone, :quantum, :cinder].each do |service|
desc "Build #{service.to_s}"
task "build_#{service.to_s}_package".to_sym do 
  project_path = Dir.pwd
  component = service.to_s
  desc "Build the packages for #{component}"

  dir = "build/opt/cloudbau/#{component}-virtualenv"
  mkdir_p dir
  rm_rf dir
  sh "virtualenv #{dir}"

  if component == "nova"
    # Special treatment for libvirt
    # Copy libvirt stuff because it's not available via pypi
    sh "cp /usr/lib/python2.7/dist-packages/libvirt* #{dir}/lib/python2.7/site-packages/"
  end

  pip_install = "#{dir}/bin/pip install --upgrade"
  sh "#{dir}/bin/easy_install -U distribute"
  sh "#{pip_install} #{PyDependencies['all'].join(' ')}"
  sh "#{pip_install} #{PyDependencies[component].join(' ')}"
  sh "cd #{dir}; bin/pip install #{project_path}/#{component}*.tar.gz"
  sh "virtualenv --relocatable #{dir}"
  
  create_upstart_scripts(component)
  start_scripts = component == "quantum" ? "build/etc/init.d/#{component}*" : "build/etc/init/#{component}*.conf build/etc/init.d/#{component}*"

  dependencies = SysDependencies["all"] + SysDependencies[component]
  sh "fpm  -v #{Version} -x .git -s dir -t deb -a all  -d #{dependencies.join(" -d ")}  -n cloudbau-#{component} -C build etc opt" 
end
end

def create_upstart_file_content(component, service_name)
  exec_name = service_name
  exec_name += "-all" if service_name == 'keystone'

  config_name = service_name
  config_name = "nova" if service_name =~ /nova/
  config_name = "cinder" if service_name =~ /cinder/

virtenv="/opt/cloudbau/#{component}-virtualenv"
file=<<EOF
description "#{component} #{service_name}"
author "Hendrik Volkmer <h.volkmer@cloudbau.de>"

start on runlevel [2345]
stop on runlevel [016]

chdir /var/run

pre-start script
  mkdir -p /var/log/#{component}
  mkdir -p /var/run/#{component}
  chown #{component}:root /var/log/#{component}
  chown #{component}:root /var/run/#{component}
end script

exec start-stop-daemon --start --chuid #{component} --exec #{virtenv}/bin/#{exec_name} -- --config-file=/etc/#{component}/#{config_name}.conf --log-file=/var/log/#{component}/#{service_name}.log
EOF
file
end

def create_upstart_scripts(component)
  Services[component].each do |service|
    mkdir_p "build/etc/init"
    mkdir_p "build/etc/init.d"
    File.open("build/etc/init/#{service}.conf", "w") do |f|
      f.puts create_upstart_file_content(component, service)
    end
    sh "ln -fs /lib/init/upstart-job build/etc/init.d/#{service}"
  end

end
