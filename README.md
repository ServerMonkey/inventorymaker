# inventorymaker(1) -- CSV to Ansible inventory file converter

Handle your complete Single Source of Truth (SSOT) using CSV files that are
both readable and writable by humans.

## FEATURES

* Generates Ansible inventory file
* Uses 'pass' to manage
  passwords: [www.passwordstore.org](https://www.passwordstore.org/)

## SETUP

Inventorymaker uses the 'pass' CLI tool to get passwords for the Ansible
inventory file. The 'pass' program requires a working GPG setup.

Typical setup:

Create a GPG password and key for the user:  
`gpg --full-gen-key`

Then init pass tool with that email:  
`pass init your.name@example.org`

Correctly configure gpg-agent with:  
`inventorymaker-init`

If you have issues with GPG permissions, this might help:  
`find ~/.gnupg -type d -exec chmod 700 {} ; find ~/.gnupg -type f -exec chmod 600 {}`

GPG
Manual: [wiki.archlinux.org/title/GnuPG](https://wiki.archlinux.org/title/GnuPG)

## BASIC USAGE

### Syntax:

`inventorymaker -h, --help`

`inventorymaker <FOLDER_WITH_CSVS | template> <OUTPUT_FILE> [OPTIONS]`

`Options can be: 'o' (force overwrite OUTPUT_FILE) and/or 'd' (debug)`

Inventorymaker will parse the CSVs and apply custom parsing rules that will
result in a directly usable Ansible inventory file. Do not edit the final
Ansible inventory file manually. During conversion the following things happen:

* Inventorymaker will import passwords from the 'pass' tool. This is a password
  manager that uses GPG to encrypt passwords. inventorymaker will look for
  passwords in: `~/.password-store/ansible/<PASS_PATH>.gpg`
  If the GPG password is set to a none-empty string, inventorymaker will ask
  for your GPG password. For automated servers it is recommended to set the GPG
  password to an empty string.
* Due to speed considerations, passwords will be stored in plain text in the
  output Ansible inventory file (file attribute will be set to 600, only
  readable by the current user).
* Will automatically add 'local' and 'localhost' to the inventory file.

## CSV FORMAT

* Each folder represents a confined SSOT source.
* Each CSV file in that folder represents a group of hosts.
* The CSV file must use commas as separators.
* Filenames must end in .csv
* The first line always contains the (Ansible) variable names.
* The second line always contains group values for all hosts below.
* All following lines contain individual host values.
* Individual host values will override group values but try to avoid using
  group values and individual host values together. Choose one or the other.
* Inline comments can be added by starting a cell with '#'.

## VARIABLES

Variables in the CSV file are not exactly the same as Ansible variables. They
are simplified by removing the prefix 'ansible_'. Meaning the variable
'HOSTNAME' will result in the output variable 'ansible_hostname'.

* The minimum required variables are: HOSTNAME and, USER or CHILD
* The CHILD variable can be used instead of HOSTNAME, to group host together.
  This can simply point to another CSV file in the same directory. Just use the
  name of the file without the .csv extension. For example node1.csv node2.csv
  can be grouped together in the file rack.csv. Ansible can now target the
  name 'racks', see the example below. This can be very useful when defining a
  large number of hosts that are similar.

| CHILD  |
|--------|
| node1  |
| node2  |

* There are new variables that extend Ansible's functionality. They start with
  'im_' or 'tag_', see list below.
* Ansible variables that are not included in the list below, can be utilized by
  adding the complete variable with the 'ansible_' prefix.
* You can use your own custom variables, they will automatically get the
  prefix 'im_' in the final Ansible inventory .ini file.
* Recommended to only use static variables that don't change during the
  lifetime of a host. Otherwise, it is better to define variables in a
  playbook.
* Subsequent occurrences of the same variable, for example MAC, NET or CD,
  shuld be named with an underscore and a number like this:  
  CD,CD_2,CD_3 ... or NET,NET_2 ...  
  Don't call them CD1, CD2, or CD_1. The first occurrence of the variable shall
  not contain an underscore or a number.

Variables:

| IM VARIABLE        | ANSIBLE VARIABLE           | WILL BE PARSED |
|--------------------|----------------------------|----------------|
| BECOME             | ansible_become             | Yes            |
| BECOME_METHOD      | ansible_become_method      | Yes            |
| BECOME_PASSWORD    | ansible_become_password    | Yes            |
| CD                 | im_cd                      |                |
| CHILD              | tag_child                  | Yes            |
| CPU                | im_cpu                     |                |
| DESCRIPTION        | im_description             |                |
| DISK_SIZE          | im_disk_size               |                |
| DISK_TYPE          | im_disk_type               |                |
| DNS                | im_dns                     |                |
| FACTORY_HOST       | im_factory_host            |                |
| FACTORY_PASSWORD   | im_factory_password        | Yes            |
| FACTORY_USER       | im_factory_user            | Yes            |
| HARDWARE           | im_hardware                |                |
| HOST               | ansible_host               | Yes            |
| HOSTNAME           | ansible_hostname           | Yes            |
| HPILO_HOST         | im_hpilo_host              |                |
| HPILO_PASSWORD     | im_hpilo_password          | Yes            |
| HPILO_USER         | im_hpilo_user              | Yes            |
| IPMI_HOST          | im_ipmi_host               |                |
| IPMI_PASSWORD      | im_ipmi_password           | Yes            |
| IPMI_USER          | im_ipmi_user               | Yes            |
| LOCATION           | im_location                |                |
| MAC                | im_mac                     |                |
| MEM                | im_mem                     |                |
| NET                | im_net                     |                |
| ORDER              | im_order                   |                |
| OS                 | im_os                      |                |
| PASSWORD           | ansible_password           | Yes            |
| PORT               | ansible_port               | Yes            |
| PYTHON_INTERPRETER | ansible_python_interpreter | Yes            |
| STATE              | im_state                   | Yes            |
| USER               | ansible_user               | Yes            |

Variables in the section 'WILL BE PARSED' will be used by inventorymaker to
customize the Ansible inventory file. The other variables will be included in
the final Ansible inventory file but will not be manipulated by inventorymaker.
'WILL BE PARSED' variables are recommended to be used in the CSV file but are
not mandatory.

To see what each variable's value does it is recommended to generate a template
and read the comments in that file. See further down under 'EXAMPLE USAGE'.

## VARIABLE VALUES

Some variables and values are more complicated, they are explained here:

**ORDER variable** : This integer value sets the order in which virtual
machines are supposed to be deployed.  
This variable can be used by other programs automate VM deployment. For example
vmh: [github.com/ServerMonkey/vmh](https://github.com/ServerMonkey/vmh)  
Settings a value here ultimately marks the host as a VM. In that case the 'OS'
variable should be the name of the VM template (Golden Image) and HOSTNAME
should be the name of the VM.  
For example to deploy a VM with the name 'myvm' first and 'myvm2' second, set
the ORDER variable to 1 for 'myvm' and 2 for 'myvm2'.  
For parallel deployment set the ORDER variable to the same value.

**PASSWORD and USER variable** : If this value is set to 'LOCAL'
inventorymaker will assume that this is a physical-access-only system, that can
be reached only via a physical terminal. This host will be excluded from the
exported Ansible inventory file. This is useful if you want to manage hosts in
your source inventory but not in Ansible.

**STATE variable** : The value for the STATE variable is used to
determine the systems state of a host. It is up to your playbooks to handle the
different states. Recommend using the purposes from the list below. When using
DOWN and EXC, the host will automatically be excluded from the exported Ansible
inventory file. This is useful if you want to manage hosts in your source
inventory but not in Ansible.

| VALUE   | PURPOSE                                              |
|---------|------------------------------------------------------|
| UP      | host is supposed to always be online / booted        |
| DOWN    | permanently offline, will be excluded from inventory |
| MAN     | manual, either booted or offline                     |
| MAINT   | maintenance, under repair or broken                  |
| EXC     | exclude, will be excluded from inventory             |
| LIQ     | liquidate, is supposed to be removed soon            |
| UNKNOWN | unknown state, existing but unknown host             |

**`$USER` value** : This value will be replaced with the current users Posix
username. This is not a shell-script variable. Example:

| HOSTNAME  | OS         | USER  | PASSWORD |
|-----------|------------|-------|----------|
| winbox    | Windows 11 | peter | PASS     |
| ubuntubox | Ubuntu     | $USER | PASS     |

**`PASS` value** : This value will be replaced with the password from the
password manager 'pass'. It will automatically retrieve the password from the
password storage by searching for the matching group and username. Works only
for the variables in the list below.

| VARIABLE         | MATCHING USER |
|------------------|---------------|
| PASSWORD         | USER          |
| BECOME_PASSWORD  | USER          |
| FACTORY_PASSWORD | FACTORY_USER  |
| IPMI_PASSWORD    | IMPI_USER     |
| HPILO_PASSWORD   | HPILO_USER    |

Let's demonstrate with this example file named 'myhosts.csv':

| HOSTNAME  | USER  | PASSWORD |
|-----------|-------|----------|
| centosbox | jack  | PASS     |

Inventorymaker will look for the password
in `~/.password-store/ansible/myhosts/centosbox/jack.gpg`.

When using value for the variable CHILD in the same row, the password path will
be the name of the child group, not the hostname.

**`PASS:<PATH>` value** : Custom path, instead of matching the username
automatically. You can specify the relative search path. For example: `PASS:
pcs/jacks-pc`, will retrieve the password
from `~/.password-store/ansible/pcs/jacks-pc.gpg`.

If you don't want to use plain-text passwords in your Ansible inventory you can
ommit pass and let Ansible handle passwords like this instead:
```"{{ lookup('passwordstore', 'ansible/myusername', errors='strict')}}"```
Be aware that when you have hundrets of passwords this will significantly slow
down your plays. Or use Ansible vaults.

## EXAMPLE USAGE

Create a folder that contains all of your CSV source files. In this example we
use the folder '~/lab/inventory_src'

Create a CSV file from the template in that folder:

`$ inventorymaker template homelab.csv`

Now open the CSV file in any CSV editor that respects the CSVs default format.
For example LibreOffice Calc, VisiData (CLI) or the 'CSV Editor' plugin for
your IDE, by Martin Sommer. There are many more.

* Add your hosts to the CSV file under 'HOSTNAME', remember to skip the second
  row because it is used for group values.
* Add passwords and other variables

It should look something like this:

| HOSTNAME          | HOST         | USER   | PASSWORD       | DESCRIPTION  |
|-------------------|--------------|--------|----------------|--------------|
| # group variables |              |        |                | Test machine |
| my-workstation    | 192.168.40.2 | peter  | aplaintextpass |              |
| ftp.example.com   |              | user34 | PASS           |              |

To convert the source inventory to an Ansible inventory file, run:

`$ inventorymaker "$HOME/lab/inventory_src" "$HOME/lab/homelab.ini"`

Now inventorymaker will abort and complain that the password for 'user34' is
missing from the pass store. Add it by running:

`$ pass insert ansible/homelab/ftp.example.com`

Then run inventorymaker again. You should now have a working Ansible inventory
file. It is a good idea to open and verify it.

## KNOWN BUGS

Avoid using group passwords and individual host passwords together. In some
cases sudo authentication will not work.

## SEE ALSO

ansible(1), wildwest(1), vmh(1), pass(1)
