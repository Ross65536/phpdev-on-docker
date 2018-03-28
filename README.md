
## Intro

This README describes how to setup the development environment for LBAW. It was prepared to run on Linux but it should be fairly easy to follow and adapt for the remaining operating systems.

* [Installing Docker and Docker Compose](#installing-docker-and-docker-compose)
* [Setting up the Development repository](#setting-up-the-development-repository)
* [Starting Docker Containers](#starting-docker-containers)
* [Development phase](#development-phase)
* [Working with pgAdmin](#working-with-pgadmin)
* [Setting up Php Interpreter and Debugger](#setting-up-php-interpreter-and-debugger)
* [Laravel Code Structure](#laravel-code-structure)
* [Publishing your image](#publishing-your-image)


## Installing Docker and Docker Compose
Before starting you'll need to have Docker and Docker Compose installed on your PC. 

You can check the official instructions here,
[Install Docker](https://docs.docker.com/install/) and [Install Docker Compose](https://docs.docker.com/compose/install/#install-compose):

    sudo apt-get update
    sudo apt-get install apt-transport-https ca-certificates curl software-properties-common
    curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
    sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
    sudo apt-get update
    sudo apt-get install docker-ce
    docker run hello-world # make sure that the installation worked

    sudo curl -L https://github.com/docker/compose/releases/download/1.18.0/docker-compose-`uname -s`-`uname -m` -o /usr/local/bin/docker-compose
    sudo chmod +x /usr/local/bin/docker-compose
    docker-compose --version # verify that you have Docker Compose installed.
    
    
    
# Setting up the Development repository    

You should have your own repository and a copy of the demo repository in the same folder in your machine.  Then, copy the contents of the demo repository to your own:

    # clone the group repository, if not yet available locally
    git clone git@github.com:<YOUR GITHUB>/lbaw17GG 
    
    # clone this branch of the demo repository
    git clone -b phpdev-on-docker git@github.com:lbaw-admin/lbaw-laravel.git
    
    # remove the git folder from the demo
    rm -rf lbaw-laravel/.git
    
    # goto your repository
    cd lbaw17GG
    
    # make sure you are using the master branch
    git checkout master 
    
    # copy all the demo files
    cp -r ../lbaw-laravel/ .
    
    # add the new files to your repository
    git add .  
    git commit -m "Base laravel structure"
    git push origin master 

**Tip**: If you're having trouble cloning from GitHub using *ssh*, check [this](https://help.github.com/articles/connecting-to-github-with-ssh/).

Notice that you need to substitute \<YOUR GITHUB\> with the username of the team member that owns the
repository. At this point you should have the project skeleton in your local machine and be ready to start.
  
 
# Starting Docker Containers
> For sake of time saving, you might just go ahead and execute `docker-compose up` at you're project root. While it runs, come back and read this section. Also make sure you're using a reasonable internet connection.
 
If you're not familiar with Docker you can have a quick grasp of the tool by looking into its documentation [What is docker?](https://www.docker.com/what-docker) or watching [Docker in 12 Minutes](https://www.youtube.com/watch?v=YFl2mCHdv24) and [Docker Compose in 12 Minutes](https://www.youtube.com/watch?v=Qw9zlE3t8Ko).

But essentially, what you'll need to know is that Docker is a tool that allows you to run  __containers__ (similar to virtual machines, but much lighter). And this is good because: 
1. We get consistent environments anywhere we want. In our case, development (your PCs) and production (xxxx.lbaw-prod.fe.up.pt).
2. Because we can share this containers __images__, it is simpler to replicate an environment setup (as we will do here).
3. And because containers are fairly light, we can split our applications into many as we want, helping us having better modularity and a consequently less dependencies related headaches (aka micro services).

__Docker Compose__ is a tool that helps us manage multiple containers at once. Specify, start, pause, stop, and so on.


### Configured Containers

```
+-------+ +-------+ +-------+ +-------+
|       | |       | |       | |       |
|  PHP  | |POSTGRE| |PG     | |MAILHOG|
|       | |SQL    | |ADMIN4 | |       |
|       | |       | |       | |       |
+-------+ +-------+ +-------+ +-------+
+-------------------------------------+
|                                     |
+-------------------------------------+
```

For development purposes we have 4 configured containers. Which are specified under services at docker-compose.yml 
1. __php__ It is were your app source-code lives.
2. __postgres__ It hosts your (local) database.
3. __pgadmin__ It is a tool that that helps you interacting with your Database.
4. __mailhog__ It offers a "fake" e-mail server and client.


__We setup the PHP container__ so that the project folder is shared with the container i.e. it both lives in your PC(s) and in the container. So when you change your code it also changes inside the container. Ports are also opened and forwarded so that, for instance, when you go to http://localhost:8000 on your browser, you're requests are redirected to the php container (that is where the php server lives).

__To start the servers__ you simply need to run 
```
docker-compose up
```

This will automagically do everything for you. But it is important that you know a thing or two about what's going on under the hood: 
1. Docker-compose will read the docker-compose.yml file and know what services (or containers *) it needs to start up.
2. For each service it will see what __image__ this container is based on, and fetch it (from Docker Hub). In the case of an image not being provided, e.g. the php container, a Dockerfile is. This file is like a shell script that instructs what needs to be done to create the respective image. 
3. Once docker-compose has the __image__ of each service, it will spin up the respective containers. 
4. Around this phase, docker-compose sets up a network among all the containers and configures the respective internal DNS entries so that from one container you can reference the remaining by its service name. e.g if from the php container you `ping postgres` you'll hit the database server host. 
5. When the php container is launched, the __docker_run-dev.sh__ script is run within the container (check the Dockerfile-dev for more details). It will wait for the database server to be ready and then, if it is the first start up, it will install the project dependencies (`composer install`) and seed your database (`php artisan db:seed`). And finally, it launches the php server (`php artisan serve`).




__Everything should now be up and running.__ Checkout your web server at `http://localhost:8000`, the phpadmin at `http://localhost:5050` and mailhog at `http://localhost:8025`. To stop the servers just hit __Ctrt^C__.

__To restart the containers__, you just issue `docker-compose up` again. This docker-compose up followed by Ctrl^C is similar to turn on and off your computer, meaning that everything will be kept including the data at your database. But if for some reason you want to start fresh you can run `docker-compose down` which will destroy your the containers. The next time you run `docker-compose up`, docker instantiates brand new containers from the previously compiled __images__. (But you'll probably just want to re seed the database [publish your image](#publishing-your-image)) Either way, your code will always be kept because it lives in the host (your computer) and only is shared with the container.


__When you [publish your image](#publishing-your-image),__ the project source code will be copied to a brand new PHP container, slightly differently configured, and uploaded to Docker Hub. Latter on it will be pulled by an automated process to the production server and, due to the different setup configurations, it will connect to your database at the dbm.fe.up.pt, using the credentials previously given to you.

> Services and containers are not the same thing. If you have a website with a lot of visits, for instance, you might want to run it on multiple computers but only having one address test.com. So you would be providing a service, but running on multiple containers. But for lbaw, thinking container == service is good enough.



# Development phase

During the development you might need to re seed your database, or reinstall php dependencies i.e., interact with php that is running inside the container. Thankfully docker provides a quick way of executing commands inside a container from the host.
```
    docker exec lbaw_php php artisan db:seed
    docker exec lbaw_php composer install
```
Another approach is to "enter" the container by executing
```
    docker exec lbaw_php -it bash
```
you might notice that the terminal prompt changes to something like `root@acf3dbd56f07:/app# `. So, you can execute
```
    php artisan db:seed
    composer install
```
to leave the container execute `exit`.

__Note that the container must be running__ in order that you can run exec. Therefore if you pause the containers by hitting Ctrl^C, for example, it won't work. This means that you'll have to have one terminal for running the containers and another to exec commands onto the php container. I prefer the first approach but another possible one is to run docker in detached/background mode
```
    docker-compose -d up
    # do other things such exec into a container
    
    docker-compose stop # pauses the containers
```

> This happens because in Docker each container is running a main never-ending process, being that the life cicle of a container is tied to that process. In other words, a container only exists to provide an environment in which that process can be run. In fact when you hit `docker-composer up` and launch the containers, each has one process associated e.g., the php container has the php server, postgres the postgres server, and so on.

__You might run into file permissions issues__ if you use `php artisan make` to generate files. This happens because these files will be created by the php container root user. To solve that you can simply run `sudo chown -R $USER ` after each file(s) generation.

__When you save your changes to git__ simply stage, commit, merge and push your changes as usually. The only caveat is, if you doing that with terminal, and you are using `docker ... bash` to enter the containers, do not forget that you need to be at the host (your pc) when commiting and pushing as thats where your git creadentials and on the other hand if your're not at the host, you might run you'll run into permission issues with the .git files, analogously to the previous paragraph.

# Working with pgAdmin
 
You will be using _PostgreSQL_ to implement this project. If you've followed the previous sections namely  [Starting Docker Containers](#starting-docker-containers) you should now have both _PostgreSQL_ and _pgadmin 4_ running locally.
 
 The database's username is `postgres` and the password `pg!fcp`. You can access http://localhost:5050 to access _pgadmin 4_ and manage your database.

On the
first usage you will need to add the connection to the database using the following attributes:

    hostname: postgres
    username: postgres
    password: pg!fcp

Hostname is _postgres_ instead of _localhost_ since _docker composer_ creates an internal _DNS_ entry to
facilitate connection between linked containers.

# Setting up Php Interpreter and Debugger

### PhpStorm

You can check the official instructions [here](https://blog.jetbrains.com/phpstorm/2016/11/docker-remote-interpreters/), but it roughly translates into:
1. File > Settings
2. Languages & Frameworks > PHP
3. On CLI Interpreter, click (...)
4. Click (+) > From Docker, Vagrant, VM, Remote ...
5. Choose __Docker__ and select __lbawlaravel_php:latest__ as the __Image Name__
6. Hit Ok, and it automatically should detect PHP 7.1.15 and Xdebug 2.6.0

# Laravel Code Structure

In Laravel, a typical web request involves the following steps and files:

### 1) Routes

The webpage is processed by *Laravel*'s [routing](https://laravel.com/docs/5.5/routing) mechanism.
By default, routes are defined inside *routes/web.php*. A typical *route* looks like this:

    Route::get('cards/{id}', 'CardController@show');

This route receives a parameter *id* that is passed on to the *show* method of a controller
called *CardController*.

### 2) Controllers

[Controllers](https://laravel.com/docs/5.5/controllers) group related request handling logic into
a single class. Controllers are normally defined in the *app/Http/Controllers* folder.

    class CardController extends Controller
    {
        public function show($id)
        {
          $card = Card::find($id);

          $this->authorize('show', $card);

          return view('pages.card', ['card' => $card]);
        }
    }

This particular controller contains a *show* method that receives an *id* from a route. The method
searches for a card in the database, checks if the user as permission to view the card, and then
returns a view.

### 3) Database and Models

To access the database, we will use the query builder capabilities of [Eloquent](https://laravel.com/docs/5.5/eloquent) but the initial database seeding will still be done
using raw SQL (the script that creates the tables can be found in *resources/sql/seed.sql*).

    $card = Card::find($id);

This line tells *Eloquent* to fetch a card from the database with a certain *id* (the primary key of the
table). The result will be an object of the class *Card* defined in *app/Card.php*. This class extends
the *Model* class and contains information about the relation between the *card* tables and other tables:

    /* A card belongs to one user */
    public function user() {
      return $this->belongsTo('App\User');
    }

    /* A card has many items */
    public function items() {
      return $this->hasMany('App\Item');
    }

### 4) Policies

[Policies](https://laravel.com/docs/5.5/authorization#writing-policies) define which actions a user
can take. You can find policies inside the *app/Policies* folder. For example, in the *CardPolicy.php*
file, we defined a *show* method that only allows a certain user to view a card if that user is the
card owner:

    public function show(User $user, Card $card)
    {
      return $user->id == $card->user_id;
    }

In this example policy method, *$user* and *$card* are models that represent their respective tables,
*$id* and *$user_id* are columns from those tables that are automatically mapped into those models.

To use this policy, we just have to use the following code inside the *CardController*:

    $this->authorize('show', $card);

As you can see, there is no need to pass the current *user*.

### 5) Views

A *controller* only needs to return HTML code for it to be sent to the *browser*. However we will
be using [Blade](https://laravel.com/docs/5.5/blade) templates to make this task easier:

    return view('pages.card', ['card' => $card]);

In this example, *pages.card* references a blade template that can be found at *resources/views/pages/card.blade.php*. The second parameter is the data we are sending to the template.

The first line of the template states it extends another template:

    @extends('layouts.app')

This second template can be found at *resources/views/layouts/app.blade.php* and is the basis
of all pages in our application. Inside this template, the place where the page template is
introduced is identified by the following command:

    @yield('content')

Besides the *pages* and *layouts* template folders, we also have a *partials* folder where small
snippets of HTML code can be saved to be reused in other pages.    

### 6) CSS

The easiest way to use CSS is just to edit the CSS file found at *public/css/app.css*.

If you prefer to use [less](http://lesscss.org/), a PHP version of the less command-line tool as
been added to the project. In this case, edit the file at *resources/assets/less/app.less* instead and
keep the following command running in a shell window so that any changes to this file can be
compiled into the public CSS file:

    ./compile-assets.sh

### 7) Javascript

To add Javascript into your project, just edit the file found at *public/js/app.js*.

## Publishing your image

You should keep your git's master branch always functional and frequently build and deploy your
code. To do so, you will create a _docker_ image for your project and publish it at
[docker hub](https://hub.docker.com/). LBAW's teachers will frequently pull all these images and
make them available at http://<YOUR_GROUP>.lbaw-prod.fe.up.pt/. This demo repository is available at
[http://demo.lbaw-prod.fe.up.pt/](http://demo.lbaw-prod.fe.up.pt/). Make sure you are inside FEUP's 
network or VPN.

First thing you need to do is create a [docker hub](https://hub.docker.com/) account and get your
username from it. Once you have a username, let your docker know who you are by executing:

    docker login

Once your docker is able to communicate with the docker hub using your credentials configure the
`upload_image.sh` script with your username and group's identification as well. Example configuration

    DOCKER_USERNAME=johndoe # Replace by your docker hub username
    IMAGE_NAME=lbaw17GG # Replace by your lbaw group name

Afterwards, you can build and upload the docker image by executing that script from the project root:

    ./upload_image.sh

You can test the locally by running:

    docker run -it -p 8000:80 -e DB_DATABASE=<your db username> -e DB_USERNAME=<your db username> -e DB_PASSWORD=<your db password> <DOCKER_USERNAME>/<IMAGE NAME>

The above command exposes your application on http://localhost:8000. The `-e` argument creates environment variables inside the container, used to provide laravel with the required database configurations. 

Note that during the build process we adopt the production configurations configured in the `.env_production` file. **You should not add your database username and password to this file, your configuration will be provided as an environment variable to your container on execution time**. This prevents anyone else but us from running your container with your database. 

There should be only one image per group. One team member should create the image initially and add his team to the repository at docker hub. You should provide your teacher the details for accessing your docker image, namely, docker username and repository (DOCKER_USERNAME/lbaw17GG).
