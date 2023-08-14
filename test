#!/bin/bash
# RKE2 install script for Linux System

# Only Install Env
KUBECTL=/var/lib/rancher/rke2/bin/kubectl
KUBE_CFILE=/etc/rancher/rke2/rke2.yaml
KUBECONFIG="--kubeconfig $KUBE_CFILE"

EXIST_KUBECONFIG=false
ON_ERROR=false
# Helm Install path
HELM=/usr/local/bin/helm
# Start User Home Path Check
USER_HOME=$(eval echo ~$SUDO_USER)

# CUSTOM VARIABLE
RKE2_VER=v1.24
MAIN_CR_NAME="security365cr"
# Longhorn Disk Path (default : /var/lib/longhorn/ ; The path you want to change to must be pre-existing.)
LONGHORN_DATA_PATH=/longhorn/

RKE_TYPE=$1
COMMAND=$2
PARAM1=$3
PARAM2=$4
PARAM3=$5

function set_linux_distribution() {
  DISTRO=$( cat /etc/*-release | tr [:upper:] [:lower:] | grep -Poi '(debian|ubuntu|red hat|centos|nameyourdistro)' | uniq )
  if [ -z $DISTRO ]; then
      DISTRO='unknown'
  fi
  echo "Detected Linux distribution: $DISTRO"  
}

function print_parameter_error() {
    echo "================================================================================================================================="
    echo " >"
    echo " > sudo ./rke2.sh RKE_TYPE COMMAND PARAM1, PARAM2"
    echo " >"
    echo " - RKE_TYPE(required) : server, agent"
    echo " - COMMAND(required)"
    echo " -    server : install, uninstall, reset(by ip changed) or reinstall(uninstall and install), add(master-n node), remove(master-n node)"
    echo " -    agent : install, uninstall, reinstall(uninstall and install)"
    echo " -  SERVER PARAMS"
    echo " -   server install/reinstall PARAM1 PARAM2"
    echo " -      PARAM1(option) : RANCHER_URL or not set"
    echo " -      PARAM2(option) : HARBOR_URL or not set"
    echo " -   server uninstall"
    echo " -   server reset PARAM1"
    echo " -      PARAM1(required) : ipchanged"
    echo " -   server add PARAM1 PARAM2"
    echo " -      PARAM1(required) : Master Node IP"
    echo " -      PARAM2(required) : Master Node Token"
    echo " -   server remove PARAM1"
    echo " -      PARAM1(required) : nodeDeleted"
    echo " -  AGENT PARAMS"
    echo " -   agent install/reinstall PARAM1 PARAM2"
    echo " -      PARAM1(required) : Master Node IP"
    echo " -      PARAM2(required) : Master Node Token"
    echo " -   agent uninstall"
    echo "================================================================================================================================="
    echo " - Master Node Token  : RKE-SERVER CONNECT TOKEN, check MASTER SERVER '/var/lib/rancher/rke2/server/node-token'"
    echo "================================================================================================================================="
}

function server_param_check() {
  if [ "$COMMAND" = "install" -o "$COMMAND" = "reinstall" -o "$COMMAND" = "uninstall" ]
  then
    if [ "$PARAM1" = "" ]
    then
      PARAM1="rancher.local"
    fi
    if [ "$PARAM2" = "" ]
    then
      PARAM2="harbor.local"
    fi
  elif [ "$COMMAND" = "reset" ]
  then
    if [ "$PARAM1" != "ipchanged" ]
    then
      ON_ERROR=true
    fi
  elif [ "$COMMAND" = "add" ]
  then
    if [ "$PARAM1" = "" -o "$PARAM2" = "" ]
    then
      ON_ERROR=true
    fi
  elif [ "$COMMAND" = "remove" ]
  then
    echo "================================================================================================================================="
    echo " Warning  : You need to work after deleting the node in the Kubernetes Cluster."
    echo "  - 1 : Rancher UI Login"
    echo "  - 2 : Cluster -> Nodes"
    echo "  - 3 : Current Nodes Deleted"
    echo "  - 4 : call sudo ./rek2.sh server remove nodeDeleted"
    echo "================================================================================================================================="
    if [ "$PARAM1" != "nodeDeleted" ]
    then
      ON_ERROR=true
    fi
  else
    ON_ERROR=true
  fi
}

function agent_param_check() {
  if [ "$COMMAND" = "install" -o "$COMMAND" = "reinstall" ]
  then
    if [ "$PARAM1" = "" -o "$PARAM2" = "" ]
    then
      ON_ERROR=true
    fi
  elif [ "$COMMAND" = "uninstall" ]
  then
    ON_ERROR=false
  else
    ON_ERROR=true
  fi
}

function check_and_set_default_param() {
  if [ "$EUID" -ne 0 ]
  then 
    echo "================================================================================================================================="
    echo "==== access denied : only sudo run"
    echo "================================================================================================================================="
    print_parameter_error
    exit
  fi
  if [ -r "$KUBE_CFILE" ]
  then
    EXIST_KUBECONFIG=true
  fi
  if [ "$RKE_TYPE" = "server" ]
  then
    server_param_check
  elif [ "$RKE_TYPE" = "agent" ]
  then
    agent_param_check
  else
    ON_ERROR=true;
  fi
  if [ "$ON_ERROR" = "true" ]
  then
    echo "================================================================================================================================="
    echo "==== rke type, command or parameter failed"
    echo "================================================================================================================================="
    print_parameter_error
    exit
  fi
  echo "================================================================================================================================="
  echo " START : sudo $0 $RKE_TYPE $COMMAND $PARAM1 $PARAM2" 
  echo "================================================================================================================================="
}

function sleep_message() {
  echo "**** wait $1 seconds ****"
  for (( c=1; c<=$1; c++ ))
  do
    if [ $((c%10)) -eq 0 ]
    then
      echo "**** $c seconds passed."
    fi
    sleep 1s
  done  
}

function check_nodes_pods() {
  echo "================================================================================================================================="
  echo "========== Check Nodes & Pods ==================================================================================================="
  echo "================================================================================================================================="
  if [ "$EXIST_KUBECONFIG" = "true" ]
  then
    $KUBECTL $KUBECONFIG get nodes
    $KUBECTL $KUBECONFIG get pods -A
  fi
}

function delete_node() {
  echo "================================================================================================================================="
  echo "========== Delete Node =========================================================================================================="
  echo "================================================================================================================================="
  if [ "$EXIST_KUBECONFIG" = "true" ]
  then
    $KUBECTL $KUBECONFIG get nodes
    $KUBECTL $KUBECONFIG drain `hostname` --force --ignore-daemonsets
    $KUBECTL $KUBECONFIG delete node `hostname`
    $KUBECTL $KUBECONFIG get nodes
  fi
}

function uninstall_previous_helmcharts() {
  if [ "$EXIST_KUBECONFIG" = "true" ]
  then
    echo "================================================================================================================================="
    echo "========== previous harbor, longhorn, rancher uninsatll ========================================================================="
    echo "================================================================================================================================="
    $HELM $KUBECONFIG uninstall $MAIN_CR_NAME -n harbor
    $KUBECTL $KUBECONFIG -n longhorn-system patch -p '{"value": "true"}' --type=merge lhs deleting-confirmation-flag
    $HELM $KUBECONFIG uninstall longhorn -n longhorn-system
    $HELM $KUBECONFIG uninstall rancher -n cattle-system
  fi
}

function uninstall_previous_rke2() {
  echo "================================================================================================================================="
  echo "========== previous rke2 uninsatll =============================================================================================="
  echo "================================================================================================================================="
  if [ "$DISTRO" = "centos" ]
  then
    UNINSTALL_PATH=/usr/bin
  else
    UNINSTALL_PATH=/usr/local/bin
  fi
  if [ -r "$UNINSTALL_PATH/rke2-killall.sh" ]
  then
    $UNINSTALL_PATH/rke2-uninstall.sh
  fi
  if [ -r "$UNINSTALL_PATH/rke2-uninstall.sh" ]
  then
    $UNINSTALL_PATH/rke2-killall.sh
    $UNINSTALL_PATH/rke2-uninstall.sh
  fi
  sleep_message 20
  echo "**** Delete existing garbage data and directories."
  LONGHORN_SEARCH=/var/lib/kubelet/plugins/kubernetes.io/csi/driver.longhorn.io/*/globalmount
  rm -rf /etc/rancher
  rm -rf /etc/cni
  rm -rf /opt/cni/bin
  rm -rf /var/lib/rancher
  rm -rf /var/lib/kubelet
  rm -rf /var/lib/longhorn
  rm -rf /var/lib/calico
  rm -rf "$USER_HOME/.kube/"
  echo "**** Check for undeleted longhorn devices !!!!"
  for dir in $LONGHORN_SEARCH; do
    if [ -d "$dir" ]
    then
      echo "**** unmount longhorn device $dir"
      umount $dir
      sleep 1s
    fi
  done
  sleep 2s
  rm -rf /var/lib/kubelet
}

function rke2_server_reset_by_ip_changed() {
  echo "================================================================================================================================="
  echo "========== RKE2 SERVER RESET By Local Host IP Changed ==========================================================================="
  echo "================================================================================================================================="
  echo "**** RKE2 SERVER STOP !!!!"
  systemctl stop rke2-server.service
  echo "**** RKE2 SERVER CLUSTER RESET !!!!"
  rke2 server --cluster-reset
  echo "**** RKE2 SERVER START !!!!"
  systemctl start rke2-server
}

function install_longhorn_required() {
  if [ "$DISTRO" = "ubuntu" ]
  then
    install_longhorn_required_ubuntu
  elif [ "$DISTRO" = "centos" ]
  then
    install_longhorn_required_centos
  else
    echo "Not support : $'$DISTRO"
  fi
}

function install_longhorn_required_ubuntu() {
  echo "================================================================================================================================="
  echo "========== Update system, longhorn rwx support tools install for ubuntu ========================================================="
  echo "================================================================================================================================="
  systemctl disable --now ufw
  apt update && apt install nfs-common open-iscsi -y && apt list --upgradable && sudo apt install curl && sudo apt upgrade -y && sudo apt autoremove -y 
}

function install_longhorn_required_centos() {
  echo "================================================================================================================================="
  echo "========== Update system, longhorn rwx support tools install for centos ========================================================="
  echo "================================================================================================================================="
  systemctl disable --now firewalld
  yum install -y nfs-utils cryptsetup iscsi-initiator-utils
  echo "InitiatorName=$(/sbin/iscsi-iname)" > /etc/iscsi/initiatorname.iscsi
  systemctl enable --now iscsid.service
  yum update -y
  yum clean all
}

function install_rke2_server() {
  echo "================================================================================================================================="
  echo "========== Install RKE2 Server and System Start ================================================================================="
  echo "================================================================================================================================="
  curl -sfL https://get.rke2.io | INSTALL_RKE2_CHANNEL=$RKE2_VER INSTALL_RKE2_TYPE=server sh -
  systemctl enable rke2-server.service
  systemctl start rke2-server.service
}

function add_rke2_server() {
  echo "================================================================================================================================="
  echo "========== Add RKE2 Server and System Start ====================================================================================="
  echo "================================================================================================================================="
  curl -sfL https://get.rke2.io | INSTALL_RKE2_CHANNEL=$RKE2_VER INSTALL_RKE2_TYPE=server sh -
  systemctl enable rke2-server.service
  echo "server: https://$PARAM1:9345" > /etc/rancher/rke2/config.yaml
  echo "token: $PARAM2" >> /etc/rancher/rke2/config.yaml
  systemctl start rke2-server.service
}

function install_rke2_agent() {
  echo "================================================================================================================================="
  echo "========== Install RKE2 Agent and System Start =================================================================================="
  echo "================================================================================================================================="
  curl -sfL https://get.rke2.io | INSTALL_RKE2_CHANNEL=$RKE2_VER INSTALL_RKE2_TYPE=agent sh -
  systemctl enable rke2-agent.service
  echo "server: https://$PARAM1:9345" > /etc/rancher/rke2/config.yaml
  echo "token: $PARAM2" >> /etc/rancher/rke2/config.yaml
  systemctl start rke2-agent.service
}

function install_helm3() {
  echo "================================================================================================================================="
  echo "========== Install Helm3 ========================================================================================================"
  echo "================================================================================================================================="
  # helm chart 
  curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
}

function user_env_setting() {
  echo "================================================================================================================================="
  echo "========== User Environment Settings ============================================================================================"
  echo "================================================================================================================================="
  ln -s /var/lib/rancher/rke2/bin/kubectl /usr/local/bin/kubectl
  if [ -r "$USER_HOME/.bashrc.rke2" ]
  then
    echo "Previous setting $USER_HOME/.bashrc"
  else
    cp "$USER_HOME/.bashrc" "$USER_HOME/.bashrc.rke2"
    echo 'export PATH=/var/lib/rancher/rke2/bin:$PATH' >> $USER_HOME/.bashrc
    source "$USER_HOME/.bashrc"
  fi
  # $KUBECTL config setting
  if [ -r "$USER_HOME/.kube/config" ]
  then
    rm -rf "$USER_HOME/.kube/config"
  else
    mkdir -p "$USER_HOME/.kube"
  fi
  cp "$KUBE_CFILE" "$USER_HOME/.kube/config"
  chown $(id -u $SUDO_USER):$(id -g $SUDO_USER) "$USER_HOME/.kube" -R
  chmod 755 "$USER_HOME/.kube/config"
  if [ -r "$KUBE_CFILE" ]
  then
    EXIST_KUBECONFIG=true
    echo 'alias k=kubectl' >> $USER_HOME/.bashrc
  else
    echo "================================================================================================================================="
    echo "========== INSTALL ABORT   : RKE2, RANCHER, LONGHORN, HARBOR ===================================================================="
    echo "================================================================================================================================="
    exit
  fi
}

function install_cert_manager() {
  echo "================================================================================================================================="
  echo "========== Install Cert Manager ================================================================================================="
  echo "================================================================================================================================="
  $KUBECTL $KUBECONFIG apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.11.0/cert-manager.yaml
}

function install_rancher() {
  echo "================================================================================================================================="
  echo "========== Install Rancher ======================================================================================================"
  echo "**** Rancher Url : $1"
  echo "================================================================================================================================="
  $HELM $KUBECONFIG repo add rancher-latest https://releases.rancher.com/server-charts/latest
  $HELM $KUBECONFIG repo update
  $HELM $KUBECONFIG install rancher rancher-latest/rancher --create-namespace --namespace cattle-system --set hostname=$PARAM1 --set bootstrapPassword=admin --set replicas=1 --set global.cattle.psp.enabled=false
}

function install_longhorn() {
  echo "================================================================================================================================="
  echo "========== Install Longhorn ====================================================================================================="
  echo "================================================================================================================================="
  $HELM $KUBECONFIG repo add longhorn https://charts.longhorn.io
  $HELM $KUBECONFIG repo update
  $HELM $KUBECONFIG install longhorn longhorn/longhorn --namespace longhorn-system --create-namespace --version 1.4.2 --set persistence.defaultClassReplicaCount=1 --set defaultSettings.defaultDataPath=$LONGHORN_DATA_PATH
}

function install_harbor() {
  echo "================================================================================================================================="
  echo "========== Install Longhorn ====================================================================================================="
  echo "**** Harbor Url : $1"
  echo "================================================================================================================================="
  $HELM $KUBECONFIG repo add harbor https://helm.goharbor.io
  $HELM $KUBECONFIG repo update
  $HELM $KUBECONFIG install $MAIN_CR_NAME harbor/harbor --namespace harbor --create-namespace --set chartmuseum.enabled=true \
  --set trivy.enabled=true --set notary.enabled=false --set metrics.enabled=false --set trace.enabled=false \
  --set persistence.persistentVolumeClaim.registry.storageClass=longhorn \
  --set persistence.persistentVolumeClaim.registry.size=100Gi \
  --set persistence.persistentVolumeClaim.jobservice.jobLog.storageClass=longhorn \
  --set persistence.persistentVolumeClaim.jobservice.jobLog.size=10Gi \
  --set persistence.persistentVolumeClaim.database.storageClass=longhorn \
  --set persistence.persistentVolumeClaim.database.size=10Gi \
  --set persistence.persistentVolumeClaim.redis.storageClass=longhorn \
  --set persistence.persistentVolumeClaim.redis.size=50Gi \
  --set persistence.persistentVolumeClaim.trivy.storageClass=longhorn \
  --set persistence.persistentVolumeClaim.trivy.size=10Gi \
  --set expose.ingress.hosts.core=$1 \
  --set externalURL=https://$1
}

function create_docker_registry() {
  echo "================================================================================================================================="
  echo "========== Create Secret Docker Registry ========================================================================================"
  echo "**** Secret Name : $1"
  echo "**** Namespace   : $2"
  echo "**** CR URL      : $3"
  echo "**** Username    : $4"
  echo "**** password    : $5"
  echo "================================================================================================================================="
  # check namespace
  $KUBECTL $KUBECONFIG create namespace $2
  $KUBECTL $KUBECONFIG create secret docker-registry $1 \
    --namespace $2 \
    --docker-server=$3 \
    --docker-username=$4 \
    --docker-password=$5
}

function hosts_setting() {
  echo "================================================================================================================================="
  echo "========== Set Hosts ============================================================================================================"
  echo "**** Hosts 127.0.0.1 : $1 $2"
  echo "================================================================================================================================="
	echo "127.0.0.1 $1 $2" >> /etc/hosts
	cat /etc/hosts
}

function multipath_setting() {
  if [ "$DISTRO" = "centos" ]
  then
    return
  fi
  echo "================================================================================================================================="
  echo "========== Set multipath.conf ==================================================================================================="
  echo "**** 1. config file check"
  echo "**** 2. sudo systemctl restart multipathd"
  echo "================================================================================================================================="
	# Append multipath set
  if [ -r "/etc/multipath.conf.rke2" ]
  then
    cat /etc/multipath.conf
  else
    cp /etc/multipath.conf /etc/multipath.conf.rke2
    echo '
blacklist {
    devnode "^(ram|raw|loop|fd|md|dm-|sr|scd|st)[0-9]*"
    devnode "^sd[a-z]?[0-9]*"
}
' >> /etc/multipath.conf
    cat /etc/multipath.conf
  fi
}

function show_rancher_password() {
  echo "=================================== rancher password ============================================================================"
  $KUBECTL $KUBECONFIG get secret --namespace cattle-system bootstrap-secret -o go-template='{{.data.bootstrapPassword|base64decode}}{{"\n"}}'
  echo "================================================================================================================================="
  echo "**** CMD : $KUBECTL $KUBECONFIG get secret --namespace cattle-system bootstrap-secret -o go-template='{{.data.bootstrapPassword|base64decode}}{{\"\n\"}}'"
  echo "================================================================================================================================="
}
function docker_install() {
  echo "================================================================================================================================="
  echo "========== Docker Install ======================================================================"
  echo "================================================================================================================================="
  yum -y update
  yum install -y yum-utils
  yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
  yum-config-manager --enable docker-ce-nightly
  yum -y install docker-ce docker-ce-cli containerd.io
  systemctl start docker
  systemctl enable docker
}
function skopeo_install() {
  echo "================================================================================================================================="
  echo "========== Docker Image Copy Program Install ======================================================================"
  echo "================================================================================================================================="
  yum -y install skopeo
} 
function rke2_image_registries() {
  echo "================================================================================================================================="
  echo "========== RKE2 Image pull ======================================================================"
  echo "================================================================================================================================="
cat << EOF  > /etc/rancher/rke2/registries.yaml
mirrors:
  $PARAM2:
    endpoint:
      - "http://$PARAM2"
configs:
  "$PARAM2":
    tls:
      insecure_skip_verify: true
EOF
}
function repo_login() {
  echo "================================================================================================================================="
  echo "========== Image Repository Login ======================================================================"
  echo "================================================================================================================================="
  docker login $PARAM2 -u admin -p Harbor12345
  docker login scharbor.security365.com -u admin -p '!QA2ws3ed4rf'
  kubectl create ns idgp
  kubectl create secret docker-registry softcamp-secret --docker-server=harbor.local --docker-username=admin --docker-password=Harbor12345 -n idgp
 
} 
function image_list() {
  echo "=================================================================================================================================" 
  echo "========== Docker Image List ======================================================================"
  echo "================================================================================================================================="
    
	curl -X 'GET' 	'https://scharbor.security365.com/api/v2.0/projects/security365/logs?q=username%3Dadmin%2Coperation%3Dpull&page=1&page_size=100'   -H 'accept: application/json'   -H 'authorization: Basic YWRtaW46IVFBMndzM2VkNHJm' | jq '.' > repo.json

	JSON="repo.json"
	TYPE="resource"

	image=$(grep -o "\"$TYPE\": \"[^\"]*" $JSON | grep -o "[^\"]*$")

	for imagename in $image

	do

	echo ""

	done
 
}
function image_copy() {
  echo "================================================================================================================================="
  echo "========== Docker Image pull ======================================================================"
  echo "================================================================================================================================="

value=`cat imagelist.txt`

for image in $value

do

skopeo copy --dest-tls-verify=false docker://scharbor.security365.com/$image docker://$PARAM2/$image

done
}
function jq_install() {
  echo "================================================================================================================================="
  echo "========== Jq install ======================================================================"
  echo "================================================================================================================================="
  yum install -y epel-release
  yum install -y jq
}
function create_repo() {
  echo "=================================================================================================================================" 
  echo "========== Harbor Repository Create Project ======================================================================"
  echo "================================================================================================================================="
curl -X 'POST' \
  "https://$PARAM2/api/v2.0/projects" \
  -H 'accept: application/json' \
  -H 'X-Resource-Name-In-Location: false' \
  -H 'authorization: Basic YWRtaW46SGFyYm9yMTIzNDU=' \
  -H 'Content-Type: application/json' \
  -d '{
  "project_name": "security365",
  "cve_allowlist": {
    "update_time": "2023-08-14T00:53:05.035Z",
    "project_id": 2,
    "creation_time": "2023-08-14T00:53:05.035Z",
    "id": 2
  },
  "public": true,
  "metadata": {
    "enable_content_trust": "false",
    "enable_content_trust_cosign": "false",
    "auto_scan": "false",
    "severity": "low",
    "prevent_vul": "false",
    "public": "true",
    "reuse_sys_cve_allowlist": "true"
  }
}'

}
function server_install() {
  echo "================================================================================================================================="
  echo "========== INSTALL START : RKE2, RANCHER, LONGHORN, HARBOR, Docker ======================================================================"
  echo "================================================================================================================================="
  install_longhorn_required
  install_rke2_server
  install_helm3
  sleep_message 60
  check_nodes_pods  
  user_env_setting
  install_cert_manager
  sleep_message 60
  check_nodes_pods
  install_longhorn
  sleep_message 60
  check_nodes_pods
  install_harbor $PARAM2
  sleep_message 30
  check_nodes_pods
  install_rancher $PARAM1
  sleep_message 10
  docker_install
  sleep_message 5
  skopeo_install
  create_docker_registry security365acr harbor security365acr.azurecr.io security365acr fomXnaJ6kP2VuyhLOfEgNQ8HlU=lLzoJ
  create_docker_registry localcr harbor $PARAM2 admin Harbor12345
  create_docker_registry security365acr security365 security365acr.azurecr.io security365acr fomXnaJ6kP2VuyhLOfEgNQ8HlU=lLzoJ
  create_docker_registry localcr security365 $PARAM2 admin Harbor12345
  check_nodes_pods
  hosts_setting $PARAM1 $PARAM2
  multipath_setting
  show_rancher_password
  sleep 30
  repo_login
  sleep 10
  rke2_image_registries
  systemctl restart rke2-server
  sleep 30
  create_repo
  sleep 10
  jq_install
  sleep 10
  image_list > imagelist.txt
  sleep 10
  image_copy
  echo
  "================================================================================================================================="
  echo "========== INSTALL END   : RKE2, RANCHER, LONGHORN, HARBOR ======================================================================"
  echo "================================================================================================================================="
}

function server_uninstall() {
  echo "================================================================================================================================="
  echo "========== UNINSTALL START : RKE2, RANCHER, LONGHORN, HARBOR ===================================================================="
  echo "================================================================================================================================="
  check_nodes_pods  
  uninstall_previous_helmcharts
  sleep_message 10
  uninstall_previous_rke2
  echo "================================================================================================================================="
  echo "========== UNINSTALL END   : RKE2, RANCHER, LONGHORN, HARBOR ===================================================================="
  echo "================================================================================================================================="
}

function server_reset() {
  echo "================================================================================================================================="
  echo "========== IPCHANGE START : RKE STOP & CLUSTER RESET ============================================================================"
  echo "================================================================================================================================="
  rke2_server_reset_by_ip_changed  
  echo "================================================================================================================================="
  echo "========== IPCHANGE END   : RKE STOP & CLUSTER RESET ============================================================================"
  echo "================================================================================================================================="
}

function server_add() {
  echo "================================================================================================================================="
  echo "========== ADD MASTER NODE START : RKE MASTER NODE ADD =========================================================================="
  echo "================================================================================================================================="
  install_longhorn_required
  add_rke2_server
  user_env_setting
  echo "================================================================================================================================="
  echo "========== ADD MASTER NODE END   : RKE MASTER NODE ADD =========================================================================="
  echo "================================================================================================================================="
}

function server_remove() {
  echo "================================================================================================================================="
  echo "========== REMOVE MASTER NODE START : RKE MASTER NODE REMOVE ===================================================================="
  echo "================================================================================================================================="
  delete_node
  sleep_message 20
  uninstall_previous_rke2
  echo "================================================================================================================================="
  echo "========== REMOVE MASTER NODE END   : RKE MASTER NODE REMOVE ===================================================================="
  echo "================================================================================================================================="
}

function agent_install() {
  echo "================================================================================================================================="
  echo "========== INSTALL START : RKE2-AGENT ==========================================================================================="
  echo "================================================================================================================================="
  install_longhorn_required
  install_rke2_agent
  echo "================================================================================================================================="
  echo "========== INSTALL END   : RKE2_AGENT ==========================================================================================="
  echo "================================================================================================================================="
}

function agent_uninstall() {
  echo "================================================================================================================================="
  echo "========== UNINSTALL START : RKE2-AGENT ========================================================================================="
  echo "================================================================================================================================="
  uninstall_previous_rke2
  echo "================================================================================================================================="
  echo "========== UNINSTALL END   : RKE2-AGENT ========================================================================================="
  echo "================================================================================================================================="
}



function main() {
  set_linux_distribution
  check_and_set_default_param
  if [ "$RKE_TYPE" = "server" ]
  then
    if [ "$COMMAND" = "install" ]
    then
      server_install
    elif [ "$COMMAND" = "uninstall" ]
    then
      server_uninstall
    elif [ "$COMMAND" = "reinstall" ]
    then
      server_uninstall
      server_install
    elif [ "$COMMAND" = "reset" ]
    then
      server_reset
    elif [ "$COMMAND" = "add" ]
    then
      server_add
    elif [ "$COMMAND" = "remove" ]
    then
      server_remove
    fi
  elif [ "$RKE_TYPE" = "agent" ]
  then
    if [ "$COMMAND" = "install" ]
    then
      agent_install
    elif [ "$COMMAND" = "uninstall" ]
    then
      agent_uninstall
    elif [ "$COMMAND" = "reinstall" ]
    then
      agent_uninstall
      agent_install
    fi
  fi
}

main
