# How to run your own ngrokd server

Running your own ngrok server is really easy! The instructions below will guide you along your way!

## 1. Get an SSL certificate
ngrok provides secure tunnels via TLS, so you'll need an SSL certificate. Assuming you want to create
tunnels on *.example.com, buy a wildcard SSL certificate for *.example.com. Note that if you
don't need to run https tunnels that you don't need a wildcard certificate. (In fact, you can
just use a self-signed cert at that point, see the section on that later in the document).

## 2. Modify your DNS
You need to use the DNS management tools given to you by your provider to create an A
record which points *.example.com to the IP address of the server where you will run ngrokd.

## 3. Compile it
You can compile an ngrokd server with the following command:

	make release-server

Make sure you compile it with the GOOS/GOARCH environment variables set to the platform of
your target server. Then copy the binary over to your server.

## 4. Run the server
You'll run the server with the following command.


	./ngrokd -tlsKey="/path/to/tls.key" -tlsCrt="/path/to/tls.crt" -domain="example.com"

### Specifying your TLS certificate and key
ngrok only makes TLS-encrypted connections. When you run ngrokd, you'll need to instruct it
where to find your TLS certificate and private key. Specify the paths with the following switches:

	-tlsKey="/path/to/tls.key" -tlsCrt="/path/to/tls.crt"

### Setting the server's domain
When you run your own ngrokd server, you need to tell ngrokd the domain it's running on so that it
knows what URLs to issue to clients.

	-domain="example.com"

## 5. Configure the client
In order to connect with a client, you'll need to set two options in ngrok's configuration file.
The ngrok configuration file is a simple YAML file that is read from ~/.ngrok by default. You may specify
a custom configuration file path with the -config switch. Your config file must contain the following two
options.

	server_addr: example.com:4443
	trust_host_root_certs: true

Substitute the address of your ngrokd server for "example.com:4443". The "trust_host_root_certs" parameter instructs
ngrok to trust the root certificates on your computer when establishing TLS connections to the server. By default, ngrok
only trusts the root certificate for ngrok.com.

## 6. Connect with a client
Then, just run ngrok as usual to connect securely to your own ngrokd server!

	ngrok 80

# ngrokd with a self-signed SSL certificate
It's possible to run ngrokd with a a self-signed certificate, but you'll need to recompile ngrok with your signing CA.
If you do choose to use a self-signed cert, please note that you must either remove the configuration value for
trust_host_root_certs or set it to false:

    trust_host_root_certs: false

Special thanks @kk86bioinfo, @lyoshenka and everyone in the thread https://github.com/inconshreveable/ngrok/issues/84 for help in writing up instructions on how to do it:

https://gist.github.com/lyoshenka/002b7fbd801d0fd21f2f
https://github.com/inconshreveable/ngrok/issues/84

# Misc clues

## ngrok as a systemd service

You can write this unit to /etc/systemd/system/ngrok.service :
```
[Unit]
Description=ngrok server

[Service]
ExecStart=/full/path/to/ngrokd -domain=example.com

[Install]
WantedBy=multi-user.target
```

Don't forget to `systemctl reload-daemon` before `systemctl start ngrok`.

It is important not to use quotes in the arguments. You can use other arguments, for instance -httpsArg= to disable https.

## Using an auth_token

If you self host, you probably have a public server, but don't want everyone to be able to use your server to tunnel. You can set an auth_token, that has to be the same on the server and on the client.

On the server, it must be set as an env variable `AUTH_TOKEN`. If you use the unit file above, add a line in the `[Service]` block : `Environment=AUTH_TOKEN=mysecrettoken`.

On the client, it must be set either using ~/.ngrok with `auth_token: mysecrettoken` at the root of the yml, or in the command with `-authtoken mysecrettoken`.

## Using a vhost

If your public address is not the same as the one set in httpArg or httpsArg, you can use the env variable `VHOST`. For instance, you can use `-httpArg=:8000`, `VHOST=ngrok.example.com:80` and a proxy that redirects ngrok.example.com/\* requests to localhost:8000 (it must keep the Host header). The corresponding Apache vhost would be :
```
<VirtualHost *>
        ServerAlias *.ngrok.example.com
        ProxyPreserveHost On
        ProxyPass / http://localhost:8000
        ProxyPassReverse http://localhost:8000 /
</VirtualHost>
```
