# Developing on OMERO - Creating your own development server with docker-compose
This section targets any development you want to do on any OMERO web application or any OMERO plugins (omero-iviewer, omero-figure, tagsearch, etc).

## Docker installation
You need to download and install [Docker desktop](https://www.docker.com/products/docker-desktop/).

> For Windows user, make sure that you also have [Windows subsystem Linux (WSL)](https://learn.microsoft.com/en-us/windows/wsl/install) installed on your machine.

## Step 1 - Creating a running omero server with docker compose
1. Clone the following repository on your computer: <kbd>git clone https://github.com/ome/docker-example-omero </kbd>.
2. Open `Docker desktop`.

> You always need to have Docker desktop open and ready to interact with the Docker engine.


3. Open a terminal in the repository of the ``docker-compose.yaml``.
4. Write down <kbd>docker compose pull</kbd> to get the Docker images (they are downloaded from Docker Hub).
5. Write down <kbd>docker compose build</kbd> to build the images.
6. Write down <kbd>docker compose up -d</kbd> to create and start the containers in detached mode. 
7. Check the *container* tab of your *docker desktop application*; you should have one *compose* made of 3 different containers.
8. Open a web browser and enter the following address to connect to your local OMERO: http://localhost:4080/ and connect using the credentials of the ``root`` user.

> **Note:** It can take few minutes for the server to be ready to accept connection. During that time, you may encounter login errors.

You now have an OMERO server ready for testing. This server, built with the standard `docker-compose.yaml`, doesn't allow you to modify plugin's version nor to develop and test your own code i.e. it is just a running instance of OMERO. To make it configurable, the following steps are required.

## Step 2 : Specifying a version of a plugin
This step shows how it is possible to create an omero-server with a specific version of a plugin e.g.  install ``omero-figure`` version 7.0.0.

1. In the same directory of `docker-example-omero`, create a new folder called `docker-omero-web` (the name has no importance; feel free to modify).
2. Create a new file called `Dockerfile`, without any extension and save it under `docker-omero-web` directory.
3. Edit the `Dockerfile` as follows:

````
FROM openmicroscopy/omero-web-standalone:latest
USER root
RUN /opt/omero/web/venv3/bin/pip install omero-figure==7.0.0
USER omero-web
````
- Line 1: Indicates which Docker image of omero-web you are using as reference. The version number can be changed according to the version you want to run AND the version available on Docker Hub. For simplicity, we are using the `latest` tag.
- Line 2: Indicates who runs the following command ; it has to be **root** to make any modification.
- Line 3: Command to install a plugin with a specific version. As the `pip install...` has to be run in a Python virtual environment, you need to specify its location by `/opt/omero/web/venv3/bin/`.
- Line 4: Switch the user to **omero.web**, as it is the user required to start the application at runtime.

4. Back to the `docker-compose.yaml` file, you need to edit it as follows: 

- Under `omeroweb:` replace
````
omero-web:
	image: "openmicroscopy/omero-web-standalone:5"
```` 
  - by
````
omero-web:
	build:
		context: ../docker-omero-web
		dockerfile: Dockerfile
````
- Line `context`: Indicate the relative path to the folder where the Dockerfile you want to build is located.
- Line `dockerfile`: Give its name `Dockerfile`, as we want to build it.

5. Follow **Step 1 steps (from 3 to 8)** to re-build and re-launch your containers. **Don't forget to shut down your containers before!**


## Step 3 - Adding a new plugin

1. Download the file [01-default-webapps.omero](https://github.com/ome/omero-web-docker/blob/master/standalone/01-default-webapps.omero) into the `docker-omero-web` folder, next to `Dockerfile`.
This file contains all configurations for the plugins that will be run in your docker. If you don't need some of the plugins, delete the corresponding lines.

As an example, the [omero-autotag](https://github.com/German-BioImaging/omero-autotag) plugin will be added.

2. In the `01-default-webapps.omero` file, add the configuration lines listed in the [GitHub repository of the plugin](https://github.com/German-BioImaging/omero-autotag?tab=readme-ov-file#installation).

> **Note:** Those line always begin with `omero config`.

> **Note:** Remove `omero` from the config line before pasting it in the `01-default-webapps.omero` file.


3. Edit the `Dockerfile` accordingly to `01-default-webapps.omero` file i.e. all plugins declared in `01-default-webapps.omero` have to be installed in the `Dockerfile`.

````
FROM openmicroscopy/omero-web:latest

USER root
RUN /opt/omero/web/venv3/bin/pip install \
        omero-figure==7.0.0 \
        omero-iviewer \
        omero-fpbioimage \
        omero-parade \
        omero-autotag==4.0.1 \
        omero-tagsearch==4.0.1 \
        omero-autotag \
        whitenoise

ADD 01-default-webapps.omero /opt/omero/web/config/
USER omero-web
````
- Line 14: Copy the full content of `01-default-webapps.omero` file (from your local computer) to `/opt/omero/web/config/` in the Docker container.

6. Follow **Step 1 steps (from 3 to 8)** to re-build and re-launch your containers. **Don't forget to shut down your containers before!**

## Step 4 - Develop and test your own code
In this section, you will see how to test the code you are developing within your ``omero`` docker.

> The example of OMERO.figure is used here but any other OMERO plugins can work.

1. Under `docker-omero-web` folder, clone the GitHub repository of [omero-figure](https://github.com/ome/omero-figure).

>  TIP: with Dockerfile, it is not possible to ``COPY`` files that are located upstream, but only files located downstream. This is the reason why we need to clone the ``omero-figure`` repository at the same level as the dockerfile i.e. in  the `docker-omero-web` folder.

2. Add the following lines (3 to 5) to the Dockerfile, immediately after adding the config file.

````
ADD 01-default-webapps.omero /opt/omero/web/config/

COPY omero-figure/. /home/figure/src
WORKDIR /home/figure/src
RUN /opt/omero/web/venv3/bin/pip install -e .

USER omero-web
````

- Line 3: Copying the content of the local ``omero-figure`` to the `/home/figure/src` folder in your container. This is necessary if you would like to test the code you will develop under `omero-figure`.
- Line 5: Installing the `-e .` indicates that we are editing this directory.

3. At the end of the `01-default-webapps.omero` file, add the following configuration lines to run the OMERO server in development mode.
````
config set omero.web.application_server development
config set omero.web.debug true
````

4. In order to dynamically see changes in the running Docker containers when you are developing an application e.g. ``omero-figure``, you need to add a `volume` that synchronizes local file with the mounted volume.
Adding a volume is achieved in the `docker-compose.yaml` file.

````
    ports:
      - "4080:4080"
    volumes:
		- type: bind
		  source: "../docker-omero-web/omero-figure"
          target: "/home/web/src"
````

5. Follow **Step 1 steps (from 3 to 8)** to re-build and re-launch your containers. **Don't forget to shut down your containers before!**

5. Modify any file under `omero-figure/src` and save the change(s).
6. Build the plugin files; have a look to the GitHub repository of the plugin to know how to build it.

> **Note:** This is an important step. Only files under `plugin-name/plugin_name` will be read and executed. So, if you don't build the plugin, then you will not see any changes even if you modified files under `plugin-name/src` and it is not always possible to modify the files directly under `plugin-name/plugin_name`.

8. In Docker desktop, restart the **OMERO-WEB container ONLY**.
