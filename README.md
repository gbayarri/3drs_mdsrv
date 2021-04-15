# 3drs_mdsrv

This is a summary with some fixes of the [MDsrv installation instructions](https://nglviewer.org/mdsrv/installation.html) of the MDsrv official site.

## Software installation instructions 

### Step 1: install mdsrv

Install via pip3 in the server as root:

```shell
pip3 install mdsrv
```

### Step 2: test mdsrv

Test it simply executing `mdsrv`. As it's a server installation it won't open the browser, but it should show the next output:

```shell
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

```shell
sudo apt-get install apache2 libapache2-mod-wsgi-py3
```

Then activate wsgi with:

```shell
sudo a2enmod wsgi
```

### Step 2: create Apache2 conf file

Add the content of [apache.conf](config/apache.conf) into */etc/apache2/sites-available/000-default.conf* 

If new conf file is created, execute (from the */etc/apache2/sites-available/* folder):

```shell
sudo a2ensite newFile.conf
```

### Step 3: create python app.cfg file

Add the file [app.cfg](config/app.cfg) into some protected folder. In the **DATA_DIRS** variable of this file, there must be the route to the files folder.

More information about this file can be found [here](https://nglviewer.org/mdsrv/configuration.html).

### Step 4: create mdsrv.wsgi file

Create a folder and place the file [mdsrv.wsgi](config/mdsrv.wsgi). This folder must be the one defined in the [apache.conf](config/apache.conf) file: */path/to/mdsrv/mdsrv.wsgi*. 

In the **APP_CFG** variable of this file, there must be the route to the [app.cfg](config/app.cfg) file.

### Step 5: restart Apache

Execute:

```shell
sudo service apache2 restart
```

### Step 6: test server

Test the server accessing the next address (assuming that the installation is in localhost, if not, put the correct URL address):

> http:<span>//</span>localhost/**\<api\>**/dir/**\<myproject\>**/

Where **\<api\>** is the alias defined in the [apache.conf](config/apache.conf):

```shell
WSGIScriptAlias /api /path/to/mdsrv/mdsrv.wsgi
```
And **\<myproject\>** is the path defined in the variable **DATA_DIRS** of the [app.cfg](config/app.cfg) file:

```shell
DATA_DIRS = {
    "myproject": os.path.abspath("/path/to/myproject/files"),
}
```

The address above should show a list of the files stored in the */path/to/myproject/* files folder.

## Launch server

First off, make sure that the files of the [files](files/README.md) folder are in the in the path (**myproject**) defined in the variable **DATA_DIRS** of the [app.cfg](config/app.cfg) file:

```shell
DATA_DIRS = {
    "myproject": os.path.abspath("/path/to/myproject/files"),
}
```

Then, place the files of the [examples](examples/README.md) folder in the */var/www/html* folder of the server.

Finally execute the server on a browser:

> http:<span>//</span>localhost/