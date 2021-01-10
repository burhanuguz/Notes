
# Nginx Custom Error Pages
- Custom nginx error pages should be added to a folder and it should be defined then like in below.
```nginx
error_page  403 404 500 502 503 504  /error.html;
location = /error.html {
  root   /usr/share/nginx/nginx_error;
}

location ~ "^/webimages" {
  root  /usr/share/nginx/nginx_error;
}

location ~ "^/webscripts" {
  root  /usr/share/nginx/nginx_error;
}
location ~ "^/webstyles" {
  root  /usr/share/nginx/nginx_error;
}
```
> Test it with 
> ```bash
> nginx -t
> ```
> After that restart the service.
