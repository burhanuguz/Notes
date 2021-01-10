# Secure Docker Registry

- Create self-signed certificate first.
```bash
openssl req -newkey rsa:4096 -nodes -sha256 -keyout client.key -x509 -days 99999 -out client.crt
```
![1](https://user-images.githubusercontent.com/59168275/104117415-31912c80-5332-11eb-9aaa-0dd0d1b22823.png)
> Except from **Common Name** part, there is no important pont. **Common Name** will be the DNS address of the registry.
- Registry certificate should be trusted like below.
 ```bash
cp registery.cert /etc/pki/ca-trust/source/anchors/
update-ca-trust extract
```
- After that **docker daemon** service should be restarted to trust the certificate.
- Write hosts file the DNS address of the registry. In this example localhost is chose.
![2](https://user-images.githubusercontent.com/59168275/104117494-c85de900-5332-11eb-8be3-3ed27aabf16b.png)
- In case another client wants to reach this registry, use your own DNS or just simply write it down like that to hosts file.
![3](https://user-images.githubusercontent.com/59168275/104117495-c98f1600-5332-11eb-9027-fde03ba77cad.png)
- Start registry with volumes to prevent the images from being deleted.
```bash
docker run -d \
--restart=always \
--name registry \
-v /path/to/certs:/certs \
-v /path/to/volume:/var/lib/registry/ \
-e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/registry.cert \
-e REGISTRY_HTTP_TLS_KEY=/certs/registry.key \
-e REGISTRY_HTTP_ADDR=0.0.0.0:443 \
-p 443:443 \
registry
```
![4](https://user-images.githubusercontent.com/59168275/104117573-3d312300-5333-11eb-8655-5ed5a0dac05e.png)

- It can be tested with nginx image to be pushed to registry and pulling it from the registry again.
```bash
docker pull nginx
docker tag nginx master.registry/nginx
docker push master.registry/nginx
docker rmi nginx master.registry/nginx
docker pull master.registry/nginx
```
![5](https://user-images.githubusercontent.com/59168275/104117611-87b29f80-5333-11eb-82cd-bc115f5d4e26.png)

# Docker Registry with Credentials

- If needed **Docker Registry** can be used with authentication, though it is a simple authentication method it is better than nothing.
- User and password will be getting from **htpasswd** like down below.
```bash
docker run --entrypoint htpasswd registry -Bbn user password > /path/to/htpasswd
```
- **user:password** pair is created, now with a little tweak registry can be created again.
```bash
docker run -d \
--restart=always \
--name registry \
-v /path/to/auth:/auth \
-v /etc/docker/certs.d/master.registry:/certs \
-v /path/to/volume:/var/lib/registry/ \
-e "REGISTRY_AUTH_HTPASSWD_REALM=Registry Realm" \
-e REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd \
-e REGISTRY_AUTH=htpasswd \
-e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/registry.cert \
-e REGISTRY_HTTP_TLS_KEY=/certs/registry.key \
-e REGISTRY_HTTP_ADDR=0.0.0.0:443 \
-p 443:443 \
registry
```
- After it starts, you can login like down below and do the testing again.
```bash
docker login master.registry
```
![6](https://user-images.githubusercontent.com/59168275/104117662-10c9d680-5334-11eb-879c-b879eca615ac.png)
