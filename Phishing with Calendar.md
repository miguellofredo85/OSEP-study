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



