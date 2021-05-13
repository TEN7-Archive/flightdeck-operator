FROM quay.io/operator-framework/ansible-operator:latest

COPY requirements.yml ${HOME}/requirements.yml
RUN ansible-galaxy collection install -r ${HOME}/requirements.yml \
 && chmod -R ug+rwx ${HOME}/.ansible

COPY watches.yaml ${HOME}/watches.yaml
COPY roles/ ${HOME}/roles/
COPY playbooks/ ${HOME}/playbooks/

USER root
ADD https://dev.mysql.com/get/mysql80-community-release-el8-1.noarch.rpm /tmp/mysql80-community-release-el8-1.noarch.rpm
RUN yum install -y /tmp/mysql80-community-release-el8-1.noarch.rpm && \
    yum install -y mysql-community-client mysql-community-common mysql-community-libs python38-PyMySQL

USER ansible
