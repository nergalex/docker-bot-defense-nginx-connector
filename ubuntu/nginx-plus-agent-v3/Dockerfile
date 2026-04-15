ARG RELEASE=24.04
FROM ubuntu:${RELEASE} as nginx-installer

LABEL maintainer="F5 SE"

# Define NGINX versions for NGINX Plus and NGINX Plus modules
ARG NGINX_AGENT_VERSION=3.4.2~noble

# Set the debconf frontend to Noninteractive
RUN echo 'debconf debconf/frontend select Noninteractive' | debconf-set-selections

# Install prerequisite packages
RUN  apt-get update \
    && apt-get install --no-install-recommends --no-install-suggests -y \
                        ca-certificates \
                        lsb-release \
                        curl \
                        sudo \
                        vim  \
                        procps  \
                        gnupg  \
                        gnupg1 \
                        gnupg2 \
                        binutils  \
                        net-tools  \
                        jq  \
                        tcpdump \
                        apt-utils \
                        apt-transport-https \
                        distro-info-data \
                        gettext-base \
                        brotli \
                        wget \
                        git \
                        libxml2 \
                        gettext-base \
                        ubuntu-keyring \
                        dnsutils \
                        iproute2 \
                        inotify-tools \
                        luarocks \
                        luajit

# Install Lua package to make http request
RUN luarocks install lua-resty-http

# Get other files required for NGINX installation
RUN wget -q -O - http://nginx.org/keys/nginx_signing.key | apt-key add -
RUN wget -q -O /etc/apt/apt.conf.d/90nginx https://cs.nginx.com/static/files/90nginx
RUN printf "deb https://plus-pkgs.nginx.com/ubuntu `lsb_release -cs` nginx-plus\n" >/etc/apt/sources.list.d/nginx-plus.list

# Update the repository and install the most recent version of the NGINX Plus
RUN --mount=type=secret,id=nginx-crt,dst=/etc/ssl/nginx/nginx-repo.crt,mode=0644 \
    --mount=type=secret,id=nginx-key,dst=/etc/ssl/nginx/nginx-repo.key,mode=0644 \
    apt-get update \
    && apt-get install -y nginx-plus

# Install the Lua module package nginx-plus-module-lua from the official NGINX Plus
RUN apt install nginx-plus-module-lua

# Remove any previous NGINX Plus repository and apt configuration files:
RUN rm /etc/apt/sources.list.d/nginx-plus.list \
    && rm /etc/apt/apt.conf.d/90pkgs-nginx

FROM nginx-installer as nginx-agent-installer

# Download and add NGINX signing key:
RUN wget -qO - https://cs.nginx.com/static/keys/nginx_signing.key | gpg --dearmor | sudo tee /usr/share/keyrings/nginx-archive-keyring.gpg >/dev/null

# Create apt configuration /etc/apt/apt.conf.d/90pkgs-nginx
RUN echo "Acquire::https::pkgs.nginx.com::Verify-Peer \"true\";" > /etc/apt/apt.conf.d/90pkgs-nginx \
    && echo "Acquire::https::pkgs.nginx.com::Verify-Host \"true\";" >> /etc/apt/apt.conf.d/90pkgs-nginx \
    && echo "Acquire::https::pkgs.nginx.com::SslCert     \"/etc/ssl/nginx/nginx-repo.crt\";" >> /etc/apt/apt.conf.d/90pkgs-nginx \
    && echo "Acquire::https::pkgs.nginx.com::SslKey      \"/etc/ssl/nginx/nginx-repo.key\";" >> /etc/apt/apt.conf.d/90pkgs-nginx

# Add the nginx-agent repository
RUN echo "deb [signed-by=/usr/share/keyrings/nginx-archive-keyring.gpg] \
    https://pkgs.nginx.com/nginx-agent/ubuntu/ `lsb_release -cs` agent" \
  | sudo tee /etc/apt/sources.list.d/nginx-agent.list

# Install NGINX agent
RUN --mount=type=secret,id=nginx-crt,dst=/etc/ssl/nginx/nginx-repo.crt,mode=0644 \
    --mount=type=secret,id=nginx-key,dst=/etc/ssl/nginx/nginx-repo.key,mode=0644 \
    apt-get update \
    && apt-get install -y nginx-agent=$NGINX_AGENT_VERSION

FROM nginx-agent-installer as nginx-initial-config
RUN mkdir -p /nginx-initial-config
COPY nginx.conf /nginx-initial-config/

# Add NGINX User to nginx-agent Group
RUN groupadd nginx-agent \
    && usermod -a -G nginx-agent nginx \
    && usermod -a -G nginx-agent root

RUN cp /etc/nginx/uwsgi_params /nginx-initial-config/ \
    && cp /etc/nginx/fastcgi_params /nginx-initial-config/ \
    && cp /etc/nginx/mime.types  /nginx-initial-config/ \
    && cp /etc/nginx/scgi_params /nginx-initial-config/ \
    && chmod -R g+rw /nginx-initial-config/ \
    && chown -R :nginx-agent /nginx-initial-config/

FROM nginx-initial-config as nginx-agent-initial-config
RUN chown -R :nginx-agent /etc/nginx-agent \
    && chmod -R g+rw /etc/nginx-agent \
    && chown -R :nginx-agent /opt \
    && chmod -R g+rw /opt \
    && chown -R :nginx-agent /var/run/nginx-agent  \
    && chmod -R g+rw /var/run/nginx-agent \
    && chown -R :nginx-agent /var/lib/nginx-agent  \
    && chmod -R g+rw /var/lib/nginx-agent \
    && chown -R :nginx-agent /etc/nginx  \
    && chmod -R g+rw /etc/nginx

FROM nginx-agent-initial-config as cleaner
# cleanup sensitive nginx-plus data
RUN apt-get autoremove --purge -y \
      gnupg \
      apt-transport-https \
  && rm -rf /root/.gnupg \
  && rm -rf /var/lib/apt/lists/* \
  && apt-get remove --purge -y lsb-release \
  && apt-get remove --purge --auto-remove -y && rm -rf /var/lib/apt/lists/* /etc/apt/sources.list.d/nginx-plus.list \
  && rm -rf /etc/apt/apt.conf.d/90pkgs-nginx /etc/ssl/nginx

FROM cleaner as runner
# Forward request logs to Docker log collector
RUN ln -sf /dev/stdout /var/log/nginx/access.log \
  && ln -sf /dev/stderr /var/log/nginx/error.log \
  && ln -sf /dev/stdout /var/log/nginx-agent/agent.log \
  && chmod -R g+rw /var/log/ \
  && chown -R :nginx-agent /var/log/

COPY entrypoint.sh /
COPY remove.sh /

# App services
EXPOSE 8000
EXPOSE 4443
# HEALTHCHECK
EXPOSE 8081
# NGINX-agent API
EXPOSE 8032
# NGINX API
EXPOSE 9000
# NGINX sync zones
EXPOSE 9001
STOPSIGNAL SIGTERM
# USER nginx
ENTRYPOINT ["bash", "/entrypoint.sh"]
