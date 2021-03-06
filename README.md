This is a node.js command line utility for configuring new sudo mesh nodes.

Early alpha status. Things may break.

# About

makenode combines a set of configuration file templates with information like generated SSH keys, assigned IP address ranges, private wifi pasword, etc. then bundles the resulting configuration files into an ipk package, sends the ipk to the node using scp and installs the package. makenode can also optionally flash the node using the sudomesh firmware to prepare it for configuration.

# Install Dependencies

System dependencies

* dropbear

```
sudo apt-get install dropbear
```
OR for OSX (w/ brew)
```
brew install dropbear
```

* fakeroot

```
sudo apt-get install fakeroot
```
OR for OSX (w/ brew)
```
brew install fakeroot
```

* gcc

```
sudo apt-get install build-essential
```
OR for OSX
```
xcode-select --install
```
(I think?)


Install node.js packages:

```
npm install
```

# Settings

Copy and tweak settings file

```
cp settings.js.example settings.js
```

Add your public ssh key to `./configs/authorized_keys`

# Usage

```
Usage: ./makenode.js

Options:
  --firmware: Flash firmware before configuring. See ubi-flasher for relevant command line arguments.
  --ip: Router IP address (default: 172.22.0.1)
  --port: SSH port (default: 22)
  --password: SSH root password (default: meshtheplanet)
  --detectOnly: Report hardware detection results and exit.
  --detectOnlyJSON <file>: Write hardware detection results in JSON format to file and exit.
  --hwInfo <file>: Read hardware info results from file instead of detecting.
  --offline: Prompt user for the parameters usually provided by the meshnode database.
  --ipkOnly: Generate .ipk file but don't automatically upload or install to node.
```

Defaults and other settings can be overwritten in the settings.js file.

# Patching
makenode is used to rapidly integrate updates, patches, and fixes into sudowrt firmware. However, instead of re-running makenode on every node on your network, you may find it more convenient to create patch that can be pushed remotely. Existing patches and instructions for executing them can be found in the [patches](https://github.com/sudomesh/makenode/tree/master/patches) directory. To create the most simple patch of altering a single file, use the following instructions,

1. create two copies of the files you would like to patch (one before, one after), either from a test node or modified from this repo (note: it is inadvisable to use `git diff`, since changes are made to the files before they are packed by makenode)

2. Delete any lines from these copies that are user/node specifc (e.g. IP address, hostname, hashed passwords)

3. send the ouput of `diff -Naur` to a patchfile, for example,
```
diff -Naur tunneldigger_before tunneldigger_after > bug0017.patch 
```

4. edit the patch so the file path is global and the file replaces itself (assuming that is the desired behavior), for example,
```
--- tunneldigger_before    2018-02-23 05:49:40.677861696 -0800
+++ tunneldigger_after    2018-02-23 05:49:55.302282993 -0800
```
would become,
```
--- /etc/config/tunneldigger    2018-02-23 05:49:40.677861696 -0800
+++ /etc/config/tunneldigger    2018-02-23 05:49:55.302282993 -0800
```

5. copy the patch file to a test node that is in the "before" state.

6. test the patch with dry run by sshing into your test node and running,
```
patch --dry-run -p1 < bug0017.patch
```
note: may first need to install the `patch` package as this has yet to be rolled into makenode or the sudowrt firmware. To install `patch`, run the following on your node,
```
opkg update
opkg install patch
```

7. If the dry-run succeeds, attempt to run the patch with something like,
```
patch -p1 < bug0017.patch
``` 
You may see a warning about "fuzz", this is triggered when the original file does not match patch exactly. Potential side effects of this fuzz are unknown. 

8. Check that the file was correctly patched.

Every patch should come with clear instructions on usage and/or a shell script that automates most of the process for node whisperers.   

Additionally, if patching a file that contains or modifies user/node specific information, it may be necessary to write helper scripts that discovers such information and inserts it into the patch file.

# Details

makenode gathers data in the following ways

* Hardware detection by running commands on the node
* User input
* Requesting unique IP assignment from a remote meshnode database
* Key generation (for SSH host keys)
* Password generation (human readable or XKCD style)

The data is gathered and combined with templates by recursively walking through the configs/ dir and subdirs. For each dir the following occurs:

If present, the config.js file is imported and the function it exports is run. That function will receive the hardware info gathered during hardware detection (the hwInfo object) and can run information gathering functions (or schedule async information gathering to be run later) and return a configuration object with values to be combined with template files. If a templates/ directory exists in the same subdir as a config.js file, and the config.js file does not return null, then those templates are included in the ipk. This allows the inclusion of different versions of configuration templates based on e.g. user input or hardware.

The config objects returned from all config.js files are combined into a single config object and all templates (except for those that were ignored) are combined with the config object using the underscore.js template compiler. 

The resulting files are packaged as an ipk, sent to the node and installed.

## Offline Mode

Usage: ./makenode.js --offline offlineConfig.json

offlineConfig.json contains object properties that you'd like to override/set.
offlineConfig.json is a sample of such a file.
Looking through /configs/templates/\* will give you a pretty good idea of what variables are templated and can be set. Adding templated
variables in those files and then adding them to offlineConfig.json *should* add those variables to the configured node.

Here's a sample offlineConfig.json file with commenting because we can't have comments in a json file:
```
{ 
  "mesh_addr_ipv4": "100.1.2.65",
  "mesh_subnet_ipv4": "100.0.0.0",
  "mesh_subnet_ipv4_bitmask": "12",
  "mesh_subnet_ipv4_mask": "255.240.0.0",
  "adhoc_addr_ipv4": "100.1.2.65",
  "adhoc_subnet_ipv4_bitmask": "32",
  "adhoc_subnet_ipv4_mask": "255.255.255.255",
  "tun_addr_ipv4": "100.1.2.65",
  "tun_subnet_ipv4_bitmask": "32",
  "tun_subnet_ipv4_mask": "255.255.255.255",
  "open_addr_ipv4": "100.1.2.65",
  "open_subnet_ipv4": "100.1.2.64",
  "open_subnet_ipv4_mask": "255.255.255.192",
  "open_subnet_ipv4_bitmask": "26",
  "mesh_addr_ipv6": "a237:473:2349:a1::30:1:92",
  "exit_node_mesh_ipv4_addr": "100.0.0.1",
  "relay_node_inet_ipv4_addr": "104.236.181.226",
  "tx_power": "20",
  "id": "Sudomesh-Node-Sample",
  "operator": {
        "name": "Sample Node Operator",
        "email": "node_operator1@sudomesh.org",
        "phone": null
    }
}
```

# License

GPLv3

Copyright 2014 Marc Juul
