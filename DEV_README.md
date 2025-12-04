# Developping on OMERO - Creating your own dev server with docker-compose
This section targets any developments you want to do on any OMERO web application or any OMERO plugins (omero-iviewer, omero-figure, tagsearch...)

## Docker installation
You need to download and install [Docker desktop](https://www.docker.com/products/docker-desktop/) 

> For Windows user, make sur that you also have [Windows subsystem Linux (WSL)](https://learn.microsoft.com/en-us/windows/wsl/install) installed on your machine.

## Step 1 - Create a running omero server with docker compose
1. Clone the following repo on your computer : <kbd>git clone https://github.com/ome/docker-example-omero </kbd>
2. Open `Docker desktop`

> You always need to have Docker desktop open and ready to make anything with docker.
{.is-info}

3. Open a git terminal at the ``docker-compose.yaml`` layer, in the `docker-example-omero` folder <kbd>right-click + open-git-bash-here</kbd>
4.  Write down <kbd>docker compose pull</kbd> to get the docker images (they are are downloaded from docker hub)
5. Write down <kbd>docker compose build</kbd> to build the images
6. Write down <kbd>docker compose up -d</kbd> to create the containers. 
7. Have a look to the *container* tab of your *docker desktop application* ; you should have one *compose* made of 3 different containers.
8. Open a web browser and enter the following address to connect to your local OMERO : http://localhost:4080/ and connect with the root credentials (u: root ; p: omero)

> It can take a while since the server is ready. During that time, you may have login errors.
{.is-info}

You have now one OMERO server that you can use to try out any stuff you want. This server, build with the standard `docker-compose.yaml`, doesn't allow you to modify plugin's version or to develop and test your own code (i.e. it is just a running instance of OMERO). Let's do the others steps to make it configurable.

## Step 2 : Specify a version of a plugin
This step to show how it is possible to create an omero-server app with a specific version of a plugin (ex: omero-figure 7.0.0) running on it.


1. In the same directory of `docker-example-omero`, create a new folder called `docker-omero-web` (the name has no importance ; if you want to name it differently, fell free)
2. Create a new file `Dockerfile`, without any extension and save it under `docker-omero-web`.
3. Edit the `Dockerfile` as following

````
FROM openmicroscopy/omero-web-standalone:latest
USER root
RUN /opt/omero/web/venv3/bin/pip install omero-figure==7.0.0
USER omero-web
````
- Line 1 : Indicate which docker image you are using as reference. The version number can be changed according to the version you want to run AND the version available on docker-hub. For simplicity, we are using the keyword `latest`.
- Line 2 : Indicate the user who runs the following command.
- Line 3 : Command to install a plugin with a specifc version. As the `pip install...` has to be run in a python environment, one need to specify its location by `/opt/omero/web/venv3/bin/`

4. Back to the `docker-compose.yaml` file, one need to edit it as following: 

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
- Line `context` : Indicate the relative path to the folder where the Dockerfile you want to build is located
- Line `dockerfile`: Give its name `Dockerfile`, as we want to build it.

5. Follow **Step 1 steps (from 3 to 8)** to re-build and re-launch your containers. **Don't forget to shut down your containers before !**


## Step 3 - Add a new plugin

1. Copy the file [01-default-webapps.omero](https://github.com/ome/omero-web-docker/blob/master/standalone/01-default-webapps.omero) into the `docker-omero-web`folder, next to `Dockerfile`.
2. Modify the `Dockerfile` such that the plugins you're installing are the same as the one listed in the `01-default-webapps.omero` file AND the image pulled in `omero-web` (not `omero-web-standalone`anymore)

> You can simply replace your Dockerfile by the one given [here](https://github.com/ome/omero-web-docker/blob/master/standalone/Dockerfile)
{.is-info}

3. As an example, we'll add [omero-autotag](https://github.com/German-BioImaging/omero-autotag) plugin

4. In the `01-default-webapps.omero` file, add the configuration lines. 
Please refer to the github repository of the plugin to get the configuration lines.

````
# omero-autotag
config append omero.web.apps '"omero_autotag"'
config append omero.web.ui.center_plugins '["Auto Tag", "omero_autotag/auto_tag_init.js.html", "auto_tag_panel"]'
````

5. In the Dockerfile, add an entry to pip install omero-mapr and to add the configs.

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
- Line 14 : Copy the full content of `01-default-webapps.omero` file (from your local computer) to `/opt/omero/web/config/` in the docker container.

6. Follow **Step 1 steps (from 3 to 8)** to re-build and re-launch your containers. **Don't forget to shut down your containers before !**

## Step 4 - Develop and test your own code
In this section, you'll see how to test the code you're developping within your omero docker.

> The example of OMERO.figure is used here but any other OMERO plugins can work
{.is-info}


1. Under `docker-omero-web` folder, clone the github repository of omero-figure https://github.com/ome/omero-figure

>  TIP : with Dockerfile, it is not possible to COPY files that are located upstream, but only files located downstream. This is the reason why we need to clone ``omero-figure`` repo at the same level as the dockerfile i.e. in `docker-omero-web` folder.
{.is-info}

2. Add the following lines (3 to 5) to the Dockerfile, right-after adding the confg file.

````
ADD 01-default-webapps.omero /opt/omero/web/config/

COPY omero-figure/. /home/figure/src
WORKDIR /home/figure/src
RUN /opt/omero/web/venv3/bin/pip install -e .

USER omero-web
````

- Line 3 : This will copy the content of your local omero-figure to the `/home/figure/src` folder in your container. This is necessary if you would like to test the code you'll develop under `omero-figure`.
- Line 5 : Installing the `-e .` indicates that we are editing this directory.

3. At the end of the `01-default-webapps.omero` file, add the following configuration lines. They tell OMERO that the OMERO server is used for development
````
config set omero.web.application_server development
config set omero.web.debug true
````

4. In order to dynamically see changes in your docker when you are coding under omero-figure, you need to add a `volume` that synchronize local file with your docker. Adding a volume is done in the `docker-compose.yaml` file.

````
    ports:
      - "4080:4080"
    volumes:
		- type: bind
		  source: "../docker-omero-web/omero-figure"
          target: "/home/web/src"
````
- Line 4 : It is important here to match the sames folders as the one you copied in the Dockerfile

5. Follow **Step 1 steps (from 3 to 8)** to re-build and re-launch your containers. **Don't forget to shut down your containers before !**

5. Modify any file under `omero-figure/src` and save them.
6. Build the new *readable* files ; have a look to the github repo of the plugin to know how to build it. For omero-figure, see below.

> This is an important step. Only files under `plugin-name/plugin_name` will be read and executed. So, if you don't build the plugin, then you'll not see any changes even if you modified files under `plugin-name/src` and it is not always possible to modify the files directly under `plugin-name/plugin_name`
{.is-warning}