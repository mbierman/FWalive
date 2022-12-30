# Introduction: Update a Google Spreadsheet with Firewalla's status

This is a script for updating a Firewalla box's status in a Google Spreadsheet and sending an alert when it is down longer than any period you choose. It is tested on  Firewalla 1.975.

Technically, Firewalla is sending an "alive" message to the spreadsheet via IFTTT and IFTTT monitors if the spreadsheet ever says Firewalla is down. 


# Notes
- This requires an IFTTT account to work.
- Firewalla has some of this kind of functionality built in. But I wanted faster notifications and to be able to define how I got notified (email, slack, notification, etc.) 

# Installation
To install:
1. Create a Google Spreadsheet [like this](https://docs.google.com/spreadsheets/d/1-31nv9rvPw_RRu9565bGzuondch9V0y8jQglDQo21uM/edit?usp=sharing)
<img width="699" alt="image" src="https://user-images.githubusercontent.com/1205471/210029758-6edacdee-c632-497a-8acc-0cdccd2aee48.png">
You can copy mine, if you wish. 

2. Go to [IFTTT.com](https://IFTTT.com) and create an applet as shown below.
<img width="828" alt="image" src="https://user-images.githubusercontent.com/1205471/210029514-f6eea947-b819-41a5-8d30-ff53eb90f331.png">
The filter code looks like this: 

```
let mytime = Meta.currentUserTime.format('l LT');
GoogleSheets.updateCellInSpreadsheet.setPath('IFTTT');
GoogleSheets.updateCellInSpreadsheet.setFilename('Alive');
GoogleSheets.updateCellInSpreadsheet.setCell('A1');
GoogleSheets.updateCellInSpreadsheet.setValue(mytime);
```
and the Spreadsheet action looks like this: 
![image](https://user-images.githubusercontent.com/1205471/210031071-378c7bc4-adc1-4836-8f6d-30296211703c.png)

The aplet will receive a webhook from Firewalla on a schedule you choose and update the Google spreasheet. Once you creeate it you can use curl or what have you to test updating the spreadsheet. 
 
This assumes a setup like the spreadsheet provided. If you make any changes you have to adjust accordingly. 

3. Next, create another applet that will monitor the Google Spreadsheet and notify you (or whatever you choose. 
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
#!/bin/bash

curl https://maker.ifttt.com/trigger/alive/json/with/key/YOURIFTTTKEY
```

Note "alive" is the trigger and YOURIFTTTKEY can be found under Documentation https://ifttt.com/maker_webhooks 

3. Next, you will want to create a cron job to call the webhook we created earlier on a schedule of your choosing. You do this by going to ~/.firewalla/config
   
   ```
  cd ~/.firewalla/config 
  vi user_crontab
   ```
   
   Add a line something like this
   ```
   */1 * * * * /data/fwalive.sh
   ```
   
   which runs once every minute (or however often you want it to run). 
   
   Save the file and restart Firewalla. You can confirm the cronjob is in place by doing `crontab -l` after Firewalla restarts.
