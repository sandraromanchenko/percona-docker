FROM golang:1.19 AS go_builder
WORKDIR /go/src/github.com/percona/percona-xtradb-cluster-operator/src

RUN export GO111MODULE=off \
    && go get k8s.io/apimachinery/pkg/util/sets \
    && curl -Lf -o /go/src/github.com/percona/percona-xtradb-cluster-operator/src/peer-list.go https://raw.githubusercontent.com/percona/percona-xtradb-cluster-operator/main/cmd/peer-list/main.go \
    && go build peer-list.go

FROM redhat/ubi8-minimal

LABEL org.opencontainers.image.authors="info@percona.com"

ENV PXC_VERSION 8.0.30-22.1
ENV PROXYSQL_VERSION 2.4.8-1.1
ENV OS_VER el8
ENV FULL_PERCONA_VERSION "$PXC_VERSION.$OS_VER"
ENV FULL_PROXYSQL_VERSION "$PROXYSQL_VERSION.$OS_VER"

# check repository package signature in secure way
RUN set -ex; \
    export GNUPGHOME="$(mktemp -d)"; \
    gpg --batch --keyserver keyserver.ubuntu.com --recv-keys 430BDF5C56E7C94E848EE60C1C4CBDCDCD2EFD2A 99DB70FAE1D7CE227FB6488205B555B38483C65D 94E279EB8D8F25B21810ADF121EA45AB2F86D6A1 76FD3DB13AB67410B89DB10E82562EA9AD986DA3; \
    gpg --batch --export --armor 430BDF5C56E7C94E848EE60C1C4CBDCDCD2EFD2A > ${GNUPGHOME}/RPM-GPG-KEY-Percona; \
    gpg --batch --export --armor 99DB70FAE1D7CE227FB6488205B555B38483C65D > ${GNUPGHOME}/RPM-GPG-KEY-centosofficial; \
    gpg --batch --export --armor 94E279EB8D8F25B21810ADF121EA45AB2F86D6A1 > ${GNUPGHOME}/RPM-GPG-KEY-EPEL-8; \
    gpg --batch --export --armor 76FD3DB13AB67410B89DB10E82562EA9AD986DA3 > ${GNUPGHOME}/RPM-GPG-KEY-oracle; \
    rpmkeys --import ${GNUPGHOME}/RPM-GPG-KEY-Percona ${GNUPGHOME}/RPM-GPG-KEY-centosofficial ${GNUPGHOME}/RPM-GPG-KEY-EPEL-8 ${GNUPGHOME}/RPM-GPG-KEY-oracle; \
    microdnf install -y findutils; \
    curl -Lf -o /tmp/percona-release.rpm https://repo.percona.com/yum/percona-release-latest.noarch.rpm; \
    rpmkeys --checksig /tmp/percona-release.rpm; \
    rpm -i /tmp/percona-release.rpm; \
    rm -rf "$GNUPGHOME" /tmp/percona-release.rpm; \
    rpm --import /etc/pki/rpm-gpg/PERCONA-PACKAGING-KEY; \
    percona-release setup pxc80; \
    percona-release enable tools testing

RUN set -ex; \
    curl -Lf -o /tmp/jq.rpm http://vault.centos.org/centos/8/AppStream/x86_64/os/Packages/jq-1.5-12.el8.x86_64.rpm; \
    curl -Lf -o /tmp/oniguruma.rpm http://vault.centos.org/centos/8/AppStream/x86_64/os/Packages/oniguruma-6.8.2-2.el8.x86_64.rpm; \
    rpmkeys --checksig /tmp/jq.rpm /tmp/oniguruma.rpm; \
    rpm -i /tmp/jq.rpm /tmp/oniguruma.rpm; \
    rm -rf /tmp/jq.rpm /tmp/oniguruma.rpm

RUN set -ex; \
    microdnf install -y \
        percona-xtradb-cluster-client-${FULL_PERCONA_VERSION} \
        shadow-utils \
        which \
        hostname \
        procps-ng \
        perl-DBD-MySQL \
        perl-DBI \
        cpio \
        tar; \
    microdnf clean all; \
    rm -rf /var/cache/dnf /var/cache/yum

RUN groupadd -g 1001 proxysql; \
    useradd -u 1001 -r -g 1001 -s /sbin/nologin \
        -c "Default Application User" proxysql

# we need licenses from docs
RUN set -ex; \
	curl -Lf -o /tmp/krb5-libs.rpm https://yum.oracle.com/repo/OracleLinux/OL8/baseos/latest/x86_64/getPackage/krb5-libs-1.18.2-22.0.1.el8_7.x86_64.rpm; \
    curl -Lf -o /tmp/util-linux.rpm http://vault.centos.org/centos/8/BaseOS/x86_64/os/Packages/util-linux-2.32.1-28.el8.x86_64.rpm; \
    curl -Lf -o /tmp/proxysql2-${FULL_PROXYSQL_VERSION}.rpm http://repo.percona.com/proxysql/yum/testing/8/RPMS/x86_64/proxysql2-${FULL_PROXYSQL_VERSION}.x86_64.rpm; \
    curl -Lf -o /tmp/percona-xtradb-cluster-server-${FULL_PERCONA_VERSION}.rpm http://repo.percona.com/pxc-80/yum/release/8.0/RPMS/x86_64/percona-xtradb-cluster-server-${FULL_PERCONA_VERSION}.x86_64.rpm; \
    rpmkeys --checksig /tmp/percona-xtradb-cluster-server-${FULL_PERCONA_VERSION}.rpm /tmp/proxysql2-${FULL_PROXYSQL_VERSION}.rpm /tmp/util-linux.rpm /tmp/krb5-libs.rpm; \
    rpm -iv /tmp/proxysql2-${FULL_PROXYSQL_VERSION}.rpm /tmp/util-linux.rpm --nodeps; \
	rpm -U /tmp/krb5-libs.rpm; \
    rm -rf /tmp/proxysql2-${FULL_PROXYSQL_VERSION}.rpm /tmp/util-linux.rpm /tmp/krb5-libs.rpm; \
    microdnf clean all; \
    rm -rf /var/cache/dnf /var/cache/yum /etc/proxysql /var/lib/proxysql; \
    rpm -ql percona-xtradb-cluster-client | egrep -v "mysql$|mysqldump$" | xargs rm -rf; \
    install -o 1001 -g 0 -m 775 -d /etc/proxysql /var/lib/proxysql; \
    rpm2cpio /tmp/percona-xtradb-cluster-server-${FULL_PERCONA_VERSION}.rpm | cpio -idv ./usr/bin/my_print_defaults; \
    rm -rf /tmp/percona-xtradb-cluster-server-${FULL_PERCONA_VERSION}.rpm

COPY LICENSE /licenses/LICENSE.Dockerfile
RUN cp /usr/share/doc/proxysql2/LICENSE /licenses/LICENSE.proxysql

COPY dockerdir /
COPY --from=go_builder /go/src/github.com/percona/percona-xtradb-cluster-operator/src/peer-list /usr/bin/
RUN chown 1001:1001 /etc/proxysql/proxysql.cnf /etc/proxysql-admin.cnf; \
    chmod 664 /etc/proxysql/proxysql.cnf /etc/proxysql-admin.cnf

USER 1001

VOLUME /var/lib/proxysql

ENTRYPOINT ["/entrypoint.sh"]

EXPOSE 3306 6032
CMD ["/usr/bin/proxysql", "-f", "-c", "/etc/proxysql/proxysql.cnf", "--reload"]
