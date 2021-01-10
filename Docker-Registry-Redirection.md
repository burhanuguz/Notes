
# Proxy Default Docker Registry to your own Registries (Workaround method)
- By default docker daemon pulls images from docker.io site like down below. 
```bash
docker pull nginx                         ###     By default it will be pulled from docker.io    ###
docker pull my-private-registry.com/nginx ### If it is wanted to be pulled other than Docker Hub ###
```
- It can be changed with reverse proxies, like nginx. To do that certificate should be created.
```bash
openssl req -newkey rsa:2048 -nodes -keyout key.pem -x509 -days 365 -out certificate.pem
```
![1](https://user-images.githubusercontent.com/59168275/104118122-969b5100-5337-11eb-9580-42ff15f64b70.png)
> Except from **Common Name** part, there is no important pont. **Common Name** will be the DNS address of the registry.
- Registry certificate should be trusted like below.
 ```bash
cp registery.cert /etc/pki/ca-trust/source/anchors/
update-ca-trust extract
```
- Nginx **server** block should be defined to make reverse-proxy like down below.
```nginx
server {
    listen       443 ssl;
    server_name  registry-1.docker.io;

    location / {
	    proxy_pass http://my-private-registry.com;
    }
    ssl_certificate "/etc/nginx/certs/certificate.pem";
    ssl_certificate_key "/etc/nginx/certs/key.pem";
    error_page 500 502 503 504 /50x.html;
        location = /50x.html {
    }
}
```
>In this example I did reverse-proxy to my own Nexus Repository.
- After that start nginx.
```bash
systemctl start nginx
```
- **docker daemon** service should be restarted to trust the certificate.
- Write hosts file the DNS address of the registry. In this example localhost is chose because Nginx is listening from localhost too.
```bash
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4 registry-1.docker.io
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
```
- Testing if it is working or not.
```bash
$ ping registry-1.docker.io
PING localhost (127.0.0.1) 56(84) bytes of data.
64 bytes from localhost (127.0.0.1): icmp_seq=1 ttl=64 time=0.097 ms
64 bytes from localhost (127.0.0.1): icmp_seq=2 ttl=64 time=0.075 ms

$ curl https://registry-1.docker.io
<html>
<head>
<meta http-equiv="Content-Type" content="text/html;charset=utf-8"/>
<title>Error 400 Not a Docker request</title>
</head>
<body><h2>HTTP ERROR 400</h2>
<p>Problem accessing /. Reason:
<pre>    Not a Docker request</pre></p><hr><a href="http://eclipse.org/jetty">Powered by Jetty:// 9.4.18.v20190429</a><hr/>

</body>
</html>
```
![2](https://user-images.githubusercontent.com/59168275/104118189-322cc180-5338-11eb-8e1e-d83181193536.png)

- Now it can be tested again and can be seen if it is working or not like down below.
![3](https://user-images.githubusercontent.com/59168275/104118190-32c55800-5338-11eb-9781-38e1b50287a3.png)

- It can be tested with nginx image to be pushed to registry and pulling it from the registry again.
```bash
docker pull nginx
docker tag nginx master.registry/nginx
docker push master.registry/nginx
docker rmi nginx master.registry/nginx
docker pull master.registry/nginx
```
![5](https://user-images.githubusercontent.com/59168275/104117611-87b29f80-5333-11eb-82cd-bc115f5d4e26.png)

# Proxy Well-known Registries to your own Registries (Workaround method)

- If other than docker.io registry needs to be proxied, you should have one certificate and as many as reverse proxy address to proxy them.
- To self sign certificate with more than one DNS, we need to create our own **Root** certificate. 
```bash
openssl req -newkey rsa:2048 -nodes -keyout RootCA.key -new -x509 -days 99999 -out RootCA.pem
```
- To create a certificate with SANs or with more DNS lets say, we need to create a configuration file for openssl like down below. It will be named as **registry.cnf**
```ini
[ req ]
default_bits       = 2048
distinguished_name = req_distinguished_name
req_extensions     = req_ext
[ req_distinguished_name ]
countryName                 = Country Name (2 letter code)
stateOrProvinceName         = State or Province Name (full name)
localityName                = Locality Name (eg, city)
organizationName            = Organization Name (eg, company)
commonName                  = Common Name (e.g. server FQDN or YOUR name)
[ req_ext ]
subjectAltName = @alt_names
[alt_names]
DNS.1 = registry-1.docker.io
DNS.2 = *.quay.io
DNS.3 = k8s.gcr.io
DNS.4 = docker.elastic.co
DNS.5 = docker.io
DNS.6 = mcr.microsoft.com
DNS.7 = gcr.io
DNS.8 = registry.redhat.io
DNS.9 = quay.io
DNS.10 = registry.redhat.io
```
- With newly created **registry.cnf** a certificate signing request (.csr)
and key file can be generated.
```bash
openssl req -newkey rsa:2048 -nodes -keyout docker-proxy.key -new -out docker-proxy.csr -config registry.cnf -extensions req_ext
```
- You will be asked to fill infos like you did before. Below down you will see an example.
```bash
$ openssl req -newkey rsa:2048 -nodes -keyout docker-proxy.key -new -out docker-proxy.csr -config registry.cnf -extensions req_ext
Generating a 2048 bit RSA private key
.....................................................................+++
.....................................................................................................................................................................................................................................................................................................+++
writing new private key to 'docker-proxy.key'
-----
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) []:
State or Province Name (full name) []:
Locality Name (eg, city) []:
Organization Name (eg, company) []:
Common Name (e.g. server FQDN or YOUR name) []:docker-proxy
```
- There will be docker-proxy.csr and .key file after successfull creation.
```bash
$ ll docker-proxy.*
-rw-r--r-- 1 root root 1058 Feb 17 10:13 docker-proxy.csr
-rw-r--r-- 1 root root 1704 Feb 17 10:13 docker-proxy.key
```
- With our own CA .csr will be signed and certificate will be created.
```bash
openssl x509 -req -days 9999 -in docker-proxy.csr -CA RootCA.pem -CAkey RootCA.key -set_serial 0101 -extfile registry.cnf -extensions req_ext -out docker-proxy.pem
```
- With the command below you can check wheter it is properly created or not.
```bash
openssl x509 -req -days 9999 -in docker-proxy.csr -CA RootCA.pem -CAkey RootCA.key -set_serial 0101 -extfile registry.cnf -extensions req_ext -out docker-proxy.pem
Signature ok
subject=/CN=docker-proxy
Getting CA Private Key
```
- And with final control you can check wheter your newly created certificate has proper SANs or not.
```bash
openssl req -in docker-proxy.csr -noout -text | grep DNS
                DNS:registry-1.docker.io, DNS:*.quay.io, DNS:k8s.gcr.io, DNS:docker.elastic.co, DNS:docker.io, DNS:mcr.microsoft.com, DNS:gcr.io, DNS:registry.redhat.io, DNS:quay.io
```
- Registry certificate and ROOTCA should be trusted like below.
 ```bash
cp RootCA.pem docker-proxy.pem /etc/pki/ca-trust/source/anchors/
update-ca-trust extract
```
- Nginx server block would be edited like below.
```nginx
server {
    listen       443 ssl;
    server_name  registry-1.docker.io *.quay.io k8s.gcr.io docker.elastic.co docker.io mcr.microsoft.com gcr.io registry.redhat.io quay.io;

    location / {
    proxy_pass http://my-private-registry.com;
    }
    ssl_certificate "/etc/nginx/certs/docker-proxy.pem";
    ssl_certificate_key "/etc/nginx/certs/docker-proxy.key";
    error_page 500 502 503 504 /50x.html;
        location = /50x.html {
    }
}
```
- Nginx and docker should be restarted.
```bash
sudo systemctl restart nginx 
sudo systemctl restart docker
```
- After that you can succesfully pull any image you want.
