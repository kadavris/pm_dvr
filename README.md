# Smarthome components: poor man's IP cameras DVR with cloud backup
## Scripts for IP cameras recording, backup and housekeeping.
Short description:
* cam_service.pl - main component. service daemon which records single camera rtsp stream.  
* cam_cleaner.pl - keep footage to a specified minimum size. Run from cron or manually.
* cam_proxy.pl - Service daemon to provide a rtsp proxy, allowing multiple connections for dumber cameras.  
* cam_event.pl - Event processor/initiator.  
* cam_sync.pl - Another service daemon. Runs rclone to backup entire event directory to cloud or remote storage.

## Detailed overview
By default I use systemd services. Look into "service" directory for working samples.   
All daemons run using 'smarthome' username and group. Change stuff to your preference before activating the services.

### cam_service: RTSP stream recorder
Main workhorse. Uses ffmpeg to suck in stream from a single camera and save it into neatly sized clips for later.
Start separate services via cam_service@CAM_ID  
Run with -h switch to get help   
Main configuration is in the cam_service.config.
It includes cam_common.config for base settings and cam_rtsp.config for camera-specific stuff.

### cam_event: External event processor
Use to reserve recorded clips at the time of event or incident.  
The method is to hardlink the footage of specified cameras around the current time into the designated directory.  
1) Creates series of existing (pre-event) clips, starting N seconds _before_ the current time.  
2) Loops and waits for a current clip to be finished. Links it too.  
3) Repeating for another N seconds of post-event footage.  
This enables user to see what's got the event to run at the first place and what happened some time after.  

Use cases:
* When run manually it does just that and exits.  
* If persistent recording: -p=on switch is used, the script will run indefinitely in foreground (until the persistent-recording-flag file is gone).  
You should run script again with -p=off switch to stop
* When run with the -m switch it:
  1) Connects to a MQTT server
  2) Subscribes to specified topic, waiting for the event message to come.
  3) When the message arrived it forks and runs the clips hardlinking procedure (see above)
  4) Loop

### cam_sync: Cloud syncing daemon.
When "sync flag" file detected, it runs rclone to do a backup for entire event directory to the cloud or other remote storage.  
_NOTE: Must be run from account owning rclone's configuration._

### cam_cleaner: Cleans up old recordings.
Expected to be run from cron.hourly. Ready-made shell script included.

### cam_proxy: RTSP proxy frontend
The idea is to make it possible for dumb cameras' footage to be viewed and recorded in background at the same time.  
Yet it seems that many cheap, Chinese devices have very broken RTSP implementation, so live555 constantly whines
about inconsistensies and usually re-streams broken picture.  
Maybe it is in fact live555's problem as ffmpeg works just fine. The last time I tried to use it was around 2018.

This stuff is disabled by default, and you don't need it if your camera(s) support multiple RTSP connections from the box.

### Directories:
* service - .service samples for systemd
* configs - .config file samples
* testing - test scripts

### Dependencies:
* perl5
* Net::MQTT::Simple
* ffmpeg
* live555 (optional)
* sendmail (optional)

Repo is at https://github.com/kadavris/pm_dvr
