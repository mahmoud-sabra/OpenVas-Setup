# OpenVas-Setup

## Install dependencies

There are a few dependencies required for the following steps like `curl`, which is required for downloading files.

```
sudo apt install curl docker-ce docker-ce-cli containerd.io docker-compose-plugin
``` 
To allow the current user to run Docker and therefore start the containers, they must be added to the Docker user group.

```
sudo usermod -aG docker $USER && su $USER
```

# Downloading the Greenbone Community Edition docker compose file requires creating a destination directory.
```
export DOWNLOAD_DIR=$HOME/greenbone-community-container && mkdir -p $DOWNLOAD_DIR
```
# Download the file.
```
cd $DOWNLOAD_DIR && curl -f -L https://greenbone.github.io/docs/latest/_static/docker-compose-22.4.yml -o docker-compose.yml
```
# Expose the gvmd Unix socket for GMP access from the docker host, follow these steps:

 Create a directory for the gvmd Unix socket && Adjust the permissions of the directory to allow access
  ```
   mkdir -p /tmp/gvm/gvmd
   chmod -R 777 /tmp/gvm
  ```
# In the next step, the docker compose file must be changed as follows:
  ```
  gvmd:
    image: greenbone/gvmd:stable
    restart: on-failure
    volumes:
       - gvmd_data_vol:/var/lib/gvm
       - vt_data_vol:/var/lib/openvas
       - psql_data_vol:/var/lib/postgresql
-      - gvmd_socket_vol:/run/gvmd  # before
+      - /tmp/gvm/gvmd:/run/gvmd   # after
       - ospd_openvas_socket_vol:/run/ospd
       - psql_socket_vol:/var/run/postgresql
     depends_on:
      - pg-gvm
```
```
  gsa:
    image: greenbone/gsa:stable
    restart: on-failure
     ports:
       - 9392:80
     volumes:
-      - gvmd_socket_vol:/run/gvmd # before
+      - /tmp/gvm/gvmd:/run/gvmd  # after
     depends_on:
       - gvmd
```

# To allow remote access to the Greenbone Web Interface, you need to modify the docker compose file to configure the web server (gsad) to listen on all network interfaces. 

1. Open the docker-compose.yml file in a text editor.

2. Locate the section for the gsa service (Greenbone Security Assistant) within the file.

3. Modify the ports configuration to remove the local address (127.0.0.1) and allow access on all host interfaces.


``` 
  gsa:
    image: greenbone/gsa:stable
    restart: on-failure
    ports:
      - 127.0.0.1:9392:80 #before
      - 9392:80     #After
    volumes:
      - gvmd_socket_vol:/run/gvmd
    depends_on:
      - gvmd
```
# Start the Greenbone Community Edition container.
```
docker compose -f $DOWNLOAD_DIR/docker-compose.yml -p greenbone-community-edition up -d
```
# Setting up an Admin User
```
docker compose -f $DOWNLOAD_DIR/docker-compose.yml -p greenbone-community-edition \
    exec -u gvmd gvmd gvmd --user=admin --new-password='<password>'
```

# Performing a Feed Synchronization
## To download the latest feed data container images run
```
docker compose -f $DOWNLOAD_DIR/docker-compose.yml -p greenbone-community-edition pull notus-data vulnerability-tests scap-data dfn-cert-data cert-bund-data report-formats data-objects
```
## To copy the data from the images to the volumes run

```
docker compose -f $DOWNLOAD_DIR/docker-compose.yml -p greenbone-community-edition up -d notus-data vulnerability-tests scap-data dfn-cert-data cert-bund-data report-formats data-objects
```



    
