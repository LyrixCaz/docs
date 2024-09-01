## __Method of hosting:__ Container (Rootless Podman)

### __Tidbits & notes:__ Make sure you have two domains

- One for Public access (web facing etc)
- One for file serving (securely if web facing) as minio serves the files on the clear (OTC) to whatever public url is specified, this case being an https top level domain (TLD)
- For Local interaction make sure you populate the public urls and storage urls with the local IP of host access, not the container IP.
- For the love of god please make sure you leave a trailing ' / ' on the storage url and NO trailing ' / ' on the public url. 
- In the quadlets, when specifying a path i end with :Z due to SELinux, if you do not have SELinux on your computer, then disregard putting the ' :Z '

You need to make sure of all of these for PDF export/download to work. As of 9/1/24 there are no previews in the resume dashboard, i suspect it will be implemented in the future, it only makes sense, at least to have an option to toggle such function if system resources are a concern.

---

#   
__Quadlets:__

Setting this software to use quadlets will be the best approach to "set it and forget it" do as follows:

- Review these and adjust to your paths and username.

Begin creating the quadlets in the following directory: **\~/.config/containers/systemd**

**postgresql-rx.container:**

```
[Unit]
Description=DB container for the resume builder.

[Container]
AutoUpdate=registry
Label="io.containers.autoupdate=registry"
ContainerName=postgresql-rx
Image=docker.io/library/postgres:16-alpine
Volume=/home/USER/containers/postgres-rx/pgdata16-alpine:/var/lib/postgresql/data:Z
Timezone=America/New_York
Pod=Rx.pod
EnvironmentFile=/home/USER/bin/env/psql-rx.env
PodmanArgs=--memory 2G

[Service]
Restart=unless-stopped

[Install]
WantedBy=default.target
```

**minio.container:**

```
[Unit]
Description=Storage (for image uploads) For Resume builder
After=network-online.target

[Container]
AutoUpdate=registry
Label="io.containers.autoupdate=registry"
ContainerName=minio
Image=docker.io/minio/minio:latest
Volume=/home/USER/containers/minio/data:/data:Z
Timezone=America/New_York
Pod=Rx.pod
EnvironmentFile=/home/USER/bin/env/minio.env
Exec=server /data

[Service]
Restart=unless-stopped

[Install]
WantedBy=default.target
```

**rx-web.container:**

```
[Unit]
Description=Reactive resume app Web facing
After=network-online.target

[Container]
AutoUpdate=registry
Label="io.containers.autoupdate=registry"
ContainerName=resume-web
Image=docker.io/amruthpillai/reactive-resume:latest
Timezone=America/New_York
Pod=Rx.pod
EnvironmentFile=/home/USER/bin/env/rx-web.env

[Service]
Restart=unless-stopped

[Install]
WantedBy=default.target
```

**Chromium.container:**

```
[Unit]
Description=Chrome Browser (for printing and previews)

[Container]
AutoUpdate=registry
Label="io.containers.autoupdate=registry"
ContainerName=chromium
Image=ghcr.io/browserless/chromium:v2.13.0
Timezone=America/New_York
Pod=Rx.pod
EnvironmentFile=/home/USER/bin/env/chromium.env

[Service]
Restart=unless-stopped

[Install]
WantedBy=default.target
```

Those are the 4 main containers. Now all you need is to setup your environment files (which I highly recommend), your actual pod & network to encapsulate these containers.

---

## __Setting up the Pod & its Network:__

  
Again tailor these settings as you see fit. In the same location of **\~/.config/containers/systemd/** create the following:

**Rx.pod:**

```
[Unit]
Description= Resume buildiner pod with postgres, minio, chrome, & reactive resume
After=network-online.target

[Pod]
PodName=Rx
Network=Rx.network
PublishPort=3030:3030
PublishPort=3300:3300
PublishPort=9000:9000
PodmanArgs=--cpus=2 --ip 10.89.3.15 --hostname Resume
```

**Rx.network:**

```
[Network]
Driver=bridge
IPv6=false
Internal=false
DisableDNS=false
IPAMDriver=host-local
NetworkName=Resume
Subnet=10.89.3.1/24
Gateway=10.89.3.1
DNS=9.9.9.9

[Install]
WantedBy=default.target
```

---

## env files:

As you noticed there are **EnvironmentFile** referenced in each of the quadlets. Now you can add the envionment files accordingly to each container and any time you want to change a value all you have to do is edit those .env files directly and restart the containers via **systemctl --user restart *containername*** . These are the files, again tailor to your liking; I will also use the same location referenced in the quadlets.

**psql-rx.env:**

```
POSTGRES_DB=postgres
POSTGRES_USER=postgres
POSTGRES_PASSWORD=postgres
```

**minio.env:**

```
MINIO_ROOT_USER=minioadmin
MINIO_ROOT_PASSWORD=minioadmin
```

**rx-web.env:**

```
PORT=3300
NODE_ENV=production

PUBLIC_URL=https://my1.domain.tld
STORAGE_URL=https://my2.domain.tld/default/

CHROME_TOKEN=chrome_token
CHROME_URL=ws://10.89.3.15:3000

DATABASE_URL=postgresql://postgres:postgres@localhost:5432/postgres

ACCESS_TOKEN_SECRET=access_token_secret
REFRESH_TOKEN_SECRET=refresh_token_secret

MAIL_FROM=myemail@address.com
SMTP_URL=smtp://myemail:pwd@smtp.someserver.com:587

STORAGE_ENDPOINT=10.89.3.15
STORAGE_PORT=9000
STORAGE_REGION=us-east-1 # Optional
STORAGE_BUCKET=default
STORAGE_ACCESS_KEY=minioadmin
STORAGE_SECRET_KEY=minioadmin
STORAGE_USE_SSL=false
STORAGE_SKIP_BUCKET_CHECK=false
```

**chromium.env:**

```
TIMEOUT=50000
CONCURRENT=10
TOKEN=chrome_token
EXIT_ON_HEALTH_FAILURE=true
HEALTH=true
```

Aaannd your done! Please note that I use chromium with a particular version just because I encountered the issue with the PDF downloading so to be on the safe side I remained using this version of chromium. Not sure if it still works with the latest but i just want to set this thing up and be done with it, specially in such a critical step which is the PDF downloading.     
  
Another step I took was to fire up another reactive resume container with virtually the same contents on the quadlet but with a different port (3030) and with the Public url and storage url matching my local IP so I can also use the software without an internet connection. Until now, I don't know of another way to accomplish this without doing another quadlet. If you want to do the same here is the quadlet and its env file:

**rx-local.container:**

```
[Unit]                                                                                                     
Description=Reactive resume app for local
After=network-online.target                                  

[Container]
AutoUpdate=registry
Label="io.containers.autoupdate=registry"
ContainerName=resume-local
Image=docker.io/amruthpillai/reactive-resume:latest
Timezone=America/New_York
Pod=Rx.pod
EnvironmentFile=/home/USER/bin/env/rx-local.env

[Service]
Restart=unless-stopped

[Install]
WantedBy=default.target
```

**rx-local.env:**

```
PORT=3030
NODE_ENV=production

PUBLIC_URL=http://192.168.122.1:3030
STORAGE_URL=http://192.168.122.1:9000/default/

CHROME_TOKEN=chrome_token
CHROME_URL=ws://10.89.3.15:3000

DATABASE_URL=postgresql://postgres:postgres@localhost:5432/postgres

ACCESS_TOKEN_SECRET=access_token_secret
REFRESH_TOKEN_SECRET=refresh_token_secret

MAIL_FROM=myemail@address.com
SMTP_URL=smtp://same-thing-as-the-web-version

STORAGE_ENDPOINT=10.89.3.15
STORAGE_PORT=9000
STORAGE_REGION=us-east-1 # Optional
STORAGE_BUCKET=default
STORAGE_ACCESS_KEY=minioadmin
STORAGE_SECRET_KEY=minioadmin
STORAGE_USE_SSL=false
STORAGE_SKIP_BUCKET_CHECK=false
```