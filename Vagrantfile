# WSU Server Base Vagrant Configuration
#
# Matches the WSU Server environment production setup as closely as possible.
# This will auto edit some of the local provisioning file but not the production.
# All production states/grains/pillars are to be managed by hand.
#
# We recommend Vagrant 1.3.5 and Virtualbox 4.3.
#
# -*- mode: ruby -*-
# vi: set ft=ruby :

require 'json'

#for dev
#bootstrap="washingtonstateuniversity/WSU-Web-Serverbase/master"
bootstrap="jeremyBass/WSU-Web-Serverbase/bootstrap"

#######################
# Setup
######################
# There shouldn't be anything to edit below this point config wise
####################################################################
    
    ################################################################ 
    # Setup value defaults
    ################################################################ 
    
        #base dir
        ################################################################ 
        vagrant_dir = File.expand_path(File.dirname(__FILE__))
        
        # the sub projects :: writes out the salt "config data" and 
        # sets up for vagrant.  The production is done by hand on purpose
        ###############################################################
        destroying=false
        ARGV.each do |arg|        
            if arg.include?'destroy'
                destroying=true
            end
        end
        config_str_obj=""
        configFile="config.json"
        if File.exist?("config.json")
            # See e.g. https://gist.github.com/karmi/2050769#file-node-example-json
            begin
                config_str_obj=File.read(configFile).split.join(' ')
                config_obj = JSON.parse(config_str_obj, symbolize_names: true)
                rescue Exception => e
                STDERR.puts "[!] Error when reading the configuration file:",
                e.inspect
            end
        else
            config_str_obj = '{ "vagrant_options": { "ip":"10.10.30.30", "hostname":"WSUWEB", "memory":"1024", "vram":"8", "cores":"4", "host_64bit":"true", "install_type":"testing", "minion":"vagrant", "verbose_output":"true" } }'
            begin
                config_obj = JSON.parse(config_str_obj, symbolize_names: true)
                rescue Exception => e
                STDERR.puts "[!] Error when reading the configuration file:",
                e.inspect
            end
            
            
        end
        
        #######################
        # CONFIG Values
        #####################
        CONFIG=config_obj[:vagrant_options]
        
        if !destroying        
            config_obj[:apps].each_pair do |name, obj|
                appname=name.to_s
                repolocation=obj[:repo].to_s
                if !File.exist?("app/#{appname}")
                    puts "cloning repo that was missing"
                    puts "git clone #{repolocation} app/#{appname}"
                    puts `git clone #{repolocation} app/#{appname}`
                end
            end
        end
        filename = vagrant_dir+"/provision/salt/minions/#{CONFIG[:minion]}.conf"
        text = File.read(filename)
        
        PILLARFILE=   "#PILLAR_ROOT-\n"
        PILLARFILE << "pillar_roots:\n"
        PILLARFILE << "  base:\n"
        PILLARFILE << "    - /srv/salt/base/pillar\n"

        ROOTFILE=   "#FILE_ROOT-\n"
        ROOTFILE << "file_roots:\n"
        ROOTFILE << "  base:\n"
        ROOTFILE << "    - /srv/salt/base\n"
        
        SALT_ENV=   "#ENV_START-\n"
        SALT_ENV << "  env:\n"
        SALT_ENV << "    - base\n"
        SALT_ENV << "    - vagrant\n"
    
        config_obj[:apps].each_pair do |name, obj|
            appname=name.to_s
            
            SALT_ENV << "    - #{appname}\n"

            PILLARFILE << "  #{appname}:\n"
            PILLARFILE << "    - /srv/salt/#{appname}/pillar\n"
            
            ROOTFILE << "  #{appname}:\n"
            ROOTFILE << "    - /srv/salt/#{appname}\n"
        end

        SALT_ENV << "#ENV_END-"
        PILLARFILE << "#END_OF_PILLAR_ROOT-"
        ROOTFILE << "#END_OF_FILE_ROOT-"
 
        edited = text.gsub(/\#FILE_ROOT-.*\#END_OF_FILE_ROOT-/im, ROOTFILE)
        edited = edited.gsub(/\#PILLAR_ROOT-.*\#END_OF_PILLAR_ROOT-/im, PILLARFILE)
        edited = edited.gsub(/\#ENV_START-.*\#ENV_END-/im, SALT_ENV)
        File.open(filename, "w") { |file| file << edited }
    
    ################################################################ 
    # Start Vagrant
    ################################################################   
    Vagrant.configure("2") do |config|

        # check all versions of vagrant and plugins first
        ################################################################ 

        if Gem::Version.new(Vagrant::VERSION) > Gem::Version.new('1.4.0')
            $ok_msg = "Vagrant is #{Vagrant::VERSION}"
            puts $ok_msg
        else
        $err_msg = <<ERR
        
The Version of Vagrant is to old to be affective.  Please use 1.4.0 and above.

Visit http://www.vagrantup.com/downloads.html and update to continue

ERR
            puts $err_msg
            abort()
        end


        if Vagrant.has_plugin?("vagrant-hosts")
            $ok_msg = "The vagrant-hosts plugin is loaded"
            puts $ok_msg
        else
        $err_msg = <<ERR
    
WARNING

The vagrant-hosts plugin is required to ensure proper functionality. Use the
following command to install this plugin before continuing:

$ vagrant plugin install vagrant-hosts      
        
ERR
            puts $err_msg
            abort()
        end

        if defined? VagrantPlugins::HostsUpdater
            $ok_msg = "The vagrant-hostsupdater plugin is loaded"
            puts $ok_msg
        else
        $err_msg = <<ERR
    
WARNING

The vagrant-hostsupdater plugin is required to ensure proper functionality. Use the
following command to install this plugin before continuing:

$ vagrant plugin install vagrant-hostsupdater      
        
ERR
            puts $err_msg
            abort()
        end



        # Virtualbox specific settings for the virtual machine.
        ################################################################ 
        config.vm.provider :virtualbox do |v|
            v.gui = false
            v.name = CONFIG[:hostname]
            v.memory = CONFIG[:memory].to_i
            cores= CONFIG[:cores].to_i
            if cores>1
                v.customize ["modifyvm", :id, "--vram", CONFIG[:vram].to_i]
                v.customize ["modifyvm", :id, "--cpus", cores ]
                if CONFIG[:host_64bit] == 'true'
                    v.customize ["modifyvm", :id, "--ioapic", "on"]
                end
            end
        end

        # CentOS 6.4, 64 bit release
        ################################################################  
        config.vm.box     = "centos-64-x64-puppetlabs"
        config.vm.box_url = "http://puppet-vagrant-boxes.puppetlabs.com/centos-64-x64-vbox4210-nocm.box"
        
        # Set networking options
        ################################################################           
        if !(CONFIG[:hostname].nil? || !CONFIG[:hostname].empty?)
            config.vm.hostname = CONFIG[:hostname]
        end
        config.vm.network :private_network, ip: CONFIG[:ip]

        # register hosts for all hosts for apps and the server
        ################################################################
        # Local Machine Hosts
        # Capture the paths to all `hosts` files under the repository's provision directory.

        hosts = []
        config_obj[:apps].each do |app,obj|
            hosts.concat obj[:hosts]
        end
        
        if defined? VagrantPlugins::HostsUpdater
            config.hostsupdater.aliases = hosts
        end
        
        config.vm.provision :hosts do |provisioner|
          provisioner.add_host '127.0.0.1', hosts
        end

        # Set file mounts
        ################################################################           
        # Mount the local project's app/ directory as /var/app inside the virtual machine. This will
        # be mounted as the 'vagrant' user at first, then unmounted and mounted again as 'www-data'
        # during provisioning.
        config.vm.synced_folder "app", "/var/app", :mount_options => [ "uid=510,gid=510", "dmode=775", "fmode=774" ]

        # Provisioning: Salt 
        ################################################################              
        $provision_script=""
        $provision_script<<"curl -L https://raw.github.com/#{bootstrap}/bootstrap.sh | sudo sh -s -- -m #{CONFIG[:minion]} "
        
        # Set up the web apps
        #########################
        config_obj[:apps].each_pair do |appname, obj|
            $provision_script<<" -a #{appname}:#{obj[:repoid]} "
        end
        
        $provision_script<<" -i -b bootstrap -o jeremyBass \n"
        
        if !destroying
            puts "running : #{$provision_script}"
            config.vm.provision "shell", inline: $provision_script
        else
            puts "Destroyed the local server, now, on to the world."
        end
        puts "at the end of it"
    end
    