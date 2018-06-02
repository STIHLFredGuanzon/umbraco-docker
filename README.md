# Umbraco in Docker

## Overview

* Single-instance for administration and scheduling (aka "master server")
* Multiple load-balanced instances (aka "front-end" servers)
* All instances point at the same SQL container
* Static content is stored on EFS
* Explicit configuration of master server to avoid complexities of auto-configuration

## Starting from Scratch

When starting a new site, you need to get your SQL Server container running, and then run the Umbraco startup wizard on the host to establish the SQL connection string and machine key.  At that point, you can create a Docker image.  When you create the master/slave images, you will set the `LoadBalancer.*` web.config settings appropriately.

`>> todo: set in Dockerfile based on env var`

### Run SQL Container

Put your strong `sa` account password in the **sa_password** environment variable, start the SQL container, and inspect to find its IP address.

```bash
export sa_password=MySuperStrongPassword
docker-compose up sql
docker inspect umbraco-docker_sql_1
```

Use the IP address and `sa` account to connect SSMS and create a new, empty database named **umbraco** with data and log files in **C:\data**.

### Initialize Umbraco

Download [Umbraco](https://umbraco.com/) and unzip to **src/umbraco/site**.  [Run the site in IIS Express](https://our.umbraco.org/documentation/Getting-Started/Setup/Install/install-umbraco-with-vs-code) (or however you'd like) and proceed through the setup wizard.

At the first step, select **Customize**.  Next, select **Custom connection string** and enter the following, substituting the SQL container IP address and your `sa` password:

```bash
Server=123.123.123.123;Database=umbraco;User Id=sa;Password=MySuperStrongPassword;MultipleActiveResultSets=True
```

**Important!** Allow Umbraco to generate an ASP.NET machine key for your site.  Installing a starter kit site is optional.

### Create Umbraco Docker Image

We had to use an IP address in the SQL connection string for Umbraco to access that container from the host.  Before making the Docker image, we need to update the web.config to replace that IP with the docker container name.

Open the web.config and change the connection string to:

```bash
Server=umbraco-docker_sql_1;Database=umbraco;User Id=sa;Password=MySuperStrongPassword;MultipleActiveResultSets=True
```

Now build the Umbraco image:

```bash
docker build -t umbraco-admin ./src/umbraco
```

### Run Everything in Docker

If the SQL container is still running, stop it:

```bash
docker stop umbraco-docker_sql_1
```

Now you can use docker-compose to start both containers, and then inspect to find the Umbraco container IP address:

```bash
docker-compose up
docker inspect umbraco-docker_rendering_1
```

Navigate to http://**{umbraco-ip}**:8000 to confirm the site is working.

## References

* [Umbraco flexible load balancing](https://our.umbraco.org/documentation/getting-started/setup/server-setup/load-balancing/flexible)
* [Explicit master scheduling server](https://our.umbraco.org/documentation/getting-started/setup/server-setup/load-balancing/flexible-advanced#explicit-master-scheduling-server)