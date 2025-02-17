FROM redhat/ubi9-minimal

LABEL name="Percona Toolkit" \
        release="3.5.1" \
        vendor="Percona" \
        summary="Percona Toolkit is a collection of advanced command-line tools used by Percona" \
        description="Percona Toolkit is derived from Maatkit and Aspersa, two of the best-known toolkits for MySQL server administration. It is developed and supported by Percona." \
        maintainer="Percona Development <info@percona.com>"

RUN microdnf -y update; \
    microdnf -y install glibc-langpack-en

ENV PS_VERSION 8.0.32-24.1.el9
ENV PT_VERSION 3.5.2-2.el9

# check repository package signature in secure way
RUN set -ex; \
    export GNUPGHOME="$(mktemp -d)"; \
    gpg --batch --keyserver keyserver.ubuntu.com --recv-keys 430BDF5C56E7C94E848EE60C1C4CBDCDCD2EFD2A 99DB70FAE1D7CE227FB6488205B555B38483C65D; \
    gpg --batch --export --armor 430BDF5C56E7C94E848EE60C1C4CBDCDCD2EFD2A > ${GNUPGHOME}/RPM-GPG-KEY-Percona; \
    gpg --batch --export --armor 99DB70FAE1D7CE227FB6488205B555B38483C65D > ${GNUPGHOME}/RPM-GPG-KEY-centosofficial; \
    rpmkeys --import ${GNUPGHOME}/RPM-GPG-KEY-Percona ${GNUPGHOME}/RPM-GPG-KEY-centosofficial; \
    microdnf install -y findutils; \
    curl -Lf -o /tmp/percona-release.rpm https://repo.percona.com/yum/percona-release-latest.noarch.rpm; \
    rpmkeys --checksig /tmp/percona-release.rpm; \
    rpm -i /tmp/percona-release.rpm; \
    rm -rf "$GNUPGHOME" /tmp/percona-release.rpm; \
    rpm --import /etc/pki/rpm-gpg/PERCONA-PACKAGING-KEY; \
    percona-release enable ps-80 release; \
    percona-release enable tools release

RUN set -ex; \
    microdnf -y update; \
    microdnf -y install  \
        shadow-utils \
        tar \
        percona-server-client-${PS_VERSION} 

RUN   set -ex; \
      microdnf -y install percona-toolkit-${PT_VERSION}; \
      microdnf clean all; \
      rm -rf /var/cache/dnf /var/cache/yum


RUN useradd -u 1001 -r -g 0 -s /sbin/nologin \
            -c "Default Application User" perconatoolkit

USER 1001
