#Puppet Training
___
##Certs
View your cert `certinfo /var/lib/puppet/ssl/cert`
Show the cert `certinfo /var/lib/puppet/ssl/certs/puppet.learnpuppet.net.pem`

##Facter
###Exposes system configuration

Get all facts with `facter`

Or just get one

	[root@puppet vagrant]# facter fqdn
	puppet.learnpuppet.net
	
You can export your own env vars into facter

	[root@puppet vagrant]# export FACTER_stuff=things
	[root@puppet vagrant]# facter stuff
	things

Define facts in a file in an executable file within `/etc/facter/facts.d/` in any language

It must run and output to stdout in the format of key=value

	my_fact=something
	foo=bar
	mem=1024
	
Can also use a .txt file

	key1=value
	key2=value
	key3=value
	
Or .json

	{
		"key1": "value",
		"key2": "value"
	}
	
Or .yaml

	---
	key1: value 
	key2: value

###Accessing facts
You can access facts in various formats:

```
[root@puppet vagrant]# facter system_uptime --yaml
---
system_uptime:
  hours: 1
  seconds: 4463
  days: 0
  uptime: 1:14 hours
[root@puppet vagrant]# facter system_uptime --json
{
  "system_uptime": {
    "seconds": 4485,
    "hours": 1,
    "uptime": "1:14 hours",
    "days": 0
  }
}
[root@puppet vagrant]# facter system_uptime --json^C
[root@puppet vagrant]# ^C
[root@puppet vagrant]# facter os --json
{
  "os": {
    "name": "CentOS",
    "family": "RedHat",
    "release": {
      "major": "6",
      "full": "6.7",
      "minor": "7"
    },
    "lsb": {
      "distdescription": "CentOS release 6.7 (Final)",
      "minordistrelease": "7",
      "release": ":base-4.0-amd64:base-4.0-noarch:core-4.0-amd64:core-4.0-noarch:graphics-4.0-amd64:graphics-4.0-noarch:printing-4.0-amd64:printing-4.0-noarch",
      "distid": "CentOS",
      "distcodename": "Final",
      "majdistrelease": "6",
      "distrelease": "6.7"
    }
  }
}
```

##Vocabulary
###Types
Useful to reference the [types](https://docs.puppetlabs.com/references/latest/type.html) document. Types are first class in the puppet DSL and specificy what type of thing you are configuring, i.e. service, file, cron etc.

##Puppet resource

Allows you to inspect the puppet config for something

```	
[root@puppet vagrant]# puppet resource service ntpd
service { 'ntpd':
  ensure => 'running',
  enable => 'true',
}
```

Or leave off the title to get everything for the type `puppet resource service`

Inspect the root user's puppet config
```
[root@puppet vagrant]# puppet resource user root
user { 'root':
  ensure           => 'present',
  comment          => 'root',
  gid              => '0',
  home             => '/root',
  password         => '$1$DPZPr3J6$szhkx2kA0sO4C1Y9gug1O0',
  password_max_age => '99999',
  password_min_age => '0',
  shell            => '/bin/bash',
  uid              => '0',
}
```

You can use it to change the system such as creating a user

```
[root@puppet vagrant]# puppet resource user gh
user { 'gh':
  ensure => 'absent',
}
[root@puppet vagrant]# puppet resource user gh ensure=present
Notice: /User[gh]/ensure: created
user { 'gh':
  ensure => 'present',
}
[root@puppet vagrant]# puppet resource user gh ensure=present
user { 'gh':
  ensure => 'present',
}
```

Give them a password and chage their shell

```
[root@puppet vagrant]# grub-md5-crypt
Password:
Retype password:
$1$dXbrX$IKBGdI6xtmaf8kIqxR3o31
[root@puppet vagrant]# puppet resource user gh password='$1$dXbrX$IKBGdI6xtmaf8kIqxR3o31'
Notice: /User[gh]/password: changed password
user { 'gh':
  ensure   => 'present',
  password => '$1$dXbrX$IKBGdI6xtmaf8kIqxR3o31',
}
[root@puppet vagrant]# puppet resource user gh shell=/bin/sh
Notice: /User[gh]/shell: shell changed '/bin/bash' to '/bin/sh'
user { 'gh':
  ensure => 'present',
  shell  => '/bin/sh',
}
```

##Configs
Configs are set in `/etc/puppet/puppet.conf`

Print them with `puppet config print`

##Modules
Self contained dur structure for encapsulating puppet code
###Good
A `apache` module that manages apache
###Bad
A `webserver` module that maages everything on a webserver

###Create a module
Create modules in `/etc/puppet/modules`

Create the module `puppet module generate nyt-motd`
nyt is the forge name (repo)

This will generate a module skeleton.

`puppet parser validate manifests/init.pp` will validate syntax of a puppet file.

##Classes
- contains the code
- can be incuded in other classes

###Modulepath
Where puppet looks for puppet modules

```
[root@puppet motd]# puppet config print modulepath
/etc/puppet/environments/production/modules:/var/local/ghoneycutt-modules/modules
```

##Files
Stored in the `<module>/files` dir
Files are referenced like `puppet:///modules/<module>/motd`

Instead of supplying `source`, you can supply `content` and set the contents of the file to a string literal or variable.


##Packages
###Do NOT use ensure => latest
If you do, you will end up updating software unintentionally whenever someone (anyone) sends a new version to the repo
###Don't use a specific version either!
Defeats the purpose of versioning packages in a repo

Only use ensure => installed. Any upgrades should be done in a manintenance effort outside of puppet.


##Package-File-Service pattern
###This is generally the order that should be followed.
1. Install packages so default files are present
2. Place your config files next, overwriting default files from package install
3. Ensure services are running

```
class ntp (
) {
  package {'ntp':
    ensure => installed,
    before => File['/etc/ntp.conf'],
  }

  file {'/etc/ntp.conf':
    ensure => present,
    source => 'puppet:///modules/ntp/ntp.conf',
    owner  => 'root',
    group  => 'root',
    mode   => '0644',
    notify => Service['ntpd'],
  }

  service {'ntpd':
    ensure    => running,
    enable => true,
  }
}
```


##Namevar

You can name a file or service anything, so long as you specify the actual path (for file) or name of service in the definition. This means the actual service or file path could be a variable which is useful if dealing with different OSs.

```
class ntp (

) {
  package {'ntp':
    ensure => installed,
    before => File['ntpd_conf'],
  }

  file {'ntpd_conf':
    ensure => present,
    path   => '/etc/ntp.conf',
    source => 'puppet:///modules/ntp/ntp.conf',
    owner  => 'root',
    group  => 'root',
    mode   => '0644',
    notify => Service['ntpd_service'],
  }

  service {'ntpd_service':
    ensure => running,
    enable => true,
    name   => 'ntpd',
  }
}
```

##Variables
Try to set vars ar the top of the manifest and then have resources below.
###Strings
Single (no interpolation) and double (interpolation) quotes.

Surround embeeded vars in curly braces `"I have a ${name}!!!\n"`

Define a var `$user = 'root'`

Facts are variables in the root scope `$banner = "Welcome to ${}"`

###Arrays
```
$nameservers = [
	'8.8.8.8',
	'8.8.4.4',
]
```
###Use a var
```
file { '/etc/motd':
	ensure => file, 
	owner => $user,}
```

##Debugging with notify
Use notify to print to stdout during a run `notify {"fqdn = <${::fqdn}>":}`


##Defines
You can define your own types like

```
#<module>/manifests/log.pp
define debugger::log (
    $msg = ''
) {
    notify {"Message: ${msg}":}
}

```

##Forge
A [place](https://forge.puppetlabs.com/) to find puppet modules. Use with caution. Some of them are shit or configured in weird ways.

