구축은 순서대로 하시면 됩니다.

1. Docker 설치
2. Docker Compose 플러그인 설치
    1. (선택사항) docker-compose 명령어를 사용하기 위한 alias 설정
3. docker-compose 파일 작성
4. promtail 설정 파일 작성
5. 컨테이너 실행

## Docker 설치

1. Amazon Linux 2023

```bash
sudo yum update -y
sudo yum install -y docker
sudo service docker start
sudo usermod -a -G docker ec2-user

# logout and login
docker ps
```

1. Amazon Linux 2

```bash
sudo yum update -y
sudo amazon-linux-extras install docker
sudo service docker start
sudo usermod -a -G docker ec2-user

# logout and login
docker ps
```

## Docker Compose Plugin 설치

```bash
sudo mkdir -p /usr/local/lib/docker/cli-plugins/
sudo curl -SL "https://github.com/docker/compose/releases/latest/download/docker-compose-linux-$(uname -m)" -o /usr/local/lib/docker/cli-plugins/docker-compose
sudo chmod +x /usr/local/lib/docker/cli-plugins/docker-compose

docker compose version
```

- **(선택사항) docker-compose 명령어를 사용하기 위한 alias 설정**
    - 사용 중인 쉘 확인하기
    
    ```bash
    echo $SHELL
    # /bin/bash or /bin/zsh
    ```
    
    - config 파일 수정하기
        - /bin/bash면 `~/.bashrc` 를 수정하고, /bin/zsh면 `~/.zshrc` 를 수정해요.
    - 아래 코드를 맨 아랫줄에 추가합니다.
    
    ```bash
    alias docker-compose='docker compose --compatibility "$@"'
    ```
    
    - 프로파일 적용하기
    
    ```bash
    source ~/.bashrc
    # or
    source ~/.zshrc
    ```
    
    - 확인
    
    ```bash
    docker-compose version
    ```
    

## docker-compose 파일 작성

- docker-compose-grafana.yml

```yaml
version: '3'

services:
  loki:
    image: grafana/loki:main-df7a8e4
    container_name: loki
    ports:
      - "3100:3100"
    restart: always
    
  grafana:
    image: grafana/grafana:main-ubuntu
    container_name: grafana
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
    ports:
      - "3000:3000"
    restart: always
```

- docker-compose-promtail.yml

```yaml
version: '3'

services:
  promtail:
    image: grafana/promtail:main-df7a8e4
    container_name: promtail
    volumes:
      - {promtail 설정 파일 로컬경로}/promtail.yml:/etc/promtail/promtail.yml
      - {로그 로컬경로}:/mnt/logs
    command:
      - "-config.file=/etc/promtail/promtail.yml"
    restart: always
```

## promtail 설정 파일 작성

- promtail.yml

```yaml
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /tmp/positions.yaml

clients:
  - url: "http://{Loki 서버 IP}:3100/loki/api/v1/push"
    tenant_id: "default"
    batchwait: 1s
    batchsize: 102400
    timeout: 10s

scrape_configs:
  - job_name: 'gateway-log'
    static_configs:
      - targets:
          - localhost
        labels:
          job: "gateway"
          __path__: /mnt/logs/gateway.*.log
```

## 컨테이너 실행

- 로그 서버

```bash
docker-compose -f docker-compose-grafana.yml up -d
```

- API 서버

```bash
docker-compose -f docker-compose-promtail.yml up -d
```

## Grafana에서 Loki 연동

> grafana 서버와 loki 서버가 같은 Local이기 때문에 아래와 같이 컨테이너 명으로 접근 가능합니다.
> 
> - http://loki:3100


연동이 정상적으로 완료되었다면, [Explore] >> [Logs] 에서 로그 확인이 가능합니다.

![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/ae115e03-70a8-424f-a8e1-4ecf247ca623/7daab381-21de-49c0-ad92-59d71a2cbbad/image.png)
