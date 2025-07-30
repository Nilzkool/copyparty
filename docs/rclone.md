# using rclone to mount a remote copyparty server as a local filesystem

speed estimates with server and client on the same win10 machine:
* `1070 MiB/s` with rclone as both server and client
* `570 MiB/s` with rclone-client and `copyparty -ed -j16` as server
* `220 MiB/s` with rclone-client and `copyparty -ed` as server
* `100 MiB/s` with [../bin/partyfuse.py](../bin/partyfuse.py) as client

when server is on another machine (1gbit LAN),
* `75 MiB/s` with [../bin/partyfuse.py](../bin/partyfuse.py) as client
* `92 MiB/s` with rclone-client and `copyparty -ed` as server
* `103 MiB/s` (connection max) with `copyparty -ed -j16` and all the others

# creating the config file

the copyparty "connect" page at `/?hc` (for example http://127.0.0.1:3923/?hc) will generate commands to autoconfigure rclone for your server

**if you prefer to configure rclone manually, continue reading:**

replace `hunter2` with your password, or remove the `pass = hunter2` line if anonymous access is allowed

### on windows clients:

Install [WinFsp](https://github.com/billziss-gh/winfsp/releases/latest) and rclone.

Create (or edit) `%USERPROFILE%\.config\rclone\rclone.conf` with:

```
[cpp-dav]
type = webdav
vendor = owncloud
url = http://127.0.0.1:3923/
pacer_min_sleep = 0.01ms
user = k
pass = hunter2
```

### on unix clients:

```
cat > ~/.config/rclone/rclone.conf <<'EOF'
[cpp-dav]
type = webdav
vendor = owncloud
url = http://127.0.0.1:3923/
pacer_min_sleep = 0.01ms
user = k
pass = hunter2
EOF
```

# mounting the copyparty server locally

Read-write mount:
```
rclone mount --vfs-cache-mode writes --dir-cache-time 5s cpp-dav: /mnt/copyparty
```

On Windows use a drive letter:
```
rclone.exe mount --vfs-cache-mode writes --dir-cache-time 5s cpp-dav: W:
```

**Tips:**
* If your Copyparty server uses HTTPS with a self-signed certificate, add `--no-check-certificate`.
* To allow other users to access the mount on Linux, add `--allow-other`.

# sync folders to/from copyparty

Note that the up2k client [u2c.py](https://github.com/9001/copyparty/tree/hovudstraum/bin#u2cpy) (available on the "connect" page of your copyparty server) does uploads much faster and safer, but rclone is bidirectional and more ubiquitous

```
rclone sync /usr/share/icons/ cpp-dav:icons/
```

# use rclone as server too, replacing copyparty

feels out of place but is too good not to mention

```
rclone.exe serve http --read-only .
rclone.exe serve webdav .
```

# devnotes

copyparty supports and expects [the following](https://github.com/rclone/rclone/blob/46484022b08f8756050aa45505ea0db23e62df8b/backend/webdav/webdav.go#L575-L578) from rclone,

```go
case "owncloud":
    f.canStream = true
    f.precision = time.Second
    f.useOCMtime = true
    f.hasOCMD5 = true
    f.hasOCSHA1 = true
```

notably,
* `useOCMtime` enables the `x-oc-mtime` header to retain mtime of uploads from rclone
* `canStream` is supported but not required by us
* `hasOCMD5` / `hasOCSHA1` is conveniently dontcare on both ends

there's a scary comment mentioning PROPSET of lastmodified which is not something we wish to support

and if `vendor=owncloud` ever stops working, try `vendor=fastmail` instead
