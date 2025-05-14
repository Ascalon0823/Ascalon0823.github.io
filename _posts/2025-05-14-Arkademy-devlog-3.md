---
layout: single
title: Arkademy Devlog 3
date: 2025-05-13 20:19 +0800
categories: Personal
tags: ["Devlog","Arkademy"]
toc: true
toc_lable: "Contents"
---

# Updates
---
- Hosting strategy

# Backend
---

Finally I decided to host my backend server on cloud.
Currently the stack is as such:
- **Front end Unity WebGL:** still hosted on this Github page, but now its no longer using offline mode since proper https connection can be established to backend server.
- **Back end Go server:** the container is pushed to Google Cloud Artifact Registry, then hosted on Cloud Run
- **Mongo DB:** free tier cluster on mongoDB Atlas on MongoDB, hope it is really free.

# Steps
---

## DB:

This part is pretty simple. 

**1 Create an account on [MongoDB](https://www.mongodb.com/cloud/atlas/register)**

**2 Follow the steps to create a project**

**3 Create a free (M0) cluster**

**4 Set up allowed IP,use 0.0.0.0/0 to accept everyone**

**5 Get connection string**


For my server I expect a database named *arkademy* and a collection named *user*, thus I created them directly from the console page on mongoDB web.
I tested the connection in VScode MongoDB extensions with the connection string, works fine. This is the first step towards cloud.

However when I run my server locally in WSL, it refused to connect to db, complaining about DNS setup. 
After a quick search I found out that the DNS setup in WSL is not using a proper one, which is stated in [MongoDB Go driver doc](https://pkg.go.dev/go.mongodb.org/mongo-driver/mongo#hdr-Potential_DNS_Issues)

To change the DNS, I follow [this post](https://askubuntu.com/questions/1347712/make-etc-resolv-conf-changes-permanent-in-wsl-2)

``` bash
sudo rm /etc/resolv.conf
sudo bash -c 'echo "nameserver 8.8.8.8" > /etc/resolv.conf'
sudo bash -c 'echo "[network]" > /etc/wsl.conf'
sudo bash -c 'echo "generateResolvConf = false" >> /etc/wsl.conf'
sudo chattr +i /etc/resolv.conf
```

## Server:

Before I move the container to cloud, I changed the dockerfile abit to remove the LOCAL naming, and using port 1027 instead of 8080.
Test it locally everything works, even with the mongoDB on Atlas connection string, I thought it might have the DNS issue again in container.
After confirming the container is working, I started moving it to cloud.

This part takes most of my time, and I wasted a lot of time because I did not read the doc properly.

**1 Create Google Cloud Account.**

**2 Enable Artifact Registry**

**3 Create Repository**

**4 Setup Gcloud CLI**

**5 Build and Push Docker Image**

Part 5 is where I stuck...on simple mistakes.

I build an image locally, then follow the steps to tag and image and push to the repository using the copied url.

However I miss understood the url composition.

According to the documentation I should be pushing to 
> Registry_location/Project_name/Repository_name/**Image_name**

What I copied is only 
> Registry_location/Project_name/Repository_name

The error message says unauthenticated or not exist. I naturally think it is because I did not setup the Gcloud auth properly or docker is doing some weird thing with WSL.

Therefore I spent hours trying to get the Gcloud credential correct and test on both wsl on windows docker. And of course none of them works.

Finally I found [this official doc](https://cloud.google.com/artifact-registry/docs/docker/pushing-and-pulling#tag), which points out the problem I had.

After rebuild and tag, finally push to the correct url, my image is on cloud.

Continue to Cloud Run...

**6 Deploy a repository**

This part is quite straight forward, except some Env setup for the server can be easily done in the form.

The first deployment went wrong, I quickly check the log and found out the connection to mongoDB Atlas failed.
I paniced a bit as I thought it is because of some complex networking issue, and quickly realize that my server's ip is not listed in my mongoDB cluster network access.
So I just add 0.0.0.0/0 into it and tested again.

**7 Server active.**

I quickly tested in my editor by copy the Cloud Run url into my unity env scriptable object.
Everything works. Now it is 95% done.

## WebGL

This part was quite easy.

**1 Set the url into prod env scriptable object and set to online**

**2 Build**

**3 Upload to github page**

After page's action, I can have my game running fully on cloud.

You can also try [here](/Arkademy/index.html)

# Next Step

## CI/CD
Although it is quite fast to push new image and redeploy a revision, I still want to automate this part.

I guess I need to change the git action on my current CI. So that every push in git will auto update the Registry and Cloud Run

In the future I will also need to separate the pipeline for production, staging and dev. Dev will mostly be local, which was used before this update.

However since I can't have another free atlas, I guess I will have to make a new database in my cluster for staging data. Not sure about the container and cloud run though

## Security

Currently every part of the web app is pretty unsafe

- Server code accepts all origin for CORS
- Mongo Cluster accepts all ip
- No rate limiting etc to limit the bill


# Tasks
---

- [x] Client build and deploy flow
- [x] 2D Game basics (Control, character, camera)
- [x] Basic game loop
- [x] Continuous ability
- [ ] Rift rewards (Currencies, Equipments) 
    - [ ] Equipment and items
- [x] Character building (Level up, Shops)
    - [x] Character Ability tree
    - [x] Character HUD


- [x] DB setup
- [x] User register and authentication
- [x] Find a working http package for webgl
- [x] Hosting on Cloud
- [ ] CI/CD for cloud hosting
- [ ] Rift reward generator API
- [ ] ~~Character building API~~
