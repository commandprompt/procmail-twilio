# procmail-twilio

Ansible playbook for procmail-twilio, a simple way to generate Twilio phone calls based on the contents of e-mail.

## Dependencies

* Python 2.7
* Python modules: Flask >= 0.8, twilio >= 3.3.6
* procmail >= 3.22
* shuf (part of coreutils on Debian/Ubuntu and RHEL/CentOS)
* HTTP server (Apache, nginx, etc.)
* BASH and SH shells
* Twilio account
* Ansible >= 2.0
* Redmine, JIRA, BugZilla, etc. (optional)

## What is procmail-twilio?
procmail-twilio is a combination of procmail recipes and a set of (BASH and Python) scripts that use Twilio Python REST API Client
to generate phone calls.

The procmail recipes distributed as part of this playbook are specifically designed to work with Redmine e-mail notifications for
tickets that were assigned a custom "Emergency (Critical)" priority. 

However, it is only an example. Just replace the ~/.procmailrc with one of your own and you can generate calls for almost any event
that registers in e-mail.

## How does it work exactly?

It all works together rather simply.

* All e-mail (generated by Redmine, JIRA, BugZilla, your personal GMail account etc.) goes through a procmail filter
* which looks for an Emergency priority ticket, or some other string that matches a regex in ~/.procmailrc
* and, when found, parses it
* and extracts key information about the ticket: ticket number, project name, who created or updated the ticket,
* generates a TwiML file based on the extracted information,
* executes a BASH script that initializes a Python virtual environment
* and then runs a Python script that makes an actual call.

The HTTP server is required to serve the TwiML file generated by procmail to Twilio via a REST call.

That's it in a nutshell.

To get a closer look at each stage of this process and see how procmail-twilio is used in real-life scenario by Command Prompt, Inc.
see this blog post: https://www.commandprompt.com/blog/generate-phone-calls-for-redmine-emergency-tickets-using-twilio/

## Redmine requirements

It depends on how your Redmine is configured.

One way to set things up is to create a dedicated Redmine user, say procmail-twilio@yourdomain.com, and make it a member of all
projects. You could use a dedicated group for this, configuring permissions accordingly, or use one of the pre-existing suitable groups.
This approach ensures that the dedicated user receives e-mail notificaitons for all ticket updates in all projects.

Additionally, an e-mail account is required for the dedicated Redmine user. This is where all Redmine e-mail notifications will
be sent to and analyzed by procmail.

## HTTP server

In a simple configuration scenario, HTTP server runs on the same server where procmail processes e-mail notifications. 
As procmail writes out a TwiML file to disk, which is eventually served over HTTP to Twilio, the HTTP server must be able to
locate and read this file.

If you need to write out the TwiML file to a remote filesystem consider using NFS, sshfs or some other solution.

## Install procmail-twilio

You need Ansible to install procmail-twilio. 

Just run these commands in your terminal:

```
$ git clone https://github.com/commandprompt/procmail-twilio.git
$ cd procmail-twilio
$ ansible-playbook run.yml --ask-vault-pass -K
```

### Configure playbook

Edit roles/procmail-twilio/vars/main.yml and set system username, filesystem paths, etc. as needed.

We highly recommend that you ecnrypt the vars file after entering your Twilio account credentials:

```
$ ansible-vault encrypt roles/procmail-twilio/vars/main.yml
```

You may also want to modify some template files but you don't really have to.

## Known Issues

### E-mail Quote Chain
If a matching string is found in an e-mail quote, procmail will dispatch a call.

Sometimes people will reply to an Emergency ticket e-mail notification via e-mail by quoting the original e-mail message.
They can do so several times sending replies to the same discussion thread, causing a growing number of e-mail messages that include the original e-mail body and has a matching regex line that triggers a phone call.

This doesn't work well with the procmail recipes you'll find in this repo. Such e-mail messages will generate essentially a false-positive match. So, in addition to the original call more calls may be made, if people keep including e-mail body(-ies) in their replies that have matching procmail recipe conditions.

This is a bit of a corner case, though.

Consider this real-life example:

```
...

Issue #86581 has been updated by Joe Doe.

I have added the mount details to the /etc/fstab.

Joe

-----Original Message-----
From: projectid@yourdomain.com [mailto:projectid@yourdomain.com] 
Sent: Monday, March 06, 2017 8:44 AM
Subject: [Project - Support #86581] Fwd: File_system_alert

Issue #86581 has been updated by Joe Doe.

The /x89 mount was not included in /etc/fstab as it was a temporary design for backup until we can get our backup solution working and tested. I have mounted the /x89 mount point.

Joe

-----Original Message-----
From: projectid@yourdomain.com [mailto:projectid@yourdomain.com] 
Sent: Monday, March 06, 2017 8:37 AM
Subject: [Project - Support #86581] Fwd: File_system_alert

Issue #86581 has been updated by Joe Doe.

Assignee set to Joe Doe

Do you have any insight as to what has happened with our backup mount.

Please advise.

Joe

-----Original Message-----
From: projectid@yourdomain.com [mailto:projectid@yourdomain.com] 
Sent: Monday, March 06, 2017 8:15 AM
Subject: [Project - Support #86581] Fwd: File_system_alert

Issue #86581 has been updated by Steve Danaval.

Priority changed from Medium to Emergency (Critical)

/x89 is empty when viewed directly and does not appear on a df -hk listing. 
```

The original e-mail (sent at 8:15 AM) was included 4 times by various parties in their e-mail replies. procmail knows nothing about this, though. All it cares about is if there is a matching line (Priority changed from Medium to Emergency (Critical)) in an e-mail or not. So, 4 calls would be generated in total.
