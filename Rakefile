require 'erb'

STDOUT.sync = true
BASEDIR = File.dirname(__FILE__)

desc "Generate puppet hiera module"
task :build do
  build
end

desc "Generate puppet enterprise hiera module"
task :build_pe do
  @pe=true
  build
  patch_bin_hiera
end

desc "Remove puppet hiera module"
task :clean do
  clean
end

def build
  @moduledir = ENV['USER'] + "-hiera"
  @moduledir = @moduledir + '_pe' if @pe
  if File.directory?(@moduledir)
    puts "Cleaning up old module directory"
    FileUtils.rm_rf(@moduledir)
  end
  puts "Pulling hiera.git"
  if File.directory?("hiera-git")
    system("cd hiera-git && git pull")
  else
    system("git clone http://github.com/puppetlabs/hiera.git hiera-git")
  end
  puts "Pulling hiera-puppet.git"
  if File.directory?("hiera-puppet-git")
    system("cd hiera-puppet-git && git pull")
  else
    system("git clone http://github.com/puppetlabs/hiera-puppet.git hiera-puppet-git")
  end
  puts "Generating module structure"
  system("envpuppet puppet-module generate #{@moduledir}")
  puts "Syncing lib/ files to module"
  system("rsync -azPH hiera-git/lib/ #{@moduledir}/lib")
  system("rsync -azPH hiera-puppet-git/lib/ #{@moduledir}/lib")
  # TODO: remove gem hiera req from functions
  puts "Creating module manifests"
  # TODO ^
  FileUtils.mkdir("#{@moduledir}/templates")
  FileUtils.mkdir("#{@moduledir}/files")
  create_init_pp
  create_hiera_yaml
  create_common_yaml
  puts "Copying cli tool"
  FileUtils.cp("hiera-git/bin/hiera","#{@moduledir}/files/bin_hiera")
  puts "Cleaning up unneeded module files"
  # TODO ^
  puts "Module built in #{@moduledir}/"
end

def clean
  @moduledir = ENV['USER'] + "-hiera"
  puts "Cleaning up"
  FileUtils.rm_rf(@moduledir) if File.directory?(@moduledir)
  FileUtils.rm_rf(@moduledir + '_pe') if File.directory?(@moduledir + '_pe')
  FileUtils.rm_rf("hiera-git") if File.directory?("hiera-git")
  FileUtils.rm_rf("hiera-puppet-git") if File.directory?("hiera-puppet-git")
end

def patch_bin_hiera
  # Make template based on PE or not
  # if $puppetversion =~ "Puppet Enterprise"
  File.open(@moduledir + '/files/bin_hiera', 'r+') do |f|
    lines = (["#!/opt/puppet/bin/ruby\n","require 'puppet'\n","\n"] + f.readlines)
    f.rewind
    f.puts lines
  end
  # TODO: remove gem hiera req
end

def create_init_pp
   File.open("#{@moduledir}/manifests/init.pp",'w') do |f|
     f.write <<-EOF
class hiera(
  $hiera_yaml='/etc/#{"puppetlabs/" if @pe}puppet/hiera.yaml',
  $hiera_data='/etc/#{"puppetlabs/" if @pe}puppet/hieradata'
) {
  File {
    owner => '0',
    group => '0',
    mode  => '0644',
  }
  file { "/usr/bin/hiera":
    ensure => present,
    source => 'puppet:///modules/hiera/bin_hiera',
    mode   => '0755',
  }
  file { $hiera_data:
    ensure => directory,
  }
  file { "${hiera_data}/common.yaml":
    ensure  => present,
    replace => false,
    source  => 'puppet:///modules/hiera/common.yaml',
  }
  file { $hiera_yaml:
    ensure  => present,
    content => 'hiera/hiera.yaml.erb',
    replace => false,
    source  => $hiera_yaml
  }
  file { "/etc/hiera.yaml":
    ensure => symlink,
    target => $hiera_yaml,
  }
}
    EOF
  end
end

def create_hiera_yaml
   File.open("#{@moduledir}/templates/hiera.yaml.erb",'w') do |f|
     f.write <<-EOF
---
:backends: - yaml
:logger: console
:hierarchy:
- common

:yaml:
   :datadir: <%= hiera_data %>
    EOF
  end
end

def create_common_yaml
   File.open("#{@moduledir}/files/common.yaml",'w') do |f|
     f.write <<-EOF
---
hiera_data: "${calling_module}/files/hiera"
    EOF
  end
end
