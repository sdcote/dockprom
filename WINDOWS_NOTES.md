# Problems with Volumes
1. Go into the Docker toolkit, under settings.Shared drives and make sure the drive on which the volumes are specified are set to shared. 
    - If they are and it is still does not work, try "Reset Credentials" and reshare the drive entering in your credentials.
1. Make sure there is a `.env` file in the root of the project which set the environment variables to `COMPOSE_CONVERT_WINDOWS_PATHS=1`


# Invalid Volume Specification: '/host_mnt/rootfs:ro'
The compose file is desigend to run on Unix systems. There are at least two locations in the `docker-compose.yml` which refer to the root drive of the Docker host "/" which should be "c:/". You will get errors similar to the following:
```
ERROR: for nodeexporter  Cannot create container for service nodeexporter: invalid volume specification: '/host_mnt/rootfs:ro'
ERROR: for cadvisor  Cannot create container for service cadvisor: invalid volume specification: '/host_mnt/rootfs:ro'
```
Edit the `docker-compose.yml` file and replace
```
volumes:
  - /:/rootfs:ro
```
with
```
volumes:
  - c:\:/rootfs:ro
```


