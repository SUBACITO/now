# docker-compose.yml thêm vào
services:
  nginx_active:
    image: nginx:alpine
    network_mode: host  # cần host network để Keepalived quản lý VIP
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf

  nginx_standby:
    image: nginx:alpine
    network_mode: host
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf

  keepalived_active:
    image: osixia/keepalived
    network_mode: host
    cap_add: [NET_ADMIN, NET_BROADCAST]
    environment:
      KEEPALIVED_VIRTUAL_IPS: "192.168.1.100"  # Virtual IP
      KEEPALIVED_UNICAST_PEERS: "192.168.1.11"  # IP của standby node
      KEEPALIVED_PRIORITY: "150"  # active có priority cao hơn

  keepalived_standby:
    image: osixia/keepalived
    network_mode: host
    cap_add: [NET_ADMIN, NET_BROADCAST]
    environment:
      KEEPALIVED_VIRTUAL_IPS: "192.168.1.100"
      KEEPALIVED_UNICAST_PEERS: "192.168.1.10"  # IP của active node
      KEEPALIVED_PRIORITY: "100"  # thấp hơn → standby