# TSQA 2020-09-24

## Lab: docker-compose

To view slides local website:

```bash
docker run --name lab-nginx -p 8000:80 -v $(pwd)/export:/usr/share/nginx/html:ro -d nginx
```