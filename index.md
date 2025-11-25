# Locally archive emails from Google Workspace or other cloud accounts.

## Benefits:
If you want to save emails locally and stop paying for those accounts. Or protect against risk of loss of those accounts.

## Use Cases
Say an employee leaves and you want to stop paying monthly for their cloud account, but need to preserve the emails.

Or, what happens if the vendor (Google or Microsoft) decides they don't want you to use their system anymore? You would lose all your emails - what a disaster that would be.

## Alternatives
I found a couple apps that address these use cases and archive your emails accounts. I found and tried <a href='https://github.com/s1t5/mail-archiver'>https://github.com/s1t5/mail-archiver</a> and <a href='https://github.com/LogicLabs-OU/OpenArchiver'>https://github.com/LogicLabs-OU/OpenArchiver</a>, and Wayne Sheng (of the latter project) even wrote a nice blog post talking about his motivation for creating OpenArchiver: <a href='https://www.indiehackers.com/post/open-archiver-open-source-email-archiving-tool-with-full-text-search-1df19e095e'>https://www.indiehackers.com/post/open-archiver-open-source-email-archiving-tool-with-full-text-search-1df19e095e</a>

## Risks of using a complicated solution
But what happens if Wayne or @s1t5 and company decide to stop maintaining either of these two projects? Well, you'd be in the same vendor lock-in situation that is partly the goal of these projects to avoid (not getting locked into Google Workspace or Microsoft 365 or whatever). Sure, they are open source projects and you can maintain it yourself, but that's not so accessible if you're not a software engineer or don't want to fathom the whole project to maintain it for yourself.

I tried both projects and they work, but the user interfaces were a bit clunky in that I wanted to access the accounts like webmail, not like an archiving tool. Also, OpenArchiver was very resource heavy. My docker host was groaning under the weight - it wanted a lot of CPU and RAM and it appeared to be a fairly complex app with multiple orchestrated containers.

## A more durable solution
I wanted a stripped down, fast, and durable setup. Enter dovecot. Widely used and around since 2002 - I can forsee it being supported well into the future. It's lightweight, fast, and compatible with standard email clients.

## The architecture
1. Use `mbsync` to download the emails from Google/Microsoft to your computer.
2. Use `dovecot` to serve the emails in a standard email archive.
3. Use `roundcube` to provide a gui to access the accounts.

## Downside
The only downside I can think of for this setup versus the projects above is that if you want webmail client only you need to log in separately to each account, instead of being able to search and query across all of your archived emails. I suppose you could set up multiple docker instances of `roundcube` and log into those separately. But you could also just set up a desktop client like Thunderbird and add all your accounts there. p.s. I changed the session lifetime for roundcube from the default 10 minutes to 1 year so I don't get logged out. If you don't want that feature or want to change it to a different timeout level, you can do so easily in the docker-compose.yml file.


# Install

## 1. Set up the folder structure
```
mkdir -p $HOME/email-archive/dovecot
mkdir -p $HOME/email-archive/mail
mkdir -p $HOME/email-archive/mbsync
mkdir -p $HOME/email-archive/roundcube_build
```
### Create user folders and set permissions
```
mkdir -p $HOME/email-archive/mail/username1
mkdir -p $HOME/email-archive/mail/username2
chown -R 1000:1000 $HOME/email-archive/mail/username1
chown -R 1000:1000 $HOME/email-archive/mail/username2
```
### Set permissions (we are using a virtual user 1000 here)
```
chown -R 1000:1000 $HOME/email-archive/mail
chown -R 1000:1000 $HOME/email-archive/mbsync
cd $HOME/email-archive
```
### Save roundcube settings to persistent location
```
mkdir -p $HOME/email-archive/roundcube/db
chown -R 33:33 $HOME/email-archive/roundcube/db   # Make it owned by UID 33 (www-data) which is the user roundcube runs as
```

## 2. Define `mbsync` in a Dockerfile
```
 cat > $HOME/email-archive/mbsync/Dockerfile <<'EOT'
FROM alpine:latest
# Install isync (which contains the mbsync executable) and CA certificates for SSL
RUN apk add --no-cache isync ca-certificates
# Create the virtual user (ID 1000) inside the container
RUN adduser -D -u 1000 -h /home/app appuser
WORKDIR /home/app
USER appuser
# Default command (can be overridden)
CMD ["mbsync", "-a"]
EOT
```

## 3. OPTIONAL: Define `roundcube` that extends session lifetime from 10 min to 1 year
p.s. If you don't want this, then change the docker-compose file to comment out the `build:` line and uncomment the `image:` line.
```
 cat > $HOME/email-archive/roundcube_build/Dockerfile <<'EOT'
FROM roundcube/roundcubemail:latest

# Permanently patch the default session lifetime from 10 to 525600 (1 year)
# We modify the source template so it survives resets
RUN sed -i "s/\$config\['session_lifetime'\] = 10;/\$config\['session_lifetime'\] = 525600;/g" /usr/src/roundcubemail/config/defaults.inc.php
EOT
```

# 4. Create `dovecot.conf`
## This config tells Dovecot to look for emails in the `Maildir` format in `/srv/mail`
```
 cat > dovecot/dovecot.conf <<'EOT'
#dovecot_config_version = 2.4.2   # Only works for versions >2.4.x (does not work for 2.3.x). Leaving here in case of later upgrade of dovecot to 2.4.x
listen = *
log_path = /dev/stdout
# Note: Dovecot sees the volume at /srv/mail
# Mail Location (Maildir format)
# Expects folders like: `/srv/mail/username1@gmail.com`
# `:LAYOUT=fs` to allow reading standard folders
# `:INBOX=...` so Dovecot finds the Inbox folder created by mbsync
mail_location = maildir:/srv/mail/%u:LAYOUT=fs:INBOX=/srv/mail/%u/Inbox

# Authentication: Allow plain text (okay since it's internal docker network)
disable_plaintext_auth = no
auth_mechanisms = plain login

# Force Dovecot to use ID 1000 (The Virtual User)
# User Database (Static: All users share the same UID/GID for simplicity)
userdb {
  driver = static
  args = uid=1000 gid=1000 home=/srv/mail/%u
}

# Password Database (File-based)
passdb {
  driver = passwd-file
  args = scheme=PLAIN username_format=%u /etc/dovecot/users
}
protocols = imap
EOT
```

# 5. Create `dovecot/users` file
### Define the user for your local email archive
This is the login you will use in Roundcube or your whatever desktop email GUI client to access the local files after you sync from your cloud provider.
### DO NOT DELETE USERS FROM THIS FILE.
If you remove them from this file, you will lose the ability to log in and view emails, even though the files are still on the disk. You must add new users to this file, not replace them.
### CRITICAL Concept
As you add more accounts later (Bob, Alice), simply append them to this list. Do not remove a user when you are done syncing that account.
```
 cat > $HOME/email-archive/dovecot/users <<'EOT'
username1:{PLAIN}redactedlocalpassword
username2:{PLAIN}redactedlocalpassword
EOT
```

# 6. Create `.mbsyncrc` config file
### If you simply sync * (everything), at least for gmail, you will download every email at least twice (once in its specific folder, and once in All Mail), doubling your storage requirement and cluttering your search results.
### For a usable archive that looks like your old inbox, you should sync everything except Sent Mail (included in All Mail) and Important (which is Google's algorithmic noise).
### You can delete account blocks after you are done syncing, if you don't need to sync them anymore (e.g., after you delete those accounts from your cloud provider).
```
 cat > $HOME/email-archive/mbsync/.mbsyncrc <<'EOT'
# Group 1 (example of a long-term sync account): For username1@gmail.com
# Group 2 (example of one-time sync, then delete from cloud account): For username2@yourdomain.com
# --- GLOBAL DEFAULTS ---
Create Near    # Create local directories if they don't exist
Sync Pull      # Only pull from Google, never push back deletions. Change to Sync All for long-term maintained (instead of one-time and deleted) accounts.
Expunge None   # Do not delete local files if they vanish from Google (Double safety for archiving).
               # `Expunge Near` means if the gmail version is missing, then delete it locally.
               # `Expunge Far` means if a file is missing on the Near side (local), delete it from Far (gmail).
               # We defined `Channel username1-main` below:
               # Far :username1-cloud:   <-- We defined this as the IMAPStore (Google)
               # Near :username1-local:  <-- We defined this as the MaildirStore (Your Disk)

# ===================================================================
# ACCOUNT 1: username1@gmail (Long-Term Maintained Account Example)
# ===================================================================
IMAPAccount username1-remote
Host imap.gmail.com
User username1@gmail.com
Pass "redacted_app_password"   # search for how to create an app password, it's not hard. on google set up 2FA then create an app password.
TLSType IMAPS
AuthMechs LOGIN
#PipelineDepth 50   # PipelineDepth 50 helps speed up the transfer on high latency connections.
PipelineDepth 1     # PipelineDepth 1 is slow, but it is a safer way to download a large mailbox without getting banned.

IMAPStore username1-cloud
Account username1-remote

MaildirStore username1-local
# Path is the Maildir itself. Ensure this path matches the volume you mount to Dovecot
Path /home/app/mail/username1/
Inbox /home/app/mail/username1/Inbox
# Verbatim - This creates real folders.
SubFolders Verbatim

# Channel 1: The Main Folders (Inbox, Sent, Drafts, Trash)
Channel username1-main
Far :username1-cloud:
Near :username1-local:
Pattern "[Gmail]/Drafts" "[Gmail]/Trash" "[Gmail]/Starred" "[Gmail]/Spam" "[Gmail]/All Mail"   # these are gmail examples, adapt for Microsoft, etc.
# Expunge Near   # Deletes local copy (Near) if deleted on gmail (Far). But if the account is deleted, it will [auto-]delete locally! (if automated sync via cron, etc.)

# Channel 2: Custom Labels (excluding All Mail/Spam) - User Created Labels (The "Work", "Project X" stuff)
Channel username1-labels
Far :username1-cloud:
Near :username1-local:
# This captures "INBOX" and any custom user folders (e.g. "Work", "Receipts")
Patterns * !"[Gmail]*"

Group long-term-username1-gmail
Channel username1-main
Channel username1-labels


# ======================================================================
# ACCOUNT 2: username2@yourdomain (One-Time Clone Then Delete Example)
# ======================================================================
IMAPAccount username2-remote
Host imap.gmail.com
User username2@yourdomain.com
Pass "redacted_app_password"
TLSType IMAPS
AuthMechs LOGIN
PipelineDepth 50   # PipelineDepth 50 helps speed up the transfer on high latency connections.
#PipelineDepth 1   # PipelineDepth 1 is slow, but it is a safer way to download a large mailbox without getting banned.

IMAPStore username2-cloud
Account username2-remote

MaildirStore username2-local
Path /home/app/mail/username2/
Inbox /home/app/mail/username2/Inbox
SubFolders Verbatim

Channel username2-main
Far :username2-cloud:
Near :username2-local:
# --- FOLDER LOGIC ---
# "[Gmail]/All Mail"    - REQUIRED if you want to save "Archived" emails (no label).
#                       - If YES: You get everything, but it duplicates Inbox/Sent (uses ~2x space).
#                       - If NO:  You save space, but lose all archived/unlabeled mail.
# "[Gmail]/Sent Mail"   - If you skip this (to save space), you can search All Mail to find sent items and move them to Sent (in Roundcube).
# "[Gmail]/Drafts"      - Download (Unique data, not included in All Mail)
# "[Gmail]/Trash"       - Download (Unique data, not included in All Mail)
# "[Gmail]/Spam"        - Download (Unique data, not included in All Mail)
# "[Gmail]/Starred"     - Download (Duplicative, but probably small)
# "[Gmail]/Important"   - SKIP. It is 100% duplicated in All Mail/Inbox and is just algorithmic noise.
# "INBOX"               - SKIP. It is already caught by the 'username2-labels' channel below `Pattern *`.

# --- CHOOSE YOUR PATTERN ---
# OPTION 1: The "Clean & Efficient" (Recommended if you don't care about 'Archived' mail)
# result: Perfect Roundcube usability. Zero duplication. Loses hidden/archived mail.
#Pattern "[Gmail]/Drafts" "[Gmail]/Trash" "[Gmail]/Starred" "[Gmail]/Spam" "[Gmail]/Sent Mail"

# OPTION 2: The "Completionist" (Recommended for maximum safety)
# result: Perfect Roundcube usability. Saves 100% of history. High duplication (Inbox/Sent stored twice).
#Pattern "[Gmail]/Drafts" "[Gmail]/Trash" "[Gmail]/Starred" "[Gmail]/Spam" "[Gmail]/Sent Mail" "[Gmail]/All Mail"

# OPTION 3: The "Space Saver" (Inbox is duplicated in All Mail, but some slight de-duplication of Sent Mail)
# result: 100% History. Lower disk usage. "Sent" folder will be empty - but can manually move sent emails (search `from:username2@yourdomain.com`) from All Mail to Sent folder.
# Note that if you manually move Sent emails, then the All Mail folder in Roundcube will be misleading because it will be missing Sent Mail.
Pattern "[Gmail]/Drafts" "[Gmail]/Trash" "[Gmail]/Starred" "[Gmail]/Spam" "[Gmail]/All Mail"

Channel username2-labels
Far :username2-cloud:
Near :username2-local:
# This captures "INBOX" and any custom user folders (e.g. "Work", "Receipts")
Patterns * !"[Gmail]*"

Group one-time-username2
Channel username2-main
Channel username2-labels
EOT
# Set permissions again just in case
chown 1000:1000 $HOME/email-archive/mbsync/.mbsyncrc
chmod 644 $HOME/email-archive/mbsync/.mbsyncrc
```

# x. Define the `docker-compose.yml` file
```
 cat > $HOME/email-archive/docker-compose.yml <<'EOT'
services:
  # 1. The Fetcher (mbsync)
  fetcher:
    build: ./mbsync                             # Builds the Dockerfile we just made
    container_name: email_archive_fetcher
    user: "1000:1000"                           # Run as Virtual User 1000
    volumes:
      - ./mail:/home/app/mail                   # Mail storage
      - ./mbsync:/home/app/config:ro            # Config folder mount - User 1000
    # No restart policy because this is a run-on-demand tool
    # We don't want this running 24/7 yet, so we leave 'restart' off.
    # It acts like a tool we invoke when needed.

  # 2. The IMAP Server (Dovecot)
  dovecot:
    image: dovecot/dovecot:2.3.21.1         # the latest version of dovecot 2.3.x
    container_name: email_archive_dovecot
    restart: unless-stopped
    environment:
      - DOVECOT_PASSDB_PASSWD_FILE_PATH=/etc/dovecot/users
    volumes:
      - ./mail:/srv/mail                    # Where the emails live and Dovecot sees them
      - ./dovecot/users:/etc/dovecot/users:ro
      - ./dovecot/dovecot.conf:/etc/dovecot/dovecot.conf:ro
    ports:
      - "143:143"                           # IMAP (StartTLS/Plain)

  # 3. The Web Client (Roundcube)
  roundcube:
    # BUILD OUR CUSTOM IMAGE
    build: ./roundcube_build                # comment this if you don't want to use custom Roundcube Dockerfile above (that changes timeout from 10 min to 1 year)
#    image: roundcube/roundcubemail:latest  # uncomment this if you don't want to use custom Roundcube Dockerfile above (that changes timeout from 10 min to 1 year)
    container_name: email_archive_roundcube
    restart: unless-stopped
    environment:
      - ROUNDCUBEMAIL_DEFAULT_HOST=dovecot  # Connects to dovecot container
      - ROUNDCUBEMAIL_SMTP_SERVER=None      # Disable sending
      - ROUNDCUBEMAIL_PLUGINS=archive,zipdownload
      - ROUNDCUBEMAIL_DB_TYPE=sqlite        # Tell Roundcube to use SQLite
    volumes:
      - ./roundcube/db:/var/roundcube/db    # Map directly to the internal default location
    ports:
      - "8000:80"
    depends_on:
      - dovecot
EOT
```

# Check if app password works

## Use `mbsync` to ask server for list of all folders
```
docker compose run --rm fetcher mbsync -c /home/app/config/.mbsyncrc -l long-term-username1-gmail
docker compose run --rm fetcher mbsync -c /home/app/config/.mbsyncrc -l one-time-username2
```


# Sync!! This copies email to your local system (Execution Commands - Runbook)
## Build & Start Infrastructure (make sure you're in the `email-achiver` folder: `cd $HOME/email-archiver`
docker compose build fetcher
docker compose up -d
## Run the "One-Time" Archive (username2) This runs the sync only for the one-time group defined in the config.
docker compose run --rm fetcher mbsync -c /home/app/config/.mbsyncrc one-time-username2
## Wait for this to finish. Log into Roundcube with user username2 / redactedlocalpass to verify everything is there.
## If this is taking too long and you want to kill the process:
docker ps
docker kill <CONTAINER_ID>
Example: `docker kill 7d0faaa765ac`

## Repeat for the "Long-Term" Sync (username1)
```
docker compose run --rm fetcher mbsync -c /home/app/config/.mbsyncrc -l long-term-username1-gmail   # Optional Dry Run `-l switch`
docker compose run --rm fetcher mbsync -c /home/app/config/.mbsyncrc long-term-username1-gmail      # Real Download
```


# Check Everything to see if it worked!

## VERIFY: Check size of mailboxes on disk
`du -sh $HOME/email-archive/mail/*`

## CHECK the Google Admin web console for size of gmail usage. Check against the size on disk and number of emails downloaded locally.
## This creates a temporary container, installs openssl, and drops you into a shell
```
docker compose run --rm -u root --entrypoint /bin/sh fetcher
apk add openssl
## Connect to gmail
openssl s_client -connect imap.gmail.com:993 -crlf -quiet
a login username@yourdomain.com redacted_app_password
b STATUS "INBOX" (MESSAGES)
c STATUS "[Gmail]/All Mail" (MESSAGES)
d logout
```

## Check by logging into Roundcube to verify everything is there.
Go to: http://your.docker.host.ip:8000
User: username1
Pass: redactedlocalpassword
## Turn on folders:
Roundcube > Settings > Folders > flip on all the gmail folders
Roundcube > Settings > Preferences > Displaying Messages > Allow remote resources (images, styles) > always > Save


# Delete the account - be careful!
Check all the above. Once satisfied, you can delete the Google Workspace or other cloud account.

Honestly I would wait a month or three to see if this is a stable solution for you. Try the backup and restore procedures (below) and make sure you're comfortable with everything. And only then delete the cloud account.

## Clean Up Local Account (After deleting the cloud account)
Once you are happy that your account(s) are safely in dovecot and you have deleted the Google Workspace or other cloud account:
`nano $HOME/email-archive/mbsync/.mbsyncrc`
### Delete the entire "Account 1" or whatever section from `.mbsyncrc`.
This prevents mbsync from trying to connect to a deleted account in the future.


# For Long-Run Sync Accounts: You can set up a simple cronjob on your host machine to trigger the sync daily.
Adapt the folder path below to wherever you set up your `email-archive` folder.
## Add to crontab (This runs every day at 2:00 AM)
`0 2 * * * cd /path/to/email-archive && docker compose run --rm fetcher mbsync -c /home/app/config/.mbsyncrc long-term-username2-gmail`
## If you want to sync all the blocks in `.mbsyncrc` then use `-a` above instead of `-c /home/app/config/.mbsyncrc long-term-username2-gmail`


# Process the "next" account
Let's say an employee parts ways and you want to stop paying for their Google Workspace account, just edit the above files, search for all instances of say `username2` and update to the next account you want to process.


# Backup
## Backup each account into a separate .tar.gz file
(along with all associated infrastructure files)
```
cd $HOME/email-archive
docker compose stop
tar -czvf      username1-gmail-backup-$(date +%F).tar.gz docker-compose.yml mbsync/ dovecot/ roundcube/ roundcube_build/ mail/username1/
tar -czvf username2-yourdomain-backup-$(date +%F).tar.gz docker-compose.yml mbsync/ dovecot/ roundcube/ roundcube_build/ mail/username2/
docker compose up -d
```
## Safely move the `.tar.gz` archives somewhere for long-term storage

# Restore
Copy the `.tar.gz` file to your new linux docker host.
```
mkdir -p email-archive
tar -xzvf username2-yourdomain-backup*.tar.gz -C email-archive/
cd email-archive
docker compose up -d
```
Open roundcube http://<whatever-new-ip>:8000
User: username2
Pass: redactedlocalpassword


# Update passwords for local users
## If you want to change the password for a local email archive, edit this file:
`nano $HOME/email-archive/dovecot/users`


# Possible Future Enhancements:
- consider `imapsync` instead of `mbsync`
- upgrade `dovecot` to v2.4.x
- replace cron job with systemd timer
- create script to ask user for username, app password, sync options, and create the config files automatically
- stripped down variant without roundcube, if you want to just use a desktop email client
