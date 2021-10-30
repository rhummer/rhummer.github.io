---
layout: post
title:  "TeslaMate - Google Hosting"
date:   2021-10-29 21:15:10 -0500
categories: tesla gcp free-tier teslamate
excerpt_separator: <!--end_excerpt-->
---
I've been using the amazing [Teslamate](https://github.com/adriankumpf/teslamate) software to keep track of charing/drives and other interesting data on my Tesla for a few years now. Hosted first on a RaspberryPi, then moved to my Synology server for something with a bit more horse power. Finally I wanted to move it to the Cloud for 100% uptime, and remote access. With out messing around with home network shenanigas.

<!--end_excerpt-->
I found the awesome setup article(s) here: 
* [https://www.teslaev.co.uk/how-to-setup-and-run-teslamate-for-free-on-google-cloud/](https://www.teslaev.co.uk/how-to-setup-and-run-teslamate-for-free-on-google-cloud/)
* [https://www.teslaev.co.uk/how-to-perform-an-automatic-teslamate-backup-to-google-drive/](https://www.teslaev.co.uk/how-to-perform-an-automatic-teslamate-backup-to-google-drive/)

The guides worked, but they needed a bit more fenessing, so figured I'd make some corrections

First the key reminders
* **Follow the instructions** - They are very good! You'll want to get the details right or you will be in a bit of a bind. The big call outs:
  * e2-micro for the vm instance type!
  * 30 GB **Standard persistent disk**!

Beyond that, getting the google drive sync was a bit annoying. Here's my notes/tweaks to the setup steps.

### Setting up the GCP Client ID & Secret
I'll repost the guide steps, with adendums:
* Select an existing project, or create a new one.
* Under **"ENABLE APIS AND SERVICES"** search for "Drive", and enable the "Google Drive API".
* Click "Credentials" in the left-side panel (not "Create credentials", which opens the wizard), then "Create credentials"
* Click on **"CONFIGURE CONSENT SCREEN"** button (near the top right corner of the right panel), then select "External" and click on "CREATE"; on the next screen, enter an "Application name" ("rclone" is OK), complete the support and developer email addresses (just your normal email) then click on "Save & Continue" Click "Save & Continue" again on the Scopes screen, click "Save & Continue" again on the Test Users screen - It should then show you a summary of what you have just setup.
* Click again on "Credentials" on the left panel to go back to the "Credentials" screen.
* Click on the **"+ CREATE CREDENTIALS"** button at the top of the screen, then select "OAuth client ID".
* Choose an application type of "Desktop app" if you using a Google account or "Other" if you using a Google Workspace account and click "Create". (the default name is fine - feel free to enter TeslaMate if you wish)
* ~~Click on the OAuth Consent Screen and Click PUBLISH APP and confirm you wish to publish.~~ Do **NOT** publish, Google requires a review process with a YouTube video, seems like more hasses than needed. You should clik the + ADD USERS under the Test users section so you are able to log into the service.

### Installing and configuring rclone
The rclone setup wasn't too bad to start with, the number is different, you can just type 'drive' instead of a number and that will start the google drive selection.

When it comes times to hit the OAuth portal. I ran into issue. I couldn't connect to the URL at all. So I stopped the setup at that step.

What I ended up doing was running the rclone steps locally via my local WSL install. If you are already on Linux/Mac just fire up your bash/shell/terminal instance and follow the steps.

I was able to then see the browser popup to login. I then ran the following command `rclone config file` to print out the location of the config file. I copyed the contents of the file into a file I created on the GCP VM as that has all the auth information you need. You can then continue the steps by setting up the backup dir and testing out the command.

I also updated the backup script. I'll show the script below, but how I setup my instance and backu is as follows:
* All the files (docker-compose, .env, .htpasswd, backup scripts, and a db restore script) live under `~/teslamate`
* The backup is exported to `~/teslamate/backup`
* Each backup is stored as a tar after being exported
* The whole `~/teslamate` folder is stored in a tar, then that file is send to Google Drive
* Besides the script. I enabled snapshots for the VM as you are allowed 5GB of snapshots for the Free Tier. I make a backup every week and keep two for now. As each backup is ~1.3GB currently
* With a weeks work of backups the resuting tar.gz is ~300MB. Each individual database dump is about 400MB before it is tar'd, after being compressed a single archive is ~45MB

Here's my script:
```bash
#!/bin/bash
PATH=/usr/local/bin:/usr/bin:/bin:/usr/local/games:/usr/games

# make our date var _<0-6 day of the week>_<String for day of the week> ie: _0_Sunday
now=`date +"_%w_%A"`

cd ~/
mkdir -p teslamate_backup
mkdir -p teslamate/backup/tmp

echo "Backing up database..."
cd teslamate
sudo /usr/local/bin/docker-compose exec -T database pg_dump -U <DB_USERNAME> <DB_NAME> ~/teslamate/backup/tmp/teslamate.pg_dump

echo "Compressing db export..."
cd backup/tmp
tar --use-compress-program='gzip -9' -cvf ../teslamate${now}.tar.gz *

echo "Archiving files..."
cd /~
sudo tar --exclude='teslamate/import' --exclude='teslamate/backup/tmp' -zcvf teslamate_backup/teslamate.tar.gz teslamate/
echo "Syncing files..."
rclone copy --max-age 24h --update --progress ~/teslamate_backup teslamate-gdrive:teslamate

echo "Cleaning files..."
rm -R -f teslamate_backup
```
