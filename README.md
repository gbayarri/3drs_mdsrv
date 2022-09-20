# MMB MDsrv

This is a summary with some fixes of the [MDsrv installation instructions](https://nglviewer.org/mdsrv/installation.html) of the MDsrv official site.

The [MMB (IRB Barcelona)](https://mmb.irbbarcelona.org/) technical team has deployed a version of MDsrv:

[https://mmb.irbbarcelona.org/mdsrv](https://mmb.irbbarcelona.org/mdsrv)

## Installation steps

* [Software installation instructions](#software-installation-instructions)
  1. [Step 1: install mdsrv](#step-1-install-mdsrv)
  2. [Step 2: test mdsrv](#step-2-test-mdsrv)
* [Server installation instructions](#server-installation-instructions) 
  1. [Step 1: install Apache2 and mod-wsgi](#step-1-install-apache2-and-mod-wsgi)
  2. [Step 2: create Apache2 conf file](#step-2-create-apache2-conf-file)
  3. [Step 3: create python app.cfg file](#step-3-create-python-appcfg-file)
  4. [Step 4: create mdsrv.wsgi file](#step-4-create-mdsrvwsgi-file)
  5. [Step 5: restart Apache](#step-5-restart-apache)
  6. [Step 6: test server](#step-6-test-server)
* [Launch server](#launch-server)
  1. [Step 1: place structure and trajectory files](#step-1-place-structure-and-trajectory-files)
  2. [Step 2: create HTML & JS client files](#step-2-create-html--js-client-files)
  3. [Step 3: execute client](#step-3-execute-client)
* [Install MDAnalysis](#install-mdanalysis)
* [Install GridFS Fuse](#install-gridfs-fuse)
    + [Installation](#installation)
    + [Mounting](#mounting)
    + [Unmounting](#unmounting)

## Software installation instructions 

### Step 1: install mdsrv

Install via pip3 in the server as root:

```bash
sudo pip3 install mdsrv
```

### Step 2: test mdsrv

Test it simply executing `mdsrv`. As it's a server installation it won't open the browser, but it should show the next output:

```console
* Serving Flask app "mdsrv.mdsrv" (lazy loading)
 * Environment: production
   WARNING: This is a development server. Do not use it in a production deployment.
   Use a production WSGI server instead.
 * Debug mode: off
INFO:werkzeug: * Running on http://127.0.0.1:39605/ (Press CTRL+C to quit)
```

## Server installation instructions 

### Step 1: install Apache2 and mod-wsgi

Execute:

```bash
sudo apt-get install apache2 libapache2-mod-wsgi-py3
```

Then activate wsgi with:

```bash
sudo a2enmod wsgi
```

### Step 2: create Apache2 conf file

Add the content of [apache.conf](config/apache.conf) into */etc/apache2/sites-available/000-default.conf*:

```apacheconf
<VirtualHost *:80>
    ServerName localhost
    #LogLevel info ssl:warn
    ErrorLog ${APACHE_LOG_DIR}/error.log
    CustomLog ${APACHE_LOG_DIR}/access.log combined

    # the wsgi process will run with the user & group specified below,
    # so make sure that the files and directories you want to serve
    # are accessible with that user & group combination
    # please also adjust the python-path or remove it to use the default
    WSGIDaemonProcess mdsrv user=www-data group=www-data threads=5
    WSGIScriptAlias /api /path/to/mdsrv/mdsrv.wsgi

    <Directory /path/to/mdsrv>
        WSGIProcessGroup mdsrv
        WSGIApplicationGroup %{GLOBAL}
        WSGIScriptReloading On
        WSGIPassAuthorization On
        Require all granted
    </Directory>

    DocumentRoot /var/www/html
    <Directory /var/www/html>
        Options Indexes FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>

</VirtualHost>
```

If new .conf file is created, execute (from the */etc/apache2/sites-available/* folder):

```bash
sudo a2ensite newFile.conf
```

### Step 3: create python app.cfg file

Add the file [app.cfg](config/app.cfg) into some protected folder. In the **DATA_DIRS** variable of this file, there must be the route to the files folder:

```python
import os

DEBUG = False
HOST = "127.0.0.1"
PORT = 8010

MAX_CONTENT_LENGTH = 64 * 1024 * 1024
SEND_FILE_MAX_AGE_DEFAULT = 0

DATA_DIRS = {
    "myproject": os.path.abspath("/path/to/myproject/files"),
    # "_hidden": os.path.abspath("/path/hidden/from/dir/listing"),
    # "protected": os.path.abspath("/path/protected"),
}

# Note that only one of REQUIRE_AUTH and REQUIRE_DATA_AUTH
# can be true with the former taken precedence

REQUIRE_AUTH = False
USERNAME = "user"
PASSWORD = "pass"

REQUIRE_DATA_AUTH = True
DATA_AUTH = {
        "protected": [ "user", "test123" ]
}
```

More information about this file can be found [here](https://nglviewer.org/mdsrv/configuration.html).

### Step 4: create mdsrv.wsgi file

Create a folder and place the file [mdsrv.wsgi](config/mdsrv.wsgi). This folder must be the one defined in the [apache.conf](config/apache.conf) file: */path/to/mdsrv/mdsrv.wsgi*. 

```python
#####
# configuration for the mdsrv wsgi app

# leave empty if no virtualenv is needed
APP_ENV = ''

# leave empty to use default config
APP_CFG = '/path/to/app.cfg'


#####
# do not change anything below unless you are sure
# see http://flask.pocoo.org/docs/deploying/mod_wsgi/


import os, sys

if APP_ENV:
    if sys.version_info > (3,):
        activate_this = os.path.join( APP_ENV, 'activate_this.py' )
        with open( activate_this ) as f:
            code = compile( f.read(), activate_this, 'exec' )
            exec( code, dict( __file__=activate_this ) )
    else:
        activate_this = os.path.join( APP_ENV, 'activate_this.py' )
        execfile( activate_this, dict( __file__=activate_this ) )

from mdsrv import app as application
if APP_CFG:
    application.config.from_pyfile( APP_CFG )
```

In the **APP_CFG** variable of this file, there must be the route to the [app.cfg](config/app.cfg) file.

### Step 5: restart Apache

Execute:

```bash
sudo service apache2 restart
```

### Step 6: test server

Test the server accessing the next address (assuming that the installation is in localhost, if not, put the correct URL address):

> http:<span>//</span>localhost/**\<api\>**/dir/**\<myproject\>**/

Where **\<api\>** is the alias defined in the [apache.conf](config/apache.conf):

```apacheconf
WSGIScriptAlias /api /path/to/mdsrv/mdsrv.wsgi
```
And **\<myproject\>** is the path defined in the variable **DATA_DIRS** of the [app.cfg](config/app.cfg) file:

```python
DATA_DIRS = {
    "myproject": os.path.abspath("/path/to/myproject/files"),
}
```

The address above should show a list of the files stored in the */path/to/myproject/* files folder.

## Launch server

### Step 1: place structure and trajectory files

First off, make sure that the files of the [files](files/README.md) folder are in the path (**myproject**) defined in the variable **DATA_DIRS** of the [app.cfg](config/app.cfg) file:

```python
DATA_DIRS = {
    "myproject": os.path.abspath("/path/to/myproject/files"),
}
```

### Step 2: create HTML & JS client files

Then, place the files of the [examples](examples/README.md) folder in the */var/www/html* folder of the server.

Below you can find the code for a basic **HTML + JS** example:

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <title>MDsrv/NGL - embedded</title>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, user-scalable=no, minimum-scale=1.0, maximum-scale=1.0">
</head>
<body>
    <!-- NGL -->
    <script src="ngl.2.0.0-dev.36.js"></script>

    <script>
        // Datasources
        var mdsrvDatasource = new NGL.MdsrvDatasource( "http://localhost/api/" )
        NGL.DatasourceRegistry.add("file", mdsrvDatasource)
        NGL.setListingDatasource(mdsrvDatasource)
        NGL.setTrajectoryDatasource(mdsrvDatasource)
        
        document.addEventListener( "DOMContentLoaded", function(){
            stage = new NGL.Stage( "viewport" );
            stage.loadFile( "/api/file/myproject/md_1u19.gro", { defaultRepresentation: false } ).then( function( comp ){
                comp.setName( "comp_name" );
                comp.setSelection( "" );
                comp.addRepresentation( "tube", { sele: "*", opacity: 1, side: "front" } );
                var t = comp.addTrajectory( "myproject/md_1u19.xtc", {centerPdb: true, removePbc: true, superpose: true, initialFrame: 0} );      
                // tricky way for getting the total number of frames
                setTimeout( function() {
                    document.getElementById( "clipRange" ).max = (t.trajectory.frameCount - 1)
                }, 200);
                stage.setParameters( { backgroundColor: "#f1f1f1" } );
                comp.autoView();
            } );
            var toggleRunMDs = document.getElementById( "play" );
            var isRunning = false;
            toggleRunMDs.addEventListener( "click", function(){
                var trajComp = stage.getComponentsByName("comp_name").list[0].trajList[0];
                var player = new NGL.TrajectoryPlayer(trajComp.trajectory, {timeout: 10, mode: "once", interpolateType: "spline",step: 1, interpolateStep: 5});
                if( !isRunning ){
                    player.play();
                    isRunning = true;
                    var clipNear = 0;
                    trajComp.signals.frameChanged.add(function(){
                        var fnum=trajComp.trajectory.currentFrame;                  
                        clipRange.value = fnum;
                        clipRange_val.innerHTML = parseInt(clipRange.value) + 1;
                    });
                }else{
                    player.play();
                    player.pause();
                    isRunning = false;
                }
            } );
            var clipRange = document.getElementById( "clipRange" );
            var clipRange_val = document.getElementById( "clipRange_val" );

            clipRange.oninput = function( e ){
                var trajComp = stage.getComponentsByName("comp_name").list[0].trajList[0];
                trajComp.setFrame(e.target.value)
                clipRange_val.innerHTML = parseInt(e.target.value) + 1;
            };
        } );
    </script>

    <h2>MDsrv Test:</h2>
    
    <div style="display:inline;text-align: left;float: left; width: 305px;">

        <div id="viewport" style="width:600px; height:600px;"></div>
        <div style="width:600px;">
            <input id="clipRange" type="range" value="0" min="0" max="100" step="1" style="width:80%;"></input><span id="clipRange_val">1</span>fs
            <button id="play">&#9658; / &#9613;&#9613;</button>
        </div>
    </div>
    
</body>
```

### Step 3: execute client

Finally execute the server on a browser:

> http:<span>//</span>localhost/

## Install MDAnalysis

By default, MDsrv supports the following formats: DCD, NCTRAJ/NetCDF, TRR, XTC, LAMMPSTRJ, XYZ, BINPOS, HDF5, DTR, ARC, TNG, GRO. Additionally, installing MDAnalysis, more formats will be supported: MDCRD/CRD, DMS, TRJ, ENT, NCDF.

First off, create a **/var/www/.local** folder with permissions for the **www-data** user.

Login as **www-data** user:

```bash
sudo su www-data
```

Install MDAnalysis:

```bash
pip3 install --user MDAnalysis[analysis] MDAnalysisTests
```

Execute tests to check that MDAnalysis is working properly (it takes between 15 and 20 minutes):

```bash
pytest --disable-pytest-warnings --pyargs MDAnalysisTests
```

Restart Apache:

```bash
sudo /etc/init.d/apache2 restart
```

## Install GridFS Fuse

As seen in the [Step 3: create python app.cfg file](#step-3-create-python-appcfg-file), the route to the files folder must be a path in the machine where MDsrv is installed. In order to store the trajectories in a **MongoDB GridFS**, there is a workaround: install **GridFS Fuse** to access the MongoDB GridFS files as if they were in the machine file system.

We have chosen **this version** of GridFS Fuse:

https://github.com/jmfernandez/py_gridfs_fuse

### Installation

Install libfuse:

```bash
sudo apt-get install python3-pip libfuse3-dev
```

Install py_gridfs_fuse:

```bash
sudo -H pip3 install git+https://github.com/jmfernandez/py_gridfs_fuse.git@v0.4.0
```

### Mounting

Mount MongoDB GridFS in a folder of our MDsrv machine (execute it as **sudo user**):

```bash
mount.gridfs_naive mongodb://<user>:<password>@<host>:<port>/<dbname>?authSource=admin /mnt/gridfs_fuse -o allow_other
```

or 

```bash
naive_gridfs_fuse --mongodb-uri="mongodb://<user>:<password>@<host>:<port>/<dbname>?authSource=admin" --database="dbname" --mount-point="/mnt/gridfs_fuse" --show-versions -o allow_other
```

Now, all the files of the GridFS in the **dbname** database will we available in **/mnt/gridfs_fuse**. Take into account that all the files must have different filename (no repeated names allowed at this moment).

This installation of GridFS Fuse has been only used for reading.

### Unmounting

Standard way:

```bash
umount gridfs
```

If target is busy:

```bash
lsof | grep fuse
kill -9 <pid>
umount /mnt/gridfs_fuse
```