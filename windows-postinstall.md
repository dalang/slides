% Introduce to Windows Postinstall
% Dong Guoxing (gxdong@ctrip.com)
% 2014.1.13


## Agenda

* Components involved in Implementation
* Scripts in Postinstall
* Windows Postinstall Workflow
* Dive into Puppet
    - Master/Agent mode
    - Puppet on Windows
    - Example
* Customize Scripts

## Components

In our Windows Postinstall Provisioning, the following components will be involved:

* Razor Server
    - Control Baremetal Provisioning Process
* Samba Server
    - Host files (drivers, installers and scripts) used during provisioning
    - Serve as Share Folder in Windows Postinstall
* Puppet
    - master node
    - agent nodes

Normally, razor, samba and puppet master will be hosted on the same node.

# Scripts

## Types

We divide the scripts into **two** types:

* Dynamic
    - Parameters passed from Front-End
    - Each provision instance has his own values
* Static
    - Immutable
    - All provision instances shared

## Comparision

In aspect of location, obtain and invoke method.

<br/>

***

<table><thead>
<tr>
<th>Aspects</th>
<th>Dynamic</th>
<th>static</th>
</tr>
</thead><tbody>
<tr>
<td>*Location*</td>
<td>.erb Templates in Razor</td>
<td>Samba</td>
</tr>
<tr>
<td>*Obtain-method*</td>
<td>send http request to Razor</td>
<td>copy from Samba</td>
</tr>
<tr>
<td>*Invoke-method*</td>
<td>Puppet agent service</td>
<td>Puppet agent service</td>
</tr>
</tbody></table>

***

#
## Postinstall Workflow

flow chat

<iframe frameborder='0' src='https://mural.ly/mural/dalang/1389535010456/embed?b=2912&amp;l=1308&amp;r=6408&amp;t=732' width='510px' height='315px'></iframe>

checkout [the complete view of the flow chat](http://mrl.li/1eOjfCs)

<!-- Dive into Puppet -->
#
## master/agent mode

![Compilation](fig/manifest_to_defined_state_unified.png)

## master/agent mode (Cont'd)

* Puppet agent runs as a service, and triggers a Puppet run about every half hour (configurable).
* Puppet agent requests a pre-compiled catalog from a puppet master.
* The puppet master always use one special manifest (site.pp), to compile a catalog and sends back to the agent.
* After getting the catalog, the agent applies it.

## Puppet on Windows

* Puppet runs as a 32-bit process.
    - when run on 64-bit windows, the [File System Redirector](http://msdn.microsoft.com/en-us/library/aa384187(v=vs.85).aspx) will redirect access to `%windir\system32` to `%windir%\SysWOW64` instead
* [reboot module from PuppetLabs](https://forge.puppetlabs.com/puppetlabs/reboot)
    - Check if need OS reboot after Patching
    - Do reboot operation if needed
    - After OS restart, puppet agent service will be trigger again.
    - Exit loop until there is no patches to be install.

## Puppet on Windows (Cont'd)

Puppet Resources

* Package
    - mainly used to install software

```ruby
package { "mcafee-agent":
  name            => "McAfee Agent", # The software's name
  ensure          => present,
  provider        => windows,
  source          => 'C:\oem\McAgent\setup\FramePkg.exe', # The software installer location
  install_options => [ # additional options to pass when installing
    '/SILENT',
    {
      'INSTALL'   => 'AGENT',
    }
  ],
  require         => Exec["checkin"],
}
```

## Puppet on Windows (Cont'd)

Puppet Resources

* Exec
    - run commands directly or execute a .bat file

```ruby
exec { "config-os-template":
  command => 'C:\oem\svrtemp\svrtemp.cmd',
  path    => 'C:\Windows\System32',
  require => Package['megaraid-manager'],
}

exec { "add-wmi-user":
  command => 'C:\oem\wmiUser\adduser.cmd',
  require => Exec["config-os-template"],
}
```

## Puppet on Windows (Cont'd)

Exec Resource

>Executes external commands.
>It is critical that all commands executed using this mechanism can be run multiple times without harm,
>i.e., they are idempotent.


>There is a strong tendency to use exec to do whatever work Puppet can’t already do;
>while this is obviously acceptable (and unavoidable) in the short term,
>it is highly recommended to migrate work from exec to native Puppet types as quickly as possible.

## Resource Ordering

```ruby
file {'/tmp/test1': #1
  ensure  => present,
  content => "Hi.",
}

file {'/tmp/test2': #2
  ensure => directory,
  mode   => 644,
}

notify {"I'm notifying you.":} #3
notify {"So am I!":} #4
```

When we ran this, the resources weren’t synced in the order we wrote them:
it went `#1`, `#2`, `#4` and `#3`.

So when dealing with **related resources**, we have to express their relationships.
We usually declare resource relationship with **require** metaparameters.

## Troubleshooting

* create .pp file on agent node with you own manifest, such as test.pp
* Start Command Prompt with Puppet (run in 32-bit process)
* run "puppet apply test.pp --debug --trace --verbose" in Puppet Console (output debug info)

***

**P.S.** when invoke .bat file, all commands must run in **non-interactive (or silence) mode**.

## Example 1
Zabbix Installation failed

```bat
:: /oem/zabbix/zabbix_agent_install.bat
@ECHO OFF
pushd "%~dp0"

md "C:\Program Files\Zabbixagentd\"
xcopy /Y zabbix_agentd.conf "C:\Program Files\Zabbixagentd\"
xcopy /Y zabbix_agentd.exe "C:\Program Files\Zabbixagentd\"

"%programfiles%"\Zabbixagentd\zabbix_agentd.exe --install -c "%programfiles%"\Zabbixagentd\zabbix_agentd.conf
"%programfiles%"\Zabbixagentd\zabbix_agentd.exe --start -c "%programfiles%"\Zabbixagentd\zabbix_agentd.conf

popd
exit /b
```

* HINT: The Value of "%programfiles%" is "C:\\Program Files (x86)\\" in Puppet context

## Example 2

Puppet agent service hanged while installing MegaRAID Storage Manager (MSM)

```text
To install the product in a silent mode, use the following commands:
       a. Install the VC Redist Package from command line "vcredist_x86.exe /Q",
          vcredist_x86.exe is available in \DISK1\ISSetupPrerequisites\VC Redist Installation.
           If vcredist_x86.exe is already installed, run the setup command.
       b. setup.exe /s /v"/qn SETUPTYPE="<setuptype>" [REMOVEUTIL="<util1>[,<util2>]"]"  from the installation disk.
           The REMOVEUTIL option will remove the utility from install list.
           <utilx> can be either Popup or SNMP
           <setuptype> can be StandAlone
```

* HINT: Missing VC Redist Package, and MSM setup.exe would try to install it automaticly, **but NOT in silence mode**.

#
## Customize Scripts

* Which type of scripts?
    - Dynamic: add (or edit) .erb template file to Razor
    - Static: add (or edit) .bat file to Samba Server

* Then modify Puppet manifest to invoke these scripts properly.

## Sample Case

HuaWei Server(windows2k8) nic teaming

* TODO
    1. Install `Broadcom Management Programs`
    1. Invoke `bcmteam.cmd`

* Static Scripts
    - Copy required files to Samba Server

## Broadcom Management Programs

```ruby
package { "broadcom":
  name => 'Broadcom Management Programs',
  ensure => present,
  provider => windows,
  source => 'C:\oem\huawei\BCM\BCSP.exe',
  install_options => ['/s', '/v/qn'],
  require => Package['megaraid-manager'],
}
```

## Invoke `bcmteam.cmd`

```ruby
exec { "nic-team":
  command => 'C:\oem\WinTeam\bcmteam.cmd 1',
  require => Exec["turn-off-nic"],
}
```

## Test in Puppet Standalone Mode

run `"puppet apply test.pp --debug --trace --verbose"` on Puppet Console

```ruby
# test.pp

package { "broadcom":
  name => 'Broadcom Management Programs',
  ensure => present,
  provider => windows,
  source => 'C:\oem\huawei\BCM\BCSP.exe',
  install_options => ['/s', '/v/qn'],
}

exec { "turn-off-nic":
command => 'C:\oem\NICOff\NICOff.cmd',
require => Package['broadcom'],
}

exec { "nic-team":
  command => 'C:\oem\WinTeam\bcmteam.cmd 1',
  require => Exec["turn-off-nic"],
}
```

#
## References

* [Postinstall Workflow of Windows Provision](https://mural.ly/#/dalang/1389535010456)
* [Puppet Resource Type Reference](http://docs.puppetlabs.com/references/latest/type.html)
* [Writing Manifests for Windows](http://docs.puppetlabs.com/windows/writing.html)
* [Troubleshooting Puppet on Windows](http://docs.puppetlabs.com/windows/troubleshooting.html)

#
## Thanks

> Q&A
