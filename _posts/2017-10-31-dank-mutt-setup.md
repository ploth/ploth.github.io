---
title: My dank mutt setup
---

## My dank mutt setup

In meinem Umkreis scheint niemand so richtig E-Mails zu mögen. Ich möchte
einfach mal behaupten dies liegt an ausschließlich auf dem Mobilgerät
eingerichteter Postfächer oder an der Verwendung webbasierter Lösungen. Beides
sehr belastende Umstände. 

Kommen wir an dieser Stelle zu Mutt. *("All mail clients suck. This one just
sucks less.")*  

Als erstes möchte ich einen kleinen Überblick über die benötigten Programme
verschaffen. [Mutt](http://www.mutt.org/) ist mein E-Mail Client, welcher
theoretisch in der Lage wäre sich mit einem IMAP Server zu verbinden um
entsprechende Nachrichten anzuzeigen, doch wären diese dann nicht offline
verfügbar. Hier hilft ein tolles Tool namens
[OfflineIMAP](https://github.com/OfflineIMAP/offlineimap). Unerwarteterweise
lassen sich damit noch keine E-Mails versenden. Hierfür benötigen wir einen SMTP
Client. Es bietet sich [msmtp](http://msmtp.sourceforge.net/) an. Natürlich
möchte man innerhalb von Mutt auch komfortablen Zugriff auf sein Adressbuch
haben. Dies ermöglicht das CLI [khard](https://github.com/scheibler/khard).
Optional zu installieren ist das CLI [khal](https://github.com/pimutils/khal),
welches uns auch gleich einen Kalender für das Terminal bringt. Khard kann sich
von Haus aus nicht mit einem CalDAV Server synchronisieren; khal auch nicht mit
einem CardDAV. Beide Programme bieten nur Schnittstellen um mit entsprechenden
Datenbanken zu interagieren. Die Synchronisation mit einem Cal/CardDAV Server
übernimmt [vdirsyncer](https://github.com/pimutils/vdirsyncer). Mit Mutt lassen
sich durchaus die E-Mails durchsuchen, doch besser, effizienter und schneller
geht dies mit [notmuch](https://notmuchmail.org/).

### Installation

Mutt, msmtp und notmuch sollten sich über eure Paketmanager installieren lassen.
OfflineIMAP, khard, khal und vdirsyncer sollten in einem [Python Virtual
Environment](https://docs.python.org/3/library/venv.html) installiert werden.  

Das *Virtual Environment* `office` kann natürlich auch an geignet anderer Stelle
erstellt werden. Bedenkt die Anpassung der Symlinks. In dem folgenden Beispiel
wird davon ausgegangen, dass sich `~/bin/` in eurem `PATH` befindet.

{% highlight Shell linenos %}
mkdir ~/.venvs
cd ~/.venvs
python3 -m venv office
source ./office/bin/activate
pip install offlineimap vdirsyncer khard khal
deactivate

ln -s ~/.venvs/office/bin/offlineimap ~/bin/offlineimap
ln -s ~/.venvs/office/bin/vdirsyncer ~/bin/vdirsyncer
ln -s ~/.venvs/office/bin/khard ~/bin/khard
ln -s ~/.venvs/office/bin/khal ~/bin/khal
ln -s ~/.venvs/office/bin/ikhal ~/bin/ikhal
{% endhighlight %}

### OfflineIMAP

Zuerst machen wir uns an die Konfiguration von OfflineIMAP.

{% highlight Shell linenos %}
mkdir ~/.mutt
mkdir ~/.mutt/accounts
mkdir ~/.mutt/cache
mkdir ~/offlineimap
vim ~/.offlineimaprc #mit dem folgenden Inhalt
{% endhighlight %}

{% highlight Shell linenos %}
[general]
# List of accounts to be synced, separated by a comma.
accounts = <YOURACCOUNTNAME>
maxsyncaccounts = 1
sockettimeout = 10

[Account <YOURACCOUNTNAME>]
# Identifier for the local repository; e.g. the maildir to be synced via IMAP.
localrepository = Local<YOURACCOUNTNAME>
# Identifier for the remote repository; i.e. the actual IMAP, usually non-local.
remoterepository = Remote<YOURACCOUNTNAME>
# Status cache. Default is plain, which eventually becomes huge and slow.
status_backend = sqlite
autorefresh = 1
quick = 10
postsynchook = /usr/bin/notmuch new

[Repository Local<YOURACCOUNTNAME>]
# Currently, offlineimap only supports maildir and IMAP for local repositories.
type = Maildir
# Where should the mail be placed?
localfolders = ~/offlineimap/Mailbox

[Repository Remote<YOURACCOUNTNAME>]
# Remote repos can be IMAP or Gmail, the latter being a preconfigured IMAP.
type = IMAP
ssl = yes
remotehost = <IMAPSERVER>
remoteuser = <MAILADDRESS> #In the most cases
remotepass = <PASSWORD>
maxconnections = 5
holdconnectionopen = yes
keepalive = 60
idlefolder = ['INBOX']
sslcacertfile = /etc/ssl/certs/ca-certificates.crt

# Automatic mailbox generation for mutt
[mbnames]
enabled  = yes
filename = /home/<USERNAME>/.mutt/mailboxes
header   = "mailboxes "
peritem  = "+%(accountname)s/%(foldername)s"
sep      = " "
footer   = "\n"
{% endhighlight %}

Now give it a try! `offlineimap -o` sollte eingestellte Postfächer abrufen. Dies
kann allerdings einige Zeit dauern.

### Mutt

Weiter geht es mit der Einrichtung von Mutt. Legt eine `~/.muttrc` mit dem
folgenden Inhalt an:

{% highlight Shell linenos %}
{% raw %}
set spoolfile       = "/home/<USER>/offlineimap/<YOURACCOUNTNAME>/INBOX"
set folder          = "/home/<USER>/offlineimap"

source /home/<USER>/.mutt/mailboxes
source /home/<USER>/.mutt/accounts/<YOURACCOUNTNAME>

#Speed up folder switch
set sleep_time = 0

# View new/flag
macro index .i  "<limit>(~N|~F)<Enter>"  "view new/flag"
macro index .a  "<limit>~A<Enter>"       "view all"

# Load corresponding settings
folder-hook <YOURACCOUNTNAME>/*   source ~/.mutt/accounts/<YOURACCOUNTNAME>

# Switch between mailboxes
macro index,pager <f2> "<change-folder>+<YOURACCOUNTNAME>/INBOX<enter>"

# Mark all emails as read
macro index A "<tag-pattern>~N|~O<enter><tag-prefix><clear-flag>N<untag-pattern>.<enter>" "mark all new as read"

# Mutt can cache headers of messages so they need to be downloaded just once.
# This greatly improves speed when opening folders again later.
set header_cache     = ~/.mutt/cache/headers
set message_cachedir = ~/.mutt/cache/bodies

# Fetch new mails via offlineimap
macro index,pager z "! /home/<USER>/bin/offlineimap -o<enter>" "Refresh offlineimap"

# Khard commands
#complete email addresses by pressing the Tab-key in mutt's new mail dialog
set query_command= "khard email --parsable '%s'"
bind editor <Tab> complete-query
bind editor ^T    complete
#add email addresses to khard's address book
macro index,pager A "<pipe-message>khard add-email<return>" "add the sender email address to khard"

# Mutt bindings
bind index p recall-message

bind index ^P print-message

# Mutt Behaviour
set sig_on_top        = yes
set mime_forward      = ask-yes
set move              = no
set sort              = 'threads'
set sort_aux          = 'reverse-last-date-received'
set sort_re           = yes
set pager_stop        = yes
set pager_index_lines = 20
set quit              = ask-yes
set fast_reply        = yes
set include           = yes
set reverse_name      = yes
set pager_context     = 3     # number of context lines to show
set pager_stop        = yes   # don't go to next message automatically
set menu_scroll       = yes   # scroll in menus
set tilde             = yes   # show tildes like in vim
set markers           = no    # no ugly plus signs
set edit_headers      = yes

auto_view application/x-pkcs7-mime

# VIM-Like Movement in Index
bind index gg first-entry
bind index G  last-entry
bind index,pager F  flag-message
bind index,pager R  group-reply
bind index N search-opposite
bind index M toggle-new

# Ignore all headers
ignore *
# Then un-ignore the ones I want to see
unignore From:
unignore To:
unignore Reply-To:
unignore Mail-Followup-To:
unignore Subject:
unignore Date:
unignore Organization:
unignore Newsgroups:
unignore CC:
unignore BCC:
unignore Message-ID:
unignore X-Mailer:
unignore User-Agent:
unignore X-Junked-Because:
unignore X-SpamProbe:
unignore X-Virus-hagbard:
# Now order the visable header lines
hdr_order From: Subject: To: CC: BCC: Reply-To: Mail-Followup-To: Date: Organization: User-Agent: X-Mailer:

set index_format      = "%Z %{%a %Y-%m-%d %H:%M} » %-20.20F › %s"
set xterm_set_titles  = yes

# Sidebar
set sidebar_visible=yes
set sidebar_divider_char='|'
set mail_check_stats=yes
set sidebar_format='%B%?F? [%F]?%* %?N?%N/?%S'
set sidebar_short_path=yes
set sidebar_delim_chars='/'
set sidebar_folder_indent=yes
set sidebar_indent_string="  "
set sidebar_width=25
set sidebar_sort_method=path

# Sidebar navigation
bind index,pager <down>   sidebar-next
bind index,pager <up>     sidebar-prev
bind index,pager <right>  sidebar-open

# View attachments properly.
set mailcap_path  = ~/.mutt/mailcap
bind attach <return> view-mailcap

# 'L' performs a notmuch query, showing only the results
macro index L "<enter-command>unset wait_key<enter><shell-escape>read -p 'notmuch query: ' x; \
  echo \$x >~/.cache/mutt_terms<enter><limit>~i \"\`notmuch search --output=messages \$(cat ~/.cache/mutt_terms) | \
  head -n 600 | perl -le '@a=<>;chomp@a;s/\^id:// for@a;$,=\"|\";print@a'\`\"<enter>" "show only messages matching a notmuch pattern"
# 'a' shows all messages again (supersedes default <alias> binding)
macro index a "<limit>all\n" "show all messages (undo limit)"

color index      brightgreen    default  ~F # Markierte Nachrichten
color index      red            default  ~N # Neue Nachrichten
color index      red            default  ~O # Ungelesene Nachrichten

### Header
color header default default "^from:"
color header default default "^to:"
color header default default "^cc:"
color header default default "^date:"
color header default default "^newsgroups:"
color header default default "^reply-to:"
color header default default "^subject:"
color header default default "^x-spam-rule:"
color header default default "^x-mailer:"
color header default default "^message-id:"
color header default default "^Organization:"
color header default default "^Organisation:"
color header default default "^User-Agent:"
color header default default "^message-id: .*pine"
color header default default "^X-Fnord:"
color header default default "^X-WebTV-Stationery:"
color header default default "^x-spam-rule:"
color header default default "^x-mailer:"
color header default default "^message-id:"
color header default default "^Organization:"
color header default default "^Organisation:"
color header default default "^User-Agent:"
color header default default "^message-id: .*pine"
color header default default "^X-Fnord:"
color header default default "^X-WebTV-Stationery:"
color header default default "^X-Message-Flag:"
color header default default "^X-Spam-Status:"
color header default default "^X-SpamProbe:"
color header default default "^X-SpamProbe: SPAM"
{% endraw %}
{% endhighlight %}

Dies konfiguriert euch ein recht schlicht aussehendes Mutt. Es treten nur
markierte, neue und ungelesene Nachrichten farbig auf. In der linken Leiste
*(Sidebar)* wird mit den Pfeiltasten navigiert. Die rechte Pfeiltaste läd den
ausgewählten Ordner.  In den Mails kann mit den standard vim Kommandos navigiert
werden. Nun sind noch mailaccountspezifische Einstellungen zu tätigen. Erstellt
die Datei `~/.mutt/accounts/<YOURACCOUNTNAME>`

{% highlight Shell linenos %}
set sendmail        = "/usr/bin/msmtp -a <YOURACCOUNTNAME>"
set realname        = "<NAME>"
set from            = "<MAILADDRESS>"
set mbox_type       = Maildir
set folder          = "/home/<USER>/offlineimap"
set spoolfile       = "+<YOURACCOUNTNAME>/INBOX"
set record          = "+<YOURACCOUNTNAME>/Sent"
set postponed       = "+<YOURACCOUNTNAME>/Drafts"
set signature       = "/home/<USER>/.mutt/accounts/Signature"
{% endhighlight %}

Nicht vergessen eine Signatur anzulegen (siehe Zeile 9). Außerdem ist es möglich über
die Datei `~/.mutt/mailcap` jeglicher Art von Anhang einer E-Mail entsprechende
Programme zuzuweisen, mit denen diese geöffnet werden sollen. Per Druck auf die
Taste `v` *(view)* werden für die ausgewählte E-Mail alle Anhänge angezeigt.
Beispielhaft kann die `mailcap` wie folgt aussehen:

{% highlight Shell linenos %}
# Plain
text/plain; vim %s

# Documents
application/pdf; zathura %s #zathura ist ein pdf-viewer
application/x-pdf; zathura %s
application/octet-stream; zathura %s

# HTML
text/html; chromium -dump %s >/dev/null 2>&1;copiousoutput
#text/html; firefox %s;copiousoutput

# Pictures
image/png; sxiv %s
image/jpg; sxiv %s
image/jpeg; sxiv %s
{% endhighlight %}

### msmtp

Legt eine `~/.msmtp` an. `from` und `user` sind in den meisten Fällen auf eure
E-Mailadresse zu setzen. Optional: `from` darf auch ein valider E-Mail Alias sein.

{% highlight Shell linenos %}
# Set default values for all following accounts.
defaults
auth           on
tls            on
tls_trust_file /etc/ssl/certs/ca-certificates.crt
logfile        ~/.msmtp.log

# Mailbox
account        <YOURACCOUNTNAME>
host           <SMTPSERVER>
port           <PORT>
from           <MAILADDRESS>
user           <USER>
password       <PASSWORD>

# Set a default account
account default : <YOURACCOUNTNAME>
{% endhighlight %}

An diesem Punkt sollte es euch möglich sein E-Mails zu empfangen und an bekannte
Adressen zu senden.

### vdirsyncer

Im Folgenden werden wir die Synchronisation mit unserem CardDav und CalDAV
Server einrichten.

{% highlight Shell linenos %}
mkdir ~/.contacts
mkdir ~/.calendars
{% endhighlight %}

Die vdirsyncer Konfiguration ähnelt der von OfflineIMAP. Editiert
die `~/.config/vdirsyncer/config` wie folgt:

{% highlight Shell linenos %}
[general]
# A folder where vdirsyncer can store some metadata about each pair.
status_path = "~/.vdirsyncer/status/"

# CARDDAV
[pair <YOURACCOUNTNAME>]
# A `[pair <name>]` block defines two storages `a` and `b` that should be
# synchronized. The definition of these storages follows in `[storage <name>]`
# blocks. This is similar to accounts in OfflineIMAP.
a = "<YOURACCOUNTNAME>Local"
b = "<YOURACCOUNTNAME>Remote"

# Synchronize all collections available on "side B" (in this case the server).
# You need to run `vdirsyncer discover` if new calendars/addressbooks are added
# on the server.

# Omitting this parameter implies that the given path and URL in the
# corresponding `[storage <name>]` blocks are already directly pointing to a
# collection each.
collections = ["from b"]

# Synchronize the "display name" property into a local file (~/.contacts/displayname).
metadata = ["<YOURACCOUNTNAME>"]

# To resolve a conflict the following values are possible:
#   `null` - abort when collisions occur (default)
#   `"a wins"` - assume a's items to be more up-to-date
#   `"b wins"` - assume b's items to be more up-to-date
conflict_resolution = "a wins"

[storage <YOURACCOUNTNAME>Local]
# A storage references actual data on a remote server or on the local disk.
# Similar to repositories in OfflineIMAP.
type = "filesystem"
path = "~/.contacts/<YOURACCOUNTNAME>/"
fileext = ".vcf"

[storage <YOURACCOUNTNAME>Remote]
type = "carddav"
url = "<CARDDAVSERVER>"
username = "<USERNAME>"
# The password can also be fetched from the system password storage, netrc or a
# custom command. See http://vdirsyncer.readthedocs.org/en/stable/keyring.html
password = "<PASSWORD>"

# CALDAV
[pair <YOURACCOUNTNAME>Calendar]
a = "<YOURACCOUNTNAME>CalendarLocal"
b = "<YOURACCOUNTNAME>CalendarRemote"
#collections = ["private", "work"]
collections = ["from b"]

# Calendars also have a color property
metadata = ["<YOURACCOUNTNAME>", "blue"]

# To resolve a conflict the following values are possible:
#   `null` - abort when collisions occur (default)
#   `"a wins"` - assume a's items to be more up-to-date
#   `"b wins"` - assume b's items to be more up-to-date
conflict_resolution = "a wins"

[storage <YOURACCOUNTNAME>CalendarLocal]
type = "filesystem"
path = "~/.calendars/<YOURACCOUNTNAME>/"
fileext = ".ics"

[storage <YOURACCOUNTNAME>CalendarRemote]
type = "caldav"
url = "<CARDDAVSERVER>"
username = "<USERNAME>"
password = "<PASSWORD>"

{% endhighlight %}

### Khard

Khard ist nicht wild zu konfigurieren. Erstellt eine
`~/.config/khard/khard.conf`

{% highlight Shell linenos %}
[addressbooks]
[[<YOURACCOUNTNAME>]]
path = ~/.contacts/<YOURACCOUNTNAME>/<PROBABLYAFOLDER>/

[general]
editor = vim
merge_editor = vimdiff
default_action = list
show_nicknames = no
{% endhighlight %}

### Khal

Auch khal ist einfach. Erstellt eine `~/.config/khal/config`

{% highlight Shell linenos %}
[calendars]
[[<YOURACCOUNTNAME>Calendar]]
path = ~/.calendars/<YOURACCOUNTNAME>/<PROBABLYAFOLDER>/
#color = blue

[sqlite]
path = ~/.khal/khal.db

[locale]
local_timezone = Europe/Berlin
default_timezone = Europe/Berlin

# I really like ISO 8601
timeformat = %H:%M
#dateformat = %d.%m.
dateformat = %Y-%m-%d
#longdateformat = %d.%m.%Y
longdateformat = %Y-%m-%d
#datetimeformat =  %d.%m. %H:%M
datetimeformat =  %Y-%m-%d
#longdatetimeformat = %d.%m.%Y %H:%M
longdatetimeformat = %Y-%m-%dT%H:%M

firstweekday = 0

[default]
default_command = calendar
default_calendar = <YOURACCOUNTNAME>Calendar
# in agenda/calendar display, shows all days even if there is no event
show_all_days = True

[view]
theme = dark
{% endhighlight %}

### notmuch

Zu guter Letzt fehlt noch notmuch. Editiert die `~/.notmuch-config` wie folgt:

{% highlight Shell linenos %}
[database]
path=/home/<USER>/offlineimap

[user]
name=<YOURNAME>
primary_email=<YOURPRIMARYEMAILADDRESS>
other_email=<ANOTHERMAILADDRESS>;<ONEMOREADDRESS>

[new]
tags=unread;inbox;
ignore=

[search]
exclude_tags=deleted;spam;

[maildir]
synchronize_flags=true

[crypto]
gpg_path=gpg
{% endhighlight %}

Nun führt noch ein `notmuch setup` aus.  
Herzlichen Glückwunsch! Welcome the to light side of the Force.  
Ihr habt da nun ein ziemlich mächtiges Set an Programmen die super miteinander
harmonieren. Das Ausführen von `khard` bringt euer komplettes Adressbuch hervor,
welches ihr über entsprechende Kommandos manipulieren könnt. `khal` zeigt euch
eine kurze Zusammenfassung der nächsten Termine. `ikhal` ist ein interaktiver
Kalender, in dem ihr direkt eure Termine verwalten könnt. Es sei vielleicht noch
erwähnt, sofern ihr es nicht in der `~/.muttrc` gelesen habt, dass ihr innerhalb
von Mutt mit `L` euere Emails durchsucht. Einigen wird auch aufgefallen sein,
dass die E-Mails nicht automatisch abgerufen werden. Hier kann je nach belieben
ein OfflineIMAP Service oder ein cronjob für `offlineimap -o` eingerichtet
werden.

Anmerkungen, Fehler, Beschwerden gerne an blog@ploth.xyz
