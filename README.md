# docker-lu project

## Introduction

docker-lu is a small GO program to adapt container files, /etc/passwd & /etc/groups, with docker host local user UID & GID.

## The use case is:

When a user execute a container as himself on his filesystem, the container can creates files.
Those files must not be owned by a user ID which is not himself.

Ex:

```bash
[me@localhost tmp ] ll
total 0
[me@localhost tmp ] docker run -it -v $(pwd):/home/me -w /home/me -u $(id -u):$(id -g) --rm alpine sh -c "echo blabla > test.txt"
[me@localhost tmp ] ll
total 4
-rw-r--r-- 1 me me 7 Jul  5 16:14 test.txt
```

As you can see, it works.

But if you call a more advanced command like `git`, you may receive an error like `fatal: unable to look up current user in the passwd file: no such user`

```bash
[me@localhost tmp ] docker run --rm -e http_proxy -e https_proxy -e no_proxy -it -u 1001:1001 forjdevops/jenkins git clone https://github.com/forj-oss/jenkins-install-inits /tmp/jenkins-install-inits
Cloning into '/tmp/jenkins-install-inits'...
remote: Counting objects: 423, done.
remote: Total 423 (delta 0), reused 0 (delta 0), pack-reused 422
Receiving objects: 100% (423/423), 340.18 KiB | 0 bytes/s, done.
Resolving deltas: 100% (171/171), done.
fatal: unable to look up current user in the passwd file: no such user
Unexpected end of command stream
```

You cannot replace the `-u $(id -u):$(id -g)` by `-u $(id -un):$(id -gn)` or you may get following error: 
`docker: Error response from daemon: linux spec user: unable to find user larsonsh: no matching entries in passwd file.`

Why are we having this issue? Because, your uid or username is not recognized in the container. ie you are not in the container passwd file.

## How to resolve it?

To resolve this, we need to ensure that UID/GID given is listed in `/etc/passwd` and `/etc/group` in the container.

Yoy can fix it easily with a basic bash script to run as root:

```bash
sed -i 's/\(devops:x:\)1000:1000/\1'"$1"':'"$2"'/g' /etc/passwd
sed -i 's/\(devops:x:\)1000/\1'"$2"'/g' /etc/group
```

If `sed` has been installed you can use that code.

but sed is not installed, you have to 
- install it
- create a script with those 2 sed commands, and probably checking parameters...

To be honest, it is not a big deal to do this.

`docker-lu` globally do exactly this.

Why don't we exposed a script to do this?

because:
- you do not need install anything else than docker-lu
- you do not need to create a script to test parameters docker-lu has that
- you limit the risk to break your container if you update wrongly those files
- docker-lu refuses to work outside a container.

## How to use docker-lu

You should use docker-lu if all following conditions are true

- if you add `-u` to `docker run`
- if your container has a local FS mounted (`-v /local/fs:/data`)
- if your container write/update the mounted path with the user rights given (`-u`)
- if the user uid is not recognized in the /etc/passwd of the container

If your use case is confirmed, do the following:

### Use case 1 - Dockerfile

1. Add `ADD https://github.com/forj-oss/docker-lu/releases/0.1/docker-lu /usr/local/bin/docker-lu`
2. Add `RUN chmod +x /usr/local/bin/docker-lu`
3. Add a call to docker-lu in the entrypoint. ex: `ENTRYPOINT [ "/usr/local/bin/entrypoint.sh" ]`

    ... assume jenkins exist as user in the image ...
    
    ```bash
    [...]
    # entrypoint.sh with docker-lu
    /usr/local/bin/docker-lu "jenkins" $UID "jenkins" $GID

    ```
4. Run it

    ```bash
    docker run -it --name test --rm -e UID=$(id -u) -e GID=$(id -g) <image> <tool> <parameters>
    ```

### Use case 2 - docker run as daemon

1. Download docker-lu. 

    `wget -O ~/bin/docker-lu https://github.com/forj-oss/docker-lu/releases/0.1/docker-lu`

2. Give executable rights

    `chmod +x ~/bin/docker-lu`
3. Start your container as docker daemon: 
    
    ex: it will use `developer` as declared user.
    
    `docker run -u $(id -u):$(id -g) --name test -it -d alpine sh -c 'adduser developer -D;cat'`
4. copy docker-lu: `docker copy ~/bin/docker-lu test:/usr/bin/docker-lu`
5. run it : `docker exec -it test docker-lu developer $(id -u) developer $(id -g)`


check it : `docker exec -it test id`

Forj Team