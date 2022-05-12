# inventorymaker
Automatically generate Ansible inventory file from CSV's.

Tested on Debian 11

### Setup
Inventorymaker uses the tool 'pass' to get passwords for the Ansible inventory file. The 'pass' program requires a working GPG setup.

Typical setup:

Create a GPG password and key for the current user:  
`gpg --full-gen-key`

Then init pass tool with  
`pass init`

Correctly configure gpg-agent with  
`inventorymaker-init`

If you have issues with GPG permissions, this might help:  
`find ~/.gnupg -type d -exec chmod 700 {} ; find ~/.gnupg -type f -exec chmod 600 {}`

GPG Manual: https://wiki.archlinux.org/title/GnuPG

### How to use
Create a folder that contains all your CSV source files.  
To convert all CSVs to a usable Ansible inventory file run:

`inventorymaker <SOURCE_FOLDER_TO_CSV> <OUTPUT_FILE>.ini`

### Scripting language
The variable '$USER' represents the current hosts username that executes Ansible.

### Known bugs
Avoid using group passwords and individual host passwords together. In some cases sudo authentication will not work.