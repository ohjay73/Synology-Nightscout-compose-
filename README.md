What’s needed

Synology NAS

This document is a walk through on how to create a nightscout service on your Synology NAS (in this case using a DS923+) and have it accessible outside of your network for use with AAPS

You will require a purchased domain from cloudflare.The process to do so is available on cloudflare’s site. www.cloudflare.com


Githubs used

https://github.com/nightscout/AndroidAPS

https://github.com/nightscout/cgm-remote-monitor

https://github.com/mrikirill/SynologyDDNSCloudflareMultidomain - used to create an internal DDNS entry in the Synology NAS for cloudflare DDNS

There is an alternate way where you use the internal NAS’s DDNS service which eliminates the need for cloudflare but also increases complexity as you will need to create certificates via lets encrypt, port forwarding in your NAS as well as to configure a reverse proxy and configure your web portal. All of which add’s unique challenges and this guide eliminates. Docs on how to do so are available online.  A good reference is here https://github.com/ohjay73/Synology_NightScout_Easy

You can also use this same process to expose any other app on your NAS including the login so you don’t have to use synology’s quickconnect option. 


This Doc assumes you have DSM 7.x installed. In this case we are using DSM 7.2.2-72806

Tools in DSM used:

File station
Container manager
Control Panel - External Access - DDNS

In file station you need to create the base folders that your docker project will reside inside

In this case we are using /volume1/docker/Mongo/nightscout-mongodb. This folder path must be created in advance either by file station or if you prefer to create it all by command line. 

You'll also need to create volume1/docker/cloudflare for the DDNS later

volume1 will only show up when logging into the nas via commandline. In filestation you will create your directory structure under the docker top level.

Docker compose YAML
This YAML will create 3 containers;
Mongodb
Nightscout instance using latest_dev for 15.0.3 and latest AAPS support
Mongo express 

I am using mongo 5.0 for this example which works with the 923+ but 4.4.18 works as well and you may have to adjust this setting to get it working on your Synology NAS. I have not had any success in building this with Mongo 6 and above. The outcome of YAML will provide you with a NS site that is open to all viewers with the appropriate url and is set to accept data from AAPS for treatments, BG and profiles. You will need to create your own Access tokens within NS to get AAPS and NS to communicate and the assumption is you are aware of how to do so. 

Please examine the ENABLE section where the environment tables are set to ensure you have what you require for options. 

see example nightscout-mongodb-mongoexpress-defaultcompose.yaml in project list

Items you must adjust;

     API_SECRET: PUTYOURSECRETHERE
     CUSTOM_TITLE: NAMEYOURSITEHERE

Once your YAML is adjusted to your liking open the Container Manager in DSM select project and click the create button. 

Project Name : whatever descriptor you need to identify the project

Path: click set path and select docker/Mongo/nightscout-mongodb

Source: Create docker_compose.yaml

Copy the above YAML into the project and click Next, Next and Done and the project will create. For this you do not need to create a web station config but if you were creating this for a fully in DSM built with its internal DDNS, and certificates you would need to configure webstation to work

Once this is built you can test your Nightscout instance by putting in the url IPTOYOURNAS:1337 in your browser and as long as everything is healthy you should have a  working website.
At this point I suggest you attempt to authenticate to the page and goto Admin tools to create an Access Token. A token with admin privileges is needed to work with AAPS. You can also setup your base profile at this time as AAPS will look to pull it from your NS instance once it connects.  

DDNS redirector for cloudflare from your NAS. 

CloudFlare has been removed as an option in DSM 7x so you will have to create it following these steps. There are perhaps easier ways but this is fully built and ready to rock.

Follow the directions HERE to enable and configure DDNS for cloudflare directly. 
https://github.com/mrikirill/SynologyDDNSCloudflareMultidomain

Cloudflare management of tunnels to your NAS

Cloudflare has a comprehensive set of options including once that will handle all of your redirects via subdomains. 

Once logged into cloudflare - In the search function look for Zero Trust and select it
Your first time in Zero Trust you may have to create a team name and your support package. All of this works just fine under the free support package. 

Once at the main Zero Trust package, 

Select Network → Tunnels and create new Tunnel and then make sure cloudflared is selected and click next

You’ll have to create a descriptor at this time for your Tunnel and click next

Now you’ll need to prepare your YAML for the cloudflare redirect. 

Select Docker and copy the command in full, you’ll only need the API Token string to complete this portion

docker run cloudflare/cloudflared:latest tunnel --no-autoupdate run --token REALLYLONGTOKENSTRINGTOCOPY


In a text doc I suggest you copy/paste this and replace the appropriate info with your actual token

version: "3.3"
services:
 cloudflared:
   image: cloudflare/cloudflared:latest
   command: tunnel run
   environment:
     - TUNNEL_TOKEN=CUTANDPASTETOKENFROMCLOUDFRONTHERE



With the above YAML you’ll need to go back to Container station and build your connector 

Open File Station and create a new folder under docker called cloudflare

Open Container station

Under project create new project

Project Name : whatever descriptor you need to identify the project

Path: click set path and select docker/cloudflare

Source: Create docker_compose.yaml

Copy the above YAML into the project and click Next, Next and Done and the project will create.

Confirm that the project builds correctly and then head back to cloudflare/Zero Trust

In the zerotrust webpage you left earlier, if your cloudflared project ran correctly, you will see that the Connector has a status of connected. 

Click next and you will now configure your routes. 

Subdomain: you can select anything here but I chose nightscout

Domain: your fully qualified domain from cloudflare

Service Type: HTTP 

URL: your direct internal path including port ie 192.1.1.50:1337

If you entered everything correctly as this point you can type nightscout.yourdomain in a browser and your nightscout instance will show up

You can now add your nightscout URL into AAPS and this will work for v1 and v3 with websockets




 



 
