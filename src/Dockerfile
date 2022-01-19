FROM quay.io/operator-framework/ansible-operator:v1.5

COPY requirements.yml ${HOME}/requirements.yml
RUN ansible-galaxy collection install -r ${HOME}/requirements.yml \
 && chmod -R ug+rwx ${HOME}/.ansible

COPY watches.yaml ${HOME}/watches.yaml
COPY roles/ ${HOME}/roles/
COPY playbooks/ ${HOME}/playbooks/

USER root
# RUN pip3 install --no-cache-dir openshift==0.12.1

ADD https://dev.mysql.com/get/mysql80-community-release-el8-3.noarch.rpm /tmp/mysql80.rpm
RUN yum install -y /tmp/mysql80.rpm && \
    yum install -y mysql-community-client mysql-community-common mysql-community-libs python38-PyMySQL && \
    pip3 install passlib --no-cache-dir && \
    curl -o /usr/local/bin/kubectl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl" && \
    chmod a=rx /usr/local/bin/kubectl

USER ansible
