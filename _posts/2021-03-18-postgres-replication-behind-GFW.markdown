---
layout: post
title: Postgres replication behind China's GFW
date:   2021-03-18 15:50:22 +0800
categories: GFW bucardo postgres
---

A draft on how to replicate Postgres database through China's Great Firewall without breaking the law.
# Requirements and issues

I had the need to setup a master to master Postgres replication system in a very specific environment.

1. One of the server is located in mainland China behind the GFW
1. The other server is running on a managed database from a famous cloud provider outside China.

The main issue is that our Chinese office often has trouble communicating with the network located outside the GFW. Sometimes the connection might drop or it will be just impossible to access our external cloud provider. China has strict regulation regarding the use of VPN/Proxy and it was important to abide the law. Hopefully we had a server in another office in a China-friendly environment (CN2). You can replicate this setup by renting a CN2 VPS outside mainland China. They are quite expensive but they work well.

{:refdef: style="text-align: center;"}
![Simple server diagram](/assets/img/2021-03-18-diagram.png)
{: refdef}

For the replication we decided to use [Bucardo](https://bucardo.org/Bucardo/). It's easy to use and seems to work pretty well for our needs.

The biggest downside of this solution is that Bucardo needs SUPER USER permission to you Postgres server. This did not work with our managed database  since the administrator did not grant us the SUPER USER access. You may get around this limitation if your provider enables the pgperl extension. For those want to go that way, you may [read this article](https://www.compose.com/articles/using-bucardo-5-3-to-migrate-a-live-postgresql-database/). 

Another solution would be to run your own Postgres server. It is simple to setup but you will need to optimise and secure it yourself . It might not be suitable for everyone and might end up increasing your workload.


# Bucardo installation

The idea is to install bucardo on the CN2 friendly server. You don't need to replicate the database on the Bucardo server. It is a standalone application.

I decided to create a Docker image with Bucardo from an existing Postgres docker image. If you do not need to need to replicate the data on the CN2 VPS you do not need to install Postgres in your docker image. You could also make 2 images, one for Postgres and one for bucardo or you could install them directly on the server without using docker.

I am still testing this solution, I will upgrade the architecture later before going to production. This is the Dockerfile I used for testing purpose:

```
FROM postgres:11-alpine

RUN echo http://dl-cdn.alpinelinux.org/alpine/v3.10/main > /etc/apk/repositories \
    && apk add --no-cache \
    git\
    gcc \ 
    make \
    build-base \
    build-base make \
    perl \
    perl-dbd-pg \
    perl-dev \
    postgresql-plperl\
    postgresql-plperl-contrib\
    postgresql-client\
    && rm -fr /var/cache/apk/* \
    && mv /usr/lib/postgresql/* /usr/local/lib/postgresql/ \
    && mv /usr/share/postgresql/extension/* /usr/local/share/postgresql/extension/
    && mkdir /var/run/bucardo\
    && mkdir /var/log/bucardo\
    && ln -s /var/run/postgresql/.s.PGSQL.5432 /tmp/.s.PGSQL.5432

# Install Bucardo
RUN git clone git://github.com/bucardo/bucardo.git
WORKDIR /bucardo
RUN perl Makefile.PL
RUN make && make install
RUN bucardo --version

# Install DBIXSafe
RUN git clone git://github.com/bucardo/dbixsafe.git /dbixsafe
WORKDIR /dbixsafe
RUN perl Makefile.PL
RUN make install

RUN perl -MCPAN -e 'install DBI'
```

You can use an entrypoint script to make bucardo default install or you can run bash in your container. I will write below the command I used to set it up, it's up to you to decide how to implement it.

```bash
bucardo install --db-user $POSTGRES_USER --db-pass $POSTGRES_PASSWORD --dbname postgres
```

You can use `yes` if you use it in an entrypoint to bypass the confirmation.

```bash
yes P | COMMAND ABOVE
```

Then add your databases. E.g for 3 databases (2 masters, 1 slave):

```bash
# Add first server database
bucardo add database srv1_db dbname=db dbhost=X.X.X.X dbport=5432 dbuser=myuser dbpass=mypassword

# Add second server database
bucardo add database srv2_db dbname=db dbhost=X.X.X.X dbport=5432 dbuser=mysuer dbpass=mypassword

# Add third server database
bucardo add database srv3_db dbname=db dbhost=X.X.X.X dbport=5432 dbuser=myuser dbpass=mypassword

# Create a group. "source" is used for master replication and target for "slave"
bucardo add dbgroup db_group srv1_db:source srv2_db:source srv3_db:target

bucardo add all tables herd=db_herd

bucardo add sync db_sync herd=db_herd dbs=db_group
```

If you did not have any error you can start Bucardo with ```bucardo start``` and check its status with ```bucardo status```

It works pretty well so far.