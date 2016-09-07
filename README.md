# web development TLS tunnel proxy

## SSH Tunneling

	ssh -R 8080:localhost:8080 -l ubuntu proxy.example.org

Alternatively, edit your `~/.ssh/config` file and add:

	Host proxy
	Hostname proxy.example.org
	User ubuntu
	IdentityFile "~/.ssh/id_rsa.pem"
	RemoteForward 8080 localhost:8080

and simply use

	ssh proxy

# Install manually

## Install nginx

	sudo apt-get install nginx

## Install certificates

	mkdir /etc/nginx/ssl
	sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
		-keyout /etc/nginx/ssl/nginx.key -out /etc/nginx/ssl/nginx.crt
	sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
		-keyout /etc/nginx/ssl/nginx.key \
		-out /etc/nginx/ssl/nginx.crt \
		-subj '/CN=proxy.example.org'
	chmod 640 /etc/nginx/ssl/nginx.key

## Configure nginx to proxy requests over the SSH tunnel

	cp default /etc/nginx/sites-available/default 
	ln -s /etc/nginx/sites-available/default /etc/nginx/sites-enabled/
	service nginx restart

Apart from a standard TLS configuration, nginx is configured as a reverse proxy for  your development server on port 8080 (through the SSH tunnel): 

	location / {
		proxy_pass http://127.0.0.1:8080;
		proxy_set_header Host $host:443;
		proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
		proxy_set_header X-Forwarded-Proto $scheme;
	}

The `X-Forwarded-Proto` header can be used to signal the use of https to the proxied server. Support for this header varies. When running an apache on your local server, you can propagate this information to your application using:

	SetEnvIf X-Forwarded-Proto https HTTPS=on
