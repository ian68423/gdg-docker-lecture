FROM nginx:alpine

# 覆寫預設的 default.conf
COPY nginx.conf /etc/nginx/conf.d/default.conf

# 把網頁放進 nginx 的 web root
COPY index.html /usr/share/nginx/html/

EXPOSE 80

# nginx:alpine 預設 CMD 已經是 nginx -g 'daemon off;'，不用寫