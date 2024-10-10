# netmaker-nginx-proxy-manager-v2
Using an existing NGINX Proxy Manager to access Netmaker UI / API

### Having an existing NPM installed to serve differents websites
### Installing Netmaker the 'Easy Way'
### Access UI / API through existing NPM instance

## Benefits : 

### Permits to run Netmaker along other services on the same "IP", installed "On prem"
### Allows to serve differents services on the same "IP"
### Permits mesh-networking along multiples sites

## Requirements : 

### Full DNS Control of a domain name
### API Access to DNS (Wildcard Let's Encrypt certificates)
### Multiples "hosts" or "virtual hosts" hosted "On Prem"


# Howto : 



### 1 - Run Installer : nm-quick.sh 

```
sudo wget -qO /root/nm-quick.sh https://raw.githubusercontent.com/gravitl/netmaker/master/scripts/nm-quick.sh && sudo chmod +x /root/nm-quick.sh && sudo /root/nm-quick.sh
```

Script V0.1.1 : 


```
#!/bin/bash

CONFIG_FILE=netmaker.env
# location of nm-quick.sh (usually `/root`)
SCRIPT_DIR=$(dirname "$(realpath "$0")")
CONFIG_PATH="$SCRIPT_DIR/$CONFIG_FILE"
NM_QUICK_VERSION="0.1.1"
LATEST=$(curl -s https://api.github.com/repos/gravitl/netmaker/releases/latest | grep "tag_name" | cut -d : -f 2,3 | tr -d [:space:],\")

if [ $(id -u) -ne 0 ]; then
	echo "This script must be run as root"
	exit 1
fi
# increase the timeouts
export DOCKER_CLIENT_TIMEOUT=120
export COMPOSE_HTTP_TIMEOUT=120
unset INSTALL_TYPE
unset BUILD_TAG
unset IMAGE_TAG
unset NETMAKER_BASE_DOMAIN
unset UPGRADE_FLAG
unset COLLECT_PRO_VARS
# usage - displays usage instructions
usage() {
	echo "nm-quick.sh v$NM_QUICK_VERSION"
	echo "usage: ./nm-quick.sh [-c]"
	echo " -c  if specified, will install netmaker community version"
	echo " -p  if specified, will install netmaker pro version"
	echo " -u  if specified, will upgrade netmaker to pro version"
	echo " -d if specified, will downgrade netmaker to community version"
	exit 1
}



# print_logo - prints the netmaker logo
print_logo() {
	cat <<"EOF"
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
                                                                                         
 __   __     ______     ______   __    __     ______     __  __     ______     ______    
/\ "-.\ \   /\  ___\   /\__  _\ /\ "-./  \   /\  __ \   /\ \/ /    /\  ___\   /\  == \   
\ \ \-.  \  \ \  __\   \/_/\ \/ \ \ \-./\ \  \ \  __ \  \ \  _"-.  \ \  __\   \ \  __<   
 \ \_\\"\_\  \ \_____\    \ \_\  \ \_\ \ \_\  \ \_\ \_\  \ \_\ \_\  \ \_____\  \ \_\ \_\ 
  \/_/ \/_/   \/_____/     \/_/   \/_/  \/_/   \/_/\/_/   \/_/\/_/   \/_____/   \/_/ /_/ 
                                                                                                                                                                                                 

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
EOF
}

# set_buildinfo - sets the information based on script input for how the installation should be run
set_buildinfo() {


	
	BUILD_TAG=$LATEST
	IMAGE_TAG=$(sed 's/\//-/g' <<<"$BUILD_TAG")

	if [ -z "$INSTALL_TYPE" ]; then
		echo "-----------------------------------------------------"
		echo "Would you like to install Netmaker Community Edition (CE), or Netmaker Enterprise Edition (pro)?"
		echo "pro will require you to create an account at https://app.netmaker.io"
		echo "-----------------------------------------------------"
		select install_option in "Community Edition" "Enterprise Edition"; do
			case $REPLY in
			1)
				echo "installing Netmaker CE"
				INSTALL_TYPE="ce"
				break
				;;
			2)
				echo "installing Netmaker pro"
				INSTALL_TYPE="pro"
				break
				;;
			*) echo "invalid option $REPLY" ;;
			esac
		done
	fi
	echo "-----------Build Options-----------------------------"
	echo "   Pro or CE: $INSTALL_TYPE"
	echo "   Build Tag: $BUILD_TAG"
	echo "   Image Tag: $IMAGE_TAG"
	echo "   Installer: v$NM_QUICK_VERSION"
	echo "-----------------------------------------------------"

}

# install_yq - install yq if not present
install_yq() {
	if [ -f /etc/debian_version ]; then
		if ! command -v yq &>/dev/null; then
			wget -qO /usr/bin/yq https://github.com/mikefarah/yq/releases/download/v4.31.1/yq_linux_$(dpkg --print-architecture)
			chmod +x /usr/bin/yq
		fi
	fi
	set +e
	if ! command -v yq &>/dev/null; then
		set -e
		wget -qO /usr/bin/yq https://github.com/mikefarah/yq/releases/download/v4.31.1/yq_linux_amd64
		chmod +x /usr/bin/yq
	fi
	set -e
	if ! command -v yq &>/dev/null; then
		echo "failed to install yq. Please install yq and try again."
		echo "https://github.com/mikefarah/yq/#install"
		exit 1
	fi
}

# setup_netclient - adds netclient to docker-compose
setup_netclient() {

	set +e
	if [ -x "$(command -v netclient)" ]; then
		netclient uninstall
	fi
	set -e

	wget -qO netclient https://github.com/gravitl/netclient/releases/download/$LATEST/netclient-linux-$ARCH
	chmod +x netclient
	./netclient install
	echo "Register token: $TOKEN"
	sleep 2
	netclient join -t $TOKEN

	echo "waiting for netclient to become available"
	local found=false
	local file=/etc/netclient/nodes.json
	for ((a = 1; a <= 90; a++)); do
		if [ -f "$file" ]; then
			found=true
			break
		fi
		sleep 1
	done

	if [ "$found" = false ]; then
		echo "Error - $file state not matching"
		exit 1
	fi
}

# configure_netclient - configures server's netclient as a default host and an ingress gateway
configure_netclient() {
	sleep 2
	NODE_ID=$(sudo cat /etc/netclient/nodes.json | jq -r .netmaker.id)
	if [ "$NODE_ID" = "" ] || [ "$NODE_ID" = "null" ]; then
		echo "Error obtaining NODE_ID for the new network"
		exit 1
	fi
	echo "register complete. New node ID: $NODE_ID"
	HOST_ID=$(sudo cat /etc/netclient/netclient.json | jq -r .id)
	if [ "$HOST_ID" = "" ] || [ "$HOST_ID" = "null" ]; then
		echo "Error obtaining HOST_ID for the new network"
		exit 1
	fi
	echo "making host a default"
	echo "Host ID: $HOST_ID"
	# set as a default host
	set +e
	nmctl host update $HOST_ID --default
	sleep 5
	nmctl node create_remote_access_gateway netmaker $NODE_ID
	#setup failOver
	sleep 5
	curl --location --request POST "https://api.${NETMAKER_BASE_DOMAIN}/api/v1/node/${NODE_ID}/failover" --header "Authorization: Bearer ${MASTER_KEY}"
	sleep 2
	# create network for internet access vpn
	if [ "$INSTALL_TYPE" = "pro" ]; then
		INET_NODE_ID=$(sudo cat /etc/netclient/nodes.json | jq -r '."internet-access-vpn".id')
		nmctl node create_remote_access_gateway internet-access-vpn $INET_NODE_ID
		out=$(nmctl node list -o json | jq -r '.[] | select(.id=='\"$INET_NODE_ID\"') | .ingressdns = "8.8.8.8"')
		curl --location --request PUT "https://api.${NETMAKER_BASE_DOMAIN}/api/nodes/internet-access-vpn/${INET_NODE_ID}" --data "$out" --header "Authorization: Bearer ${MASTER_KEY}"
		out=$(nmctl node list -o json | jq -r '.[] | select(.id=='\"$INET_NODE_ID\"') | .metadata = "This host can be used for secure internet access"')
		curl --location --request PUT "https://api.${NETMAKER_BASE_DOMAIN}/api/nodes/internet-access-vpn/${INET_NODE_ID}" --data "$out" --header "Authorization: Bearer ${MASTER_KEY}"
		curl --location --request POST "https://api.${NETMAKER_BASE_DOMAIN}/api/nodes/internet-access-vpn/${INET_NODE_ID}/inet_gw" --data '{}' --header "Authorization: Bearer ${MASTER_KEY}"
	fi
	
	set -e
}

# setup_nmctl - pulls nmctl and makes it executable
setup_nmctl() {

	local URL="https://github.com/gravitl/netmaker/releases/download/$LATEST/nmctl-linux-$ARCH"
	echo "Downloading nmctl..."
	wget -qO /usr/bin/nmctl "$URL"

	if [ ! -f /usr/bin/nmctl ]; then
		echo "Error downloading nmctl from '$URL'"
		exit 1
	fi

	chmod +x /usr/bin/nmctl
	echo "using server api.$NETMAKER_BASE_DOMAIN"
	echo "using master key $MASTER_KEY"
	nmctl context set default --endpoint="https://api.$NETMAKER_BASE_DOMAIN" --master_key="$MASTER_KEY"
	nmctl context use default
	RESP=$(nmctl network list)
	if [[ $RESP == *"unauthorized"* ]]; then
		echo "Unable to properly configure NMCTL, exiting..."
		exit 1
	fi
}

# wait_seconds - wait for the specified period of time
wait_seconds() { (
	for ((a = 1; a <= $1; a++)); do
		echo ". . ."
		sleep 1
	done
); }

# confirm - get user input to confirm that they want to perform the next step
confirm() { (
	while true; do
		read -p 'Does everything look right? [y/n]: ' yn
		case $yn in
		[Yy]*)
			override="true"
			break
			;;
		[Nn]*)
			echo "exiting..."
			exit 1
			# TODO start from the beginning instead
			;;
		*) echo "Please answer yes or no." ;;
		esac
	done
) }


save_config() { (
	echo "Saving the config to $CONFIG_PATH"
	touch "$CONFIG_PATH"
	if [ -n "$EMAIL" ]; then 
		save_config_item NM_EMAIL "$EMAIL"
	fi
	if [ -n "$NETMAKER_BASE_DOMAIN" ]; then
		save_config_item NM_DOMAIN "$NETMAKER_BASE_DOMAIN"
		save_config_item FRONTEND_URL "https://dashboard.$NETMAKER_BASE_DOMAIN"
	fi
	save_config_item UI_IMAGE_TAG "$IMAGE_TAG"
	# version-specific entries
	if [ "$INSTALL_TYPE" = "pro" ]; then
		save_config_item NETMAKER_TENANT_ID "$NETMAKER_TENANT_ID"
		save_config_item LICENSE_KEY "$LICENSE_KEY"
		if [ "$UPGRADE_FLAG" = "yes" ];then
			save_config_item METRICS_EXPORTER "on"
			save_config_item PROMETHEUS "on"
		fi
		save_config_item SERVER_IMAGE_TAG "$IMAGE_TAG-ee"
	else
		save_config_item METRICS_EXPORTER "off"
		save_config_item PROMETHEUS "off"
		save_config_item SERVER_IMAGE_TAG "$IMAGE_TAG"
	fi
	# copy entries from the previous config
	local toCopy=("SERVER_HOST" "MASTER_KEY" "MQ_USERNAME" "MQ_PASSWORD" "LICENSE_KEY" "NETMAKER_TENANT_ID"
		"INSTALL_TYPE" "NODE_ID" "DNS_MODE" "NETCLIENT_AUTO_UPDATE" "API_PORT"
		"CORS_ALLOWED_ORIGIN" "DISPLAY_KEYS" "DATABASE" "SERVER_BROKER_ENDPOINT" "VERBOSITY"
		"DEBUG_MODE"  "REST_BACKEND" "DISABLE_REMOTE_IP_CHECK" "TELEMETRY" "ALLOWED_EMAIL_DOMAINS" "AUTH_PROVIDER" "CLIENT_ID" "CLIENT_SECRET"
		"FRONTEND_URL" "AZURE_TENANT" "OIDC_ISSUER" "EXPORTER_API_PORT" "JWT_VALIDITY_DURATION" "RAC_AUTO_DISABLE" "CACHING_ENABLED" "ENDPOINT_DETECTION"
		"SMTP_HOST" "SMTP_PORT" "EMAIL_SENDER_ADDR" "EMAIL_SENDER_USER" "EMAIL_SENDER_PASSWORD")
	for name in "${toCopy[@]}"; do
		save_config_item $name "${!name}"
	done
	# preserve debug entries
	if test -n "$NM_SKIP_BUILD"; then
		save_config_item NM_SKIP_BUILD "$NM_SKIP_BUILD"
	fi
	if test -n "$NM_SKIP_CLONE"; then
		save_config_item NM_SKIP_CLONE "$NM_SKIP_CLONE"
	fi
	if test -n "$NM_SKIP_DEPS"; then
		save_config_item NM_SKIP_DEPS "$NM_SKIP_DEPS"
	fi
); }

save_config_item() { (
	local NAME="$1"
	local VALUE="$2"
	#echo "$NAME=$VALUE"
	if test -z "$VALUE"; then
		# load the default for empty values
		VALUE=$(awk -F'=' "/^$NAME/ { print \$2}"  "$SCRIPT_DIR/netmaker.default.env")
		# trim quotes for docker
		VALUE=$(echo "$VALUE" | sed -E "s|^(['\"])(.*)\1$|\2|g")
		#echo "Default for $NAME=$VALUE"
	fi
	# TODO single quote passwords
	if grep -q "^$NAME=" "$CONFIG_PATH"; then
		# TODO escape | in the value
		sed -i "s|$NAME=.*|$NAME=$VALUE|" "$CONFIG_PATH"
	else
		echo "$NAME=$VALUE" >>"$CONFIG_PATH"
	fi
); }



# install_dependencies - install necessary packages to run netmaker
install_dependencies() {

	if test -n "$NM_SKIP_DEPS"; then
		return
	fi

	echo "checking dependencies..."

	OS=$(uname)
	if [ -f /etc/debian_version ]; then
		dependencies="git wireguard-tools dnsutils jq docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin grep gawk"
		update_cmd='apt update'
		install_cmd='apt-get install -y'
	elif [ -f /etc/alpine-release ]; then
		dependencies="git wireguard jq docker.io docker-compose grep gawk"
		update_cmd='apk update'
		install_cmd='apk --update add'
	elif [ -f /etc/centos-release ]; then
		dependencies="wget git wireguard-tools jq bind-utils docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin grep gawk"
		update_cmd='yum updateinfo'
		install_cmd='yum install -y'
	elif [ -f /etc/amazon-linux-release ]; then
		dependencies="git wireguard-tools bind-utils jq docker grep gawk"
		update_cmd='yum updateinfo'
		install_cmd='yum install -y'
	elif [ -f /etc/fedora-release ]; then
		dependencies="wget git wireguard-tools bind-utils jq docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin grep gawk"
		update_cmd='dnf updateinfo'
		install_cmd='dnf install -y'
	elif [ -f /etc/redhat-release ]; then
		dependencies="wget git wireguard-tools jq bind-utils docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin grep gawk"
		update_cmd='yum updateinfo'
		install_cmd='yum install -y'
	elif [ -f /etc/arch-release ]; then
		dependencies="git wireguard-tools dnsutils jq docker.io docker-compose grep gawk"
		update_cmd='pacman -Sy'
		install_cmd='pacman -S --noconfirm'
	elif [ "${OS}" = "FreeBSD" ]; then
		dependencies="git wireguard wget jq docker.io docker-compose grep gawk"
		update_cmd='pkg update'
		install_cmd='pkg install -y'
	else
		echo "-----------------------nm-quick.sh----------------------------------------------"
		echo "OS supported and tested include:"
		echo "   Debian"
		echo "   Ubuntu"
		echo "   Fedora"
		echo "   Centos"
		echo "   Redhat"
		echo "   Amazon Linux"
		echo "   Rocky Linux"
		echo "   AlmaLinux"

		echo "Your OS system is not in the support list, please chanage to an OS in the list"
		echo "--------------------------------------------------------------------------------"
		exit 1
	fi

	if [ -z "${install_cmd}" ]; then
		echo "OS unsupported for automatic dependency install"
		# TODO shouldnt exit, check if deps available, if not
		#  ask the user to install manually and continue when ready
		exit 1
	fi
	# TODO add other supported architectures
	ARCH=$(uname -m)
    if [ "$ARCH" = "x86_64" ]; then
    	ARCH=amd64
    elif [ "$ARCH" = "aarch64" ]; then
    	ARCH=arm64
    else
    	echo "Unsupported architechure"
    	# exit 1
    fi

	# setup docker repository
	if [ "$(cat /etc/*-release |grep ubuntu |wc -l)" -gt 0 ]; then
		apt update
		apt install -y ca-certificates curl
		install -m 0755 -d /etc/apt/keyrings
		curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
		chmod a+r /etc/apt/keyrings/docker.asc
		echo \
		  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
		  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
		  tee /etc/apt/sources.list.d/docker.list > /dev/null
		apt update
	elif [ "$(cat /etc/*-release |grep debian |wc -l)" -gt 0 ]; then
		apt update
		apt install -y ca-certificates curl
		install -m 0755 -d /etc/apt/keyrings
		curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc
		chmod a+r /etc/apt/keyrings/docker.asc
		echo \
		  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/debian \
		  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
		  tee /etc/apt/sources.list.d/docker.list > /dev/null
		apt update
	elif [ -f /etc/fedora-release ]; then
		dnf -y install dnf-plugins-core
		dnf config-manager --add-repo https://download.docker.com/linux/fedora/docker-ce.repo
	elif [ -f /etc/centos-release ]; then
		yum install -y yum-utils
		yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
		if [ "$(cat /etc/*-release |grep 'release 8' |wc -l)" -gt 0 ]; then
			yum install -y elrepo-release epel-release
		elif [ "$(cat /etc/*-release |grep 'release 7' |wc -l)" -gt 0 ]; then
			yum install -y elrepo-release epel-release
			yum install -y yum-plugin-elrepo
		fi
	elif [ -f /etc/redhat-release ]; then
		yum install -y yum-utils
		yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
		if [ "$(cat /etc/*-release |grep 'release 8' |wc -l)" -gt 0 ]; then
			yum install -y elrepo-release epel-release
		fi
	fi

	set -- $dependencies

	${update_cmd}

	while [ -n "$1" ]; do
		if [ "${OS}" = "FreeBSD" ]; then
			is_installed=$(pkg check -d $1 | grep "Checking" | grep "done")
			if [ "$is_installed" != "" ]; then
				echo "  " $1 is installed
			else
				echo "  " $1 is not installed. Attempting install.
				${install_cmd} $1
				sleep 5
				is_installed=$(pkg check -d $1 | grep "Checking" | grep "done")
				if [ "$is_installed" != "" ]; then
					echo "  " $1 is installed
				elif [ -x "$(command -v $1)" ]; then
					echo "  " $1 is installed
				else
					echo "  " FAILED TO INSTALL $1
					echo "  " This may break functionality.
				fi
			fi
		else
			if [ "${OS}" = "OpenWRT" ] || [ "${OS}" = "TurrisOS" ]; then
				is_installed=$(opkg list-installed $1 | grep $1)
			elif [ -f /etc/debian_version ]; then
				is_installed=$(dpkg-query -W --showformat='${Status}\n' $1 | grep "install ok installed")
			else
				is_installed=$(yum list installed | grep $1)
			fi
			if [ "${is_installed}" != "" ]; then
				echo "    " $1 is installed
			else
				echo "    " $1 is not installed. Attempting install.
				${install_cmd} $1
				sleep 5
				if [ "${OS}" = "OpenWRT" ] || [ "${OS}" = "TurrisOS" ]; then
					is_installed=$(opkg list-installed $1 | grep $1)
				elif [ -f /etc/debian_version ]; then
					is_installed=$(dpkg-query -W --showformat='${Status}\n' $1 | grep "install ok installed")
				else
					is_installed=$(yum list installed | grep $1)
				fi
				if [ "${is_installed}" != "" ]; then
					echo "    " $1 is installed
				elif [ -x "$(command -v $1)" ]; then
					echo "  " $1 is installed
				else
					echo "  " FAILED TO INSTALL $1
					echo "  " This may break functionality.
				fi
			fi
		fi
		shift
	done

	# Startup docker daemon for OS which does not start it automatically
	if [ -f /etc/fedora-release ]; then
		systemctl start docker
		systemctl enable docker
	elif [ -f /etc/amazon-linux-release ]; then
		systemctl start docker
		systemctl enable docker
		usermod -a -G docker ec2-user
		mkdir -p /usr/local/lib/docker/cli-plugins
		curl -sL https://github.com/docker/compose/releases/latest/download/docker-compose-linux-$(uname -m) \
		-o /usr/local/lib/docker/cli-plugins/docker-compose
		chown root:root /usr/local/lib/docker/cli-plugins/docker-compose
		chmod +x /usr/local/lib/docker/cli-plugins/docker-compose
	elif [ -f /etc/centos-release ]; then
		systemctl start docker
		systemctl enable docker
	elif [ -f /etc/redhat-release ]; then
		systemctl start docker
		systemctl enable docker
	fi

	echo "-----------------------------------------------------"
	echo "dependency check complete"
	echo "-----------------------------------------------------"
}
set -e

# set_install_vars - sets the variables that will be used throughout installation
set_install_vars() {

	IP_ADDR=$(dig -4 myip.opendns.com @resolver1.opendns.com +short)
	if [ "$IP_ADDR" = "" ]; then
		IP_ADDR=$(curl -s ifconfig.me)
	fi
	if [ "$NETMAKER_BASE_DOMAIN" = "" ]; then
		NETMAKER_BASE_DOMAIN=nm.$(echo $IP_ADDR | tr . -).nip.io
	fi
	SERVER_HOST=$IP_ADDR
	if test -z "$MASTER_KEY"; then
		MASTER_KEY=$(
			tr -dc A-Za-z0-9 </dev/urandom | head -c 30
			echo ''
		)
	fi
	DOMAIN_TYPE=""
	echo "-----------------------------------------------------"
	echo "Would you like to use your own domain for netmaker, or an auto-generated domain?"
	echo "To use your own domain, add a Wildcard DNS record (e.x: *.netmaker.example.com) pointing to $SERVER_HOST"
	echo "IMPORTANT: Due to the high volume of requests, the auto-generated domain has been rate-limited by the certificate provider."
	echo "For this reason, we STRONGLY RECOMMEND using your own domain. Using the auto-generated domain may lead to a failed installation due to rate limiting."
	echo "-----------------------------------------------------"

	select domain_option in "Auto Generated ($NETMAKER_BASE_DOMAIN)" "Custom Domain (e.x: netmaker.example.com)"; do
		case $REPLY in
		1)
			echo "using $NETMAKER_BASE_DOMAIN for base domain"
			DOMAIN_TYPE="auto"
			break
			;;
		2)
			read -p "Enter Custom Domain (make sure  *.domain points to $SERVER_HOST first): " domain
			NETMAKER_BASE_DOMAIN=$domain
			echo "using $NETMAKER_BASE_DOMAIN"
			DOMAIN_TYPE="custom"
			break
			;;
		*) echo "invalid option $REPLY" ;;
		esac
	done

	wait_seconds 2

	echo "-----------------------------------------------------"
	echo "The following subdomains will be used:"
	echo "          dashboard.$NETMAKER_BASE_DOMAIN"
	echo "                api.$NETMAKER_BASE_DOMAIN"
	echo "             broker.$NETMAKER_BASE_DOMAIN"

	if [ "$UPGRADE_FLAG" = "yes" ]; then
		echo "         prometheus.$NETMAKER_BASE_DOMAIN"
		echo "  netmaker-exporter.$NETMAKER_BASE_DOMAIN"
		echo "            grafana.$NETMAKER_BASE_DOMAIN"
	fi

	echo "-----------------------------------------------------"

	if [[ "$DOMAIN_TYPE" == "custom" ]]; then
		echo "before continuing, confirm DNS is configured correctly, with records pointing to $SERVER_HOST"
		confirm
	fi

	wait_seconds 1

	unset GET_EMAIL
	while [ -z "$GET_EMAIL" ]; do
		read -p "Email Address for Domain Registration: " GET_EMAIL
	done
	EMAIL="$GET_EMAIL"

	wait_seconds 1
	if [ "$COLLECT_PRO_VARS" = "true" ]; then
		unset LICENSE_KEY
		while [ -z "$LICENSE_KEY" ]; do
			read -p "License Key: " LICENSE_KEY
		done
		unset NETMAKER_TENANT_ID
		while [ -z ${NETMAKER_TENANT_ID} ]; do
			read -p "Tenant ID: " NETMAKER_TENANT_ID
		done
	fi
	wait_seconds 1
	unset GET_MQ_USERNAME
	unset GET_MQ_PASSWORD
	unset CONFIRM_MQ_PASSWORD
	echo "Enter Credentials For MQ..."
	
	read -p "MQ Username (click 'enter' to use 'netmaker'): " GET_MQ_USERNAME
	if [ -z "$GET_MQ_USERNAME" ]; then
		echo "using default username for mq"
		MQ_USERNAME="netmaker"
	else
		MQ_USERNAME="$GET_MQ_USERNAME"
	fi

	if test -z "$MQ_PASSWORD"; then
		MQ_PASSWORD=$(
			tr -dc A-Za-z0-9 </dev/urandom | head -c 30
			echo ''
		)
	fi


	select domain_option in "Auto Generated / Config Password" "Input Your Own Password"; do
		case $REPLY in
		1)
			echo "using random password for mq"
			break
			;;
		2)
			while true; do
				echo "Enter your Password For MQ: "
				read -s GET_MQ_PASSWORD
				echo "Enter your password again to confirm: "
				read -s CONFIRM_MQ_PASSWORD
				if [ ${GET_MQ_PASSWORD} != ${CONFIRM_MQ_PASSWORD} ]; then
					echo "wrong password entered, try again..."
					continue
				fi
				MQ_PASSWORD="$GET_MQ_PASSWORD"
				echo "MQ Password Saved Successfully!!"
				break
			done
			break
			;;
		*) echo "invalid option $REPLY" ;;
		esac
	done
	

	wait_seconds 2

	echo "-----------------------------------------------------------------"
	echo "                SETUP ARGUMENTS"
	echo "-----------------------------------------------------------------"
	echo "        domain: $NETMAKER_BASE_DOMAIN"
	echo "         email: $EMAIL"
	echo "     public ip: $SERVER_HOST"
	echo "-----------------------------------------------------------------"
	echo "Confirm Settings for Installation"
	echo "- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -"

	confirm
}

# install_netmaker - sets the config files and starts docker-compose
install_netmaker() {

	echo "-----------------------------------------------------------------"
	echo "Beginning installation..."
	echo "-----------------------------------------------------------------"

	wait_seconds 3

	echo "Pulling config files..."

	
	local BASE_URL="https://raw.githubusercontent.com/gravitl/netmaker/$BUILD_TAG"
	local COMPOSE_URL="$BASE_URL/compose/docker-compose.yml"
	local CADDY_URL="$BASE_URL/docker/Caddyfile"
	if [ "$INSTALL_TYPE" = "pro" ]; then
		local COMPOSE_OVERRIDE_URL="$BASE_URL/compose/docker-compose.pro.yml"
		local CADDY_URL="$BASE_URL/docker/Caddyfile-pro"
	fi
	wget -qO "$SCRIPT_DIR"/docker-compose.yml $COMPOSE_URL
	if [ "$UPGRADE_FLAG" = "yes" ]; then
		wget -qO "$SCRIPT_DIR"/docker-compose.override.yml $COMPOSE_OVERRIDE_URL
	elif [ -a "$SCRIPT_DIR"/docker-compose.override.yml ]; then
		rm -f "$SCRIPT_DIR"/docker-compose.override.yml
	fi
	wget -qO "$SCRIPT_DIR"/Caddyfile "$CADDY_URL"
	wget -qO "$SCRIPT_DIR"/netmaker.default.env "$BASE_URL/scripts/netmaker.default.env"
	wget -qO "$SCRIPT_DIR"/mosquitto.conf "$BASE_URL/docker/mosquitto.conf"
	wget -qO "$SCRIPT_DIR"/wait.sh "$BASE_URL/docker/wait.sh"

	chmod +x "$SCRIPT_DIR"/wait.sh
	mkdir -p /etc/netmaker

	# link .env to the user config
	ln -fs "$SCRIPT_DIR/netmaker.env" "$SCRIPT_DIR/.env"
	save_config

	echo "Starting containers..."

	# start docker and rebuild containers / networks
	cd "${SCRIPT_DIR}"
	if [ -f /etc/debian_version ]; then
		docker compose up -d --force-recreate
	elif [ -f /etc/fedora-release ]; then
		docker compose up -d --force-recreate
	elif [ -f /etc/amazon-linux-release ]; then
		docker compose up -d --force-recreate
	elif [ -f /etc/centos-release ]; then
		docker compose up -d --force-recreate
	elif [ -f /etc/redhat-release ]; then
		docker compose up -d --force-recreate
	else
		docker-compose up -d --force-recreate
	fi
	cd -
	wait_seconds 2

}

# test_connection - tests to make sure Caddy has proper SSL certs
test_connection() {

	echo "Testing Caddy setup (please be patient, this may take 1-2 minutes)"
	for i in 1 2 3 4 5 6 7 8; do
		curlresponse=$(curl -vIs https://api.${NETMAKER_BASE_DOMAIN} 2>&1)

		if [[ "$i" == 8 ]]; then
			echo "    Caddy is having an issue setting up certificates, please investigate (docker logs caddy)"
			echo "    Exiting..."
			exit 1
		elif [[ "$curlresponse" == *"failed to verify the legitimacy of the server"* ]]; then
			echo "    Certificates not yet configured, retrying..."

		elif [[ "$curlresponse" == *"left intact"* ]]; then
			echo "    Certificates ok"
			break
		else
			secs=$(($i * 5 + 10))
			echo "    Issue establishing connection...retrying in $secs seconds..."
		fi
		sleep $secs
	done

}

# setup_mesh - sets up a default mesh network on the server
setup_mesh() {

	wait_seconds 5
	networks=$(nmctl network list -o json)
	if [[ ${networks} != "null" ]]; then
		netmakerNet=$(nmctl network list -o json | jq -r '.[] | .netid' | grep -w "netmaker")
		inetNet=$(nmctl network list -o json | jq -r '.[] | .netid' | grep -w "internet-access-vpn")
	fi
	# create netmaker network
	if [[ ${netmakerNet} = "" ]]; then
		echo "Creating netmaker network (100.64.0.0/16)"
		# TODO causes "Error Status: 400 Response: {"Code":400,"Message":"could not find any records"}"
		nmctl network create --name netmaker --ipv4_addr 100.64.0.0/16
	fi
	# create enrollment key for netmaker network
	local netmakerTag=$(nmctl enrollment_key list | jq -r '.[] | .tags[0]' | grep -w "netmaker")
	if [[ ${netmakerTag} = "" ]]; then
		nmctl enrollment_key create --tags netmaker --unlimited --networks netmaker
	fi

	# create internet-access-vpn
	if [ "$INSTALL_TYPE" = "pro" ]; then
		if [[ ${inetNet} = "" ]]; then
			echo "Creating internet-access-vpn network (100.65.0.0/16)"
			# TODO causes "Error Status: 400 Response: {"Code":400,"Message":"could not find any records"}"
			nmctl network create --name internet-access-vpn --ipv4_addr 100.65.0.0/16
		fi

		# create enrollment key for internet-access-vpn network
		local inetTag=$(nmctl enrollment_key list | jq -r '.[] | .tags[0]' | grep -w "internet-access-vpn")
		if [[ ${inetTag} = "" ]]; then
			nmctl enrollment_key create --tags internet-access-vpn --unlimited --networks internet-access-vpn
		fi

		# create enrollment key for both networks
		local netInetTag=$(nmctl enrollment_key list | jq -r '.[] | .tags[0]' | grep -w "netmaker-inet")
		if [[ ${netInetTag} = "" ]]; then
			nmctl enrollment_key create --tags netmaker-inet --unlimited --networks netmaker,internet-access-vpn
		fi
	fi

	if [ "$INSTALL_TYPE" = "pro" ]; then
		# create enrollment key for both setup networks
		echo "Obtaining enrollment key..."
		# key exists already, fetch token
		TOKEN=$(nmctl enrollment_key list | jq -r '.[] | select(.tags[0]=="netmaker-inet") | .token')
		
	else

		echo "Obtaining enrollment key..."
		# key exists already, fetch token
		TOKEN=$(nmctl enrollment_key list | jq -r '.[] | select(.tags[0]=="netmaker") | .token')
	fi
	
	wait_seconds 3

}

# print_success - prints a success message upon completion
print_success() {
	echo "-----------------------------------------------------------------"
	echo "-----------------------------------------------------------------"
	echo "Netmaker setup is now complete. You are ready to begin using Netmaker."
	echo "Visit dashboard.$NETMAKER_BASE_DOMAIN to log in"
	echo "-----------------------------------------------------------------"
	echo "-----------------------------------------------------------------"
}

cleanup() {
	# remove the existing netclient's instance from the existing network
	if ! command -v netclient >/dev/null 2>&1; then
		return
	fi
	if command -v nmctl >/dev/null 2>&1; then
		local node_id=$(netclient list | jq '.[0].node_id' 2>/dev/null)
		# trim doublequotes
		node_id="${node_id//\"/}"
		if test -n "$node_id"; then
			echo "De-registering the existing netclient..."
			nmctl node delete netmaker $node_id >/dev/null 2>&1
		fi
	fi

	stop_services
}

stop_services(){
	echo "Stopping all containers, this will take a while please wait..."
	local containers=("mq" "netmaker-ui" "coredns" "turn" "caddy" "netmaker" "netmaker-exporter" "prometheus" "grafana")
	for name in "${containers[@]}"; do
		local running=$(docker ps | grep -w "$name")
		local exists=$(docker ps -a | grep -w "$name")
		if test -n "$running"; then
			docker stop "$name" 1>/dev/null
		fi
		if test -n "$exists"; then
			docker rm "$name" 1>/dev/null
		fi
	done
}

upgrade() {
	print_logo
	unset IMAGE_TAG
	unset BUILD_TAG
	IMAGE_TAG=$UI_IMAGE_TAG
	semver=$(chsv_check_version_ex "$UI_IMAGE_TAG")
  	if [[ ! "$semver" ]]; then
		BUILD_TAG=$LATEST
	else
		BUILD_TAG=$UI_IMAGE_TAG
  	fi
	echo "-----------------------------------------------------"
	echo "Provide Details for pro installation:"
	echo "    1. Log into https://app.netmaker.io"
	echo "    2. follow instructions to get a license at: https://docs.netmaker.io/pro/pro-setup.html"
	echo "    3. Retrieve License and Tenant ID"
	echo "-----------------------------------------------------"
	unset LICENSE_KEY
	while [ -z "$LICENSE_KEY" ]; do
		read -p "License Key: " LICENSE_KEY
	done
	unset NETMAKER_TENANT_ID
	while [ -z ${NETMAKER_TENANT_ID} ]; do
		read -p "Tenant ID: " NETMAKER_TENANT_ID
	done
	save_config
	# start docker and rebuild containers / networks
	stop_services
	install_netmaker
}

downgrade () {
	print_logo
	unset IMAGE_TAG
	unset BUILD_TAG
	IMAGE_TAG=$UI_IMAGE_TAG

	semver=$(chsv_check_version_ex "$UI_IMAGE_TAG")
  	if [[ ! "$semver" ]]; then
		BUILD_TAG=$LATEST
	else
		BUILD_TAG=$UI_IMAGE_TAG
  	fi
	save_config
	if [ -a "$SCRIPT_DIR"/docker-compose.override.yml ]; then
		rm -f "$SCRIPT_DIR"/docker-compose.override.yml
	fi
	# start docker and rebuild containers / networks
	stop_services
	install_netmaker
}

function chsv_check_version() {
  if [[ $1 =~ ^(0|[1-9][0-9]*)\.(0|[1-9][0-9]*)\.(0|[1-9][0-9]*)(-((0|[1-9][0-9]*|[0-9]*[a-zA-Z-][0-9a-zA-Z-]*)(\.(0|[1-9][0-9]*|[0-9]*[a-zA-Z-][0-9a-zA-Z-]*))*))?(\+([0-9a-zA-Z-]+(\.[0-9a-zA-Z-]+)*))?$ ]]; then
    echo "$1"
  else
    echo ""
  fi
}

function chsv_check_version_ex() {
  if [[ $1 =~ ^v.+$ ]]; then
    chsv_check_version "${1:1}"
  else
    chsv_check_version "${1}"
  fi
}



main (){

	# read the config
	if [ -f "$CONFIG_PATH" ]; then
		echo "Using config: $CONFIG_PATH"
		source "$CONFIG_PATH"
	fi

	INSTALL_TYPE="pro"
	while getopts :cudpv flag; do
	case "${flag}" in
	c)
		INSTALL_TYPE="ce"
		;;
	u)
		echo "upgrading to pro version..."
		INSTALL_TYPE="pro"
		UPGRADE_FLAG="yes"
		upgrade
		exit 0
		;;
	d) 
		echo "downgrading to community version..."
		INSTALL_TYPE="ce"
		downgrade
		exit 0
		;;
	p)
		echo "installing pro version..."
		INSTALL_TYPE="pro"
		COLLECT_PRO_VARS="true"
		;;
	v)
		usage
		exit 0
		;;
	esac
done

	# 1. print netmaker logo
	print_logo

	# 2. setup the build instructions
	set_buildinfo
	set +e
	# 3. install necessary packages
	install_dependencies

	# 4. install yq if necessary
	install_yq

	set -e

	# 6. get user input for variables
	set_install_vars

	set +e
	cleanup
	set -e

	# 7. get and set config files, startup docker-compose
	install_netmaker

	set +e

	# 8. make sure Caddy certs are working
	test_connection

	# 9. install the netmaker CLI
	setup_nmctl

	# 10. create a default mesh network for netmaker
	setup_mesh

	set -e

	# 11. add netclient to docker-compose and start it up
	setup_netclient

	# 12. make the netclient a default host and ingress gw
	configure_netclient

	# 13. print success message
	print_success
}

main "${@}"
```



### 2 - Shutdown Netmaker : 
```
docker compose down
```
### 3 - Modify docker-compose.yml

```
version: "3.4"

services:

  netmaker:
    container_name: netmaker
    image: gravitl/netmaker:$SERVER_IMAGE_TAG
    env_file: ./netmaker.env
    restart: always
    volumes:
      - dnsconfig:/root/config/dnsconfig
      - sqldata:/root/data
    environment:
      # config-dependant vars
      - STUN_LIST=stun1.netmaker.io:3478,stun2.netmaker.io:3478,stun1.l.google.com:19302,stun2.l.google.com:19302
      # The domain/host IP indicating the mq broker address
      - BROKER_ENDPOINT=wss://broker.${NM_DOMAIN} # For EMQX broker use `BROKER_ENDPOINT=wss://broker.${NM_DOMAIN}/mqtt`
      # For EMQX broker (uncomment the two lines below)
      #- BROKER_TYPE=emqx
      #- EMQX_REST_ENDPOINT=http://mq:18083
      # The base domain of netmaker
      - SERVER_NAME=${NM_DOMAIN}
      - SERVER_API_CONN_STRING=api.${NM_DOMAIN}:443
      # Address of the CoreDNS server. Defaults to SERVER_HOST
      - COREDNS_ADDR=${SERVER_HOST}
      # Overrides SERVER_HOST if set. Useful for making HTTP available via different interfaces/networks.
      - SERVER_HTTP_HOST=api.${NM_DOMAIN}
    ports:
      - "8081:8081"
      - "50051:50051"

  netmaker-ui:
    container_name: netmaker-ui
    image: gravitl/netmaker-ui:$UI_IMAGE_TAG
    env_file: ./netmaker.env
    environment:
      # config-dependant vars
      # URL where UI will send API requests. Change based on SERVER_HOST, SERVER_HTTP_HOST, and API_PORT
      BACKEND_URL: "https://api.${NM_DOMAIN}"
    depends_on:
      - netmaker
    links:
      - "netmaker:api"
    restart: always
    ports:
      - "80:80"

  #caddy:
  #  image: caddy:2.6.2
  #  container_name: caddy
  #  env_file: ./netmaker.env
  #  restart: unless-stopped
  #  extra_hosts:
  #    - "host.docker.internal:host-gateway"
  #  volumes:
  #    - ./Caddyfile:/etc/caddy/Caddyfile
  #    - caddy_data:/data
  #    - caddy_conf:/config
  #  ports:
  #    - "80:80"
  #    - "443:443"

  coredns:
    #network_mode: host
    container_name: coredns
    image: coredns/coredns:1.10.1
    command: -conf /root/dnsconfig/Corefile
    env_file: ./netmaker.env
    restart: always
    depends_on:
      - netmaker
    volumes:
      - dnsconfig:/root/dnsconfig
  mq:
    container_name: mq
    image: eclipse-mosquitto:2.0.15-openssl
    env_file: ./netmaker.env
    depends_on:
      - netmaker
    restart: unless-stopped
    command: [ "/mosquitto/config/wait.sh" ]
    volumes:
      - ./mosquitto.conf:/mosquitto/config/mosquitto.conf
      - ./wait.sh:/mosquitto/config/wait.sh
      - mosquitto_logs:/mosquitto/log
      - mosquitto_data:/mosquitto/data
    ports:
      - "8883:8883"
      - "1883:1883"
    
volumes:
  caddy_data: { } # runtime data for caddy
  caddy_conf: { } # configuration file for Caddy
  sqldata: { }
  dnsconfig: { } # storage for coredns
  mosquitto_logs: { } # storage for mqtt logs
  mosquitto_data: { } # storage for mqtt data
  
```

  
 Container "Netmaker" : add ports to expose on host : 
     ports:
      - "8081:8081"
      - "50051:50051"

      
 Container "netmaker-ui" : add ports to expose on virtual host : 
     ports:
      - "80:80"

      
 Container "mq" : add ports to expose on virtual host : 
     ports:
      - "8883:8883"
      - "1883:1883"

      
 Container "caddy" : disable container : 
 
 ```
   #caddy:
  #  image: caddy:2.6.2
  #  container_name: caddy
  #  env_file: ./netmaker.env
  #  restart: unless-stopped
  #  extra_hosts:
  #    - "host.docker.internal:host-gateway"
  #  volumes:
  #    - ./Caddyfile:/etc/caddy/Caddyfile
  #    - caddy_data:/data
  #    - caddy_conf:/config
  #  ports:
  #    - "80:80"
  #    - "443:443"
```
  
 ### 4 - Configure Nginx Proxy Manager : 

  
 ### 4.1 - Request wildcard certificate for *.netmaker.domain.com

  
  SSL Certificate > Add SSL Certificate > Let's Encrypt > *.netmaker.domain.com

  
  Email : youremail@yourdomain.com
  
  Use a DNS Challenge : YES
  
  Choose provider / Provide API Keys or Logins
  
  Propagation seconds : default
  
  I Agree to the.. : YES

  
###  4.2 - Create Hosts : 

  
###  4.2.1 - api.netmaker.domain.com

  Scheme : http
  
  Address : virtual host address

  Forward Port : 8081
  
  SSL : *.netmaker.domain.com

  
 ### 4.2.2 - broker.netmaker.domain.com
 
  Scheme : http
  
  Address : virtual host address
  
  Forward port : 8883
  
  Websocket Support : YES
  
  SSL : *.netmaker.domain.com

  
 ### 4.2.3 - dashboard.netmaker.domain.com
 
  Scheme : http
  
  Address : virtual host address
  
  Forward port : 80
  
  SSL : *.netmaker.domain.com

  
###  4.2.4 - gprc.netmaker.domain.com

  Scheme : http
  
  Address : virtual host address
  
  Forward port : 50051
  
  Custom Location : /
  
  Scheme : http
  
  Address : virtual host address 

  Forward port : 50051
  
  Option :  grpc_pass <virtual-host-address>:50051;
  
  SSL : *.netmaker.domain.com
  
  
 ### 5 - Bring Netmaker back up : 
  ```
  docker compose up -d --force-recreate
  ```
 ### 6 - Log In UI
  
  Connect to : dashboard.netmaker.domain.com
  
  Create first user, it will be "Super Admin"
  
 ### 6.1 - Create a/some Network to join "self node" to it : 
  
  Create Network > Check IPV4 + IPV6 > Click "Autofill" > Name the network
  
  Default access control : ALLOW
  
 ### 6.2 - Create an Enrollment Key 
  
  Enrollment Keys > Create Key :
  
  Name : Enroll Key

  Type : <your-choice>
  
  Network : Select network to join with this key
  
  Click on Create Key.
  
 ### 7 - Connect Self-Host to Networks : 
  
  SSH on host, and run :
  ```
  netclient join -t <Join-Key>"
  ```
 ### 8 - Edit Host : 
  
  Hosts > Clic on "..." > Edit Host >
  
  - Check "Static Port"

  - Modify port, and open it on router to be routed to <Virtual Host IP>
  
  - Verbosity : 4 (keep track of connexions, huge logging)
  
  - Check "Default Host"

  - Check "Auto Update"
  
###  9 - Add another network to Netmaker (cf 6.1)
  
###  10 - Add an host to the network, and give access to it's connected networks : 
  
###  10.1 - Networks > Net-Just-Added > Hosts 
  
  --> Add Hosts 

  --> Add Existing Hosts 
  
  --> Select host to add
  
###  10.2 - Networks > Net-Just-Added > Remote Access
  
  --> Create Gateway
  
  --> "Host" : select host
  
  --> Create Gateway
  
 ### 10.3 - Networks > > Net-Just-Added > Egress
  
  --> Create egress
  
  --> Select an host
  
  --> Enable NAT for Egress Traffic : YES
  
  --> Create Egress
  
 ### 10.4 - Networks > > Net-Just-Added > Egress
  
  --> Add external route
  
  --> Enable NAT for egress traffic : YES
  
  --> +Add Range
  
  --> Enter range to be accessed through this host (eg : 172.22.10.0/24)
  
  
  -------------------------------------------
  
 ## MESH VPN : 
  
###  11 - Connect Another-Host to Networks : 
  
  SSH on host, and run : 
  ```
  netclient join -t <Join-Key>
  ```
###  12 - Edit Host : 
  
  Hosts > Clic on "..." > Edit Host >
  
  - Check "Static Port"

  - Modify port, and open it on router to be routed to <Another-Host-IP>
  
  - Verbosity : 4 (keep track of connexions, huge logging)
  
  - Check "Auto Update"
  
###    13 - Add another distant host to the network, and give access to it's connected networks : 
  
###  13.1 - Networks > Net-Just-Added > Hosts 
  
  --> Add Hosts 
  
  --> Add Existing Hosts 
  
  --> Select Another-Host to add
  
 ### 13.2 - Networks > Net-Just-Added > Remote Access
  
  --> Create Gateway
  
  --> "Host" : select Another-Host
  
  --> Create Gateway
  
 ### 13.3 - Networks > Net-Just-Added > Egress
  
  --> Create egress
  
  --> Select an host
  
  --> Enable NAT for Egress Traffic : YES
  
  --> Create Egress
  
 ### 13.4 - Networks > > Net-Just-Added > Egress
  
  --> Add external route
  
  --> Enable NAT for egress traffic : YES
  
  --> +Add Range
  
  --> Enter range to be accessed through this host (eg : 172.30.10.0/24)
  
  
