ARG IMAGE_REPO
FROM ${IMAGE_REPO:-lagoon}/commons as commons
FROM alpine:latest

MAINTAINER amazee.io

# Copying commons files
COPY --from=commons /lagoon /lagoon
COPY --from=commons /bin/fix-permissions /bin/ep /bin/docker-sleep /bin/
COPY --from=commons /sbin/tini /sbin/
COPY --from=commons /home /home

RUN chmod g+w /etc/passwd \
    && mkdir -p /home

# When Bash is invoked via `sh` it behaves like the old Bourne Shell and sources
# a file that is given in `ENV`. When Bash is invoked as non-interactive (like
# `bash -c command`) it sources a file that is given in `BASH_ENV`.
ENV TMPDIR=/tmp TMP=/tmp HOME=/home ENV=/home/.bashrc BASH_ENV=/home/.bashrc

ENV BACKUPS_DIR="/var/lib/mysql/backup"

ENV MARIADB_DATABASE=lagoon \
    MARIADB_USER=lagoon \
    MARIADB_PASSWORD=lagoon \
    MARIADB_ROOT_PASSWORD=Lag00n

# fix some environmental items
RUN mkdir -p /usr/include/bits; mkdir -p /usr/
COPY root/usr/include/bits/wordsize.h /usr/include/bits/wordsize.h

RUN \
    apk add --no-cache --virtual .common-run-deps \
    bash \
    curl \
    mariadb \
    mariadb-client \
    mariadb-common \
    net-tools \
    pwgen \
    tzdata \
    wget; \
    rm -rf /tmp/* /var/tmp/* /var/cache/apk/* /var/cache/distfiles/*; \
    rm -rf /var/lib/mysql/* /etc/mysql/; \
    curl -sSL http://mysqltuner.pl/ -o mysqltuner.pl


COPY logging.bash           /lagoon/entrypoints/10-mariadb-logging
COPY docker-entrypoint.bash /lagoon/entrypoints/9999-mariadb-entrypoint
COPY mysql-backup.sh /lagoon/
COPY my.cnf /etc/mysql/my.cnf

RUN for i in /var/run/mysqld /var/lib/mysql /etc/mysql/conf.d /docker-entrypoint-initdb.d/ "${BACKUPS_DIR}"; \
        do mkdir -p $i; chown mysql $i; /bin/fix-permissions $i; \
    done && \
    ln -s /var/lib/mysql/.my.cnf /home/.my.cnf && \
    sed -i 's/#!\/bin\/bash -ue/#!\/bin\/bash -e/' /usr/bin/wsrep_sst_rsync

COPY root/usr/share/container-scripts/mysql/readiness-probe.sh /usr/share/container-scripts/mysql/readiness-probe.sh
RUN /bin/fix-permissions /usr/share/container-scripts/mysql/
RUN touch /var/log/mariadb-slow.log && /bin/fix-permissions /var/log/mariadb-slow.log
RUN touch /var/log/mariadb-queries.log && /bin/fix-permissions /var/log/mariadb-queries.log

# We cannot start mysql as root, we add the user mysql to the group root and
# change the user of the Docker Image to this user.
RUN addgroup mysql root
USER mysql
ENV USER_NAME mysql

WORKDIR /var/lib/mysql
VOLUME /var/lib/mysql
EXPOSE 3306

ENTRYPOINT ["/sbin/tini", "--", "/lagoon/entrypoints.bash"]
CMD ["mysqld"]
