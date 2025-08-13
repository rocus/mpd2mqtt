
# Personal Remarks #

At first I make some remarks on the original work of https://github.com/orbifly/mpd2mqtt. I made some small changes that might make it easier to understand and some extensions that may make it more usefull. 

This program is meant to be a bridge between "The Music Player Daemon" on linux (short MPD) and a MQTT broker and MQTT client. It supports two-way communication between the MPD and the MQTT client. The program can run anywhere in your network although the MPD system or the MQTT broker are the most obvious choice. In the configuration file (./data/mpd2mqtt.conf) of this program (./mpd2mqtt.sh) are lines that describe the location of the MPD server and the MQTT broker. The communication of this program with the broker is by means of calls to mosquitto_sub and mosquitto_pub. The communication with the MPD server is by calls to the mpc program. So these programs must be installed in your Linux system.

The program is written in bash and is not so easy to understand. At startup the program checks the existence of the mpd system and the broker system and then runs an interpreter that waits for mqtt commands. But before that it forks a subprogram that just listens for information from the mpd server (it uses for this the output of a "mpc idleloop" call).

The original program implements the most obvious mpd commands (such as next song in the queue, volume up/down, stop, play, pause, repeat, random and some others). I added a "load" feature. You need the mpc load <file> call to add a playlist file to the (play-)queue. A playlist file can contain songs/mp3's but can also contain an url from a radio station. Because I very often use mpd to play radio stations and because I have many radio stations stored in several subdirectories I also implemented a loaddir command that places m3u files from one directory in the play queue. With a play command you can then select a specific radio station. Because the tuning to a radio station sometimes fails I set the replay and single option first. (see the loaddir command, see also my welle-cli github project).

There are many ways to interact with MPD over a network: there are many clients (windows/linux/android) and this program just adds one more way i.e. by means of MQTT (IoT_MQTT_panel on my android devices). I added a backup file of the data of IoT_MQTT_panel to this project. You only have to import it and fill in the MQTT settings. 

# mpd2mqtt #
## Connects a MPD (music player) with MQTT (broker) in both ways. ##

This can be helpful if you want to connect a MPD with a node-red.
The connection is both-ways: If any client changes the state of MPD the MQTT will be informed. On the other hand you can operate your MPD by sending messages to MQTT broker.

## MPD -> MQTT ##
On change of the current title or toggle between playing and pause on MPD you will get a message on MQTT at topic *music/mpd/get* like:

    {
      "player": {
        "current": {
          "name": "",
          "artist": "Billy Talent",
          "album": "Billy Talent III",
          "albumartist": "",
          "comment": "",
          "composer": "",
          "date": "2009",
          "originaldate": "%originaldate%",
          "disc": "",
          "genre": "",
          "performer": "",
          "title": "Saint Veronika",
          "track": "",
          "time": "4:10",
          "file": "Alben/Billy Talent - Billy Talent III/03 - Billy Talent - Saint Veronika.mp3",
          "position": "16",
          "id": "67",
          "prio": "0",
          "mtime": "Mon Dec 27 16:42:15 2010",
          "mdate": "12/27/10"
        },
        "state": "paused"
      }
    }

The current options of MPD will be send in a message like:

    {
      "options": {
        "volume": "98%",
        "repeat": "on",
        "random": "off",
        "single": "off",
        "consume": "off"
      }
    }

About the playlist/queue it's not so easy to get something out of MPD, so there could be 3 cases:
* We don't know anything about the playlist: `{ "playlist": { "type": "unknown", "displayName" : "<mixed>" } }`
* All songs are from the same album: `{"playlist": { "type": "album", "album" : "The Rasmus - Dead Letters", "displayName" : "The Rasmus - Dead Letters" } }`
* All songs are from a common folder: `{"playlist": { "type": "folder", "folder" : "MixedMusic/preferredSongs", "displayName" : "MixedMusic/preferredSongs" } }`
In my eyes it's not a good way to send the whole list of songs in playlist to MQTT.

To debug or check it you can use the commandline mqtt client:
`mosquitto_sub -h localhost -t "music/mpd/get"`

## MQTT -> MPD ##
You can change the payers state by, sending something to the topic *music/mpd/set*.
* `{"player": "play"}`
* `{"player": "pause"}`
* `{"player": "toggle"}`
* `{"player": "next"}`
* `{"player": "prev"}`
* `{"player": "stop"}`
* `{"player": "update"}`

It's possible to change options by:
* `{"options": { "random": "on"} }`
* `{"options": { "replaygain": "track"} }`
* `{"options": { "consume": "off"} }`
* `{"options": { "repeat": "on"} }`
* `{"options": { "single": "off"} }`
* `{"options": { "volume": "+3"} }`

You can change your queue:
* `{"queue": { { "clear": "true", "add": "MixedMusic/preferredSongs", "play": "true" } } }`
* `{"queue": { { "clear": "false", "del": "3" } } }`
* `{"queue": { { "clear": "false", "insert": "MixedMusic/preferredSongs/I want to hear next.mp3", "play": "true" } } }`

To debug or check it you can use the commandline mqtt client:
   `mosquitto_pub -h localhost -t "music/mpd/set" -m '{"player":"toggle"}'`

## How to checkout and create a docker container and run it: ##
      cd /opt
      git clone https://github.com/orbifly/mpd2mqtt.git
      cd ./mpd2mqtt/
      docker build -t mpd2mqtt .
      #On your host, create a folder for a config file
      mkdir /opt/config_mpd2mqtt
      #Start the continer once 
      docker run --name my_mpd2mqtt -v /opt/config_mpd2mqtt:/data mpd2mqtt
      #Stop the container after some seconds
      docker container stop my_mpd2mqtt
      #Now there is a file /opt/config_mpd2mqtt/mpd2mqtt.config
      #Edit this file to fit your requirements
      nano /opt/config_mpd2mqtt/mpd2mqtt.config
      #Now the container is ready to run
      docker run --name my_mpd2mqtt -v /opt/config_mpd2mqtt:/data mpd2mqtt
      
### How to integrate in docker-compose: ###
Create a docker-compose.yml or extend a existing one. Add the following lines if you build the container as described above.

    mpd2mqtt:
      image: "mpd2mqtt"
      volumes:
          - /opt/config_mpd2mqtt:/data
      restart: always

This project fits to [ct-Smart-Home](https://github.com/ct-Open-Source/ct-Smart-Home). A container is available on Docker Hub, so you just have to add the following lines to docker-compose.yml

    mpd2mqtt:
      image: "orbifly/mpd2mqtt:latest-armv7"
      volumes:
        - ./data/mpd2mqtt:/data
      restart: always

## Some words about security: ##
The idea for this is to make the connection between a frontend system (in my case node-red) and a MPD backend lightweight and indirect. So it is easy to run it on different server, just connected over MQTT. It will increase security as well, because you don't need access to a command line from node-red (exec-node).
I tried to avoid to hand over unquoted strings from MQTT to MPD in my script. But I cannot guarantee for a security. So this software is not for any sensitive use.

## History ##
I started in 2020 with the [ct-Smart-Home](https://github.com/ct-Open-Source/ct-Smart-Home) and some zigbee devices. Now it is grown including MPD and [Lirc](https://www.lirc.org/). A lot of good stuff I found in this [article](https://www.heise.de/ct/artikel/c-t-Smart-Home-4249476.html) and some related. My smartphone is invlolved by [Tasker](https://play.google.com/store/apps/details?id=net.dinglisch.android.taskerm&hl=de&gl=US) and [M.A.L.P.](https://play.google.com/store/apps/details?id=org.gateshipone.malp&hl=de&gl=US) as frontend for MPD. My desktop frontend for MPD is [Cantata](https://linuxreviews.org/Cantata).
