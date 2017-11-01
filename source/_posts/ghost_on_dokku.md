---
title: ghost on dokku
date: 2016/8/9 07:49:00
tags:
- dokku
- docker
---
What could be more appropriate for a first post than to walk through how I set up this site? I decided to use ghost after a quick search and found this [great tutorial](https://adrianperez.org/advanced-deployment-of-ghost-in-2-minutes-with-docker/) from [Adrian Perez](https://adrianperez.org/#open) on setting it up with Docker. I already had a dokku server running so I decided to adapt his method to work a little better for dokku. I'm assuming you have some familiarity with dokku and have a dokku server running somewhere. If you have no idea what dokku is, I recommend you [check it out](http://dokku.viewdocs.io/dokku/): basically it's like Heroku but you run and manage it yourself.

**Goals**

* Configuration via environment variables instead of code and hardcoded values (config changes should not require a code change)
* Persistent storage that would be easy to scale to multiple web workers

Since dokku now supports Dockerfile and docker image deployments, I first tried to use the official ghost image on docker hub but ran into a few issues. The main issue with this is that in order to use a custom config.js, you need to place the custom `config.js` into your content directory (discussed below). This might be fine for you, but I didn't like the idea that to deploy a change I would need push to git and then log into the server and pull it down. Wouldn't it be better if `git push dokku master` still worked?

The other reason I didn't like the original setup is that I wanted to set my configuration the way I normally do via `dokku config:set blog URL=<my-site> ...`. In order to do that, I created a new Dockerfile that inherits from the main ghost Dockerfile but it includes a `config.js` that loads from the environment instead. This `config.js` is placed in the ghost source code folder instead of content because it is no longer customized per deployment.

**Deploying**

Now to set up the app:

* Log into your dokku server and create the app
* Create a storage mount and link it to your new app at `/var/lib/ghost`
  * This will create a directory on your host server that will be mounted into your ghost container and survive restarts/redeploys. This directory is where you will install your themes, and where images are stored by default.
* Create a database and link it to your newly created app 
  * The default `sqlite` db will persist in your storage mount, but a db may make it easier to scale the app in the future
* Deploy the app
* Proxy the port


```bash
# Create the app
dokku apps:create blog

# Create the storage and mount it
mkdir -p  /var/lib/dokku/data/storage/blog
chown -R 32767:32767 /var/lib/dokku/data/storage/ghost
dokku storage:mount blog /var/lib/dokku/data/storage/ghost:/var/lib/ghost

# Deploy with git/Dockerfile
git clone git@github.com:lamflam/ghost_dokku.git
cd ghost_dokku
git remote add dokku dokku@<your_domain>
git push dokku master
```
or
```bash
# Deploy with docker image (I didn't actually test this one)
docker pull lamflam/ghost_dokku
docker tag lamflam/ghost_dokku:latest dokku/blog:latest
dokku tags:deploy blog latest
```

**Themes**

Installing themes is very easy to do from your host server because of the mounted storage directory. All you need to do is unzip the theme into the themes directory. If you have the URL for your theme, it's as easy as:

```bash
cd /var/lib/dokku/data/storage/ghost/themes
curl -L http://path/to/theme.zip >> theme.zip
unzip theme.zip
rm theme.zip
dokku ps:restart blog
```

Now refresh your blog settings page and the theme should be available!
