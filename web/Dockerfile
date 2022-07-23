# Base image
ARG BASE_IMAGE=almalinux:latest
FROM $BASE_IMAGE as base

# Some packages require building, so use different stage for that
FROM base as builder
RUN dnf install -y --setopt=tsflags=nodocs --setopt=install_weak_deps=False gcc make && \
    useradd --create-home --system --user-group build

# Build su-exec
COPY su-exec.c /tmp/
RUN gcc -Wall -Werror -g -o /usr/local/bin/su-exec /tmp/su-exec.c && \
    chmod u+x /usr/local/bin/su-exec

# Build PHP extensions that are not included in packages
FROM builder as php-build
COPY bin/misp_compile_php_extensions.sh /build/
RUN --mount=type=tmpfs,target=/tmp \
    dnf module enable -y php:7.4 && \
    dnf install https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm -y && \
    bash /build/misp_compile_php_extensions.sh

# MISP image
FROM base as misp

# Install required system and Python packages
COPY packages /tmp/packages
COPY requirements.txt /tmp/
RUN dnf install https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm -y && \
    dnf module -y enable mod_auth_openidc php:7.4 python39 && \
    dnf install --setopt=tsflags=nodocs --setopt=install_weak_deps=False -y $(grep -vE "^\s*#" /tmp/packages | tr "\n" " ") && \
    alternatives --set python3 /usr/bin/python3.9 && \
    pip3 --no-cache-dir install --disable-pip-version-check -r /tmp/requirements.txt && \
    rm -f /etc/httpd/conf.d/ssl.conf && \
    rm -rf /var/cache/dnf /tmp/packages

# /usr/libexec/httpd-ssl-gencerts && \
# rm -f /etc/httpd/conf.d/ssl.conf && \

COPY --from=builder /usr/local/bin/su-exec /usr/local/bin/
COPY --from=php-build /build/php-modules/* /usr/lib64/php/modules/
COPY bin/ /usr/local/bin/
COPY misp.conf /etc/httpd/conf.d/misp.conf
COPY httpd-errors/* /var/www/html/
COPY rsyslog.conf /etc/
COPY snuffleupagus-misp.rules /etc/php.d/
COPY supervisor.ini /etc/supervisord.d/misp.ini
COPY logrotate/* /etc/logrotate.d/
COPY crontab.txt /etc/crontab

ARG CACHEBUST=1
ARG MISP_VERSION=develop
ENV MISP_VERSION $MISP_VERSION

RUN chmod u=rwx,g=rx,o=rx /usr/local/bin/* && \
    /usr/local/bin/misp_install.sh

COPY Config/* /var/www/MISP/app/Config/
RUN chmod u=r,g=r,o=r /var/www/MISP/app/Config/* && \
    chmod 644 /etc/supervisord.d/misp.ini && \
    chmod 644 /etc/rsyslog.conf && \
    chmod 644 /etc/httpd/conf.d/misp.conf && \
    chmod 644 /etc/php.d/snuffleupagus-misp.rules && \
    chmod 644 /etc/logrotate.d/* && \
    chmod 644 /etc/crontab && \
    mkdir /run/php-fpm

# Verify image
FROM misp as verify
RUN touch /verified && \
    su-exec apache /usr/local/bin/misp_verify.sh

# Final image
FROM misp
# Hack that will force run verify stage
COPY --from=verify /verified /

ENV GNUPGHOME /var/www/MISP/.gnupg

VOLUME /var/www/MISP/app/tmp/logs/
VOLUME /var/www/MISP/app/files/certs/
VOLUME /var/www/MISP/app/attachments/
VOLUME /var/www/MISP/.gnupg/

WORKDIR /var/www/MISP/

# Web server
EXPOSE 80
EXPOSE 443

# ZeroMQ
EXPOSE 50000

# This is a hack how to go through mod_auth_openidc
HEALTHCHECK CMD su-exec apache curl -H "Authorization: dummydummydummydummydummydummydummydummy" --fail http://127.0.0.1/fpm-status || exit 1
ENTRYPOINT ["/usr/local/bin/docker-entrypoint.sh"]
CMD ["supervisord", "-c", "/etc/supervisord.conf"]
