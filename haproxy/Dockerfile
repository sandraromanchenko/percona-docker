FROM golang:1.20 AS go_builder
WORKDIR /go/src/github.com/percona/percona-xtradb-cluster-operator/src

RUN export GO111MODULE=off \
    && go get k8s.io/apimachinery/pkg/util/sets \
    && curl -Lf -o /go/src/github.com/percona/percona-xtradb-cluster-operator/src/peer-list.go https://raw.githubusercontent.com/percona/percona-xtradb-cluster-operator/main/cmd/peer-list/main.go \
    && go build peer-list.go

FROM redhat/ubi9-minimal AS ubi9

LABEL name="HAproxy" \
      description="TCP proxy loadbalancer for Percona Xtradb Cluster" \
      vendor="Percona" \
      summary="TCP proxy for mysql protocol" \
      org.opencontainers.image.authors="info@percona.com"

# check repository package signature in secure way
RUN export GNUPGHOME="$(mktemp -d)" \
	&& microdnf install -y findutils \
	&& gpg --keyserver keyserver.ubuntu.com --recv-keys 430BDF5C56E7C94E848EE60C1C4CBDCDCD2EFD2A 3E6D826D3FBAB389C2F38E34BC4D06A08D8B756F \
	&& gpg --export --armor 430BDF5C56E7C94E848EE60C1C4CBDCDCD2EFD2A > ${GNUPGHOME}/RPM-GPG-KEY-Percona \
	&& gpg --export --armor 3E6D826D3FBAB389C2F38E34BC4D06A08D8B756F > ${GNUPGHOME}/RPM-GPG-KEY-oracle \
	&& rpmkeys --import ${GNUPGHOME}/RPM-GPG-KEY-Percona ${GNUPGHOME}/RPM-GPG-KEY-oracle \
	&& curl -Lf -o /tmp/percona-release.rpm https://repo.percona.com/yum/percona-release-latest.noarch.rpm \
	&& rpmkeys --checksig /tmp/percona-release.rpm \
	&& rpm -i /tmp/percona-release.rpm \
	&& rm -rf "$GNUPGHOME" /tmp/percona-release.rpm \
	&& rpm --import /etc/pki/rpm-gpg/PERCONA-PACKAGING-KEY \
	&& percona-release setup pdpxc-8.0.32

# install exact version of PS for repeatability
ENV PERCONA_VERSION 8.0.32-24.1.el9

RUN set -ex; \
	curl -Lf -o /tmp/krb5-libs.rpm https://yum.oracle.com/repo/OracleLinux/OL9/baseos/latest/x86_64/getPackage/krb5-libs-1.19.1-24.0.1.el9_1.x86_64.rpm; \
	rpmkeys --checksig /tmp/krb5-libs.rpm; \
	rpm -U /tmp/krb5-libs.rpm; \
	rm -rf /tmp/krb5-libs.rpm; \
    microdnf install -y \
        shadow-utils \
        percona-haproxy \
        percona-xtradb-cluster-client-${PERCONA_VERSION} \
        which \
        tar \
        socat \
        procps-ng \
        vim-minimal \
        policycoreutils; \
    \
    microdnf clean all; \
    rm -rf /var/cache

RUN groupadd -g 1001 mysql
RUN useradd -u 1001 -r -g 1001 -s /sbin/nologin \
        -c "Default Application User" mysql

STOPSIGNAL SIGUSR1

RUN set -ex; \
    mkdir -p /etc/haproxy/pxc /etc/haproxy-custom; \
    chown -R 1001:1001 /run /etc/haproxy /etc/haproxy/pxc /etc/haproxy-custom
COPY LICENSE /licenses/LICENSE.Dockerfile
RUN cp /usr/share/licenses/percona-haproxy/LICENSE /licenses/LICENSE.haproxy

COPY dockerdir /
COPY --from=go_builder /go/src/github.com/percona/percona-xtradb-cluster-operator/src/peer-list /usr/bin/

RUN set -ex; \
    chown 1001:1001 /etc/haproxy/haproxy.cfg /etc/haproxy/haproxy-global.cfg /usr/local/bin/check_pxc.sh

USER 1001

VOLUME ["/etc/haproxy/pxc"]

ENTRYPOINT ["/entrypoint.sh"]
CMD ["haproxy"]
