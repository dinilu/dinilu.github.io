---
layout: post
title: Setting a PostgreSQL server for the European Pollen Database (EPD)
date: 2016-11-04 16:52
author: dnietolugilde
comments: true
categories: [Botanica y Ecologia, Datos, EPD, Scripts]
---
In this vignette we will explain and illustrate how to install a PostgreSQL server in Windows and Ubuntu, and how to mirror the European Pollen Database (EPD) into the server. The general procedure include the next steps:
<ol>
	<li>Download and install PostgreSQL server</li>
	<li>Download EPD data for PostgreSQL</li>
	<li>Setup PostgreSQL for EPD:
<ul>
	<li>Create mandatory users for EPD</li>
	<li>Create a database for EPD into the server</li>
</ul>
</li>
	<li>Dump EPD data into the new database</li>
</ol>
<!-- # Download and install PostgreSQL server {.tabset} -->
<h1>Download and install PostgreSQL server</h1>
The first step is to download and install the PostgreSQL server. This step changes depending on your operating system (OS). Select your OS in the tabs below.
<h2>Windows</h2>
You can download Windows binary files of PostgreSQL server latest version, which include the graphical interface pgAdminIII, from <a href="https://www.postgresql.org/download/windows/">https://www.postgresql.org/download/windows/</a>. Currently (2016/06/13), there are two providers: EnterpriseDB and BigSQL. BigSQL provides a much richer installation of the server with a lot of complements that are intended for developers and that will be useless for a low profile user. Hence, I recommend to use binary files from EnterpriseDB. The file name should looks like <code>postgresql-XXXX-windows-YYY.exe</code> (Xs indicate the version number and Ys the computer architecture: x86 or x64).

[caption id="attachment_536" align="alignnone" width="600"]<img class="alignnone size-full wp-image-536" src="https://dnietolugilde.files.wordpress.com/2018/07/postgresql-website.png" alt="PostgreSQL-website" width="600" height="394" /> Download PostgreSQL website.[/caption]

Installation wizard starts with a double click on the binary file (as any other Windows program). Then, it allows you to change the values for different configuration parameters: installation directory, data directory, or the communication port. You can change those or use the default values. It is important to remember those values (specially if you change them). Next, the installation wizard ask you for a password (has to be filled twice). This password is for a special user of the database called “postgres”. This user is the default superuser or administrator of the database server. User “postgres” can create new users, databases, etc. It is important to provide a password for security reasons and to remember it, otherwise you won't be able to make any operation with the server.

[caption id="attachment_542" align="alignnone" width="600"]<img class="alignnone size-full wp-image-542" src="https://dnietolugilde.files.wordpress.com/2018/07/3.png" alt="3" width="600" height="466" /> Installation windows to specify PostgreSQL installation folder.[/caption]

[caption id="attachment_543" align="alignnone" width="600"]<img class="alignnone size-full wp-image-543" src="https://dnietolugilde.files.wordpress.com/2018/07/4.png" alt="4" width="600" height="463" /> Installation windows to specify PostgreSQL data folder.[/caption]

[caption id="attachment_545" align="alignnone" width="600"]<img class="alignnone size-full wp-image-545" src="https://dnietolugilde.files.wordpress.com/2018/07/6.png" alt="6" width="600" height="466" /> Installation windows to specify the password for postgres user.[/caption]

[caption id="attachment_546" align="alignnone" width="600"]<img class="alignnone size-full wp-image-546" src="https://dnietolugilde.files.wordpress.com/2018/07/7.png" alt="7" width="600" height="464" /> Installation windows to specify communication port number.[/caption]

[caption id="attachment_547" align="alignnone" width="600"]<img class="alignnone size-full wp-image-547" src="https://dnietolugilde.files.wordpress.com/2018/07/8.png" alt="8" width="600" height="467" /> Installation windows to specify regional configuration for the server.[/caption]

When the installation wizard is over, you have to modify the setup in Windows path to include the <code>bin</code> folder in the installation directory of PostgreSQL. This allows you to run PostgreSQL commands in the command line from any folder in your computer. To do so, you need to right click on <code>System</code>, then sequentially click on <code>Properties</code>, <code>Advanced Setup</code>, <code>Env. Variables</code>, <code>System Variables</code>, and <code>Edit</code>. Then, at the end of the text box you have to include the address of the installation directory you especified during the installation wizard plus <code>\bin</code>. If you used the default values it should be something like this:

[sourcecode]; C:\Program Files\PostgreSQL\X.X\bin
[/sourcecode]

Note that the directory address has to be separated by semicolumn (;) from the rest of directories in your system path. Note also that X.X has to be changed to match the version of PostgreSQL you installed.
<h2>Ubuntu</h2>
Ubuntu, and most Linux distributions, include PostgreSQL server in their package repositories. Hence, installing PostgreSQL server is straightforward with the default package manager of the distribution. In Ubuntu, you can use Synaptic to search for the <code>postgresql</code> package and the <code>pgAdminIII</code> package. Installing these two packages should install the server and all necessary dependencies, and setup the required configuration. Here, the installation will create a superuser (or administrator) account called “postgres” in the server and will ask you to specify a password for this user. It is important to provide a password for security reasons and remember it, otherwise you won't be able to make any operation with the server.
<h1>Download EPD data for PostgreSQL</h1>
Now you have to download the lastest version of the EPD for Postgres from <a href="http://www.europeanpollendatabase.net/data/downloads/">http://www.europeanpollendatabase.net/data/downloads/</a>. Save the file <code>epd-postgres-distribution.zip</code> in a specific folder of your choice and unzip the file. Now you should have four files <code>dumpall_epd_db.sql.gz</code>, <code>dump_epd_db_schema_only.sql</code>, <code>dumps_epd_all_tables.zip</code>, and <code>README</code>. The file <code>dumpall_epd_db.sql.gz</code> is also compressed and has to be uncompressed to obtain the file <code>dumpall_epd_db.sql</code>.

[caption id="attachment_538" align="alignnone" width="600"]<img class="alignnone size-full wp-image-538" src="https://dnietolugilde.files.wordpress.com/2018/07/epdwebsite2.png" alt="EPDwebsite2" width="600" height="379" /> EPD website for downloads[/caption]
<h1>Setup PostgreSQL for EPD:</h1>
Before we can dump the EPD data into the server, you need to perform two basic configuration steps:
<ol>
	<li>Create two specific users in the PostgreSQL server</li>
	<li>Create an empty database in the server</li>
</ol>
These two steps can be done from the command line or from the graphical user interface pgAdminIII.
<h2>Command line</h2>
Once you are in the command line, use command <code>cd</code> to change to the same directory you unzipped the database files.

[sourcecode]diego@server:~$ cd epd-postgres-distribution
[/sourcecode]

Then, use the <code>psql</code> command to start working with the PostgreSQL server, which will change the prompt to let you know you are in a PostgreSQL prompt. Specifically, you specify that want to connect to the server with the superuser <code>-U postgres</code> and the address of the server (<code>-h localhost</code>).

[sourcecode]diego@server:~/epd-postgres-distribution$ psql -U postgres -h localhost
Password for user postgres:
psql (9.5.3)
SSL connection (protocol: TLSv1.2, cipher: ECDHE-RSA-AES256-GCM-SHA384, bits: 256, compression:off)
Type "help" for help.

postgres=#
[/sourcecode]

<h3>Create mandatory users for EPD</h3>
First, you should create two specific users in your local PostgreSQL server: <code>wwwadm</code> and <code>epd</code>. These two users are defined in the original EPD database server and are required to properly dump the data into the database in your local server. Users in PostgreSQL are also know as roles. To create a new role for the server we use the <code>CREATE ROLE</code> command, with the argument <code>WITH PASSWORD</code> to specify the desired password for each user. You can use any password of your choice, but do not forget these passwords.

[sourcecode]postgres=# CREATE USER wwwadm WITH PASSWORD 'wwwadm_password';
CREATE ROLE
postgres=# CREATE USER epd WITH PASSWORD 'epd_password';
CREATE ROLE
[/sourcecode]

<h3>Create a database for EPD into the server</h3>
Now, you can create a database for the EPD with the <code>CREATE DATABASE</code> command. In the example, we created a database called <code>epd</code> but you can select a different name if you want. Note that commands in postgresql prompt end with semicolom <code>;</code>.

[sourcecode]postgres=# CREATE DATABASE epd;
CREATE DATABASE
[/sourcecode]

<h2>pgAdminIII</h2>
pgAdminIII is a graphical user interface to manage PostgreSQL servers. It eases basic operations with PostgreSQL databases for no-advanced users that are used to work with the command line. It is also usefull to visually inspect tables in the database. If you want to give it a try or use it instead of the command line, you should look for pgAdminIII in your system and run it.

The program interface have three different parts: a side bar pannel in the left (<code>Object browser</code>) with the structure of the server and all its elements (databases, users, etc) and two pannels at the right. The right upper pannel provides details (<code>Properties</code>, <code>Statistics</code>, <code>Dependencies</code>, and <code>Dependents</code>) for the element that is selected in the left pannel and the right bottom pannel (<code>SQL pane</code>) shows text details of the selected element in SQL language.

[caption id="attachment_539" align="alignnone" width="600"]<img class="alignnone size-full wp-image-539" src="https://dnietolugilde.files.wordpress.com/2018/07/pgadminiii.png" alt="pgAdminIII" width="600" height="452" /> PGAdmin III[/caption]
<h3>Create mandatory users for EPD</h3>
To create new users (or roles) you have to right click on <code>Login Roles</code> element in the <code>Object browser</code> pannel and, then, left click on <code>New Login Role...</code>. A new window popup. In the first tab of the window (<code>Properties</code>) you have to provide a <code>Role name</code>, which is just the name for the new user. Here, you have to write <code>epd</code>. In the next tab (<code>Definition</code>) you have to provide twice the password for the new user. You can use any password of your choice. Do not change any other option in this window and click <code>Ok</code>.

Repeat the same proccess to create a new role called <code>wwwadm</code>.
<h3>Create a database for EPD into the server</h3>
To create a new database you have to right click in <code>Databases</code> element in the <code>Object browser</code> pannel and, then, left click on <code>New Database...</code>. A new window popup. In the first tab of the window (<code>Properties</code>) you have to provide a <code>Name</code> for the database. You can use any name of you choice, but it is a good practice to provide an informative name (e.g., <code>epd</code>). Do not change any other option and click <code>Ok</code>.
<h1>Dump EPD data into the new database</h1>
Dumping EPD data into the DDBB server requires to use the command line, and cannot be accomplished in pgAdminIII. To do so, once you are in the command line, use the <code>cd</code> command to move into the same folder you downloaded and uncompressed the EPD database for PostgreSQL. Then, you use the <code>psql</code> command to connect to the <code>epd</code> database in the PostgreSQL server as the <code>postgres</code> user. In the same line of code you use the <code>&lt;</code> symbol to parse all the <code>dumpall_epd_db.sql</code> script for PostgreSQL into the database you just created (<code>epd</code>).

[sourcecode]diego@server:~$ cd epd-postgres-distribution
diego@server:~/epd-postgres-distribution$ psql -U postgres -h localhost -d epd &lt; dumpall_epd_db.sql
.
.
.
REVOKE
REVOKE
GRANT
GRANT
REVOKE
REVOKE
GRANT
GRANT
REVOKE
REVOKE
GRANT
GRANT
REVOKE
REVOKE
GRANT
GRANT
REVOKE
REVOKE
GRANT
GRANT
diego@server:~/epd-postgres-distribution$
[/sourcecode]

Now, you can use the database from the pgAdminIII software or using the EPDr package for R.
