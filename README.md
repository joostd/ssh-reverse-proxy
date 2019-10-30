# HTTPS-to-HTTP Reverse proxy through an SSH tunnel

This nginx-based reverse proxy terminates HTTPS and proxies to a local server through an SSH tunnel. This is useful when developing a web application on a local system (on a loopback interface or on a local VM), and you need to test your application using HTTPS from a different client.

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

# Install automatically

You can install your proxy using ansible.

	ansible-playbook playbook.yml

or simply

	./playbook.yml

You may need to edit your `ansible.cfg` file when using a different inventory file, or specify one explicitely:

	ansible-playbook playbook.yml -i hosts

## replacing certificates

The ansible `playbook` will generate self-signed certificates. These will not be accepted by browsers without warnings. "Real" certificates can be installed by placing `nginx.crt` and `nginx.key` in the `files` directory.


# SSH Tunneling

To create a tunnel from your reverse proxy's loopback interface to your local server, use your OpenSSH client:

	ssh -R 8080:localhost:8080 -l ubuntu proxy.example.org

Alternatively, edit your `~/.ssh/config` file and add:

	Host proxy
	Hostname proxy.example.org
	User ubuntu
	IdentityFile "~/.ssh/id_rsa.pem"
	RemoteForward 8080 localhost:8080

and simply use

	ssh proxy

## Proxying a Vagrant VM

If the server you want to proxy is running on a local VM, you need to forward port 8080 on your local system to your VM.

In your VM's `Vagrantfile`, you can forward port 8080 to a port on your guest using

	config.vm.network "forwarded_port", guest: 80, host: 8080

## Alternatives

I recently discovered that for limited use, there are free services available, such as:
- [ngrok](https://ngrok.com)
- [serveo](https://serveo.net)
- and [several others](https://www.chenhuijing.com/blog/tunnelling-services-for-exposing-localhost-to-the-web/#🏀)

Also, some suggestions for improvements are described [here](https://dev.to/k4ml/poor-man-ngrok-with-tcp-proxy-and-ssh-reverse-tunnel-1fm).
