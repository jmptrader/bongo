---
title: Implementing M-PESA processing system in GO
section: blog 
---

## How the infastructure  works

* The smsd daemon checks for rexeived sms, If it finds any, It stores to the databsse backend in our case `postgresql` and the name of the database will be in the `.gammu-smsdrc` configuration file.

* After saving the sms, the daemon executes `pesambili` binary

* `pesambili` binary calls post request to the `sweetjesus` localserver running on port 8000.

* The `sweetjesus` server when receives the ping at the  `/mpesa` endpoint, it assumes there is an inbox, goes to the sms daemon server database and collects all inbox messages and process them.

* Well, we should make sure processed messages are moved to another table to avoid mayhem

It is beyond the scope of this post to go in great details how each component work, I will rather focus on how to work with each compoent to make the system tick. Oh! well, I will try to dig a bit about important bits though.


###  Installing and configuring gammu-smsd
______

The first challenge is to install and configure gammu-smsd. SMSD stands for SMS daemon.
as the name suggests, it runs cron jobs on top of gammu. Gammu is the software 
for accessing functionality of mobile phones in the desktop computers.

I am using linux mint, and I assume you are a technical reader, so enough of talking
Time to write some code

Install gammu, wammu, and gammu-smsd

    sudo apt-get install wammu gammu gammu-smsd

Noye that wammu is optional, but its good to use wammu configuring the devices. When the
installation is complete,

Start wammu with privillege

    sudo wammu

Now the wammu window will appear, and follow The instruction to configure your device
I am using a ZTE modem and it is mounted at `/dv/tty/usb2`.

When you have finished configuring gammu then a file `.gammurc` will be created at the home
path of your pc.

    $ cat .gammurc
    [gammu]
    port=/dev/ttyUSB2
    connection=at
    name=Vodafone_Modem

I have given the name `vodacom_modem` to my modem, you can name whatever you want.

So, the Real part was making gammu-smsd to work.

First create a database `smsd` I am using postgres database so I did like this

    $ createdb -O postgres smsd

The default user is postgres, so I create a database names `smsd` and give ownership
to postgres.

Now, there is a modified postgresl schema gist here [  gammu-smsd postgres schema](https://gist.github.com/gernest/a8b9f1fc8474bd92c75c), download it
and upack somewhere you can access.

Navigate to the psql.sql file

    $ cd path/to/psql/file


Execute the following command

    $ psql -d smsd -f pgsql.sql

You will see output on the terminal showing what tables are creates

Ok, time to write a gammu-smsd configuration.
First navigate back home.

    $ cd /path/to/home

Orjust type

    $ cd ~

Okay, I will use linux magic to copy fontent of our `.gammurc` file we created
ealier to our new `.gammu-smsdrc` file

    $ cat .gammurc > .gammu-smsdrc

Okay, we will need now to edit our .gammu-smsdrc

    $ nano .gammu-smsdrc

Edit it to look like this

    [gammu]
    port=/dev/ttyUSB2
    connection=at
    name=Vodafone_Modem

    [smsd]
    service=sql
    driver=native_pgsql
    host=localhost
    database=smsd
    user=postgres
    password=postgres
    logfile=/home/YOUR_USERNAME/sms.log


You can change the values whatever you like, and everything is clear, Note that
I am running a test database with user postgres and password postgres so you can put whatever you
like. And dont forget to replace `YOUR_USERNAME` with your pc username

Okay, we almost done, time to start our service.

    $ gammu-smsd -c .gammu-smsdrc

You will see a path to log file, and If nothing else is printed just know we have configured everything right.

What gammu-smsd will do is listen for incoming sms and after receiving them stores them in the database. Smsd has many cool stuffs, but In only interested in processing the received sms which are stores in the inbox table.



### The `Pesambili` script

Well, there is a hook on the `smsd` daemon called `onReceive` . It tells the daemon what to call when there is a received sms. Shell scripting is the way to go, but I wanted to focus more on go, so instead of writing a shell script, I wrote just a wrapper in go, In the hood there is a `curl` command invoked hence `pesambili` go package.

You can install this package by this command

    $ go get github.com/gernest/pesambili

If I have time I will remove the `curl` dependency and use `net/http` package instead. You should note that, I assume you have added `$GOSRC/bin` to your system path environment variables(I mean you have a working `go` environment)

__suorce code__ for `pesambili` is available here [github.com/gernest/pesambili](https://github.com/gernest/pesambili)


### The `sweetjesus` server

This is a very crucial part of the system, Its the place where actual work is done. This server uses the `net/http` package. It listens on port `8000` and register only one handler at the endpoint `/mpesa`.

So, the only varid connection to the server is through the `url` `http://localhost:8000/mpesa`. For the moment, the server only accepts  post request, and funny enough it doesn't give a shit about the received data.

The point of the request is just to make it know that there is an sms in our inbox.

You can install this server by running the following command.

    $ go get github.com/gernest/sweetjesus

There is still room for improvement though, so contributions are welcome

__source code__ for `sweetjesus` server is available here [github.com/gernest/sweetjesus](https://github.com/gernest/sweetjesus)

### The `jesus` library

This is backbone of the `sweetjesus` server. I Contains the utility functions, the table structures and many other goodies. Its in an awkward state right now but I think its design makes sense.

There is a great deal in parsing and extracting information from a received m-pesa sms. So instead of creating a lexer and parser, I just rolled an ad-hoc implementation for extracting the data in the `processor.go` file.

I will try to improve stuffs when I get the time. The part that I will be happy to receive contributions is for the `FilterFunctions`. I will write more about the filter functions when I write a a dedicated post about the library.

_Be Warned_ This is a one week hack, please dont blame me for anything weird in it, but dont be selfish to report any bugs

You can install this by the following command

    $ go get github.com/gernst/jesus

__source code__ for `jesus` packageis available here [github.com/gernest/jesus](https://github.com/gernest/jesus)
