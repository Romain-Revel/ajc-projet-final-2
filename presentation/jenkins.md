---
marp: true
theme: uncover
header: 'PROJET FINAL DEVOPS - Jenkins'
footer: 'Groupe 3'
paginate: true
---

# Jenkins

---

## Infrastructure

- Serveur 1 : **192.168.99.12** Jenkins
    https://github.com/sadofrazer/jenkins-frazer.git
- Serveur 2 : **192.168.99.21** Applications web site vitrine + pgadmin4
- Serveur 3 : **192.168.99.20** Application Odoo (PostgreSQL)

---

Serveurs 2 et 3 - boucle

```ruby
Vagrant.configure("2") do |config|
  array = ["odoo", "pgadmin-icwebapp"]
  # Boucle pour créer les 2
  array.each_with_index do |val, index|
        config.vm.define "docker-#{val}" do |docker|
                #...
                    docker.vm.network "private_network",  type: "static", ip: "192.168.99.2#{index}", name: "vboxnet0"
                #...
                        docker.vm.network "private_network",  type: "static", ip: "192.168.99.2#{index}"
                #...
                docker.vm.hostname = "docker-#{val}"
                #...
                        v.name = "docker-#{val}"
                #...
```

---

## Jenkins - Script d'installation

```bash
#!/bin/bash
# Script d'installation de Jenkins fourni par Dirane
yum -y update
yum -y install epel-release
# install ansible
yum -y install ansible
# retrieve ansible code
yum -y install git
git clone https://github.com/diranetafen/cursus-devops.git
cd cursus-devops/ansible
ansible-galaxy install -r roles/requirements.yml
ansible-playbook install_docker.yml
sudo usermod -aG docker vagrant
cd ../jenkins
/usr/local/bin/docker-compose up -d
echo "For this Stack, you will use $(ip -f inet addr show enp0s8 | \
sed -En -e 's/.*inet ([0-9.]+).*/\1/p') IP Address"
```

---

## Interface

![bg w:100%](./images/Jenkins-configuration-05-bienvenue.png)

---

## Jenkins - Inconvénients

- Version datée
- Interface web **laide** et **non ergonomique**
- Les installations ne partagent pas:
    - Comptes
    - Plugins
    - Jobs
    - Secrets
    - Configuration globale

---

## Jenkins-custom - Automatisation de l'installation

- Docker
- jenkins/jenkins:lts-jdk11

---

```docker
FROM jenkins/jenkins:lts-jdk11
USER root
ENV JAVA_OPTS -Djenkins.install.runSetupWizard=false
ENV CASC_JENKINS_CONFIG /var/jenkins_home/jenkins.casc.yml
# Installation
RUN apt-get update && \
    apt-get install -qy curl python3 python3-pip sshpass shellcheck && \
    pip3 install ansible && \
    curl -sSL https://get.docker.com/ | sh
USER jenkins
# Plugins Jenkins
COPY jenkins.plugins.txt /usr/share/jenkins/ref/jenkins.plugins.txt
RUN jenkins-plugin-cli --list && \
    jenkins-plugin-cli --plugin-file /usr/share/jenkins/ref/jenkins.plugins.txt && \
    jenkins-plugin-cli --list
# Configuration as code Jenkins
COPY jenkins.casc.yml /var/jenkins_home/jenkins.casc.yml
```

---

## Plugins par défaut

```text
antisamy-markup-formatter:latest
build-timeout:latest
cloudbees-folder:latest
credentials-binding:latest
email-ext:latest
git:latest
github-branch-source:latest
mailer:latest
pam-auth:latest
pipeline-github-lib:latest
pipeline-stage-view:latest
ssh-slaves:latest
timestamper:latest
workflow-aggregator:latest
ws-cleanup:latest
```

---

## Plugins supplémentaires

```text
ansible:latest
authorize-project:latest
configuration-as-code:latest
docker-plugin:latest
docker-workflow:latest
matrix-auth:latest
```

---

![bg w:100%](./images/Jenkins-lastest.png)

---

## jenkins-cli

- https://www.jenkins.io/doc/book/managing/cli/

- https://medium.com/@muku.hbti/export-import-jenkins-job-and-their-plugins-53cafa5869fa

- https://www.digitalocean.com/community/tutorials/how-to-automate-jenkins-setup-with-docker-and-jenkins-configuration-as-code

---

```bash
#/bin/bash
# Téléchargement de jenkins-cli
if [[ -z "${JENKINS_USERNAME}" ]]; then
    echo "La variable JENKINS_USERNAME n'existe pas. Veuillez l'exporter.";
    exit 1;
fi
if [[ -z "${JENKINS_PASSWORD}" ]]; then
    echo "La variable JENKINS_PASSWORD n'existe pas. Veuillez l'exporter.";
    exit 1;
fi
# Valeurs par défaut
[[ ! -z "${JENKINS_IP}" ]] || JENKINS_IP="192.168.99.13";
[[ ! -z "${JENKINS_PORT}" ]] || JENKINS_PORT="8080";
URL="http://${JENKINS_IP}:${JENKINS_PORT}";
JOB_NAME="TEST";
# Téléchargement de la cli jenkins
if [[ ! -f "jenkins-cli.jar" ]]; then
    wget "${URL}/jnlpJars/jenkins-cli.jar";
fi
```

---

```bash
# Pour lister les jobs
java -jar jenkins-cli.jar -s "${URL}" -auth "${JENKINS_USERNAME}":"${JENKINS_PASSWORD}" list-jobs;

# Récupérer la liste de tous les jobs et les exporter
JOBS=$(java -jar jenkins-cli.jar -s "${URL}" -auth "${JENKINS_USERNAME}":"${JENKINS_PASSWORD}" list-jobs);
mkdir -p "jobs";
for JOB_NAME in $JOBS
do
    java -jar jenkins-cli.jar -s "${URL}" -auth "${JENKINS_USERNAME}":"${JENKINS_PASSWORD}" get-job "${JOB_NAME}" > "jobs/${JOB_NAME}.xml";
done
```

---

```bash
# Restaurer tous les jobs
for JOB_FILE in $(cd "jobs"; ls *.xml)
do
    # echo $JOB_FILE
    # echo ${JOB_FILE%%.xml}
    java -jar jenkins-cli.jar -s "${URL}" -auth "${JENKINS_USERNAME}":"${JENKINS_PASSWORD}" create-job "${JOB_FILE%%.xml}" < "jobs/${JOB_FILE}"
done
```

---

## Configuration de Jenkins

Plugin [Configuration as code](https://plugins.jenkins.io/configuration-as-code/)
- Compte(s) admin
- URL
- Credentials
- Security (bonus)

---

```groovy
jenkins:
  securityRealm:
    local:
      allowsSignup: false
      enableCaptcha: false
      users:
        - id: admin
          password: password
          properties:
          - "apiToken"
          - mailer:
              emailAddress: "admin@hotmail.fr"
          - preferredProvider:
              providerId: "default"
```

---

```groovy
unclassified:
  location:
    url: http://192.168.99.13:8080/
```

---

```groovy
credentials:
  system:
    domainCredentials:
    - credentials:
      - string:
          description: "Token dockerhub"
          id: "dockerhub"
          scope: GLOBAL
          secret: "{AQAAABAAAAAw632BD8V0u3jO0s90yYiMwllBr6OzIrtmGMqWAEvtIDcqXa2XCyH2WBJPrmSdH9fPnShuX2v4AMjjUbicqwo2Ag==}"
      - usernamePassword:
          id: "ansible_user_credentials"
          password: "vagrant"
          scope: GLOBAL
          username: "vagrant"
          usernameSecret: true
      - usernamePassword:
          id: "pgadmin_credentials"
          password: "pgadmin"
          scope: GLOBAL
          username: "pgadmin@local.domain" // @ Très important
          usernameSecret: true
```

---

```groovy
jenkins:
  //...
  authorizationStrategy:
    globalMatrix:
      permissions:
        - "USER:Overall/Administer:admin"
        - "GROUP:Overall/Read:authenticated"
  remotingSecurity:
      enabled: true

security:
  queueItemAuthenticator:
    authenticators:
    - global:
        strategy: triggeringUsersAuthorizationStrategy
```

---

### Rendre Jenkins accessible avec Ngrok

Ngrok est un reverse-proxy qui permet d'ouvrir sur internet des ports d'une machine

Utilisé pour la partie **webhook**

Alternatives à ngrok:

- Vagrant share
    <https://www.vagrantup.com/docs/share>

- Localtunnel
    <https://github.com/localtunnel/localtunnel>

---

```bash
#!/bin/bash
# Script d'installation et de lancement de ngrok
FILE_NAME="ngrok-v3-stable-linux-amd64.tgz";
if [[ ! -f "/usr/local/bin/ngrok" ]]; then
    # Téléchargement
    # https://ngrok.com/download
    if [[ ! -f "${FILE_NAME}" ]]; then
        wget "https://bin.equinox.io/c/bNyj1mQVY4c/${FILE_NAME}";
    fi

    # Décompression et installation
    sudo tar xvzf "${FILE_NAME}" -C /usr/local/bin;
fi
sleep 3
# Enregistrement
ngrok config add-authtoken $(cat token.txt);
# Lancement
nohup ngrok http 8080 &
# Récupération de l'URL
curl "http://localhost:4040/api/tunnels";
```

---

![bg w:100%](./images/Jenkins-ngrok-1.png)

---

![bg w:100%](./images/Jenkins-ngrok-2.png)

---

## Pipeline(s)

---

### Structure

```groovy
// Jenkinsfile
pipeline {

    environment {
        IMAGE_NAME = "ic-webapp"
        IMAGE_TAG = "${sh(returnStdout: true, script: 'cat ic-webapp/releases.txt \
        |grep version | cut -d\\: -f2|xargs')}"
        CONTAINER_NAME = "ic-webapp"
        USER_NAME = "sh0t1m3"
    }

    agent any

    stages {
        // stage 1...
    }
}
```

---

### Lint YAML

```groovy
stage('Lint yaml files') {
    when { changeset "**/*.yml"}
    agent {
        docker {
            image 'sdesbure/yamllint'
        }
    }
    steps {
        sh 'yamllint --version'
        sh 'yamllint ${WORKSPACE} >report_yml.log || true'
    }
    post {
        always {
            archiveArtifacts 'report_yml.log'
        }
    }
}
```

---

### Lint markdown

```groovy
stage('Lint markdown files') {
    when { changeset "**/*.md"}
    agent {
        docker {
            image 'ruby:alpine'
        }
    }
    steps {
        sh 'apk --no-cache add git'
        sh 'gem install mdl'
        sh 'mdl --version'
        sh 'mdl --style all --warnings --git-recurse ${WORKSPACE} > md_lint.log || true'
    }
    post {
        always {
            archiveArtifacts 'md_lint.log'
        }
    }
}
```

---

### Lint ansible

```groovy
stage("Lint ansible playbook files") {
    when { changeset "ansible/**/*.yml"}
    agent {
        docker {
            image 'registry.gitlab.com/robconnolly/docker-ansible:latest'
        }
    }
    steps {
        sh '''
            cd "${WORKSPACE}/ansible/"
            ansible-lint play.yml > "${WORKSPACE}/ansible-lint.log" || true
        '''
    }
    post {
        always {
            archiveArtifacts "ansible-lint.log"
        }
    }
}
```

---

### Lint shell scripts

```groovy
stage('Lint shell script files') {
    when { changeset "**/*.sh" }
    agent any
    steps {
        sh 'shellcheck */*.sh >shellcheck.log || true'
    }
    post {
        always {
            archiveArtifacts 'shellcheck.log'
        }
    }
}
```

---

### Lint shell scripts - checkstyle

```groovy
stage('Lint shell script files - checkstyle') {
    when { changeset "**/*.sh" }
    agent any
    steps {
        catchError(buildResult: 'SUCCESS') {
            sh """#!/bin/bash
                # The null command `:` only returns exit code 0 to ensure following task executions.
                shellcheck -f checkstyle */*.sh > shellcheck.xml || true
            """
        }
    }
    post {
        always {
            archiveArtifacts 'shellcheck.xml'
        }
    }
}
```

---

### Lint docker files

```groovy
stage ("Lint docker files") {
    when { changeset "**/Dockerfile"}
    agent {
        docker {
            image 'hadolint/hadolint:latest-debian'
        }
    }
    steps {
        sh 'hadolint $PWD/**/Dockerfile | tee -a hadolint_lint.log'
    }
    post {
        always {
            archiveArtifacts 'hadolint_lint.log'
        }
    }
}
```

---

### Push docker image

```groovy
stage ('Login and push docker image') {
    when { changeset "ic-webapp/releases.txt"}
    agent any
    environment {
        DOCKERHUB_PASSWORD  = credentials('dockerhub')
    }
    steps {
        script {
        sh '''
            echo "${DOCKERHUB_PASSWORD}" | docker login -u ${USER_NAME} --password-stdin;
            docker push ${USER_NAME}/${IMAGE_NAME}:${IMAGE_TAG};
        '''
        }
    }
}
```

---

![bg w:100%](./images/Jenkins-skip-docker.png)

---

![bg w:100%](./images/Jenkins-run-docker-after-release-change.png)

---

![bg w:100%](./images/docker-hub-1.0.png)

---

![bg w:100%](./images/docker-hub-1.2.png)

---

### Déploiement avec Ansible

Credentials à déclarer dans Jenkins pour déployer par Ansible

---

![bg w:100%](./images/Jenkins-credentials.png)

---

```groovy
stage ('Deploy to prod with Ansible') {
    steps {
        withCredentials([
            usernamePassword(credentialsId: 'ansible_user_credentials', usernameVariable: 'ansible_user', passwordVariable: 'ansible_user_pass'),
            usernamePassword(credentialsId: 'pgadmin_credentials', usernameVariable: 'pgadmin_user', passwordVariable: 'pgadmin_pass'),
            usernamePassword(credentialsId: 'pgsql_credentials', usernameVariable: 'pgsql_user', passwordVariable: 'pgsql_pass'),
            string(credentialsId: 'ansible_sudo_pass', variable: 'ansible_sudo_pass')
        ])
        {
            ansiblePlaybook (
                disableHostKeyChecking: true,
                installation: 'ansible',
                inventory: 'ansible/prods.yml',
                playbook: 'ansible/play.yml',
                extras: '--extra-vars "NETWORK_NAME=network \
                        IMAGE_TAG=${IMAGE_TAG} \
                        ansible_user=${ansible_user} \
                        ansible_ssh_pass=${ansible_user_pass} \
                        ansible_sudo_pass=${ansible_sudo_pass} \
                        PGADMIN_EMAIL=${pgadmin_user} \
                        PGADMIN_PASS=${pgadmin_pass} \
                        DB_USER=${pgsql_user} \
                        DB_PASS=${pgsql_pass}"'
            )
        }
    }
}
```

---

### Test final

```groovy
stage ('Test full deployment') {
    steps {
        sh '''
            sleep 10;

            curl -LI http://192.168.99.21 | grep "200";
            curl -L http://192.168.99.21 | grep "IC GROUP";

            curl -LI http://192.168.99.20:8081 | grep "200";
            curl -L http://192.168.99.20:8081 | grep "Odoo";

            curl -LI http://192.168.99.21:8082 | grep "200";
            curl -L http://192.168.99.21:8082 | grep "pgAdmin 4";
        '''
    }
}
```

---

### Troubleshooting

- Attention aux redirections HTTP
- Attention au délai entre la fin d'Ansible et la disponibilité du site web

---

### Trivy

Scanner de vulnérabilité
- code source (fs, repo)
- dépendances
- images conteneur
- ...

https://semaphoreci.com/blog/continuous-container-vulnerability-testing-with-trivy

---

### Déclenchement automatique

- Webhook Github

- Tâche planifiée qui vérifie régulièrement les modifications du SCM

- Tâche planifiée qui build régulièrement (MAUVAIS)

---

### Problèmes rencontrés avec Jenkins

- Version pas à jour
- Agent docker ne fonctionne pas pour (shellcheck, trivy)
- Jenkins difficile à configurer automatiquement

---

### Astuces

- Commencer simplement, par des étapes qui fonctionnent
- when **{ changeset "\*\*/*.ext"}** pour éviter de relancer inutilement certains stages
- Il est possible de rejouer un build à partir d'une étape
  mais sur le **même** commit
- Créer un **job manuel** pour faire des tests...
- ...et qui ne teste que l'étape souhaitée
- Exporter, versionner, importer les jobs

---

### TODO

- Webook github
- Healthcheck sur les conteneurs
- Auto-merge sur main à la réussite du pipeline
- Utiliser un master Jenkins et un ou plusieurs slaves
- Finir d'implémenter les tests avec Trivy
- Tester l'installation avec un role ansible
  https://github.com/geerlingguy/ansible-role-jenkins
- Job qui exporte ET versionne automatiquement les pipelines
