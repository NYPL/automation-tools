[![Travis CI](https://travis-ci.org/artefactual/automation-tools.svg?branch=master)](https://travis-ci.org/artefactual/automation-tools)

Automation Tools
================

The Automation Tools project is a set of python scripts, that are designed to automate the processing of transfers in an Archivematica pipeline.

<!-- doctoc: https://www.npmjs.com/package/doctoc -->
<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->


- [Installation](#installation)
- [Customizing](#customizing)
  - [Hooks](#hooks)
    - [get-accession-id](#get-accession-id)
    - [pre-transfer hooks](#pre-transfer-hooks)
    - [user-input](#user-input)
  - [Logs](#logs)
  - [Multiple automated transfer instances](#multiple-automated-transfer-instances)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

Installation
------------

1. Create github ssh credentials on the Archivematica instance and clone this repository
```
sudo mkdir /usr/lib/archivematica/automation-tools
sudo chmod 777 /usr/lib/archivematica/automation-tools
git clone --recursive git@github.com:NYPL/automation-tools /usr/lib/archivematica/automation-tools
```
2. Switch to the correct branch for the pipeline and check to make sure the configurations were downloaded.
  * If there is no `configuration` folder, contact the owner of this repo to request the required permissions.
```
cd /usr/lib/archivematica/automation-tools
git checkout <branch>
ls configuration
```
3. Install required python packages
```
sudo virtualenv /usr/share/python/automation-tools
cd /usr/share/python/automation-tools
source bin/activate
pip install -r /usr/lib/archivematica/automation-tools/requirements.txt
deactivate
```
4. Create correct folders with expected ownership
```
sudo install -d -o archivematica -g archivematica /var/log/archivematica/automation-tools
sudo install -d -o archivematica -g archivematica /var/archivematica/automation-tools
sudo install -d -o archivematica -g archivematica /etc/archivematica/automation-tools
```
5. Copy pipeline-specific configuration to expected location
```
sudo cp /usr/lib/archivematica/automation-tools/configuration/* /etc/archivematica/automation-tools
sudo chown -R archivematica:archivematica /etc/archivematica/automation-tools/
```

6. Test the installation of the tools (/etc/archivematica/automation-tools/transfer-script.sh should contain a completed version)
```
cd /usr/lib/archivematica/automation-tools/
/usr/share/python/automation-tools/bin/python -m transfers.transfer --user <user> --api-key <apikey> --ss-user <user> --ss-api-key <apikey> --ss-url <url> --transfer-source <transfer_source_uuid> --config-file /etc/archivematica/automation-tools/transfers.conf
```
7. Create a crontab entry to schedule a repeated runs of the script.
```
*/5 * * * * /etc/archivematica/automation-tools/transfer-script.sh
```

Customizing
------------

### Hooks

During processing, automate transfers will run scripts from several places to customize behaviour. These scripts can be in any language. If they are written in Python, we recommend making them source compatible with python 2 or 3.

There are three places hooks can be used to change the automate tools behaviour.

* `transfers/get-accession-number` (script)
* `transfers/pre-transfer` (directory)
* `transfers/user-input` (directory)

Any new scripts added to these directories will automatically be run alongside the existing scripts.

There are also several scripts provided for common use cases and examples of processing that can be done.
These are found in the `examples` directory sorted by their usecase and can be copied or symlinked to the appropriate directory for automation-tools to run them.
If you write a script that might be useful for others, please make a pull request!

#### get-accession-id

* _Name:_ `get-accession-id`
* _Location:_ Same directory as transfers.py
* _Parameters:_ [`path`]
* _Return Code:_ 0
* _Output:_ Quoted value of the accession number (e.g. `"ID 42"`)

`get-accession-number` is run to customize the accession number of the created transfer. Its single parameter is the path relative to the transfer source location.  Note that no files are locally available when `get-accession-id` is run. It should print to standard output the quoted value of the accession number (e.g. `"ID42"`), `None`, or no output. If the return code is not 0, all output is ignored. This is POSTed to the Archivematica REST API when the transfer is created.

#### pre-transfer hooks

* _Parameters:_ [`absolute path`, `transfer type`]

All executable files found in `pre-transfer` are executed in alphabetical order when a transfer is first copied from the specified Transfer Source Location to the Archivematica pipeline. The return code and output of these scripts is not evaluated.

All scripts are passed the same two parameters:

* `absolute path` is the absolute path on disk of the transfer
* `transfer type` is transfer type, the same as the parameter passed to the script. One of 'standard', 'unzipped bag', 'zipped bag', 'dspace'.

There are some sample scripts in the pre-transfers directory that may be useful, or models for your own scripts.

* `00_file_to_folder.py`: If the transfer is a single file (eg a zipped bag or DSpace transfer), it moves it into an identically named folder. This is not required for processing, but allows other pre-transfer scripts to run.
* `00_unbag.py`: Repackages a bag as a standard transfer, writing md5 hashes from bag manifest into metadata/checksum.md5 file. This enables use of scripts such as add_metadata.py with bags, which would otherwise cause failure at the bag validation job.
* `add_metadata.py`: Creates a metadata.json file, by parsing data out of the transfer folder name.  This ends up as Dublin Dore in a dmdSec of the final METS file.
* `archivesspace_ids.py`: Creates an archivesspaceids.csv by parsing ArchivesSpace reference IDs from filenames.  This will automate the matching GUI if a DIP is uploaded to ArchivesSpace.
* `default_config.py`: Copies the included `defaultProcessingMCP.xml` into the transfer directory. This file overrides any configuration set in the Archivematica dashboard, so that user choices are guaranteed and avoided as desired.

#### user-input

* _Parameters:_ [`microservice name`, `first time at wait point`, `absolute path` , `unit UUID`, `unit name`, `unit type`]

All executable files in the `user-input folder` are executing in alphabetical order whenever there is a transfer or SIP that is waiting at a user input prompt. The return code and output of these scripts is not evaluated.

All scripts are passed the same set of parameters.

* `microservice name` is the name of the microservice awaiting user input. E.g. Approve Normalization
* `first time at wait point` is the string "True" if this is the first time the script is being run at this wait point, "False" if not. This is useful for only notifying the user once.
* `absolute path` is the absolute path on disk of the transfer
* `unit UUID` is the SIP or transfer's UUID
* `unit name` is the name of the SIP or transfer, not including the UUID.
* `unit type` is either "SIP" or "transfer"

There are some sample scripts in the pre-transfers directory that may be useful, or models for your own scripts.

* `send_email.py`: Emails the first time a transfer is waiting for input at Approve Normalization.  It can be edited to change the email addresses it sends notices to, or to change the notification message.

### Logs

Logs are written to a directory specified in the config file (or `/var/log/archivematica/automation-tools/` by default). The logging level can be adjusted, by modifying the transfers/transfer.py file. Find the following section and changed `'INFO'` to one of `'INFO'`, `'DEBUG'`, `'WARNING'`, `'ERROR'` or `'CRITICAL'`.

    'loggers': {
        'transfer': {
            'level': 'INFO',  # One of INFO, DEBUG, WARNING, ERROR, CRITICAL
            'handlers': ['console', 'file'],
        },
    },

### Multiple automated transfer instances

You may need to set up multiple automated transfer instances, for example if required to ingest both standard transfers and bags. In cases where hooks are the same for both instances, it could be achieved by setting up different scripts, each one invoking the transfers.py script with the required parameters. Example:

```
# first script invokes like this (standard transfer):
/usr/share/python/automation-tools/bin/python -m transfers.transfer --user <user>  --api-key <apikey> --ss-user <user> --ss-api-key <apikey> --transfer-source <transfer_source_uuid_for_std_xfers> --config-file <config_file>

# second script invokes like this (unzipped bags):
/usr/share/python/automation-tools/bin/python -m transfers.transfer --user <user>  --api-key <apikey> --ss-user <user> --ss-api-key <apikey> --transfer-source <transfer_source_2_uuid_for_bags> --config-file <config_file_2> --transfer-type 'unzipped bag'
```

`<config_file_1>` and `<config_file_2>` should specify different file names for db/PID/log files. See transfers.conf and transfers-2.conf in etc/ for an example

In case different hooks are required for each instance, a possible approach is to checkout a new instance of the automation tools, for example in `/usr/lib/archivematica/automation-tools-2`
