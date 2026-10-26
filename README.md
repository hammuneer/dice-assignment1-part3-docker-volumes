# Assignment 1 – Part 3: Docker Volumes

**Author:** hammuneer
**GitHub repository:** https://github.com/hammuneer/dice-assignment1-part3-docker-volumes
**Part 1 repository:** https://github.com/hammuneer/dice-assignment1-part1-docker
**Part 2 repository:** https://github.com/hammuneer/dice-assignment1-part2-docker-commands

This repository shows how a named Docker volume keeps data after the containers that use it are
gone. One volume, `my_volume`, is mounted first into an **nginx** container and then into an
**httpd** (Apache) container. Files are copied into it with `docker cp`, served by both web
servers, and still present after both containers have been removed.

For each step, this README gives a short description, the exact command and the real output it
produced. The raw output of every command is also saved in the [`logs/`](logs) folder, and the
two HTML files that were copied into the volume are in [`host-files/`](host-files).

### Summary

| # | Task | Marks | Result |
|---|------|-------|--------|
| 1 | Create the volume `my_volume` | 2 | Created ([Step 1](#step-1--create-the-docker-volume-my_volume)) |
| 2 | Run `nginx` with `my_volume` on `/usr/share/nginx/html` | 4 | Container `nginx-volume` running ([Step 2](#step-2--run-an-nginx-container-with-my_volume-mounted)) |
| 3 | nginx default page on http://localhost:8080 | 2 | "Welcome to nginx!" returned ([Step 3](#step-3--verify-the-nginx-default-page-on-port-8080)) |
| 4 | Create `index.html` on the host | 2 | [`host-files/index.html`](host-files/index.html) ([Step 4](#step-4--create-indexhtml-on-the-host)) |
| 5 | `docker cp` `index.html` into `my_volume` | 4 | Copied ([Step 5](#step-5--copy-indexhtml-into-my_volume-with-docker-cp)) |
| 6 | `index.html` on http://localhost:8080 | 2 | Custom page returned ([Step 6](#step-6--verify-indexhtml-on-port-8080)) |
| 7 | Stop and remove the nginx container | 2 | Removed, volume kept ([Step 7](#step-7--stop-and-remove-the-nginx-container)) |
| 8 | Run `httpd` with `my_volume` on `/usr/local/apache2/htdocs` | 4 | Container `httpd-volume` running ([Step 8](#step-8--run-an-httpd-container-with-my_volume-mounted)) |
| 9 | httpd page on http://localhost:8081 | 2 | Apache responds, serving the volume's `index.html` ([Step 9](#step-9--verify-the-httpd-page-on-port-8081)) |
| 10 | Create `about.html` on the host | 2 | [`host-files/about.html`](host-files/about.html) ([Step 10](#step-10--create-abouthtml-on-the-host)) |
| 11 | `docker cp` `about.html` into `my_volume` | 4 | Copied ([Step 11](#step-11--copy-abouthtml-into-my_volume-with-docker-cp)) |
| 12 | `about.html` on http://localhost:8081/about.html | 2 | Page returned ([Step 12](#step-12--verify-abouthtml-on-port-8081)) |
| 13 | Stop and remove the httpd container | 2 | Removed ([Step 13](#step-13--stop-and-remove-the-httpd-container)) |
| 14 | Both files still in `my_volume` | 4 | Both present ([Step 14](#step-14--verify-both-files-are-still-in-my_volume)) |
| 15 | Cleanup: remove `my_volume` | 2 | Removed ([Step 15](#step-15--cleanup-remove-my_volume)) |
| 16 | Document in README.md | 14 | This document plus raw logs in [`logs/`](logs) |
| 17 | Push to a new GitHub repository | – | Pushed to `main` ([Push to GitHub](#push-the-codebase-to-github)) |

### Environment

| Tool | Version |
|------|---------|
| Docker Engine / CLI | 29.2.1 (Docker Desktop on macOS) |
| nginx image | `nginx:latest` (nginx/1.31.8) |
| httpd image | `httpd:latest` (Apache/2.4.69) |
| curl | 8.7.1 |
| Git | 2.50.1 |
| GitHub CLI (`gh`) | 2.101.0 |

---

## Table of Contents

1. [Key concepts](#key-concepts)
2. [Preparation – pull the images](#preparation--pull-the-images)
3. [Step 1 – Create the Docker volume `my_volume`](#step-1--create-the-docker-volume-my_volume)
4. [Step 2 – Run an nginx container with `my_volume` mounted](#step-2--run-an-nginx-container-with-my_volume-mounted)
5. [Step 3 – Verify the nginx default page on port 8080](#step-3--verify-the-nginx-default-page-on-port-8080)
6. [Step 4 – Create `index.html` on the host](#step-4--create-indexhtml-on-the-host)
7. [Step 5 – Copy `index.html` into `my_volume` with `docker cp`](#step-5--copy-indexhtml-into-my_volume-with-docker-cp)
8. [Step 6 – Verify `index.html` on port 8080](#step-6--verify-indexhtml-on-port-8080)
9. [Step 7 – Stop and remove the nginx container](#step-7--stop-and-remove-the-nginx-container)
10. [Step 8 – Run an httpd container with `my_volume` mounted](#step-8--run-an-httpd-container-with-my_volume-mounted)
11. [Step 9 – Verify the httpd page on port 8081](#step-9--verify-the-httpd-page-on-port-8081)
12. [Step 10 – Create `about.html` on the host](#step-10--create-abouthtml-on-the-host)
13. [Step 11 – Copy `about.html` into `my_volume` with `docker cp`](#step-11--copy-abouthtml-into-my_volume-with-docker-cp)
14. [Step 12 – Verify `about.html` on port 8081](#step-12--verify-abouthtml-on-port-8081)
15. [Step 13 – Stop and remove the httpd container](#step-13--stop-and-remove-the-httpd-container)
16. [Step 14 – Verify both files are still in `my_volume`](#step-14--verify-both-files-are-still-in-my_volume)
17. [Step 15 – Cleanup: remove `my_volume`](#step-15--cleanup-remove-my_volume)
18. [Findings and observations](#findings-and-observations)
19. [Push the codebase to GitHub](#push-the-codebase-to-github)
20. [Repository structure](#repository-structure)
21. [Command reference](#command-reference)

---

## Key concepts

- **Container filesystem:** each container gets a writable layer on top of its image. That layer
  is deleted when the container is removed, so anything written there is lost.
- **Named volume:** storage managed by Docker that lives outside any container's writable layer.
  It is created with `docker volume create`, mounted with `-v <volume>:<path>`, and kept until it
  is removed with `docker volume rm`. Several containers can use the same volume, one after
  another or at the same time.
- **Mount point:** the path inside the container where the volume appears. Here it is the web
  root of each server, `/usr/share/nginx/html` for nginx and `/usr/local/apache2/htdocs` for
  httpd, so whatever is in the volume is what the server publishes.

---

## Preparation – pull the images

**Description:** `docker pull` downloads an image from Docker Hub into the local image store. The
official `nginx` and `httpd` images were pulled before they were used so that the `docker run`
output below shows only the container being created.

```bash
docker pull nginx:latest
docker pull httpd:latest
docker images --format 'table {{.Repository}}\t{{.Tag}}\t{{.ID}}\t{{.Size}}' | grep -E 'REPOSITORY|^nginx|^httpd'
```

**Output** (layer progress lines shortened, full output in [`logs/00-pull-images.log`](logs/00-pull-images.log)):

```text
$ docker pull nginx:latest
latest: Pulling from library/nginx
...
Digest: sha256:4c41e2d0a3a55b325ed572cd0200bdb72af1433c3daa2bca6ef8eea68ce504a2
Status: Downloaded newer image for nginx:latest
docker.io/library/nginx:latest

$ docker pull httpd:latest
latest: Pulling from library/httpd
...
Digest: sha256:a30cd4cd57f3f3c3be9b2d16384d2488839996bb14c67df2b51f1ecbb3e584a1
Status: Downloaded newer image for httpd:latest
docker.io/library/httpd:latest

$ docker images ... | grep -E 'REPOSITORY|^nginx|^httpd'
REPOSITORY   TAG       IMAGE ID       SIZE
nginx        latest    4c41e2d0a3a5   243MB
httpd        latest    a30cd4cd57f3   177MB
```

---

## Step 1 – Create the Docker volume `my_volume`

**Description:** `docker volume create` creates a new named volume. `docker volume ls` lists
volumes (filtered here to `my_volume`), and `docker volume inspect` shows its details: the
`local` driver and the directory inside the Docker VM where the data is stored (`Mountpoint`).

```bash
docker volume create my_volume
docker volume ls --filter name=my_volume
docker volume inspect my_volume
```

**Output** ([`logs/01-volume-create.log`](logs/01-volume-create.log)):

```text
$ docker volume create my_volume
my_volume

$ docker volume ls --filter name=my_volume
DRIVER    VOLUME NAME
local     my_volume

$ docker volume inspect my_volume
[
    {
        "CreatedAt": "2026-10-26T22:25:12Z",
        "Driver": "local",
        "Labels": null,
        "Mountpoint": "/var/lib/docker/volumes/my_volume/_data",
        "Name": "my_volume",
        "Options": null,
        "Scope": "local"
    }
]
```

---

## Step 2 – Run an nginx container with `my_volume` mounted

**Description:** `docker run` creates and starts a container from the `nginx` image.

- `-d` runs it in the background (detached).
- `--name nginx-volume` gives the container a fixed name.
- `-p 8080:80` publishes the container's port 80 on port 8080 of the host.
- `-v my_volume:/usr/share/nginx/html` mounts the volume on nginx's web root.

`docker ps` confirms the container is up, `docker inspect` shows the mount, and `docker exec ... ls`
lists the files in the web root.

```bash
docker run -d --name nginx-volume -p 8080:80 -v my_volume:/usr/share/nginx/html nginx:latest
docker ps --filter name=nginx-volume
docker inspect --format '{{json .Mounts}}' nginx-volume | python3 -m json.tool
docker exec nginx-volume ls -l /usr/share/nginx/html
```

**Output** ([`logs/02-nginx-run.log`](logs/02-nginx-run.log)):

```text
$ docker run -d --name nginx-volume -p 8080:80 -v my_volume:/usr/share/nginx/html nginx:latest
1c680e1ea633aa05b842ad16ad4ff3ba979bbf69d8cf685843bc180d84a059c8

$ docker ps --filter name=nginx-volume
CONTAINER ID   IMAGE          COMMAND                  CREATED         STATUS         PORTS                                     NAMES
1c680e1ea633   nginx:latest   "/docker-entrypoint.…"   2 seconds ago   Up 2 seconds   0.0.0.0:8080->80/tcp, [::]:8080->80/tcp   nginx-volume

$ docker inspect --format '{{json .Mounts}}' nginx-volume | python3 -m json.tool
[
    {
        "Type": "volume",
        "Name": "my_volume",
        "Source": "/var/lib/docker/volumes/my_volume/_data",
        "Destination": "/usr/share/nginx/html",
        "Driver": "local",
        "Mode": "z",
        "RW": true,
        "Propagation": ""
    }
]

$ docker exec nginx-volume ls -l /usr/share/nginx/html
total 8
-rw-r--r-- 1 root root 497 Oct 21 14:24 50x.html
-rw-r--r-- 1 root root 896 Oct 21 14:24 index.html
```

> **Finding:** `my_volume` was empty, yet the web root already contains `index.html` and
> `50x.html`. When an **empty** named volume is mounted on a directory that has files in the
> image, Docker copies those files into the volume first. The nginx default page is now stored in
> `my_volume`.

---

## Step 3 – Verify the nginx default page on port 8080

**Description:** `curl` sends an HTTP request to the published port on the host. `-i` includes the
response headers, which show that nginx answered.

```bash
curl -s -i http://localhost:8080 | head -12
curl -s http://localhost:8080 | grep -E '<title>|<h1>'
```

**Output** ([`logs/03-verify-nginx-default.log`](logs/03-verify-nginx-default.log)):

```text
$ curl -s -i http://localhost:8080 | head -12
HTTP/1.1 200 OK
Server: nginx/1.31.8
Date: Mon, 26 Oct 2026 22:25:20 GMT
Content-Type: text/html
Content-Length: 896
Last-Modified: Wed, 21 Oct 2026 14:24:51 GMT
Connection: keep-alive
ETag: "6ad8cb33-380"
Accept-Ranges: bytes

<!DOCTYPE html>
<html>

$ curl -s http://localhost:8080 | grep -E '<title>|<h1>'
<title>Welcome to nginx!</title>
<h1>Welcome to nginx!</h1>
```

The nginx default page ("Welcome to nginx!", 896 bytes) is reachable at http://localhost:8080.

---

## Step 4 – Create `index.html` on the host

**Description:** A small HTML page was written on the host machine at
[`host-files/index.html`](host-files/index.html). `ls -l` and `cat` show the file and its contents.

```bash
ls -l host-files/index.html
cat host-files/index.html
```

**Output** ([`logs/04-create-index-html.log`](logs/04-create-index-html.log)):

```text
$ ls -l host-files/index.html
-rw-r--r--@ 1 apple  staff  373 Oct 27 03:24 host-files/index.html

$ cat host-files/index.html
<!doctype html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <title>my_volume – index.html</title>
  </head>
  <body>
    <h1>Hello from my_volume!</h1>
    <p>This index.html was created on the host and copied into the Docker volume "my_volume" with docker cp.</p>
    <p>It is served by the nginx container on http://localhost:8080.</p>
  </body>
</html>
```

---

## Step 5 – Copy `index.html` into `my_volume` with `docker cp`

**Description:** `docker cp <host path> <container>:<path>` copies a file from the host into a
container. `docker cp` cannot target a volume directly, so the file is copied to the volume's
mount point inside a container that uses it (`nginx-volume:/usr/share/nginx/html`). Because that
path is the volume, the file is written to `my_volume` and not to the container's own layer. It
replaces the default `index.html`. `docker cp` prints nothing when it succeeds.

```bash
docker cp host-files/index.html nginx-volume:/usr/share/nginx/html/index.html
docker exec nginx-volume ls -l /usr/share/nginx/html
```

**Output** ([`logs/05-docker-cp-index.log`](logs/05-docker-cp-index.log)):

```text
$ docker cp host-files/index.html nginx-volume:/usr/share/nginx/html/index.html
[exit code: 0]

$ docker exec nginx-volume ls -l /usr/share/nginx/html
total 8
-rw-r--r-- 1 root root    497 Oct 21 14:24 50x.html
-rw-r--r-- 1  501 dialout 373 Oct 26 22:24 index.html
```

`index.html` is now 373 bytes, the size of the host file. It belongs to UID `501` (the macOS user)
because `docker cp` keeps the owner of the source file. GID 20 (`staff` on macOS) has the name
`dialout` inside the Debian-based image.

---

## Step 6 – Verify `index.html` on port 8080

**Description:** The same URL is requested again. nginx now serves the file from the volume.
`docker logs` shows nginx's access log for the requests.

```bash
curl -s -i http://localhost:8080
docker logs nginx-volume 2>&1 | tail -4
```

**Output** ([`logs/06-verify-index-html.log`](logs/06-verify-index-html.log)):

```text
$ curl -s -i http://localhost:8080
HTTP/1.1 200 OK
Server: nginx/1.31.8
Date: Mon, 26 Oct 2026 22:25:21 GMT
Content-Type: text/html
Content-Length: 373
Last-Modified: Mon, 26 Oct 2026 22:24:46 GMT
Connection: keep-alive
ETag: "6adfd32e-175"
Accept-Ranges: bytes

<!doctype html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <title>my_volume – index.html</title>
  </head>
  <body>
    <h1>Hello from my_volume!</h1>
    <p>This index.html was created on the host and copied into the Docker volume "my_volume" with docker cp.</p>
    <p>It is served by the nginx container on http://localhost:8080.</p>
  </body>
</html>

$ docker logs nginx-volume 2>&1 | tail -4
2026/10/26 22:25:12 [notice] 1#1: start worker process 44
172.29.0.1 - - [26/Oct/2026:22:25:20 +0000] "GET / HTTP/1.1" 200 896 "-" "curl/8.7.1" "-"
172.29.0.1 - - [26/Oct/2026:22:25:20 +0000] "GET / HTTP/1.1" 200 896 "-" "curl/8.7.1" "-"
172.29.0.1 - - [26/Oct/2026:22:25:21 +0000] "GET / HTTP/1.1" 200 373 "-" "curl/8.7.1" "-"
```

The access log shows the change: the first two requests returned the 896-byte default page, and
the request after `docker cp` returned the 373-byte custom page. nginx did not need a restart.

---

## Step 7 – Stop and remove the nginx container

**Description:** `docker stop` sends `SIGTERM` to the container's main process and waits for it to
exit. `docker rm` deletes the stopped container and its writable layer. `docker ps -a` confirms the
container is gone, and `docker volume ls` confirms that `my_volume` is still there.

```bash
docker stop nginx-volume
docker rm nginx-volume
docker ps -a --filter name=nginx-volume
docker volume ls --filter name=my_volume
```

**Output** ([`logs/07-nginx-stop-rm.log`](logs/07-nginx-stop-rm.log)):

```text
$ docker stop nginx-volume
nginx-volume

$ docker rm nginx-volume
nginx-volume

$ docker ps -a --filter name=nginx-volume
CONTAINER ID   IMAGE     COMMAND   CREATED   STATUS    PORTS     NAMES

$ docker volume ls --filter name=my_volume
DRIVER    VOLUME NAME
local     my_volume
```

`docker rm` does not delete named volumes. Only `docker volume rm` (or `docker rm -v` for
anonymous volumes) does.

---

## Step 8 – Run an httpd container with `my_volume` mounted

**Description:** A new container is created from the official Apache `httpd` image. Port 80 is
published on host port 8081, and the same volume is mounted on Apache's web root,
`/usr/local/apache2/htdocs`.

```bash
docker run -d --name httpd-volume -p 8081:80 -v my_volume:/usr/local/apache2/htdocs httpd:latest
docker ps --filter name=httpd-volume
docker inspect --format '{{json .Mounts}}' httpd-volume | python3 -m json.tool
docker exec httpd-volume ls -l /usr/local/apache2/htdocs
```

**Output** ([`logs/08-httpd-run.log`](logs/08-httpd-run.log)):

```text
$ docker run -d --name httpd-volume -p 8081:80 -v my_volume:/usr/local/apache2/htdocs httpd:latest
1455d844d8d9e82764e319ba3ec363c76a9183a238917654b2c23f9d9c1c43e4

$ docker ps --filter name=httpd-volume
CONTAINER ID   IMAGE          COMMAND              CREATED         STATUS         PORTS                                     NAMES
1455d844d8d9   httpd:latest   "httpd-foreground"   2 seconds ago   Up 2 seconds   0.0.0.0:8081->80/tcp, [::]:8081->80/tcp   httpd-volume

$ docker inspect --format '{{json .Mounts}}' httpd-volume | python3 -m json.tool
[
    {
        "Type": "volume",
        "Name": "my_volume",
        "Source": "/var/lib/docker/volumes/my_volume/_data",
        "Destination": "/usr/local/apache2/htdocs",
        "Driver": "local",
        "Mode": "z",
        "RW": true,
        "Propagation": ""
    }
]

$ docker exec httpd-volume ls -l /usr/local/apache2/htdocs
total 8
-rw-r--r-- 1 root root    497 Oct 21 14:24 50x.html
-rw-r--r-- 1  501 dialout 373 Oct 26 22:24 index.html
```

> **Finding:** Apache's web root contains the files left by the nginx container: nginx's
> `50x.html` and our custom `index.html`. Apache's own default `index.html` is not there. The volume
> was **not empty** this time, so Docker did not copy the image's files into it, and the volume's
> contents hide whatever the image has at that path.

---

## Step 9 – Verify the httpd page on port 8081

**Description:** `curl` requests the root of port 8081. The second command starts a throwaway
`httpd` container **without** the volume (`--rm` deletes it on exit) and prints the image's own
default `index.html` for comparison.

```bash
curl -s -i http://localhost:8081
docker run --rm httpd:latest cat /usr/local/apache2/htdocs/index.html
```

**Output** ([`logs/09-verify-httpd-default.log`](logs/09-verify-httpd-default.log)):

```text
$ curl -s -i http://localhost:8081
HTTP/1.1 200 OK
Date: Mon, 26 Oct 2026 22:25:31 GMT
Server: Apache/2.4.69 (Unix)
Last-Modified: Mon, 26 Oct 2026 22:24:46 GMT
ETag: "175-65ec5cc18a780"
Accept-Ranges: bytes
Content-Length: 373
Content-Type: text/html

<!doctype html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <title>my_volume – index.html</title>
  </head>
  <body>
    <h1>Hello from my_volume!</h1>
    <p>This index.html was created on the host and copied into the Docker volume "my_volume" with docker cp.</p>
    <p>It is served by the nginx container on http://localhost:8080.</p>
  </body>
</html>

$ docker run --rm httpd:latest cat /usr/local/apache2/htdocs/index.html
<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 4.01//EN" "http://www.w3.org/TR/html4/strict.dtd">
<html>
<head>
<title>It works! Apache httpd</title>
</head>
<body>
<p>It works!</p>
</body>
</html>
```

The httpd server is reachable at http://localhost:8081. The `Server: Apache/2.4.69 (Unix)` header
confirms that Apache, not nginx, answered. Because the mounted volume already held an
`index.html`, Apache serves that file (the page created in Step 4) as its default page instead of
the image's "It works!" page shown by the second command. This shows that the data written while
the nginx container was running survived its removal and is now used by a different container
running a different web server.

---

## Step 10 – Create `about.html` on the host

**Description:** A second HTML page was written on the host at
[`host-files/about.html`](host-files/about.html).

```bash
ls -l host-files/about.html
cat host-files/about.html
```

**Output** ([`logs/10-create-about-html.log`](logs/10-create-about-html.log)):

```text
$ ls -l host-files/about.html
-rw-r--r--@ 1 apple  staff  387 Oct 27 03:24 host-files/about.html

$ cat host-files/about.html
<!doctype html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <title>my_volume – about.html</title>
  </head>
  <body>
    <h1>About this page</h1>
    <p>This about.html was created on the host and copied into the Docker volume "my_volume" with docker cp.</p>
    <p>It is served by the httpd (Apache) container on http://localhost:8081/about.html.</p>
  </body>
</html>
```

---

## Step 11 – Copy `about.html` into `my_volume` with `docker cp`

**Description:** As in Step 5, `docker cp` copies the host file to the volume's mount point, this
time inside the httpd container.

```bash
docker cp host-files/about.html httpd-volume:/usr/local/apache2/htdocs/about.html
docker exec httpd-volume ls -l /usr/local/apache2/htdocs
```

**Output** ([`logs/11-docker-cp-about.log`](logs/11-docker-cp-about.log)):

```text
$ docker cp host-files/about.html httpd-volume:/usr/local/apache2/htdocs/about.html
[exit code: 0]

$ docker exec httpd-volume ls -l /usr/local/apache2/htdocs
total 12
-rw-r--r-- 1 root root    497 Oct 21 14:24 50x.html
-rw-r--r-- 1  501 dialout 387 Oct 26 22:24 about.html
-rw-r--r-- 1  501 dialout 373 Oct 26 22:24 index.html
```

---

## Step 12 – Verify `about.html` on port 8081

**Description:** `curl` requests `/about.html` from Apache. `docker logs` shows Apache's access log.

```bash
curl -s -i http://localhost:8081/about.html
docker logs httpd-volume 2>&1 | tail -3
```

**Output** ([`logs/12-verify-about-html.log`](logs/12-verify-about-html.log)):

```text
$ curl -s -i http://localhost:8081/about.html
HTTP/1.1 200 OK
Date: Mon, 26 Oct 2026 22:25:39 GMT
Server: Apache/2.4.69 (Unix)
Last-Modified: Mon, 26 Oct 2026 22:24:46 GMT
ETag: "183-65ec5cc18a780"
Accept-Ranges: bytes
Content-Length: 387
Content-Type: text/html

<!doctype html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <title>my_volume – about.html</title>
  </head>
  <body>
    <h1>About this page</h1>
    <p>This about.html was created on the host and copied into the Docker volume "my_volume" with docker cp.</p>
    <p>It is served by the httpd (Apache) container on http://localhost:8081/about.html.</p>
  </body>
</html>

$ docker logs httpd-volume 2>&1 | tail -3
[Mon Oct 26 22:25:28.846517 2026] [core:notice] [pid 1:tid 1] AH00094: Command line: 'httpd -D FOREGROUND'
172.29.0.1 - - [26/Oct/2026:22:25:31 +0000] "GET / HTTP/1.1" 200 373
172.29.0.1 - - [26/Oct/2026:22:25:39 +0000] "GET /about.html HTTP/1.1" 200 387
```

---

## Step 13 – Stop and remove the httpd container

**Description:** Same as Step 7, for the httpd container.

```bash
docker stop httpd-volume
docker rm httpd-volume
docker ps -a --filter name=httpd-volume
```

**Output** ([`logs/13-httpd-stop-rm.log`](logs/13-httpd-stop-rm.log)):

```text
$ docker stop httpd-volume
httpd-volume

$ docker rm httpd-volume
httpd-volume

$ docker ps -a --filter name=httpd-volume
CONTAINER ID   IMAGE     COMMAND   CREATED   STATUS    PORTS     NAMES
```

---

## Step 14 – Verify both files are still in `my_volume`

**Description:** No container uses `my_volume` at this point. On Docker Desktop for macOS the
volume's `Mountpoint` is inside Docker's Linux VM, not on the Mac's filesystem, so the files are
read by mounting the volume into a temporary `alpine` container (`--rm` deletes the container as
soon as the command finishes). `ls -l` lists the files and `cat` prints both pages.

```bash
docker volume ls --filter name=my_volume
docker volume inspect my_volume
docker run --rm -v my_volume:/data alpine:latest ls -l /data
docker run --rm -v my_volume:/data alpine:latest sh -c 'for f in /data/index.html /data/about.html; do echo "===== $f ====="; cat $f; done'
```

**Output** ([`logs/14-verify-volume-contents.log`](logs/14-verify-volume-contents.log)):

```text
$ docker volume ls --filter name=my_volume
DRIVER    VOLUME NAME
local     my_volume

$ docker volume inspect my_volume
[
    {
        "CreatedAt": "2026-10-26T22:25:12Z",
        "Driver": "local",
        "Labels": null,
        "Mountpoint": "/var/lib/docker/volumes/my_volume/_data",
        "Name": "my_volume",
        "Options": null,
        "Scope": "local"
    }
]

$ docker run --rm -v my_volume:/data alpine:latest ls -l /data
Unable to find image 'alpine:latest' locally
latest: Pulling from library/alpine
3da1513d2124: Download complete
3ec36f3670df: Download complete
Digest: sha256:be2656061b536c1883c4034790e21d0a09c5aaaf51968b89368a3c0371ea2c9a
Status: Downloaded newer image for alpine:latest
total 12
-rw-r--r--    1 root     root           497 Oct 21 14:24 50x.html
-rw-r--r--    1 501      dialout        387 Oct 26 22:24 about.html
-rw-r--r--    1 501      dialout        373 Oct 26 22:24 index.html

$ docker run --rm -v my_volume:/data alpine:latest sh -c 'for f in /data/index.html /data/about.html; do echo "===== $f ====="; cat $f; done'
===== /data/index.html =====
<!doctype html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <title>my_volume – index.html</title>
  </head>
  <body>
    <h1>Hello from my_volume!</h1>
    <p>This index.html was created on the host and copied into the Docker volume "my_volume" with docker cp.</p>
    <p>It is served by the nginx container on http://localhost:8080.</p>
  </body>
</html>
===== /data/about.html =====
<!doctype html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <title>my_volume – about.html</title>
  </head>
  <body>
    <h1>About this page</h1>
    <p>This about.html was created on the host and copied into the Docker volume "my_volume" with docker cp.</p>
    <p>It is served by the httpd (Apache) container on http://localhost:8081/about.html.</p>
  </body>
</html>
```

Both `index.html` (copied through the nginx container) and `about.html` (copied through the httpd
container) are still in `my_volume`, unchanged, after both containers were removed.

---

## Step 15 – Cleanup: remove `my_volume`

**Description:** `docker volume rm` permanently deletes a volume and its data. It fails if any
container, running or stopped, still uses the volume; here none does. The last two commands
confirm the volume no longer exists, so `docker volume inspect` returns an error, as expected.

```bash
docker volume rm my_volume
docker volume ls --filter name=my_volume
docker volume inspect my_volume
```

**Output** ([`logs/15-volume-rm.log`](logs/15-volume-rm.log)):

```text
$ docker volume rm my_volume
my_volume

$ docker volume ls --filter name=my_volume
DRIVER    VOLUME NAME

$ docker volume inspect my_volume
[]
Error response from daemon: get my_volume: no such volume
[exit code: 1]
```

---

## Findings and observations

1. **Volumes outlive containers.** Removing `nginx-volume` and then `httpd-volume` deleted their
   writable layers, but everything in `my_volume` was kept. Only `docker volume rm` deleted it.
2. **One volume, different images.** The same data was served by nginx (on port 8080) and later
   by Apache httpd (on port 8081), even though the two images keep their web roots at different
   paths. The mount point is set per container, so a volume is not tied to any image.
3. **Empty volumes are pre-populated from the image.** When the empty `my_volume` was first
   mounted on `/usr/share/nginx/html`, Docker copied nginx's `index.html` and `50x.html` into it.
   This happens only for named volumes that are empty; bind mounts (`-v /host/path:/container/path`)
   never do this.
4. **Non-empty volumes hide the image's files.** When `my_volume` was mounted into httpd it
   already held files, so Apache's own "It works!" page was hidden and the custom `index.html`
   was served on http://localhost:8081 instead. The leftover `50x.html` from nginx also came along.
   To see httpd's real default page with a fresh volume, it would need a new, empty volume.
5. **`docker cp` writes through the mount.** `docker cp` only knows about containers, but copying
   to a path that is a volume mount puts the file in the volume. The web servers served the new
   files at once, with no restart.
6. **`docker cp` keeps file ownership.** The copied files are owned by UID 501 / GID 20, the macOS
   user and group, not by `root`. The files are world-readable, so both servers could serve them.
7. **Volume data is not on the macOS filesystem.** With Docker Desktop, `Mountpoint`
   (`/var/lib/docker/volumes/my_volume/_data`) is inside Docker's Linux VM. The reliable way to
   inspect a volume's files is to mount it into a temporary container, as in Step 14.

---

## Push the codebase to GitHub

**Description:** `git init` creates the repository, `git add` and `git commit` record the files,
`gh repo create` creates the public repository on GitHub and adds it as the `origin` remote, and
`git push` uploads the `main` branch.

```bash
git init -b main
git add .
git commit -m "Assignment 1 Part 3: Docker volumes"
gh repo create hammuneer/dice-assignment1-part3-docker-volumes --public \
  --description "DICE Assignment 1 Part 3: Docker volumes" --source . --remote origin
git push -u origin main
gh repo view hammuneer/dice-assignment1-part3-docker-volumes --json name,visibility,url,defaultBranchRef,description
```

The `origin` URL was then switched to the `github.com-personal` SSH host alias used on this
machine for the personal GitHub account:

```bash
git remote set-url origin git@github.com-personal:hammuneer/dice-assignment1-part3-docker-volumes.git
```

**Output** (commit file list shortened, full output in
[`logs/16-git-commit.log`](logs/16-git-commit.log),
[`logs/17-gh-repo-create.log`](logs/17-gh-repo-create.log),
[`logs/18-git-push.log`](logs/18-git-push.log) and
[`logs/19-gh-repo-view.log`](logs/19-gh-repo-view.log)):

```text
$ git commit -m 'Assignment 1 Part 3: Docker volumes'
[main (root-commit) c126fb0] Assignment 1 Part 3: Docker volumes
 20 files changed, 1284 insertions(+)
 create mode 100644 .gitignore
 create mode 100644 README.md
 create mode 100644 host-files/about.html
 create mode 100644 host-files/index.html
 create mode 100644 logs/00-pull-images.log
 ...
 create mode 100644 logs/15-volume-rm.log

$ gh repo create hammuneer/dice-assignment1-part3-docker-volumes --public --description "DICE Assignment 1 Part 3: Docker volumes" --source . --remote origin
https://github.com/hammuneer/dice-assignment1-part3-docker-volumes

$ git push -u origin main
To github.com-personal:hammuneer/dice-assignment1-part3-docker-volumes.git
 * [new branch]      main -> main
branch 'main' set up to track 'origin/main'.

$ gh repo view hammuneer/dice-assignment1-part3-docker-volumes --json name,visibility,url,defaultBranchRef,description
{"defaultBranchRef":{"name":"main"},"description":"DICE Assignment 1 Part 3: Docker volumes","name":"dice-assignment1-part3-docker-volumes","url":"https://github.com/hammuneer/dice-assignment1-part3-docker-volumes","visibility":"PUBLIC"}

$ git ls-remote origin
db29560874f075977a427bba67fa8ca533d219b3	HEAD
db29560874f075977a427bba67fa8ca533d219b3	refs/heads/main

$ gh api repos/hammuneer/dice-assignment1-part3-docker-volumes/contents --jq ".[].name"
.gitignore
README.md
host-files
logs
```

The repository is public, its default branch is `main`, and the commit on GitHub (`db29560`)
matches the local commit. (The commit was amended to add a co-author line after
`16-git-commit.log` was recorded, so its hash changed from `c126fb0` to `db29560`.)

---

## Repository structure

```text
.
├── README.md                  # This document
├── host-files/
│   ├── index.html             # Created on the host, copied into my_volume via the nginx container
│   └── about.html             # Created on the host, copied into my_volume via the httpd container
└── logs/                      # Raw output of every command, in the order it was run
    ├── 00-pull-images.log
    ├── 01-volume-create.log
    ├── ...
    └── 15-volume-rm.log
```

---

## Command reference

| Command | Purpose | Used in |
|---------|---------|---------|
| `docker pull <image>` | Download an image from Docker Hub | [Preparation](#preparation--pull-the-images) |
| `docker volume create <name>` | Create a named volume | [Step 1](#step-1--create-the-docker-volume-my_volume) |
| `docker volume ls` | List volumes | Steps [1](#step-1--create-the-docker-volume-my_volume), [7](#step-7--stop-and-remove-the-nginx-container), [14](#step-14--verify-both-files-are-still-in-my_volume), [15](#step-15--cleanup-remove-my_volume) |
| `docker volume inspect <name>` | Show a volume's driver, mount point and creation time | Steps [1](#step-1--create-the-docker-volume-my_volume), [14](#step-14--verify-both-files-are-still-in-my_volume), [15](#step-15--cleanup-remove-my_volume) |
| `docker run -d -p H:C -v vol:/path <image>` | Start a container with a published port and a mounted volume | Steps [2](#step-2--run-an-nginx-container-with-my_volume-mounted), [8](#step-8--run-an-httpd-container-with-my_volume-mounted) |
| `docker run --rm -v vol:/data alpine ...` | Run a throwaway container to read a volume | [Step 14](#step-14--verify-both-files-are-still-in-my_volume) |
| `docker ps` | List containers | Steps [2](#step-2--run-an-nginx-container-with-my_volume-mounted), [7](#step-7--stop-and-remove-the-nginx-container), [8](#step-8--run-an-httpd-container-with-my_volume-mounted), [13](#step-13--stop-and-remove-the-httpd-container) |
| `docker inspect --format '{{json .Mounts}}'` | Show a container's mounts | Steps [2](#step-2--run-an-nginx-container-with-my_volume-mounted), [8](#step-8--run-an-httpd-container-with-my_volume-mounted) |
| `docker exec <container> ls -l <path>` | List files inside a running container | Steps [2](#step-2--run-an-nginx-container-with-my_volume-mounted), [5](#step-5--copy-indexhtml-into-my_volume-with-docker-cp), [8](#step-8--run-an-httpd-container-with-my_volume-mounted), [11](#step-11--copy-abouthtml-into-my_volume-with-docker-cp) |
| `docker cp <host file> <container>:<path>` | Copy a file from the host into a container (here, into the volume) | Steps [5](#step-5--copy-indexhtml-into-my_volume-with-docker-cp), [11](#step-11--copy-abouthtml-into-my_volume-with-docker-cp) |
| `curl -s -i <url>` | Request a page from the host and show headers | Steps [3](#step-3--verify-the-nginx-default-page-on-port-8080), [6](#step-6--verify-indexhtml-on-port-8080), [9](#step-9--verify-the-httpd-page-on-port-8081), [12](#step-12--verify-abouthtml-on-port-8081) |
| `docker logs <container>` | Show a container's output (the web server access logs) | Steps [6](#step-6--verify-indexhtml-on-port-8080), [12](#step-12--verify-abouthtml-on-port-8081) |
| `docker stop <container>` | Stop a running container | Steps [7](#step-7--stop-and-remove-the-nginx-container), [13](#step-13--stop-and-remove-the-httpd-container) |
| `docker rm <container>` | Remove a stopped container | Steps [7](#step-7--stop-and-remove-the-nginx-container), [13](#step-13--stop-and-remove-the-httpd-container) |
| `docker volume rm <name>` | Delete a volume and its data | [Step 15](#step-15--cleanup-remove-my_volume) |
