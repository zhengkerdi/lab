Jellyfin server for media streaming.

# Environment Variables
## PUID, PGID
`UserID`: the unique identifier of a user on this machine

`GroupID`: the unique identifier for a group on this machine

We will start a Docker container for Jellyfin. Think of a Docker container as a VM of sorts, running on this `host` machine. On the `host` machine, you log in with some user account, which has a `UserID` and `GroupID`. Call these `host.UID` and `host.GID`.

Processes inside the Docker container will run as some user. Since the container is a "VM", processes inside here run as some user, which may be assigned its own `UserID` and `GroupID`. Call these `Docker.UID` and `Docker.GID`.

The Jellyfin container may need to touch files on the `host` machine. When it does this, it will write files to the `host` machine owned by `Docker.UID` and `Docker.GID`. However, if these do not match with `host.UID` and `host.GID`, the user on `host` will need elevated permissions to access these files. That is undesirable, so to prevent this we configure `Docker.UID = host.UID` and `Docker.GID = host.GID`.

Find UserID with `id -u`, and GroupID with `id -g`. Set these variables in `.env` as `PUID` and `PGID`.

## TZ
The timezone, for Jellyfin's internal use. I set this to `America/Vancouver`.

## MEDIA_PATH
The absolute path on this host where media files are stored.

To add additional drives later, mount the new drive on `lab/Jellyfin/Media/Drive<x>`. Then add the relevant new configurations on the Jellyfin admin UI. Since I haven't done this yet, this section is left as an exercise for the reader.

## RENDER_GID, VIDEO_GID
To access the hardware-acceleration for the Jellyfin server, the Docker container processes need to be given permission. We do this by adding the Docker processes to the relevant groups.

To find the group GIDs on a particular machine, run the command `getent group render video`. This will print `<group_name>:x:<group_GID>:` Set the variables' values accordingly.