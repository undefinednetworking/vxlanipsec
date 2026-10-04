# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|
  config.vm.box = "bento/ubuntu-26.04"

  # Disable default Vagrant NAT (eth0) to enforce a single network interface
  #config.vm.network "forwarded_port", id: "ssh", host: 2222, guest: 22, disabled: true

  # VirtualBox Provider Settings (2 CPUs, 2048MB RAM, Promiscuous Mode on NIC 1)
  config.vm.provider "virtualbox" do |v|
    v.cpus = 2
    v.memory = 2048
    v.customize ["modifyvm", :id, "--nicpromisc1", "allow-all"]
  end

  # -------------------------------------------------------------------
  # Common Provisioning: Tools, Firewall, & Persistence
  # -------------------------------------------------------------------
  config.vm.provision "shell", inline: <<-SHELL
    set -e
    export DEBIAN_FRONTEND=noninteractive

    echo "=== 1. Installing Prerequisites & Network Tools ==="
    apt-get update
    #apt-get install -y openvswitch-switch openvswitch-ipsec strongswan strongswan-pqc strongswan-pki openvpn jq ufw iptables-persistent
    apt-get install -y \
    openvswitch-switch \
    openvswitch-ipsec \
    strongswan \
    strongswan-swanctl \
    libcharon-extra-plugins \
    libcharon-extauth-plugins \
    libstrongswan-standard-plugins \
    libstrongswan-extra-plugins \
    strongswan-starter \
    strongswan-pki \
    iproute2 \
    tcpdump \
    ufw

    echo "=== 2. Enabling StrongSwan PQC ML-KEM Plugin ==="
    mkdir -p /etc/strongswan.d/charon/
    echo 'mlkem { load = yes }' > /etc/strongswan.d/charon/mlkem.conf

    echo "=== 3. Setting Up UFW Firewall Rules ==="
    ufw default allow outgoing
    ufw default deny incoming
    ufw allow 22/tcp                          # SSH
    ufw allow 500/udp                         # IPsec IKE
    ufw allow 4500/udp                        # IPsec NAT-Traversal
    ufw allow proto esp from any to any       # IPsec ESP
    ufw allow 4789/udp                        # VXLAN
    ufw allow 6081/udp                        # GENEVE
    ufw allow in on br0                       # Overlay traffic
    ufw allow in on int-clear
    ufw allow in on int-ipsec
    ufw --force enable

    echo "=== 4. Preparing /etc/ipsec.d Directories ==="
    mkdir -p /etc/ipsec.d/certs /etc/ipsec.d/private /etc/ipsec.d/cacerts
    chmod 755 /etc/ipsec.d /etc/ipsec.d/certs /etc/ipsec.d/cacerts
    chmod 700 /etc/ipsec.d/private

echo "=== Persisting OpenFlow Restoration Service ==="
cat << 'EOF' > /etc/systemd/system/ovs-restore-flows.service
[Unit]
Description=Restore Open vSwitch Flows on Boot
After=openvswitch-switch.service network-online.target
Requires=openvswitch-switch.service
PartOf=openvswitch-switch.service

[Service]
Type=oneshot
ExecStart=/usr/bin/ovs-ofctl add-flows br0 /etc/openvswitch/br0-flows.txt
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
EOF

    systemctl daemon-reload
    systemctl enable ovs-restore-flows.service

    echo "=== 5. Persistence Script ==="
    cat <<'EOF' > /etc/network/if-up.d/ovs-ipsec-persist
#!/bin/bash
systemctl restart openvswitch-ipsec
ipsec rereadsecrets || true
ipsec rereadcacerts || true
EOF
    chmod +x /etc/network/if-up.d/ovs-ipsec-persist
  SHELL

  # -------------------------------------------------------------------
  # Node 1 Setup (Single Public Interface)
  # -------------------------------------------------------------------
  config.vm.define "node1" do |node1|
    node1.vm.hostname = "node1"
    
    # Public Bridged Interface with Static IP (Replace bridge name if needed)
    node1.vm.network "public_network",
      ip: "192.168.0.201",
      netmask: "255.255.255.0"

    node1.vm.provision "shell", inline: <<-SHELL
      set -e

      echo "=== [Node1] Patching ovs-monitor-ipsec for ML-KEM ==="
      if [ -f /usr/share/openvswitch/scripts/ovs-monitor-ipsec ]; then
        sed -i 's/ike=.*/ike=aes256gcm16-sha256-mlkem1024/g' /usr/share/openvswitch/scripts/ovs-monitor-ipsec
        sed -i 's/esp=.*/esp=aes256gcm16-mlkem1024/g' /usr/share/openvswitch/scripts/ovs-monitor-ipsec
      fi

      echo "=== [Node1] Generating PKI via openssl ==="
      #Create and copy keys if they don't exist

      if [ ! -f /etc/ipsec.d/private/node1-privkey.pem ]; then
        mkdir -p /vagrant/pki && cd /vagrant/pki

        #Create keys if they don't exist
        if [ ! -f /vagrant/pki/node1-privkey.pem ]; then
          # 1. Generate CA Private Key
          openssl genrsa -out ca-key.pem 2048

          # 2. Generate Self-Signed CA Certificate (valid for 10 years)
          openssl req -x509 -new -nodes -key ca-key.pem -sha256 -days 3650 \
            -subj "/C=US/ST=State/L=City/O=OVS-CA/CN=OVS-Root-CA" \
            -out cacert.pem
          # 1. Generate node1 Private Key
          openssl genrsa -out node1-privkey.pem 2048

          # 2. Create Certificate Signing Request (CSR) with CN=node1
          openssl req -new -key node1-privkey.pem \
            -subj "/C=US/ST=State/L=City/O=OVS/CN=node1" \
            -out node1.csr

          # 3. Sign the Certificate with your CA
          openssl x509 -req -in node1.csr -CA cacert.pem -CAkey ca-key.pem \
            -CAcreateserial -out node1-cert.pem -days 365 -sha256

          # 1. Generate node2 Private Key
          openssl genrsa -out node2-privkey.pem 2048

          # 2. Create CSR with CN=node2
          openssl req -new -key node2-privkey.pem \
            -subj "/C=US/ST=State/L=City/O=OVS/CN=node2" \
            -out node2.csr

          # 3. Sign the Certificate with your CA
          openssl x509 -req -in node2.csr -CA cacert.pem -CAkey ca-key.pem \
            -CAcreateserial -out node2-cert.pem -days 365 -sha256

        fi

        # Copy CA cert
        sudo cp cacert.pem /etc/ipsec.d/cacerts/

        # Copy Node Certs & Keys
        sudo cp node1-cert.pem /etc/ipsec.d/certs/
        sudo cp node2-cert.pem /etc/ipsec.d/certs/
        sudo cp node1-privkey.pem /etc/ipsec.d/private/
        sudo chmod 600 /etc/ipsec.d/private/node1-privkey.pem

        # Restart OVS IPsec service
        #sudo systemctl restart ovs-monitor-ipsec
        sudo systemctl restart openvswitch-ipsec

      fi

      chmod 644 /etc/ipsec.d/certs/*.pem /etc/ipsec.d/cacerts/*.pem
      chmod 600 /etc/ipsec.d/private/*.pem

      echo "=== [Node1] Configuring OVS SSL & Bridges ==="
      ovs-vsctl set-ssl \
        /etc/ipsec.d/private/node1-privkey.pem \
        /etc/ipsec.d/certs/node1-cert.pem \
        /etc/ipsec.d/cacerts/cacert.pem

      ovs-vsctl set Open_vSwitch . \
        other_config:certificate="/etc/ipsec.d/certs/node1-cert.pem" \
        other_config:private_key="/etc/ipsec.d/private/node1-privkey.pem" \
        other_config:ca_cert="/etc/ipsec.d/cacerts/cacert.pem"

      ovs-vsctl --if-exists del-br br0
      ovs-vsctl add-br br0

      ovs-vsctl add-port br0 int-clear -- set interface int-clear type=internal
      ovs-vsctl add-port br0 int-ipsec -- set interface int-ipsec type=internal

      ovs-vsctl add-port br0 vxlan-clear -- set interface vxlan-clear type=vxlan \
        options:remote_ip=192.168.0.202 options:key=100 options:dst_port=4789

      ovs-vsctl add-port br0 geneve-ipsec -- set interface geneve-ipsec type=geneve \
        options:remote_ip=192.168.0.202 \
        options:key=200 \
        options:ikeversion=v2 \
        options:certificate="/etc/ipsec.d/certs/node1-cert.pem" \
        options:private_key="/etc/ipsec.d/private/node1-privkey.pem" \
        options:remote_cert="/etc/ipsec.d/certs/node2-cert.pem"

      echo "=== Configuring & Persisting Flow Rules (node2) ==="
      ovs-ofctl del-flows br0
      ovs-ofctl add-flow br0 "priority=100,in_port=int-clear actions=output:vxlan-clear"
      ovs-ofctl add-flow br0 "priority=100,in_port=vxlan-clear actions=output:int-clear"
      ovs-ofctl add-flow br0 "priority=100,in_port=int-ipsec actions=output:geneve-ipsec"
      ovs-ofctl add-flow br0 "priority=100,in_port=geneve-ipsec actions=output:int-ipsec"
      ovs-ofctl add-flow br0 "priority=0 actions=NORMAL"

      # Save flows to disk for systemd auto-restore on reboot
      ovs-ofctl dump-flows br0 --no-names  --no-stats > /etc/openvswitch/br0-flows.txt


      echo "=== [Node1] Netplan Overlay Configuration ==="
      cat <<'EOF' > /etc/netplan/60-overlay.yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    int-clear:
      addresses:
        - 10.100.0.1/24
    int-ipsec:
      addresses:
        - 10.200.0.1/24
EOF
      netplan apply

      systemctl restart openvswitch-ipsec
      ipsec rereadsecrets || true
      ipsec rereadcacerts || true
    SHELL
  end

  # -------------------------------------------------------------------
  # Node 2 Setup (Single Public Interface)
  # -------------------------------------------------------------------
  config.vm.define "node2" do |node2|
    node2.vm.hostname = "node2"

    # Public Bridged Interface with Static IP
    node2.vm.network "public_network",
      ip: "192.168.0.202",
      netmask: "255.255.255.0"

    node2.vm.provision "shell", inline: <<-SHELL
      set -e

      echo "=== [Node2] Patching ovs-monitor-ipsec for ML-KEM ==="
      if [ -f /usr/share/openvswitch/scripts/ovs-monitor-ipsec ]; then
        sed -i 's/ike=.*/ike=aes256gcm16-sha256-mlkem1024/g' /usr/share/openvswitch/scripts/ovs-monitor-ipsec
        sed -i 's/esp=.*/esp=aes256gcm16-mlkem1024/g' /usr/share/openvswitch/scripts/ovs-monitor-ipsec
      fi

      echo "=== [Node2] Fetching PKI Assets ==="
      while [ ! -f /vagrant/pki/node2-privkey.pem ]; do
        echo "Waiting for node1 certificate generation..."
        sleep 3
      done

      if [ ! -f /etc/ipsec.d/certs/node2-cert.pem ]; then
        cp /vagrant/pki/cacert.pem /etc/ipsec.d/cacerts/
        cp /vagrant/pki/node2-cert.pem /etc/ipsec.d/certs/
        cp /vagrant/pki/node1-cert.pem /etc/ipsec.d/certs/
        cp /vagrant/pki/node2-privkey.pem /etc/ipsec.d/private/
      fi

      chmod 644 /etc/ipsec.d/certs/*.pem /etc/ipsec.d/cacerts/*.pem
      chmod 600 /etc/ipsec.d/private/*.pem

      echo "=== [Node2] Configuring OVS SSL & Bridges ==="
      ovs-vsctl set-ssl \
        /etc/ipsec.d/private/node2-privkey.pem \
        /etc/ipsec.d/certs/node2-cert.pem \
        /etc/ipsec.d/cacerts/cacert.pem

      ovs-vsctl set Open_vSwitch . \
        other_config:certificate="/etc/ipsec.d/certs/node2-cert.pem" \
        other_config:private_key="/etc/ipsec.d/private/node2-privkey.pem" \
        other_config:ca_cert="/etc/ipsec.d/cacerts/cacert.pem"

      ovs-vsctl --if-exists del-br br0
      ovs-vsctl add-br br0

      ovs-vsctl add-port br0 int-clear -- set interface int-clear type=internal
      ovs-vsctl add-port br0 int-ipsec -- set interface int-ipsec type=internal

      ovs-vsctl add-port br0 vxlan-clear -- set interface vxlan-clear type=vxlan \
        options:remote_ip=192.168.0.201 options:key=100 options:dst_port=4789

      ovs-vsctl add-port br0 geneve-ipsec -- set interface geneve-ipsec type=geneve \
        options:remote_ip=192.168.0.201 \
        options:key=200 \
        options:ikeversion=v2 \
        options:certificate="/etc/ipsec.d/certs/node2-cert.pem" \
        options:private_key="/etc/ipsec.d/private/node2-privkey.pem" \
        options:remote_cert="/etc/ipsec.d/certs/node1-cert.pem"

      echo "=== Configuring & Persisting Flow Rules (node2) ==="
      ovs-ofctl del-flows br0
      ovs-ofctl add-flow br0 "priority=100,in_port=int-clear actions=output:vxlan-clear"
      ovs-ofctl add-flow br0 "priority=100,in_port=vxlan-clear actions=output:int-clear"
      ovs-ofctl add-flow br0 "priority=100,in_port=int-ipsec actions=output:geneve-ipsec"
      ovs-ofctl add-flow br0 "priority=100,in_port=geneve-ipsec actions=output:int-ipsec"
      ovs-ofctl add-flow br0 "priority=0 actions=NORMAL"

      # Save flows to disk for systemd auto-restore on reboot
      ovs-ofctl dump-flows br0 --no-names  --no-stats > /etc/openvswitch/br0-flows.txt

      echo "=== [Node2] Netplan Overlay Configuration ==="
      cat <<'EOF' > /etc/netplan/60-overlay.yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    int-clear:
      addresses:
        - 10.100.0.2/24
    int-ipsec:
      addresses:
        - 10.200.0.2/24
EOF
      netplan apply

      systemctl restart openvswitch-ipsec
      ipsec rereadsecrets || true
      ipsec rereadcacerts || true
    SHELL
  end
end

