# TRINUG 2019-06-05

## Lab: docker-compose

To view slides local website:

```bash
docker run --name lab-nginx -p 8080:80 -v $(pwd)/export:/usr/share/nginx/html:ro -d nginx
```