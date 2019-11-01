# Systems Puzzle

*Note:* I knew what the `Docker` and `Flask` are, but had never used them before. Hence, some of the solution steps are not optimal or expected ones.

Following installation and startup instructions I should have `nginx` mapped to the port 8080 and get some response
in the browser or telnet connection, but the port was not opened:

	F:\sp>telnet localhost 8080
	Connecting To localhost...Could not open connection to the host, on port 8080: Connect failed

It means that either `nginx` is not running, or another port is mapped. Simple check showed that external port 80 was mapped to the internal 8080:

	F:\sp>docker ps
	CONTAINER ID   IMAGE            COMMAND                 CREATED              STATUS              PORTS                          NAMES
	046b2d0ce837   nginx:1.13.5     "nginx -g 'daemon of"   16 seconds ago       Up 14 seconds       80/tcp, 0.0.0.0:80->8080/tcp   sp_nginx_1
	249dc76836ed   sp_flaskapp      "python app.py"         19 seconds ago       Up 16 seconds       5001/tcp                       sp_flaskapp_1
	b928d7756d41   postgres:9.6.5   "docker-entrypoint.s"   About a minute ago   Up About a minute   5432/tcp                       sp_db_1
	
Based on this information, I edited `docker-compose.yml` replacing `nginx` ports mapping with `8080:80` and restarted the container.
Now I got a response `502 Bad Gateway` from `nginx`, which means that there is no response (or response is invalid) from the downstream `Flask` daemon. 

Running `flaskapp` container in a regular mode, I saw in the output that it is listening port 5000,
and not the port 5001 exposed by the `Dockerfile` and written in the `nginx` configuration file:

	flaskapp_1  |  * Running on http://0.0.0.0:5000/ (Press CTRL+C to quit)

Possible solution was either to fix the port number in `Dockerfile` and `flaskapp.conf` or to tell `Flask` framework to listen another port at `app.run(host='0.0.0.0', port = 5001)`.
I chose not to modify the original python source code, exposing 5000 port and proxying `nginx` requests to `proxy_pass http://flaskapp:5000;`

At this step, I received a response as a form, where I put some information and was redirected to the wrong server address `http://localhost,localhost:8080/success`

A fast check of the python application confirmed that it does not modify the request headers and there is something wrong with the `nginx` configuration. I found that config sets two `Host` headers, combining them into a wrong one with a comma sign. The line `proxy_set_header Host $host;` should be either removed or commented.

The next run gave me an output in the form of `[<models.Items object at 0x7fdcb1356b20>, <models.Items object at 0x7fdcb1356c40>]`, telling that there are two records in table `items` corresponding to the `Items` model. In order to see the content of the objects, `success()` function has to use an additional template tabulating the data or to iterate through the models creating a readable string. I decided to check the content of the DB directly by logging into the `sp_db_1` container:

    F:\sp>docker exec -it b928 /bin/bash
    root@b928d7756d41:/# psql flaskapp_db docker_pg
	
	flaskapp_db=# select * from items;
    id |  name  | quantity | description |         date_added
    ---+--------+----------+-------------+----------------------------
     1 | abcd   |        1 | qwert       | 2019-10-31 22:12:10.109295
     2 | test   |      123 | tset        | 2019-10-31 22:16:20.506947
    (2 rows)

Now everything is working the way it should be.
