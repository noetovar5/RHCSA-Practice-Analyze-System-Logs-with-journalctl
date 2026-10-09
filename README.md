# RHCSA-Practice-Analyze-System-Logs-with-journalctl
RHCSA Practice — Analyze System Logs with journalctl
Today’s focused objective: Locate, filter, interpret, and export system and service events from the systemd journal.
Estimated time: 45 minutes
Lab system: RHEL 9 or RHEL 10
Privileges required: Root or sudo
This lesson builds directly on our previous systemctl service-management lab. Knowing how to start a service is only half the job; when it fails, starts slowly, or behaves unexpectedly, I need to know how to find the evidence.
Important safety note
Most commands in this lesson are read-only. We will use logger to write harmless practice messages into the journal.
We will not:
- Stop or restart any service
- Modify networking, storage, boot settings, authentication, SELinux, or the firewall
- Change journal-retention settings
- Delete existing system logs
Do not use journalctl --vacuum-* during this lab. Vacuum operations can permanently remove system journal history.
1. The concept in plain language — 7 minutes
RHEL uses systemd-journald to collect events from:
- The Linux kernel
- System services
- Programs writing to standard output or standard error
- The logger command
- Boot and shutdown activity
The primary command for reading these events is:
journalctl

Think of the journal as a searchable timeline of system activity.
A typical entry resembles:
Oct 01 19:05:42 rhel9lab sshd[1842]: Accepted publickey for administrator

It contains:
- Date and time
- Hostname
- Program or service
- Process ID
- Message
Common filters
Command	Purpose
journalctl -n 20	Show the newest 20 entries
journalctl -b	Show the current boot
journalctl -b -1	Show the previous boot
journalctl -u UNIT	Show entries for one systemd unit
journalctl -p warning	Show warning and more severe events
journalctl -t TAG	Filter by program identifier
journalctl --since TIME	Set the beginning of a time range
journalctl --until TIME	Set the end of a time range
journalctl -f	Follow new events in real time
journalctl -k	Show kernel messages
journalctl --no-pager	Print directly without opening a pager


Journal priorities
Linux log priorities run from 0 through 7:
Number	Name	General meaning
0	emerg	System unusable
1	alert	Immediate action required
2	crit	Critical problem
3	err	Error
4	warning	Warning
5	notice	Significant normal event
6	info	Informational event
7	debug	Diagnostic detail


The lower the number, the more severe the event.
For example:
journalctl -p warning

selects priorities 0 through 4: warning and everything more severe.
2. Prepare the safe lab — 4 minutes
Enter a root shell so you can read the complete system journal:
sudo -i

Confirm your identity and the journal service:
whoami
systemctl is-active systemd-journald.service

Expected results:
root
active

Create a temporary workspace:
mkdir -p /tmp/rhcsa-journal-lab/reports
cd /tmp/rhcsa-journal-lab
pwd

Expected location:
/tmp/rhcsa-journal-lab

Create a unique identifier for this lab:
lab_id="RHCSA-JOURNAL-$(date +%s)-$$"
printf '%s\n' "$lab_id" > lab-id.txt
cat lab-id.txt

Why this matters:
- date +%s supplies the current Unix timestamp.
- $$ supplies the current shell’s process ID.
- Combining them produces an identifier unlikely to match older journal messages.
- Saving it in lab-id.txt lets us restore the variable later.
If you open a new shell, restore it with:
lab_id=$(cat /tmp/rhcsa-journal-lab/lab-id.txt)

3. Examine recent system activity — 5 minutes
Display the newest 20 journal entries:
journalctl -n 20 --no-pager

- -n 20 selects the newest 20 entries.
- --no-pager prevents the output from opening in less.
Display entries only from the current boot:
journalctl -b -n 20 --no-pager

List the boots retained in the journal:
journalctl --list-boots

A representative result looks like:
-1 BOOT_ID Tue 2026-09-29 18:45:10 CDT—Tue 2026-09-29 21:12:44 CDT
 0 BOOT_ID Thu 2026-10-01 18:32:01 CDT—Thu 2026-10-01 19:06:20 CDT

Here:
- 0 means the current boot.
- -1 means the previous boot.
- -2 means two boots ago.
If only boot 0 appears, older journals may not have been retained. That is not an error in today’s lab.
Display the first ten events from the current boot:
journalctl -b 0 -n 10 --no-pager

Display the previous boot, if available:
journalctl -b -1 -n 10 --no-pager

If the previous boot is unavailable, expect a message indicating that the requested boot offset has no journal data.
4. Generate controlled practice events — 5 minutes
The following commands add harmless messages to the journal. They do not change application state or system configuration.
logger -t rhcsa-lab -p user.info \
  "$lab_id InventoryApp startup completed"

logger -t rhcsa-lab -p user.warning \
  "$lab_id InventoryApp database latency detected"

logger -t rhcsa-lab -p user.err \
  "$lab_id InventoryApp database connection failed"

Command breakdown:
- logger sends a message to the logging system.
- -t rhcsa-lab assigns the identifier rhcsa-lab.
- -p user.info sets the facility and priority.
- The final quoted text is the message.
Ask the journal to synchronize pending entries:
journalctl --sync

Now display only our messages:
journalctl -t rhcsa-lab \
  --since "10 minutes ago" \
  --no-pager | grep "$lab_id"

Expected result: three lines containing:
InventoryApp startup completed
InventoryApp database latency detected
InventoryApp database connection failed

Count them:
journalctl -t rhcsa-lab \
  --since "10 minutes ago" \
  --no-pager |
grep -c "$lab_id"

Expected:
3

This combines several earlier skills:
- journalctl retrieves journal data.
- A time filter limits the search.
- A tag filter selects the application.
- A pipe sends the result to grep.
- grep -c counts matching entries.
5. Filter by time and severity — 6 minutes
Display our messages from the last five minutes:
journalctl -t rhcsa-lab \
  --since "5 minutes ago" \
  --until "now" \
  --no-pager | grep "$lab_id"

You can also use an exact timestamp:
journalctl \
  --since "2026-10-01 18:00:00" \
  --until "2026-10-01 20:00:00" \
  --no-pager

Adjust the date when practicing on another day.
Select warning and more severe messages
journalctl -t rhcsa-lab \
  -p warning \
  --since "10 minutes ago" \
  --no-pager | grep "$lab_id"

Expected: two messages:
InventoryApp database latency detected
InventoryApp database connection failed

The informational startup message should not appear.
Select errors and more severe messages
journalctl -t rhcsa-lab \
  -p err \
  --since "10 minutes ago" \
  --no-pager | grep "$lab_id"

Expected: only the database connection failure.
Specify an exact priority range
journalctl -t rhcsa-lab \
  -p warning..err \
  --since "10 minutes ago" \
  --no-pager | grep "$lab_id"

Expected: the warning and error messages.
The range notation is useful when I want particular severity levels instead of everything above a threshold.
6. Investigate service-specific events — 5 minutes
Display recent entries for the journal service itself:
journalctl \
  -u systemd-journald.service \
  -b \
  -n 20 \
  --no-pager

Options:
- -u selects one systemd unit.
- -b restricts the result to the current boot.
- -n 20 limits the result to 20 events.
Look for a currently running service:
systemctl list-units \
  --type=service \
  --state=running \
  --no-pager

Pick a safe service from the output and inspect its journal. For example:
journalctl -u chronyd.service -b -n 20 --no-pager

If chronyd is not installed or active, try:
journalctl -u systemd-logind.service -b -n 20 --no-pager

This is read-only. We are not stopping or restarting either service.
Combine service and severity filters
journalctl \
  -u systemd-journald.service \
  -b \
  -p warning \
  --no-pager

No output is a valid result. It means no warning-or-higher entries matched the filters for this boot.
Review failed services
systemctl --failed --no-pager

If a failed unit appears, inspect its evidence without changing it:
systemctl status UNIT_NAME --no-pager
journalctl -u UNIT_NAME -b --no-pager

Replace UNIT_NAME with the exact unit. This is an important troubleshooting sequence:
1. Find the failed unit.
2. Check its current state.
3. Read its journal.
4. Identify the first meaningful error.
5. Correct the cause.
6. Verify the final state.
Do not restart an unfamiliar production service simply because it appears failed.
7. Read structured journal fields — 4 minutes
Journal entries contain structured metadata beyond the one-line display.
Show one of our entries in verbose format:
journalctl -t rhcsa-lab \
  --since "10 minutes ago" \
  -o verbose \
  -n 1 \
  --no-pager

Look for fields such as:
_MESSAGE=
_PRIORITY=
SYSLOG_IDENTIFIER=
_PID=
_UID=
_GID=
_HOSTNAME=
_BOOT_ID=
_TRANSPORT=

These fields answer useful questions:
- Which process wrote the event?
- Which user ran it?
- What priority was assigned?
- During which boot did it occur?
- How did the message enter the journal?
Display our messages using an ISO-style timestamp:
journalctl -t rhcsa-lab \
  --since "10 minutes ago" \
  -o short-iso \
  --no-pager | grep "$lab_id"

This format is helpful when exact timestamps must be compared across systems or included in an incident report.
8. Export and verify an incident report — 3 minutes
Export our warning-and-higher events:
journalctl -t rhcsa-lab \
  -p warning \
  --since "10 minutes ago" \
  --no-pager |
grep "$lab_id" \
  > reports/inventory-incidents.txt

Display the report:
cat reports/inventory-incidents.txt

Verify the line count:
wc -l reports/inventory-incidents.txt

Expected:
2 reports/inventory-incidents.txt

Verify the required events:
grep -q 'database latency detected' \
  reports/inventory-incidents.txt \
  && echo "PASS: warning included"

grep -q 'database connection failed' \
  reports/inventory-incidents.txt \
  && echo "PASS: error included"

Verify that the informational message was excluded:
if grep -q 'startup completed' reports/inventory-incidents.txt; then
    echo "FAIL: informational event was incorrectly included"
else
    echo "PASS: informational event excluded"
fi

9. Check journal storage safely — 2 minutes
Display current journal disk usage:
journalctl --disk-usage

Check for persistent journal storage:
if test -d /var/log/journal; then
    echo "Persistent journal directory exists"
else
    echo "No /var/log/journal directory detected"
fi

In general:
- /run/log/journal is volatile and normally disappears after reboot.
- /var/log/journal supports persistent storage across reboots.
Today we are only inspecting the storage configuration. We are not changing it or restarting systemd-journald.
Traditional text logs might also exist:
ls -l /var/log/messages /var/log/secure 2>/dev/null

On RHEL installations with rsyslog, common locations include:
- /var/log/messages for general system messages
- /var/log/secure for authentication-related activity
- /var/log/cron for scheduled-job activity
A minimal installation may rely primarily on the system journal and may not have every traditional log file.
Independent challenge — 7 minutes
Complete this without copying the earlier message-generation and report commands.
Starting state
Remain in:
/tmp/rhcsa-journal-lab

Restore your identifier if necessary:
lab_id=$(cat lab-id.txt)

Create a second unique identifier:
challenge_id="RHCSA-CHALLENGE-$(date +%s)-$$"
printf '%s\n' "$challenge_id" > challenge-id.txt

Assignment
Using the tag:
rhcsa-challenge

write four journal messages containing your $challenge_id:
Priority	Required message
info	Report service started
notice	Report generation completed
warning	Report storage nearing capacity
err	Report upload failed


Then create:
reports/challenge-incidents.txt

The report must:
1. Include events from only the current boot.
2. Search only the last 15 minutes.
3. Select only tag rhcsa-challenge.
4. Include warning and more severe events.
5. Include only entries containing your unique $challenge_id.
6. Contain exactly two lines.
7. Include the storage warning.
8. Include the upload error.
9. Exclude the informational and notice events.
10. Use journalctl, a pipe, grep, and redirection.
Do not manually type the expected events into the report.
Independent verification
Restore the challenge identifier if necessary:
challenge_id=$(cat challenge-id.txt)

Check that the report exists and is not empty:
test -s reports/challenge-incidents.txt \
  && echo "PASS: challenge report exists" \
  || echo "FAIL: challenge report missing or empty"

Check the line count:
test "$(wc -l < reports/challenge-incidents.txt)" -eq 2 \
  && echo "PASS: report contains two incidents" \
  || echo "FAIL: incorrect incident count"

Confirm both required events:
grep -q "$challenge_id.*Report storage nearing capacity" \
  reports/challenge-incidents.txt \
  && echo "PASS: warning included" \
  || echo "FAIL: warning missing"

grep -q "$challenge_id.*Report upload failed" \
  reports/challenge-incidents.txt \
  && echo "PASS: error included" \
  || echo "FAIL: error missing"

Confirm the lower-severity events were excluded:
if grep -Eq 'Report service started|Report generation completed' \
  reports/challenge-incidents.txt; then
    echo "FAIL: lower-severity events were included"
else
    echo "PASS: lower-severity events excluded"
fi

Knowledge check
Answer these without reviewing the lesson:
1. What is the purpose of journalctl?
2. What does journalctl -b display?
3. What does boot offset -1 represent?
4. Which option filters entries for one systemd service?
5. What does journalctl -p warning include?
6. Is priority 3 more or less severe than priority 6?
7. Which option filters messages by their identifier or tag?
8. How would you search only the last 30 minutes?
9. What does --no-pager change?
10. What does journalctl -k display?
11. How do you show the newest 25 entries?
12. Why is no output sometimes a valid result?
13. What does journalctl --list-boots show?
14. Why is journalctl -f useful during troubleshooting?
15. Why should I avoid indiscriminate --vacuum-* operations?
16. What is the difference between /run/log/journal and /var/log/journal?
17. How do I combine a unit filter with a current-boot filter?
18. After finding a failed service, which two commands help me investigate it?
Mastery standard
I would consider this objective mastered when I can independently:
- Search the current and previous boot
- Filter by service, tag, time, and priority
- Interpret journal priorities
- Inspect structured journal fields
- Export selected events into a report
- Verify that the report contains only the required evidence
- Investigate a failed unit without immediately restarting it
Safe cleanup
The test messages have become normal journal entries. Individual journal entries are not meant to be selectively deleted; they will age out according to the VM’s normal retention policy.
Do not vacuum the system journal just to remove these practice messages because that could delete unrelated diagnostic history.
Remove only the temporary reports and identifier files.
Filesystem cleanup warning
Confirm that the path is exactly /tmp/rhcsa-journal-lab before running the recursive removal:
cd /tmp
rm -rf /tmp/rhcsa-journal-lab

Verify:
test ! -e /tmp/rhcsa-journal-lab \
  && echo "PASS: temporary journal lab files removed" \
  || echo "CHECK: lab directory still exists"

Expected:
PASS: temporary journal lab files removed

Leave the root shell:
exit

The next logical lesson is configuring persistent system journals safely, which will extend today’s read-and-filter skills into preserving logs across reboots.



Open chat


September 2026
RHCSA Practice — Manage Services with systemctl Today’s objective: Create, start, stop, restart, enable, inspect, and troubleshoot a harmless custom systemd service. Estimated time: 45 minutesLab system: RHEL 9 or RHEL 10Privileges required: Root or sudo This lesson moves from account administration into operating a running Linux system. Starting, stopping, checking, and enabling services are core RHCSA skills. Important system-availability warning `systemctSep 29RHCSA Practice — Configure Privileged Access with sudo Today’s objective: Safely grant a non-root user permission to run specific administrative commands through a validated /etc/sudoers.d/ rule. Estimated time: 45 minutesLab system: RHEL 9 or RHEL 10Privileges required: Root or an existing working sudo account This builds directly on your local users and groups lesson. Configuring privileged access is an explicit RHCSA objective. Critical authentication warnSep 27RHCSA Practice — Manage Local Users and Groups Today’s objective: Create and modify local users and groups, assign supplementary group membership, configure password aging, and verify access. Estimated time: 45 minutesLab system: RHEL 9 or RHEL 10Privileges required: Root or sudo This builds on your work with permissions and shell commands. Local account and group administration is an explicit RHCSA objective. Important safety warning The commands in this lessonSep 24RHCSA Practice — Process Multiple Files with a Bash for Loop Today’s focused objective: Use a Bash for loop to process multiple command-line arguments and generate a consolidated log report. Estimated time: 45 minutesLab system: RHEL 9 or RHEL 10Privileges required: NoneSafety: All work stays under /tmp/rhcsa-loop-lab. This practice does not modify networking, storage, boot configuration, authentication, SELinux, firewall rules, or system services. This buSep 22RHCSA Practice — Build a Log-Checking Bash Script Today’s focused objective: Create a simple Bash script that accepts a command-line argument, validates input with an if statement, processes command output, and returns a meaningful exit status. Estimated time: 45 minutesLab system: RHEL 9 or RHEL 10Privileges required: NoneSafety: This lesson works only under /tmp/rhcsa-script-lab. It does not modify services, networking, storage, boot configuration, authentiSep 20RHCSA Practice — Search Logs with grep and Regular Expressions Today’s focused objective: Use grep, pipes, and regular expressions to locate useful information in configuration files and application logs. Estimated time: 45 minutesLab system: RHEL 9 or RHEL 10Privileges required: NoneSafety: Everything stays under /tmp/rhcsa-grep-lab. This lesson does not change networking, storage, boot, authentication, SELinux, firewall rules, or system services. This isSep 17RHCSA Practice — Archive and Compress Files with tar Today’s focused objective: Create, inspect, extract, compress, and verify archives using tar, gzip, and bzip2. Estimated time: 45 minutesLab system: RHEL 9 or RHEL 10Privileges required: NoneExam relevance: Red Hat’s current EX200 objectives require candidates to archive, compress, unpack, and uncompress files using tar, gzip, and bzip2. [Official RHCSA EX200 objectives](https://www.redhat.com/en/Sep 15RHCSA Practice — Hard and Symbolic Links Today’s focused objective: Create, inspect, troubleshoot, and remove hard links and symbolic links. Estimated time: 45 minutesLab system: RHEL 9 or RHEL 10 VMExam relevance: Creating hard and soft links is listed under the current RHCSA EX200 essential-tools objectives. Official Red Hat EX200 objectives This lesson builds on yoSep 13RHCSA Practice — Standard Linux Permissions Today’s objective: Read, set, and verify standard user/group/other permissions using symbolic and numeric notation. Estimated time: 45 minutesLab system: Your RHEL 9 VM is suitable for this practice.Safety: This lab works entirely inside /tmp/rhcsa-permissions-lab. It does not change networking, storage, boot settings, authentication, SELinux, the firewall, or system availability. Red Hat’s current EX200 objectives includSep 10












    To pick up a draggable item, press the space bar.
    While dragging, use the arrow keys to move the item.
    Press space again to drop the item in its new position, or press escape to cancel.
