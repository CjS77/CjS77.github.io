---
layout: post
title:  "Automated backing up to dropbox"
date:   2015-08-20
header-img: img/dropboxbackup.png
author: Cayle Sharrock
categories: backup, dropbox
---

# Automatically backing up a git repo to Dropbox on Linux

Here I'll describe how I automatically backup my git repos to dropbox using

 * cron - automated job scheduler
 * rsnapshot - incremental backup tool based on rsync
 * rsnaptar - modified script to archive backups
 * upload_dropbox - script to upload files to dropbox

{:note: .note}

<div class="note" markdown='1'>
I use this method to back up my Git repositories, but you can use this approach to backup anything. 

*Hint*: I actually back up my `/etc/` folder in the same setup.
</div>

The cool properties of the backups are

* They're automated (running every night)
* Automatic rotation (I keep 6 daily backups, 4 weekly backups and 6 monthly)

Ok, let's go

## Setting up rsnapshot

This is straightforward. 

### Install rsnapshot

        sudo apt-get install rsnapshot

### Configure rsnapshot

It's easiest to use the global config to configure the backups, so you don't have to worry about permissions etc. I've set up rsnapshot to back up my `repositories` and `etc` folders.
Edit `/etc/rsnapshot.conf` and modify the following sections to your taste

    ###########################
    # SNAPSHOT ROOT DIRECTORY #
    ###########################

    # All snapshots will be stored under this root directory.
    
    snapshot_root   /var/backups/

This is where the *copies* (snapshots) will be stored. *Note*: rsnapshot uses harlinks, so only files that have changed between backups are actually copied, keeping your space used manageable.

    #########################################
    #           BACKUP INTERVALS            #
    # Must be unique and in ascending order #
    # i.e. hourly, daily, weekly, etc.      #
    #########################################

    #retain         hourly  0
    retain          daily   4
    retain          weekly  4
    retain          monthly 6

I want the last 4 days' backed up; also weekly backups going back a month; and monthly backups going back 6 months.

    ###############################
    ### BACKUP POINTS / SCRIPTS ###
    ###############################

    # LOCALHOST
    backup  /data/git-repositories/  git-repos/
    backup  /etc/                    etc/

Then specify which folders must be backup up and where.

{:note: .warning}
<div class="note warning" markdown='1'>
Note the comments at the top of the `.conf` file regarding tabs and trailing slashes.
</div>    

### Schedule the tasks with cron

Edit `/etc/cron.d/rsnapshot` and uncomment the tasks matching the backup frequencies you desire

    # This is a sample cron file for rsnapshot.
    # The values used correspond to the examples in /etc/rsnapshot.conf.
    # There you can also set the backup points and many other things.
    #
    # To activate this cron file you have to uncomment the lines below.
    # Feel free to adapt it to your needs.

    # 0 */4         * * *           root    /usr/bin/rsnapshot hourly
     30 1   * * *           root    /usr/bin/rsnapshot daily
     15 1   * * 1           root    /usr/bin/rsnapshot weekly
     0  1   1 * *           root    /usr/bin/rsnapshot monthly

### Summary

Cool. We've installed rsnapshot, configured it, and scheduled it to run automatically. Now we'll zip our backups up, encrypt them, and upload them to DropBox.

If the preceding instructions were a bit terse, there is a more detailed write up [here](https://www.howtoforge.com/set-up-rsnapshot-archiving-of-snapshots-and-backup-of-mysql-databases-on-debian)

## Packaging the snapshots

Unless you want to mirror entire directory structures to DropBox, you'll want some way of packaging your snapshots up before copying them to the cloud. Here we'll use a script I've written to handle this for us.

### Get some pre-requisites

You will need `openssl` and `dropbox_uploader`. The former is probably already installed, otherwise `sudo apt-get install openssl`. The latter is available on [GitHub](https://github.com/andreafabrizi/Dropbox-Uploader)

Follow the instructions on the [upload_dropbox](https://github.com/andreafabrizi/Dropbox-Uploader) page to set up an application key for your DropBox account.

Note that the API commands will run under your user name (hence the tarballs created in the script below are world-readable).

Make a symlink to the dropbox_uploader.sh file and make it executable:

    sudo ln -s /path/to/install/location/dropbox_uploader.sh /usr/local/bin/dropbox_uploader.sh
    sudo chmod +x /usr/local/bin/dropbox_uploader.sh

### Create a password

Create a random password that will be used to encrypt your backups:

    cd ~/.ssh
    cat `openssl rand -base64 32` > key.bin

### The backup script

Save the following script to `/usr/local/bin`, changing the variables to match your needs.
The script is based on the sample script in `/usr/share/doc/rsnapshot/examples/utils/rsnaptar`. Make it executable as well using `sudo chmod +x /usr/local/bin/rsnaptar`

    #!/bin/bash

    ##############################################################################
    # rsnaptar
    # by Cayle Sharrock
    #
    # A quick hack of a shell script to tar up backup points from the rsnapshot
    # snapshot root. 
    # 
    # Uploads tarball to DropBox when done
    #
    # http://nimbustech.biz/
    ##############################################################################

    umask 0022
    # DIRECTORIES
    TAR_DIR="/var/backups/toDropbox" # <-- CHANGE THIS
    SNAPSHOT_DIR="/var/backups"      # <-- CHANGE THIS
    DropBoxFolder=my-backups         # <-- CHANGE THIS
    uploader=/usr/local/bin/dropbox_uploader.sh   # <- check this
    # SHELL COMMANDS
    DATE=`/bin/date +%Y-%m-%d`

    # uncomment this to encrypt files
    # the e-mail address the notification is being sent to must have their GPG key
    # in the public keyring of the user running this backup
    #
    GPG="/usr/bin/openssl"          # <-- Comment this out to have unencrypted tarballs
    #The PUBLIC key to encrypt data with
    key="/home/cayle/.ssh/key.bin"  # <-- CHANGE THIS
    mkdir -p ${TAR_DIR}
    echo "" > ${TAR_DIR}/backup.log

    for freq in daily weekly monthly; do
      for i in {0..6}; do
        dest=${freq}.${i}
        if [ -d ${SNAPSHOT_DIR}/${dest} ]; then
          echo "Processing $dest"
          mkdir -p ${TAR_DIR}/${dest}/
          cd ${SNAPSHOT_DIR}
          for BACKUP_POINT in `ls ${SNAPSHOT_DIR}/${dest}`; do
            # GPG encrypt backups if $GPG is defined
            if test ${GPG}; then
                tar --numeric-owner -cf - ${dest}/${BACKUP_POINT}/ | \
                            $GPG enc -aes-256-cbc -salt -pass file:${key} | gzip > ${TAR_DIR}/${dest}/${BACKUP_POINT}.tar.enc.gz
            # just create regular tar files
            else
                tar -czf ${TAR_DIR}/${dest}/${BACKUP_POINT}.tar.gz ${dest}/${BACKUP_POINT}/
            fi
            echo "${dest}/${BACKUP_POINT}           $DATE" >> ${TAR_DIR}/backup.log
          done
        else
          echo "${dest} doesn't exist. Skipping..."
        fi
      done
    done

    echo "Uploading backups to Dropbox"
    sudo -u cayle $uploader upload ${TAR_DIR}/* ${DropBoxFolder}/            # <- CHANGE USER
    echo "Done"

    cd -

### Schedule the upload

Tack the following line on the `/etc/cron.d/rsnapshot` file you edited above:

     0  2   * * *           root    /usr/local/bin/rsnaptar


## Final notes

1. The Dropbox backups do NOT make use of incremental backups, so watch your quota if you're keeping many backups.
2. To *unencrypt* an encrypted tarball run `gunzip --stdout FILENAME.tar.enc.gz | openssl enc -d -aes-256-cbc -pass file:/path/to/key.bin | tar -xv`
  This command first unzip the tarball, sending the ecrypted stream to openssl. which decrypts the data using the password you generated earlier. 
  The decrypted data is then passed to tar which will extract it to disk. use `tar -tv` to test that a tarball is properly encrypted.
