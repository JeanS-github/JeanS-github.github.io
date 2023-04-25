title: "Removarr"
date: 2023-04-25T13:27:56+02:00
summary: "A web app to list and delete files in your download folder but not in your media folder."
aliases: ["/posts/removarr", "/removarr"]
tags: ["removarr", "Flask", "Python", "server", "web", "app"]
author: "Garnajee"
draft: true

# About

The [source code](https://github.com/garnajee/removarr) is available on github.

This is a web application, created to help you manually delete files present in the (e.g. Transmission `completed/`) download folder but not in the (e.g. Jellyfin) `medias/` folder. If you have separated folders it'll also works.

Take a look at the screenshot of the web-app [here](https://zupimages.net/up/23/15/jhb0.png).

## How it works

A recursive comparison according to the inodes of the files is made between the `completed/` (download) folder (e.g. on the Transmission side) and the folders where the media are stored (e.g. on the Jellyfin side).

Example 1:

```
├── completed       # Transmission side
│   ├── folder1
│   ├── ...
│   ├── files
│   └── ...
└── medias          # Jellyfin side
    ├── movies
    │   ├── folders
    │   └── ...
    └── series
        ├── folders
        └── ...
```

Example 2:

```
├── completed       # Transmission side
│   ├── folder1
│   ├── ...
│   ├── files
│   └── ...
├── movies          # Jellyfin side
│   ├── files
│   ├── folders
│   │   └── ...
│   └── ...
└── series          # Jellyfin side
    ├── files
    └── folders
        └── ...
```

All the files present on the side of Jellyfin are the latest versions that we want to keep. So, all we want to remove is everything that is present on the Transmission side but not on the Jellyfin side.

If the deleted file was in a folder:

* if the folder is now empty, then the folder is deleted
* if the folder does not contain other files with extension '.mkv', '.mp4', '.avi', '.mov', then the folder is deleted
* if the folder does not contain subfolder(s), the folder is deleted

### Proof it works

I'm using this app on a Synology NAS.

I have 2x6TB HDD in RAID 1. The filesystem of my main volume (`/volume1`) is in `Btrfs`.

To try if the file was properley deleted and if I recovered the storage, I ran this command before and after the deletion of the file:

```bash
$ btrfs fi usage /volume1/ -g
```

I deleted a file of about **1,9 GB**. Here is the diff before and after the deletion:

```diff
Overall:
     Device allocated:                  2884.02GiB
     Device unallocated:                2693.98GiB
     Device missing:                       0.00GiB
-    Used:                              2843.89GiB
-    Free (estimated):                  2732.34GiB      (min: 1385.35GiB)
+    Used:                              2842.04GiB
+    Free (estimated):                  2734.20GiB      (min: 1387.21GiB)
     Data ratio:                              1.00
     Metadata ratio:                          2.00
     Global reserve:                       0.28GiB      (used: 0.00GiB)

-Data,single: Size:2880.01GiB, Used:2841.64GiB
+Data,single: Size:2880.01GiB, Used:2839.79GiB
    /dev/mapper/cachedev_0      2880.01GiB
```

## How to install

Nothing more simple than to use the [docker-compose](https://github.com/garnajee/removarr/blob/master/docker-compose.yml) file.

* First option: (like the example above) you have **one** folder for all your media, use [this file](https://github.com/garnajee/removarr/blob/master/docker-compose.yml):

```yaml
version: '3.9'
services:
  removarr:
    image: ghcr.io/garnajee/removarr:latest
    container_name: removarr
    restart: always
    environment:
      - PUID=1030
      - PGID=100
    volumes:
      - '/your/path/completed/:/data/completed'
      - '/your/path/medias/:/data/medias'
    ports:
      - '127.0.0.1:8012:5000'
```

* Second option: you have several folders not in the same place, use [this file](https://github.com/garnajee/removarr/blob/master/docker-compose-2.yml):

```yaml
version: '3.9'
services:
  removarr:
    image: ghcr.io/garnajee/removarr:latest
    container_name: removarr
    restart: always
    environment:
      - PUID=1030
      - PGID=100
    volumes:
      - '/your/path/completed/:/data/completed'
      - '/your/path/movies/:/data/movies'
      - '/your/path/series/:/data/series'
    ports:
      - '127.0.0.1:8012:5000'
```

The application will be available at `<you_ip>:8012`.

* User/Group Identifiers

UserID (`PUID`) and GroupID (`PGID`) are optionals. To find these ids just type `id <username>` as below:

```bash
$ id username
    uid=1000(dockeruser) gid=1000(dockergroup) groups=1000(dockergroup)
```

* Volumes

`/tmp/test/completed/`: change this by the path of your download folder

`/tmp/test/medias/`   : change this by the path of your medias folder

* Port

`8012`: change this port to suit your needs.

## Todo

* [x] modify README (setup installation, how to use, how to build, screenshot,...) - needs to be improved
* [ ] get hash of torrent file to delete it (need to mount one more volume)
* [x] delete empty folder (ignore other files different than .mkv/avi/...)
* [x] add total size of all files
* [x] add size of each file in a column
* [ ] create a checkbox to delete multiples files at once
* [ ] create a button to delete all files at once
* [ ] sort table by alphabetic order and file size
* [ ] make the app responsive
