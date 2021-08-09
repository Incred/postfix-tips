# postfix-tips
**1) How to collect bounces in Postfix:**
  * add a new virtual domain for bounces: bounces.example.com
  * configure VERP in postfix (to send bounced emails back to our server with modified MAIL FROM according to http://www.postfix.org/VERP_README.html, so bounced email will look like:

 > From: MAILER-DAEMON@example.com (Mail Delivery System)
 > Subject: Undelivered Mail Returned to Sender
 > To: incred+123nonexistentuser123=mail.com@bounces.example.com
  * Parse TO record, where incred@bounces.example.com was FROM email and 123nonexistentuser123@mail.com was TO.

/etc/postfix/main.cf:
```
  transport_maps = hash:/etc/postfix/transport_maps
  mydestination = example.com, bounces.example.com
```

Add transport to /etc/postfix/transport_maps:

  `bounces.example.com bounces_transport:`

So all bounces send to custom transport
Add to main.cf:
```
smtpd_command_filter = pcre:/etc/postfix/append_verp.pcre
smtpd_authorized_verp_clients = $mynetworks
```

Add to /etc/postfix/append_verp.pcre:
`/^(MAIL FROM:<.*)@(reply\.)?example\.com(>.*)/ $1@bounce.example.com$3 XVERP`
 
Add bounce transport to /etc/postfix/master.cf:
```
bounce_transport   unix  -       n       n       -       -       pipe
   flags=FR user=incred  argv=python parser.py --sender ${sender} --recipient ${recipient}
```
   
And all we need to do is parse bounced email somewhere in parser.py like:
```
VERP_ADDRESS_RE = re.compile(r'\+(.+)=(.+)@.*')
match = VERP_ADDRESS_RE.search(to)
if match:
    local, domain = match.groups()
    to = '{}@{}'.format(local, domain)
```
