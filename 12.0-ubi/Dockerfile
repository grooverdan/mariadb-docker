FROM redhat/ubi9-minimal

# user 999/ group 999, that we want to use for compatibility with the ubuntu image.
RUN groupadd --gid 999 -r mysql && \
	useradd -r -g mysql mysql --home-dir /var/lib/mysql --uid 999

ENV GOSU_VERSION 1.17
RUN set -eux; \
	rpmArch="$(rpm --query --queryformat='%{ARCH}' rpm)"; \
	case "$rpmArch" in \
		aarch64) dpkgArch='arm64' ;; \
		armv7*) dpkgArch='armhf' ;; \
		i686) dpkgArch='i386' ;; \
		ppc64le) dpkgArch='ppc64el' ;; \
		s390x|riscv64) dpkgArch=$rpmArch ;; \
		x86_64) dpkgArch='amd64' ;; \
		*) echo >&2 "error: unknown/unsupported architecture '$rpmArch'"; exit 1 ;; \
	esac; \
	curl --fail --location --output /usr/local/bin/gosu https://github.com/tianon/gosu/releases/download/${GOSU_VERSION}/gosu-${dpkgArch} ; \
	curl --fail --location --output /usr/local/bin/gosu.asc https://github.com/tianon/gosu/releases/download/${GOSU_VERSION}/gosu-${dpkgArch}.asc; \
	GNUPGHOME="$(mktemp -d)"; \
	export GNUPGHOME; \
	gpg --batch --keyserver hkps://keys.openpgp.org --recv-keys B42F6819007F00F88E364FD4036A9C25BF357DD4; \
	chmod a+x /usr/local/bin/gosu; \
	gpg --batch --verify /usr/local/bin/gosu.asc /usr/local/bin/gosu; \
	gpgconf --kill all; \
	rm -rf "$GNUPGHOME" /usr/local/bin/gosu.asc; \
	gosu --version; \
	gosu nobody true

COPY --chmod=0644 docker.cnf /etc/my.cnf.d/

COPY MariaDB.repo /etc/yum.repos.d/

# HasRequiredLabel requirement from Red Hat OpenShift Software Certification
# https://access.redhat.com/documentation/en-us/red_hat_software_certification/2024/html/red_hat_openshift_software_certification_policy_guide/assembly-requirements-for-container-images_openshift-sw-cert-policy-introduction#con-image-metadata-requirements_openshift-sw-cert-policy-container-images
LABEL name="MariaDB Server" \
	vendor="MariaDB Community" \
	version="12.0.1" \
	release="Refer to Annotations org.opencontainers.image.{revision,source}" \
	summary="MariaDB Database" \
	description="MariaDB Database for relational SQL"

# OCI annotations to image
LABEL org.opencontainers.image.authors="MariaDB Community" \
      org.opencontainers.image.title="MariaDB Database" \
      org.opencontainers.image.description="MariaDB Database for relational SQL" \
      org.opencontainers.image.documentation="https://hub.docker.com/_/mariadb/" \
      org.opencontainers.image.base.name="docker.io/redhat/ubi9-minimal" \
      org.opencontainers.image.licenses="GPL-2.0" \
      org.opencontainers.image.source="https://github.com/MariaDB/mariadb-docker" \
      org.opencontainers.image.vendor="MariaDB Community" \
      org.opencontainers.image.version="12.0.1" \
      org.opencontainers.image.url="https://github.com/MariaDB/mariadb-docker"

# bashbrew-architectures: amd64 arm64v8 ppc64le s390x
ARG MARIADB_VERSION=12.0.1
ENV MARIADB_VERSION=$MARIADB_VERSION
# release-status:RC
# release-support-type:Rolling
# (https://downloads.mariadb.org/rest-api/mariadb/)

# missing pwgen(epel), jemalloc(epel) (as entrypoint/user extensions)
# procps, pv(epel) - missing dependencies of galera sst script
# tzdata re-installed as only a fake version is part of the ubi-minimal base image.
# FF8AD1344597106ECE813B918A3872BF3228467C is the Fedora RPM key
# 177F4010FE56CA3336300305F1656F24C74CD1D8 is the MariaDB Server RPM key
RUN set -eux ; \
	curl --fail https://pagure.io/fedora-web/websites/raw/master/f/sites/getfedora.org/static/keys/FF8AD1344597106ECE813B918A3872BF3228467C.txt --output /tmp/epelkey.txt ; \
	GNUPGHOME="$(mktemp -d)"; export GNUPGHOME ; \
	gpg --batch --import /tmp/epelkey.txt ; \
	gpg --batch --armor --export FF8AD1344597106ECE813B918A3872BF3228467C > /tmp/epelkey.txt ; \
	rpmkeys --import /tmp/epelkey.txt ; \
	curl --fail https://dl.fedoraproject.org/pub/epel/epel-release-latest-9.noarch.rpm --output /tmp/epel-release-latest-9.noarch.rpm ; \
	rpm -K /tmp/epel-release-latest-9.noarch.rpm ; \
	rpm -ivh /tmp/epel-release-latest-9.noarch.rpm ; \
	rm /tmp/epelkey.txt /tmp/epel-release-latest-9.noarch.rpm ; \
	curl --fail https://supplychain.mariadb.com/MariaDB-Server-GPG-KEY --output /tmp/MariaDB-Server-GPG-KEY ; \
	gpg --batch --import /tmp/MariaDB-Server-GPG-KEY; \
	gpg --batch --armor --export 177F4010FE56CA3336300305F1656F24C74CD1D8 > /tmp/MariaDB-Server-GPG-KEY ; \
	rpmkeys --import /tmp/MariaDB-Server-GPG-KEY ; \
	rm -rf "$GNUPGHOME" /tmp/MariaDB-Server-GPG-KEY ; \
	unset GNUPGHOME ; \
	microdnf update -y ; \
	microdnf reinstall -y tzdata ; \
	microdnf install -y procps-ng zstd xz jemalloc pwgen pv util-linux-core ; \
	mkdir -p /etc/mysql/conf.d /etc/mysql/mariadb.conf.d/ /var/lib/mysql/mysql /run/mariadb /usr/lib64/galera ; \
	chmod ugo+rwx,o+t /run/mariadb ; \
	microdnf install -y MariaDB-backup-${MARIADB_VERSION}  MariaDB-server-${MARIADB_VERSION} ; \
	# compatibility with DEB Galera packaging
	ln -s /usr/lib64/galera-4/libgalera_smm.so /usr/lib/libgalera_smm.so ; \
	# compatibility with RPM Galera packaging
	ln -s /usr/lib64/galera-4/libgalera_smm.so /usr/lib64/galera/libgalera_smm.so ; \
	microdnf clean all ; \
	rmdir /var/lib/mysql/mysql ; \
	chown -R mysql:mysql /var/lib/mysql /run/mariadb ; \
	mkdir /licenses ; \
	ln -s /usr/share/doc/MariaDB-server-${MARIADB_VERSION}/COPYING /licenses/GPL-2 ; \
	ln -s /usr/share/licenses /licenses/package-licenses ; \
	ln -s Apache-2.0-license /licenses/gosu

VOLUME /var/lib/mysql

RUN mkdir /docker-entrypoint-initdb.d

COPY healthcheck.sh /usr/local/bin/healthcheck.sh
COPY docker-entrypoint.sh /usr/local/bin/

ENTRYPOINT ["docker-entrypoint.sh"]

USER mysql
EXPOSE 3306
CMD ["mariadbd"]
