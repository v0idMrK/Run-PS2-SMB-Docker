So after searching everywhere for help on how to use OPL SMB functionality with docker I found myself stuck without being able to connect to it with errors (OPL would spit out error 300 and other errors).

I've searched everywhere and came to the conclusion that only some SAMBA versions are compatible with OPL (this is probably due to older SMB protocols being deprecated for security reasons - more on this later).

So using docker-compose I am now able to connect to SMB using OPL with no problems whatsoever. Even POPSTARTER works!

DISCLAIMER: I am not responsible for anything that happens using this guide. Also, be aware that you will be using a vulnerable SAMBA version. And you should never use it with files you care about and you should NEVER, EVER expose it to the internet! (it is inside a docker container, so even if someone exploits this, the damage should be kept to the container only).

Also, I am using Linux. More specifically I am using Ubuntu Server 22.04.2 LTS, but this should work on anything that supports docker.

So, first of all create a directory called "ps2smb" somewhere on your computer that we will use to hold the files that describe this docker container using docker-compose.
Then we will need to create a .env file. This .env file will hold several variables, such as timezones, config and the directory that will be shared to the PS2. The .env file should look like this:

    # Your timezone, https://en.wikipedia.org/wiki/List_of_tz_database_time_zones
    TZ=Africa/Abidjan
    # UNIX PUID and PGID, find with: id $USER
    PUID=1000
    PGID=1000
    # The directory where data and configuration will be stored.
    ROOT=/home/
    PS2GAMES=/home/PS2Games

(You will need to make sure about the PUID and PGID, these you can search on the internet on how to get them)


Then we will need to create a docker-compose.yml file that will describe the usage of our docker image, volumes, etc... It should look like this:

      samba:
        image: crazymax/samba:4.13.8
        container_name: samba
        ports:
          - "445:445/tcp"
        volumes:
          - "${ROOT}/config/PS2SMB/config:/data"
          - ${PS2GAMES}:/PS2SMB
        environment:
          - TZ=${TZ}
          - PGID=${PGID}
          - PUID=${PUID}
          - "SAMBA_LOG_LEVEL=1"
        restart: always


Lets go over some things on this docker-compose file.

* We will be using crazymax's samba docker image (https://github.com/crazy-max/docker-samba)
* The version 4.13.8 tag is very important, without this it probably won't work as it needs the older SMB protocols that aren't present in the newer SAMBA versions (don't quote me on this, I could be wrong)
* The port can be changed for any port that you would like (useful if you already have a SAMBA share running on your host). Just be aware that Windows cannot connect to SMB hosts on another port that isn't 445 (don't quote me on this).
 

After all of this is setup, be sure to start this docker container. You can do this by issuing the "docker-compose up" command and checking if the container is starting up properly. If you have any issues here make sure all the directories you are referencing in the docker-compose.yml file are created and your users have permissions to use those directories.

After making sure of all of the above, bring the container down by issuing the "docker-compose down" command as we are not done yet.

We need to go to the ${ROOT}/config/PS2SMB/config folder and create a config.yml file. This file will setup the shares you need and make sure the older SMB protocols are being used so it is compatible with the PS2. This config.yml should look like this:

    auth:
      - user: PS2SMB
        group: PS2SMB
        uid: 1000
        gid: 1000
        password: YOURPASSWORDHERE
    
    global:
      - "force user = PS2SMB"
      - "force group = PS2SMB"
      - "server min protocol = CORE"
      - "client min protocol = SMB3"
      - "client max protocol = SMB3"
      - "null passwords = yes"
      - "lanman auth = yes"
      - "ntlm auth = yes"
    
    share:
      - name: PS2SMB
        path: /PS2SMB
        browsable: yes
        readonly: no
        guestok: yes
        validusers: PS2SMB
        writelist: PS2SMB


Lets go over some things on this config.yml file:

*  uid and gid should be the same as your .env file
* valid users should be a name of your choice and it will have to match in OPL but PS2SMB is an OK name for the user
* Set your password to something other than YOURPASSWORDHERE 

Save the file and run "docker-compose up". It should detect your shares and everything should be ok. After you confirmed it you can now do "docker-compose down && docker-compose up -d" to run it in detached mode.

After all of this is done you can just connect using OPL and browse your games and stuff :D

Some things you should know:

* Sometimes, for some reason (don't ask me why), this docker container ends up in a restart loop (especially after adding games and restarting it). I fix this by deleting the "cache" and "lib" folders on the ${ROOT}/config/PS2SMB/config folder and restarting the docker container.
