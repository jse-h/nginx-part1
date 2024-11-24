# Table of Contents


# About Assignment

These instructions are set up using the Arch Linux distribution for Linux. Please refer to the appropriate commands and manuals if your distribution is different.
## Task 1 - User Setup

In this task, the system user is setup with a specified home directory structure. We are creating a **non-login system** user. This type of user is used over a regular or root user because it is being used strictly for system operations and managing our services. The system user is more controlled with limited permissions. Since it is a non-login user, there is also **no direct access available** compared to regular or root users meaning **less risk** in the user account and its files getting compromised.

### Creating the system user `webgen`

To create our system user `webgen`, copy and run the command below:

```
sudo useradd -r -d /var/lib/webgen -s /usr/sbin/nologin webgen
```

`-r` specifies it is a system account/user
`-d` specifies the path of home directory for our user `webgen`
`-s` specifies the shell using the appropriate `nologin` shell since our user is a non-login user

>[!NOTE]
> The `nologin` shell will refuse any login attempts.
### Building the home directory structure

Our new user requires some starter files and subdirectories. So, we are going to manually build the home directory for `webgen` instead of using the `-m` flag to automatically make it during the `useradd` command. After that then we can change the directories and files ownership to `webgen`.

This is the structure we want for our new user with the required files:

```
/var/lib/webgen/
├── bin/ 
│   └── generate_index 
└── HTML/ 
	└── index.html
```

First build the home directory for `webgen` by running the command below:
```
sudo mkdir -p /var/lib/webgen
```

Next, to create the `bin` and `HTML` subdirectories, run the command:
```
sudo mkdir -p /var/lib/webgen/bin /var/lib/webgen/HTML
```

Next we need to create the `generate_index` script file and `index.html` file. Run the commands separately to create them:
```
sudo touch /var/lib/webgen/bin/generate_index

sudo touch /var/lib/webgen/bin/index.html
```

The following will be our bash script that goes in the `generate_index` file. This script will be useful as it declares variables that grabs the system’s information and will output and display the information in an HTML format to our `index.html` file.

Run the command below to open up the file in the text editor:

```
sudo nvim /var/lib/webgen/bin/generate_index
```

Copy paste the code below using your text editor like `neovim` or `vim`.  For the rest of these instructions, I will be referring and using the `neovim` text editor.

```bash
#!/bin/bash

set -euo pipefail

# this is the generate_index script
# you shouldn't have to edit this script

# Variables
KERNEL_RELEASE=$(uname -r)
OS_NAME=$(grep '^PRETTY_NAME' /etc/os-release | cut -d '=' -f2 | tr -d '"')
DATE=$(date +"%d/%m/%Y")
PACKAGE_COUNT=$(pacman -Q | wc -l)
OUTPUT_DIR="/var/lib/webgen/HTML"
OUTPUT_FILE="$OUTPUT_DIR/index.html"

# Ensure the target directory exists
if [[ ! -d "$OUTPUT_DIR" ]]; then
    echo "Error: Failed to create or access directory $OUTPUT_DIR." >&2
    exit 1
fi

# Create the index.html file
cat <<EOF > "$OUTPUT_FILE"
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>System Information</title>
</head>
<body>
    <h1>System Information</h1>
    <p><strong>Kernel Release:</strong> $KERNEL_RELEASE</p>
    <p><strong>Operating System:</strong> $OS_NAME</p>
    <p><strong>Date:</strong> $DATE</p>
    <p><strong>Number of Installed Packages:</strong> $PACKAGE_COUNT</p>
</body>
</html>
EOF

# Check if the file was created successfully
if [ $? -eq 0 ]; then
    echo "Success: File created at $OUTPUT_FILE."
else
    echo "Error: Failed to create the file at $OUTPUT_FILE." >&2
    exit 1
fi
```

There is no need to copy and paste any code or text manually into `index.html` since it will be automatically filled by our `generate_index` script. 

### Setting ownership to `webgen`

Set ownership by copying and running the command below:

```
sudo chown -R webgen:webgen /var/lib/webgen
```

`-R` specifies to give ownership recursively of nested directories and files to `webgen`.

## Task 2 - Service and Timer Setup

With our new user and file structure setup, the next task is creating the service file and timer to run the `generate_index` script. Our setup will be made where the service is ran everyday at 05:00 giving us an updated display of the system information.

### Creating `generate_index.service` file

Using the text editor, create the service file in the specified path by running the command below:

```
sudo nvim /etc/systemd/system/generate_index.service
```

Then copy and paste the unit file sections below to the service file:

```shell
[Unit]
Description=Generate Index Script Service
Wants=network-online.target
After=network-online.target

[Service]
User=webgen
Group=webgen
ExecStart=/var/lib/webgen/bin/generate_index
```

In our `[Unit]` section, we use the optional dependency directive `Wants` requiring the network target dependency. Also, we use the `After` directive to tell the script the service to run only after the target is active.

In our `[Service]` section, we specify the user and group `webgen` for the service which tells the service to perform actions based on the limited permissions of `webgen`. Also, the files created by the service will be owned by the specified user and group.

### Creating `generate_index.timer` file

Now that we have our service and script files, we want the service to run that script everyday at 05:00. To do this we need to create a timer unit file similar to the service file.

Run the command below to create and edit a new file under the same path as the service file:

```
sudo nvim /etc/systemd/system/generate_index.timer
```

Then copy paste our configuration for the timer below:

```shell
[Unit]
Description=Generate Index Service Timer

[Timer]
OnCalendar=*-*-* 05:00:00
Persistent=true

[Install]
WantedBy=timers.target
```

The `Persistent` directive will tell the system to run the service if the specified timing was missed. For example, if the server was shut down before the time the service should have ran, the next time the server is up it will run the service immediately.

>[!CAUTION]
> By default, most servers are set to UTC time zone. If you want this timer to run at 05:00 during local time. Refer to the commands below to set a time zone.
> `timedatectl list-timezones ` to view your desired time zone, then:
> `sudo timedatectl set-timezone <time-zone-here>` 
> 
> For example, setting to Vancouver, Canada:
> `sudo timedatectl set-timezone America/Vancouver`



