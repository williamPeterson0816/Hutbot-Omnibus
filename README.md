# Hutbot-Omnibus
Using a bot to update the event from Slack.
1.	Modify the Objectserver configuration file to allow remote HTTP access
modify the file /opt/IBM/tivoli/netcool/omnibus/etc/AGG_P.props

NHttpd.EnableHTTP: TRUE
NHttpd.ListeningPort: 7070
NRestOS.Enable: TRUE	
2.	Restart the objectserver with the following commands:
/opt/IBM/tivoli/netcool/omnibus/bin/nco_pa_stop -process MasterObjectServer
/opt/IBM/tivoli/netcool/omnibus/bin/nco_pa_stop -process MasterObjectServer

Use the root password when asked.

3.	Test that you have remote access by running the command 
telnet localhost 7070
4.	Open a cmd window and run the following commands as root:

curl -sL https://rpm.nodesource.com/setup_6.x |sudo -E bash -
yum install -y nodejs
These commands install node.js to the server
5.	Run the following command as root:
npm install -g yo generator-hubot
This command installs a bot "generator".
6.	Run the following command as root:
npm install -g node-omnibus
This command installs a node.js library that can converse with the ObjectServer over REST
7.	The following commands must be run as localuser
su localuser
cd ~
mkdir bot
cd bot
yo hubot
 
8.	Enter your own details for the bot name and description, choose "slack" as the adaptor type
 
9.	Run the command "ls -ltr" and you will see the following files:
 
10.	Edit the file package.json and add a dependency to the node-omnibus component: 
    "node-omnibus": "^0.1.1"
 
Create the file noi.coffee in the ./scripts subdirectory and insert the following code into it:
console.log ('Omnibus = ' + process.env.HUBOT_OMNIBUS_HOST)

omnibus = require('node-omnibus')
request = require('request')
HubotSlack = require 'hubot-slack'

noiChannelName = ""
omnibusConnection = omnibus.createConnection(
   host: process.env.HUBOT_OMNIBUS_HOST
   port: '8080'
   user: process.env.HUBOT_OMNIBUS_USER
   password: process.env.HUBOT_OMNIBUS_PASSWORD)

module.exports = (robot) ->
robot.hear /noi ack (.*)/i, (msg) ->
		sql = 'UPDATE alerts.status set Acknowledged = where Serial=' +  msg.match[1]
        omnibusConnection.sqlCommand sql, (err, rows, numrows, coldesc) ->
		    console.log "err:" + err
			if numrows == 0
				msg.send "I can't find any event with a serial number of " +  msg.match[1]
				msg.send "Perhaps the event has been closed already? Otherwise, try another serial number."
			else
				msg.send "Event acknowledged"      
				
	robot.hear /noi deack (.*)/i, (msg) ->
        sql = 'UPDATE alerts.status set Acknowledged = 0 where Serial=' +  msg.match[1]
        omnibusConnection.sqlCommand sql, (err, rows, numrows, coldesc) ->
		    console.log "err:" + err
            if numrows == 0
				msg.send "I can't find any event with a serial number of " +  msg.match[1]
				msg.send "Perhaps the event has been closed already? Otherwise, try another serial number."
            else
				msg.send "Event de-acknowledged"

11.	Run the bot using the command 
HUBOT_OMNIBUS_PASSWORD= HUBOT_OMNIBUS_HOST=localhost PORT=7070 HUBOT_SLACK_TOKEN=<<YOURTOKENHERE>> ./bin/hubot --adapter slack
12.	From the Slack window, invite the bot to your slack channel with the command /invite <botname>
13.	In the DASH event viewer, choose an event and view it’s details to find the Serial number. Run the command "noi ack <serial#>" in the Slack command and verify that it has been acknowledged in the DASH event viewer.
14.	Stop the bot using ctrl-C and update the file noi.coffee with the following rows:

    robot.hear /noi journal (.*?) (.*)/i, (msg) ->
		dt = Date.now()
		dt = dt / 1000
		sql = 'Insert into alerts.journal (KeyField, Serial, UID, Chrono, Text1) values (\'' + msg.match[1] + '_' + dt + '\',' + msg.match[1] + ', 10000 ,' + dt + ', \'' + msg.match[2] + '\')'
		console.log sql
		omnibusConnection.sqlCommand sql, (err, rows, numrows, coldesc) ->
			console.log "Err=" + err
			msg.send "Journal entry added"

15.	Start the bot and run the command "noi journal <serial#> This is a test". Check in the event viewer to see if the event's journal has been updated.
16.	Stop the bot using ctrl-C and update the file noi.coffee with the following rows:
   
	robot.hear /noi sev (\d) (.*)/i, (msg) ->
        sql = 'UPDATE alerts.status set Severity = ' + msg.match[1] + '  where Serial=' +  msg.match[2]
        omnibusConnection.sqlCommand sql, (err, rows, numrows, coldesc) ->
   			console.log "Err=" + err
            if numrows == 0
				msg.send "I can't find any event with a serial number of " +  msg.match[2]
				msg.send "Perhaps the event has been closed already? Otherwise, try another serial number."
            else
				msg.send "The event severity has been set to " + msg.match[1]


17.	Start the bot and run the command "noi sev <serial#> 5". Check in the event viewer to see if the event's priority has been changed to Critical/Red.
18.	Stop the bot using ctrl-C and update the file noi.coffee with the following rows:

robot.hear /noi resolve (.*)/i, (msg) ->
		sql = 'UPDATE alerts.status set Severity = 0,CASE_ICD_Status=\'RESOLVED\'  where Serial=' +  msg.match[1]
        omnibusConnection.sqlCommand sql, (err, rows, numrows, coldesc) ->
			console.log "Err=" + err
			msg.send "Event resolved (Severity set to 0)"
19.	Start the bot and run the command "noi resolve <serial#>". Check in the event viewer to see if the event's priority has been changed to Clear/Green.
20.	Stop the bot using ctrl-C and update the file noi.coffee with the following rows:

	robot.hear /noi show (.*)/i, (msg) ->
		query = 'SELECT Serial, Summary, Severity, Node, Acknowledged, LastOccurrence from alerts.status where Serial=' +  msg.match[1]
		omnibusConnection.query query, (err, rows, numrows, coldesc) ->
			console.log err
			i = 0
			msg.send 
				text : rows[i].Summary
				attachments: [ {
					title: 'Node'
					text: rows[i].Node
					fields: [ {
						short : true
						title : 'Severity'
						value : rows[i].Severity
					},{
						title : 'Acknowledged'
						value : rows[i].Acknowledged
						short : true
					},{
						title : 'LastOccurrence'
						value : rows[i].LastOccurrence
						short : true
					},{
						title : 'Serial'
						value : rows[i].Serial
						short : true
					} ]
				} ]
				username: process.env.HUBOT_SLACK_BOTNAME
				as_user: true
				
21.	Start the bot and run the command "noi show <serial#> "
