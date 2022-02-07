# Secure shell solution

Author:
Piotr Zych <piotr@conclusive.pl>

Primary assignee:

Other contributors:

Created: 2021-12-06


## Problem Description

Current OpenBMC firmware allows the normal shell to the user, which gets the user super user privileges through which the user can perform any operation eg: 
- user can stop/start any service
- can change the network configuration files
- can read/write the private credentials
etc.


### User scenarios
1. When a service user logs into the BMC system then it cannot access any system
crucial elements. It can only use the NVIDIA debug tool and get its output.


## Background and References


### Requirements

1. Unable to create/update the files that can impact the signed files that have been authenticated during secure boot, including via the overlayFS.
2. Unable to read out the secret data including the keys (e.g., private key of certificate and other keys like the local storage key).
3. Unable to change some important data such as the factory provisioning data and other data stored on OTP (e.g., some data for DOT if any).
4. Unable to change configurations and policies required by security requirements such as the network ports.
5. Unable to create or load other executables to run on the system.
6. A service user should be able to run the debug tool, which collects the BMC dump.


### Dependencies

This feature will be controlled by build time(Distro variable) which is named as "nvidia-service-account-policy.
Distro which want to enable the secure shell have to build the image by enabling this feature.
`DISTRO_FEATURES += "nvidia-service-account-policy"`


## Proposed Design

- Create the built in service user (with default password as 0penBmc).

- Setup `PATH` variable for user in `/etc/skel/.bashrc` file. This file is a template for `.bashrc` files for every newly created user. User's `PATH` variable should not contain `/usr/bin` or `/bin` paths. Instead of this `PATH` should point to the previously prepared directory `/usr/local/bin/nvidia`. It will solve problems with common exploitation techniques.

- Block `root` user and only allow a `service` user to access the debug shell via ssh. Add:
`
do_install:append(){
    sed -i 's/-G priv-admin/ -w/g' ${D}${sysconfdir}/default/dropbear
}
`

to the `dropbear_%.bbappend` file to modify default dropbear startup switches in the configuration file. `-w` switch disallows root logins.

- Create shell script `/bin/rbash` that invokes regular bash with `-r` switch (to launch a restricted shell). Make `/bin/rbash` executable: `chmod +x /bin/rbash`. Append `/bin/rbash` to `/etc/shells` (valid login shells list). Bash builtin command index (computerhope.com). Most of the available commands are shell built in commands which are used for writing the bash scripting. A user can not edit/read any file through the debug shell as the tools/commands to read the files are not available. Prevent the writing of the contents of system files with the `echo` command - append `alias echo=':'`.
A user can copy the binary to the `$HOME` or `/tmp` as before but he would not be able to run this program as he can not execute any program from its absolute path.
Eg: /usr/bin/xxx  : User can not execute as there is a / in the path.

- Modify service user, make restricted bash default shell: `usermod service -s /bin/rbash`.

- Create directory `/usr/local/bin`. This directory will be the location of soft links to binaries available for a user.

- Make sure that `/usr/local/bin` permissions are not too open. A user should not have write/modify permissions.

- Don’t give the write permission on file `.bashrc` whether the user has been created at runtime or if it is a built-in user. By doing so, a user can only execute `create-dump-dbus` (or `scp` command) and he would not be able to run any other shell commands except those of shell's built-in. Overwrite the `$PATH` environment variable and modify permissions to user's `.bashrc` file in `/etc/skel/.bashrc` file (such that any new user created in runtime inherits changes).

- Create a soft link to the debug collector binary in `/usr/local/bin/nvidia` location. Debug collector is a client-server application. Server application runs as a system service in order to have permission to call proper DBus method "CreateDump". Client is an application that is launched in user context to trigger dump creation. Connects to server via domain socket. After launch, the client sends a message to the server with a request to call "CreateDump". After the DBus method is done, the server responds with an created entry's name or error message.

- Downloading the debug dump will be possible via scp using command:
`scp -P 2222 /var/lib/phosphor-debug-collector/dumps/$1/* user_name@machine_address:destination_direcotry`

where `$1` is the dump number (returned by debug collector).

Assumptions:
- User can not copy any file on /usr/local/bin/nvidia as it is ROFS.
- Execution commands restricted in rbash
- A user cannot change the $PATH variable as it is read only.

**Above modifications will be provided as a script invoked by single shot service (executed once at the first boot), packed as a layer in recipes-nvidia directory.**


### Implementation vs common exploits

| Exploitation Technique | Result |
|--------|--------|
| If "/" is allowed you can run /bin/sh or /bin/bash | Not allowed |
| If you can run a cp command you can copy the /bin/sh or /bin/bash into your directory. | cp is not allowed |
| From ftp > !/bin/sh or !/bin/bash | ftp server is not there on the image |
| From gdb > !/bin/sh or !/bin/bash | Gdb is not there in the image |
| From more/man/less > !/bin/sh or !/bin/bash |  restricted: cannot redirect output |
| From vim > !/bin/sh or !/bin/bash | Vim is not allowed |
| From rvim > :python import os; os.system("/bin/bash ) | Rvim is not allowed as well as python is not there for the user. |
| From scp > scp -S /path/yourscript x y: | Scp is not allowed |
| From awk > awk 'BEGIN {system("/bin/sh or /bin/bash")}' | Awk is not allowed for the user. |
| From find > find / -name test -exec /bin/sh or /bin/bash \ |  restricted: cannot specify `/' in command names |
| From except > except spawn sh then sh |  except: command not found |
| From python > python -c 'import os; os.system("/bin/sh")' | Python is not there in the image |
| From perl > perl -e 'exec "/bin/sh";' | Perl is not there in the image |
| From ssh > ssh username@IP - t "/bin/sh" or "/bin/bash" | doesn't work |
| From ssh2 > ssh username@IP -t "bash --noprofile" | doesn't work |
| From ssh3 > ssh username@IP -t "() { :; }; /bin/bash" (shellshock) | doesn't work |
| From ssh4 > ssh -o ProxyCommand="sh -c /tmp/yourfile.sh" 127.0.0.1 (SUID) | doesn't work|
| From git > git help status > you can run it then !/bin/bash |  Git is not there in the image |
| From pico > pico -s "/bin/bash" then you can write /bin/bash and then CTRL + T | Pico is not there in the image |
| From zip > zip /tmp/test.zip /tmp/test -T --unzip-command="sh -c /bin/bash" | zip is not there in the image |
| From tar > tar cf /dev/null testfile --checkpoint=1 --checkpointaction=exec=/bin/bash | tar: unrecognized option '--checkpoint=1' |


## Licensing
As this is only configuration without any source code changes then there should not be any issue with licensing.


## Testing
Functional testing is predicted using a dedicated .robot script.

Make sure:
- Service user should not be able to access the shell(sh or bash).
- Service user can not change the network configuration file.
- Service user can not change the network port configuration file.
- Service user can not change/read the private key file.
- Service user can not change/read the password file.
- Service user can not load/run other executables on the system.
- Service user is able to run a debug tool, which collects the BMC dump.
- "/" character is disallowed in paths to binaries.
- "cp" command is disallowed for service user.
- "ftp" server is disallowed for service user.
- "gdb" is not in the BMC system.
- Redirection ">", "<" operations are restricted.
- "vi", "vim" and "rvim" editors are disallowed for service user.
- "python", "perl" and other execution environments are disallowed for service user.
- "awk", "except", "find", "ssh", "ssh2", "ssh3", "ssh4", "git", "pico", "zip", "tar"  are not allowed for the user.
- "echo" command is muted for service user (to prevent read passwords/private keys).
