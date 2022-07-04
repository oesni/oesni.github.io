---
title:  "Postgresql 도커화하기"
excerpt: "Postgresql 도커화하기"

categories:
  - TIL
tags:
  - Docker
  - Postgresql
  - DB
date: 2022-07-04 20:52 +0900
last_modified_at: 2022-07-04 20:52 +0900
published: true
---

## TODO: 기존 사용중인 postgresql DB를 도커 이미지로 빌드하기

회사에서 [Postgresql](https://www.postgresql.org) DB를 사용하고 있습니다. 테스트용 서버(dev, stage, test 등등 ...)를 구축하거나 로컬 개발 환경에 테스트용 DB를 셋업하는 경우가 잦아져서, 개발 서버에서 운영중인 postgres db를 도커를 통해 쉽게 백업 & 배포할 수 있도록 해봅니다.

요구사항은 다음과 같습니다.
>1. 이미지(Docker image)에 데이터가 모두 포함되어야 함 (모든 DB 포함)
>2. user/password/port 등 설정이 자유로워야 함
>3. 이미지 빌드(백업) 및 배포를 자동화 할 수 있어야 함

다음과 같은 방식으로 구성할 계획입니다.
>1. postgres에서 제공하는 *tool*(*pg_dump or pg_dumpall or psql ...?*)을 이용해 .sql 형식으로 데이터 덤프
>2. postgres에서 제공하는 [도커 이미지](https://hub.docker.com/_/postgres?tab=description)를 이용하여 이미지 빌드
(해당 이미지는 초기화 시점에```/docker-entrypoint-initdb.d``` 디렉토리에 존재하는 ```.sql```, ```.sql.gz```, ```*.sh``` 스크립트를 실행하도록 구성되어 있습니다. - 링크의 **Initialization scripts** 항목 참고)
>3. 회사 registry 서버에 이미지 배포, git에 스크립트, Dockerfile, docker-compose.yml 등 공유
>4. 필요한 경우 Jenkins를 통해 cronjob 형태로 빌드(=백업) & 배포 자동화

## 1. DB dump하기

우선 ```pg_dumpall```을 이용하여 데이터베이스를 dump 해보았습니다.

```sh
echo "$SERVER_URL:$SERVER_PORT:*:$SERVER_USER:$SERVER_PSWD" >> ~/.pgadmin
chmod 600 ~/.pgadmin
export BACKUP_SQL="pgdump-$(date +%Y-%m-%d).sql"
pg_dumpall -h $SERVER_URL -U $SERVER_USER ./backups/$BACKUP_SQL
```

## 2. Docker 이미지 빌드하기
이미지를 빌드하기 위한 도커파일을 작성합니다.
현재 postgres 14.2 버전을 사용중이기 떄문에 해당 버전 이미지를 사용합니다. ```/docker-entrypoint-initdb.d/``` 아래에 있는 스크립트는 이미지(이미지의 entrypoint, 더 정확하게는 docker-entrypoint.sh)가 알아서 실행하기 때문에 아주 간단하게 Dockerfile 작성이 끝났습니다.
```Dockerfile
FROM postgres:14.2

COPY ./backups/backup.sql /docker-entrypoint-initdb.d/backup.sql
```

<br>

![IE002404461_STD](https://user-images.githubusercontent.com/19154301/177169294-67acdf90-eecb-4eb8-ab71-27c3ca4ca5b1.jpeg){:width="75%" height="75%"}{: .center}


이제 이미지를 빌드하고 실행 해 보겠습니다. M1 맥북을 사용하고 있기 떄문에 플랫폼을 명시 해 주었습니다.


```sh
docker build --platform=linux/amd64 -t "mypostgres" . < Dockerfile
```

참고: **POSTGRES_PASSWORD** 환경변수가 설정되어있지 않으면 설정하라고 에러(와 경고)가 나옵니다.

```sh
docker run --rm -e POSTGRES_PASSWORD=password mypostgres
```

![스크린샷 2022-07-04 오후 9 58 25](https://user-images.githubusercontent.com/19154301/177168703-4c7a02ea-0f43-4d83-a477-b87648e6358d.png)

server started 로그 후 backup.sql 파일을 실행하는데 Error와 함께 초기화에 실패합니다.


postgres docker 페이지를 확인해보니 스크립트가 실패하면 초기화가 중단됩니다.
![스크린샷 2022-07-04 오후 10 21 15](https://user-images.githubusercontent.com/19154301/177170431-65276b38-9b15-4131-9718-ab157f9de1e8.png)
>One common problem is that if one of your /docker-entrypoint-initdb.d scripts fails (which will cause the entrypoint script to exit) and your orchestrator restarts the container with the already initialized data directory, it will not continue on with your scripts.

출처: [docker hub](https://hub.docker.com/_/postgres)


sql 파일을 열어보니
다음과 같은 스크립트가 보입니다.

``` sql
--
-- Roles
--

CREATE ROLE postgres;
ALTER ROLE postgres WITH SUPERUSER INHERIT CREATEROLE CREATEDB LOGIN REPLICATION BYPASSRLS PASSWORD 'SCRAM-SHA-256$4096:KecB90rwW5X3+E/iiN3q4g==$buSYhXDusoSmrn2eqA16pZ0ImgmwwhPTk/dSjARMKmo=:7naOETynJNG17DnvtBvYeEOL/KvZFzA2NMw/eExHZhg=';

-- ...

CREATE DATABASE "mydb" WITH TEMPLATE = template0 ENCODING = 'UTF8' LOCALE = 'en_US.utf8';
-- ...
ALTER DATABASE "mydb" OWNER TO postgres;

-- ...

```

pg_dumpall로 dump 했기 때문에 global objects들도 같이 dump 되었습니다.
>pg_dumpall is a utility for writing out (“dumping”) all PostgreSQL databases of a cluster into one script file. The script file contains SQL commands that can be used as input to psql to restore the databases. It does this by calling pg_dump for each database in the cluster. pg_dumpall also dumps global objects that are common to all databases, that is, database roles and tablespaces. (pg_dump does not save these objects.)

출처: [pg_dumpall](https://www.postgresql.org/docs/current/app-pg-dumpall.html) - https://www.postgresql.org/docs/current/app-pg-dumpall.html

pg_dumpall은 기본적으로 db의 ownership도 백업하기 떄문에 요구사항 2번에 문제가 생길 수 있습니다. 따라서 dump 명령어에 --no-owner옵션을 추가합니다.
```sh
pg_dumpall --no-owner -h $SERVER_URL -U $SERVER_USER ./backups/$BACKUP_SQL
```


그러나 아무리 메뉴얼을 찾아봐도 roles를 백업하지 않는 옵션이 없고, 옵션을 아무리 이리저리 추가 해봐도 이미지 intialize단계에서 fail을 피할 수가 없습니다. 덤프된 sql을 직접 수정할 경우 요구사항 3번을 만족하지 못합니다.

![KakaoTalk_Photo_2022-07-04-22-53-22](https://user-images.githubusercontent.com/19154301/177169008-2e117bcd-da48-4e58-baa3-04d944dd1886.png)

그러던 중 pg_dumpall 문서에서 관련 항목을 발견합니다.

![스크린샷 2022-07-04 오후 10 19 48](https://user-images.githubusercontent.com/19154301/177168063-fabd5b81-d866-4d09-bccb-c9e6cc2c465f.png)

>The dump script should not be expected to run completely without errors. In particular, because the script will issue CREATE ROLE for every role existing in the source cluster, it is certain to get a “role already exists” error for the bootstrap superuser, unless the destination cluster was initialized with a different bootstrap superuser name. This error is harmless and should be ignored.

출처: [pg_dumpall](https://www.postgresql.org/docs/current/app-pg-dumpall.html) - https://www.postgresql.org/docs/current/app-pg-dumpall.html

> ... This error is **harmless** and **should be ignored**. ...

...🤨

결국 이미지의 docker-entrypoint.sh가 dump뜬 sql 스크립트를 직접 실행하지 않도록 복구 스크립트를 작성하고, Dockerfile을 수정합니다.

```sh
# restore.sh
echo "127.0.0.1:5432:*:${POSTGRES_USER}:${POSTGRES_PASSWORD}" > ~/.pgpass
chmod 600 ~/.pgpass
psql -f /sql/backup.sql
```

```Dockerfile
FROM postgres:14.2

COPY ./backups/backup.sql /sql/backup.sql
COPY restore.sh /docker-entrypoint-initdb.d/
```

다시 컨테이너를 실행시켜 로그를 확인합니다.
```
...

server started

/usr/local/bin/docker-entrypoint.sh: sourcing /docker-entrypoint-initdb.d/restore.sh
SET
SET
SET
2022-07-04 13:35:35.071 UTC [156] ERROR:  role "postgres" already exists
2022-07-04 13:35:35.071 UTC [156] STATEMENT:  CREATE ROLE postgres;
psql:/sql/backup.sql:14: ERROR:  role "postgres" already exists
ALTER ROLE

...

PostgreSQL init process complete; ready for start up.

...

2022-07-04 13:35:35.970 UTC [1] LOG:  database system is ready to accept connections
```

bakcup.sql에서 error가 발생하지만 문제 없이 해당 라인 이후 sql 스크립트가 실행되고, DB도 정상적으로 실행됩니다. DB도 제대로 복구된 것을 확인할 수 있습니다.

### 3. 빠른 배포 및 테스트를 위한 docker-compose.yml 작성하기
docker-compose 파일을 작성 해두면 로컬 및 서버에서 동일한 설정으로 빠른 테스트가 가능합니다.

*docker-compose.yml 예시*
```yaml
services:
  db:
    image: registry_url/mypostgres
    restart: always
    ports:
      - "5432:5432"
    volumes:
      - ./data:/var/lib/postgres/data
    environment:
      - POSTGRES_PASSWORD=postgres

```

정리된 내용은 아래 repository에서 확인 가능합니다.