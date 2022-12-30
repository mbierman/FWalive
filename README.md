# Introduction: Update a Google Spreadsheet with Firewalla's status

This is a script for updating a Firewalla box's status to a Google Spreadsheet and sending an alert when it is down. It is tested on  Firewalla 1.975.


# Notes
- NextDNS CLI runs fine on Purple or Gold (I hve not tested anywhere else).
- You can run NextDNS CLI and you can have Firewalla Unbound or DoH running, but any given client can only use one of these as they are mutually exclusive. By default, any device that is not in the Unbound or DoH group will use nextDNS CLI for any network segment that is set to use nextDNS and the WAN DNS will be ignored.  


# Installation
To install:
1. Create a Google Spreadsheet [like this](https://docs.google.com/spreadsheets/d/1-31nv9rvPw_RRu9565bGzuondch9V0y8jQglDQo21uM/edit?usp=sharing)
<img width="699" alt="image" src="https://user-images.githubusercontent.com/1205471/210029758-6edacdee-c632-497a-8acc-0cdccd2aee48.png">
2. Go to IFTTT.com and create an applet as shown below.
<img width="828" alt="image" src="https://user-images.githubusercontent.com/1205471/210029514-f6eea947-b819-41a5-8d30-ff53eb90f331.png">
The filter code looks like this 

```
let mytime = Meta.currentUserTime.format('l LT');
GoogleSheets.updateCellInSpreadsheet.setPath('IFTTT');
GoogleSheets.updateCellInSpreadsheet.setFilename('Alive');
GoogleSheets.updateCellInSpreadsheet.setCell('A1');
GoogleSheets.updateCellInSpreadsheet.setValue(mytime);
```
The will receive a webhook from Firewalla on a schedule you choose and update the Google spreasheet.
 
This assumes a setup like the spreadsheet provided. 
3, Next, create another applet that will monitor the Google Spreadsheet and notify you (or whatever you choose. 
<img width="816" alt="image" src="https://user-images.githubusercontent.com/1205471/210029888-310ed6ce-4bd7-40b3-8148-4decc7c4df7b.png">

Filter code:
```
let ingredient = GoogleSheets.cellUpdatedInSpreadsheet.Value;
let searchTerm = 'UP';
let mytime = Meta.currentUserTime.format('l LT');

if (ingredient.indexOf(searchTerm) !== -1) {
  IfNotifications.sendRichNotification.skip();
  Email.sendMeEmail.skip();
  GoogleSheets.appendToGoogleSpreadsheet.skip();
} else {

  IfNotifications.sendRichNotification.setMessage( 'Firewalla Gold hasn\'t been seen in more than 5 minutes. Last seen' + ingredient);
  GoogleSheets.appendToGoogleSpreadsheet.setPath('IFTTT');
  GoogleSheets.appendToGoogleSpreadsheet.setFilename("Alive");
  GoogleSheets.appendToGoogleSpreadsheet.setFormattedRow(mytime + '|||Down');
}
```

the Rich notification works like this:
<img width="779" alt="image" src="https://user-images.githubusercontent.com/1205471/210029927-5716274b-c6be-4181-815e-479d2ec5aadb.png">
And add row to spreadsheet like this
<img width="794" alt="image" src="https://user-images.githubusercontent.com/1205471/210029962-653306b4-b38a-472d-966d-2119518d46ca.png">


3. SSH into your Firewalla ([learn how](https://help.firewalla.com/hc/en-us/articles/115004397274-How-to-access-Firewalla-using-SSH-) if you don't know how already.)

2. Create a file called fwalive.sh (I recommend the `/data` directory 
```

```
3. Next, you must configure two things in `/home/pi/.firewalla/config/post_main.d/install_nextdnscli.sh`
   * **IP** the IP of your Firealla LAN
   * **id** the nextDNS id
   
   ```
   vi /home/pi/.firewalla/config/post_main.d/install_nextdnscli.sh
   ```
   
   Note you can set up different devices using different nextDNS configurations including VPN connections. For example, I have Apple TVs in a different nextDNS configration so I can tune them.  Simply add a line like this: 
   
   -config macaddress=nextdnsconfiguration ID \

4. After editing, run the script 

```
/home/pi/.firewalla/config/post_main.d/install_nextdnscli.sh
```


5. If you want to get notifications when nextDNS is not running, edit `/data/nextdnsdata.txt` to include your IFTTT API key (_optional_)

   ```
   vi /data/nextdnsdata.txt
   ```



# Testing
After completing the steps above, you can: 

1. Open `https://test.NextDNS.io/` 
![11662D0C-718C-4B7F-AC27-816FA02D4764](https://user-images.githubusercontent.com/1205471/189506662-a65c3b78-bc26-4d76-939c-1b75b9233c13.jpeg)


3. Try 
```
dig github.com
; <<>> DiG 9.10.6 <<>> github.com
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 5679
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
;; QUESTION SECTION:
;github.com.			IN	A

;; ANSWER SECTION:
github.com.		10	IN	A	192.30.255.113

;; Query time: 20 msec
;; SERVER: 192.168.241.1#53(192.168.241.1)
;; WHEN: Thu Sep 08 10:56:06 PDT 2022
;; MSG SIZE  rcvd: 55
```
The SERVER line should match the Firewalla server you set if NextDNS CLI is working. OR 

3. You can try https://my.NextDNS.io/ to see if it shows that NextDNS is working. 

Great resource for [all things NextDNS CLI](https://github.com/NextDNS/NextDNS/wiki).


**Standard disclaimer:** I can not be responsible for any issues that may result. Nothing in the script should in any way, affect firewalla as a router or comprimise security. Happy to answer questions though if I can. :)

# Monitoring

You can use the following to run a test to make sure nextDNS is running and log to `/data/logs/nextdns.log` any errors.

If you want to run the monitor automatically, you can add the following line to `/home/pi/.firewalla/config/user_crontab` and restart Firealla. From then on, every 5 minutes the test will check to see if nextDNS is running on Fireawlla. 

```
*/5 * * * *  timeout -s SIGKILL 4m /data/nextdnstest.sh
```

Make sure the "4m" is < the amount of time between runs. So here we have run every 5 minutes and don't let the script run longer than 4 minutes.

You can also send a notification via IFTT. This requires IFTTT to send the notifiction. Edit `/data/nextdnsdata.txt` to include your IFTTT API Key. 

### Known issue
The monitoring script seems to confict with Firewalla and it will often restart nextdns unnecessarily. As a result, I suggest not using the the testing script for the time being until I can figure this out. 

# Pausing
You can temporarily stop nextdns for testing etc. without uninstalltalling it. Simply run this file (or the commands in it). 

[nextdnsstop.sh](https://raw.githubusercontent.com/mbierman/Firewalla-NextDNS-CLI-install/main/nextdnsstop.sh)
 
 To Resume nextdns CLI, simply re-run the install script. Don't worry anything in place will be skipped and everything will restart and should be good. Nothing will be lost. 

# Uninstalling

You can run this by copying this line and run it on firewalla to uninstall. 
```
curl -s -L -C- https://raw.githubusercontent.com/mbierman/Firewalla-NextDNS-CLI-install/main/uninstall_NextDNS_CLI.nosh | cat <(cat <(bash))
```

This script will also be saved when you install and you can just run it locally:
```
/home/pi/.firewalla/config/post_main.d/uninstall_NextDNScli.nosh
```

There are lots of NextDNS communities on Reddit. If you have NextDNS CLI questions, please check there.
