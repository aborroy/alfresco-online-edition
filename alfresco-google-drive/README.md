# Alfresco Google Drive deployment

This project provides Docker Compose template to deploy [Google Drive](https://docs.alfresco.com/google-drive/latest/) in ACS 7.3

>> This project has been designed to test the integrations easily, deploying them in *prod* environments would require additional operations and resources.


## Google Drive

Source code is available in [https://github.com/Alfresco/googledrive](https://github.com/Alfresco/googledrive)

This module has been enabled adding following property to Alfresco Repository.

```
services:
    alfresco:
        environment:
            JAVA_OPTS : '
                -Dgoogledocs.enabled=true 
            '
```

This feature is available for Share Web application and it requires additionally a Repository addon.

* Repository addon deployment `/alfresco/modules/amps/alfresco-googledrive-repo-community-3.3.1.amp` 
* Share addon deployment `share/modules/amps/alfresco-googledrive-share-3.3.1.amp`

Once the module is ready a new action `Edit in Google Docs™` is available in Share Web Application. This module is **not** available for [Alfresco Content Application](https://github.com/alfresco/alfresco-content-app).


## Running

Start the Docker Compose using the regular command from project root folder.

```
$ docker compose up
```

Once everything is up & ready, access to following URLs using default credentials (admin/admin):

* Share Web Application http://localhost:8080/share
