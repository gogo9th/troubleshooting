
# MySQL Error Troubleshooting

- **명령어:** `sudo mysql`
- **에러메시지:** `ERROR 1045 (28000): Access denied for user 'root'@'localhost' (using password: NO)`

#### Step 1: MySQL 업그레이드 도중 에러가 발생하여서 corrupt된 것으로 의심되어서 지우고 재설치
```
$ sudo apt purge mysql-server mysql-common
$ sudo apt install mysql-server mysql-client
```

#### Step 2: MySQL 서버가 정상적인 active 상태인지 확인
```
$ sudo service mysql status
 # 
```

#### Step 3: MySQL 클라이언트를 MySQL 서버에 접속 시도
```
$ sudo mysql
# 에러 발생: ERROR 1045 (28000): Access denied for user 'root'@'localhost' (using password: NO)
```

#### Step 4: MySQL의 에러 로그 파일을 열어서 에러의 원인 분석
```
$ sudo vim /var/log/mysql/error.log
# 파일 안에 에러 메시지가 기록되지 않음
```

#### Step 5: 디버깅을 위하여 MySQL 서비스를 종료하고 단독 mysql 데몬을 실행
```
$ sudo service mysql stop
$ sudo mysqld --verbose --console
# 하지만 실행 후 1초 후 자동 종료됨
```

#### Step 6: mysql 데몬의 에러 로그 파일을 강제로 직접 설정
```
$ sudo mysqld --log-error=/tmp/mysql_error.log
```

#### Step 7: MySQL 클라이언트 재접속 시도 및 실패 후 로그 파일을 열어서 실패 원인 분석
```
$ sudo mysql
# mysql_error.log의 에러 확인: [Server] Could not create unix socket lock file /var/run/mysqld/mysqld.sock.lock
```

#### Step 8: 존재하지 않는 `/ar/run/mysql` 폴더 직접 만들어주기
```
$ sudo mkdir -p /var/run/mysqld
$ sudo chown mysql:mysql /var/run/mysqld
```

#### Step 9: mysql 데몬 재실행
```
$ sudo mysqld --log-error=/tmp/mysql_error.log
```

#### Step 10: MySQL 클라이언트 재접속 시도 및 재실패 후 실패 원인 분석
```
$ sudo mysql
# mysql_error.log의 에러 확인: 
[Server] Plugin mysql_native_password reported: ''mysql_native_password' is deprecated and will be removed in a future release. Please use caching_sha2_password instead'
```

#### Step 11: MySQL 로그인 계정/비밀번호 입력을 우회하기 위해 mysql 안전모드 데몬을 실행
```
$ sudo mysqld_safe --skip-grant-tables --skip-networking &
# 정상적인 실행 상태를 유지하지 못하고 1초 후 자동 종료됨
```

#### Step 12: mysql 안전모드 데몬이 현재 실행중인지 확인
```
$ ps -aux | grep mysqld_safe
# 실행중이 아님
```

#### Step 13: mysql 에러 로그 파일을 열어서 mysql 데몬이 자동 종료된 원인 분석
```
$ vim /var/log/mysql/error.log
# 에러 확인: [Server] unknown variable 'log-syslog=1'.
```

#### Step 14: mysql 안전모드 데몬 용도의 MySQL 설정파일에서 log-syslog 관련부분 제거
```
$ vim /etc/mysql/mysql.conf.d/mysqld_safe_syslog.cnf
#  > syslog (이 부분 comment out하기)
```

#### Step 15: MySQL 설정이 바뀌었으므로 mysql 안전모드 데몬을 재실행
```
$ sudo mysqld_safe --skip-grant-tables --skip-networking
# syslog 없이 다시 실행
```

#### Step 16: MySQL 클라이언트 재접속 시도
```
$ sudo mysql
# MySQL 접속 성공!
```

#### Step 17: MySQL의 패스워드를 `caching_sha2_password` 방식으로 재설정
```
mysql> UPDATE user SET authentication_string=PASSWORD('your_password') WHERE User='root' AND Host='localhost';
# syntax 에러: 상위 MySQL 버전에서는 PASSWORD() 함수가 더 이상 지원되지 않음

# ALTER 명령어를 사용하여서 패스워드 재설정
mysql> USE mysql;
mysql> ALTER USER 'root'@'localhost' IDENTIFIED WITH 'caching_sha2_password' BY 'your_password';
mysql> FLUSH PRIVILEGES;
```

#### Step 18: MySQL 설정을 성공적으로 바꾸었으므로 mysql 안전모드 데몬을 종료
```
$ sudo killall mysqld_safe
# 다시 spawn되어서 종료할 수 없음 
```

#### Step 19: 강력한 `kill -9` 명령어로 mysql 안전모드 데몬을 강제 종료
```
$ sudo kill -9 $(pgrep mysqld_safe)
# 완전 종료 성공
```

#### Step 20: mysql 일반 데몬 실행 후 MySQL 클라이언트 접속 시도
```
$ sudo mysqld --log-error=/tmp/mysql_error.log
$ sudo mysql
# 접속 성공
```

#### Step 21: 편의를 위해서 MySQL의 root 패스워드를 삭제 (공백으로 수정)
```
mysql> ALTER USER 'root'@'localhost' IDENTIFIED WITH 'mysql_native_password' BY '';
mysql> FLUSH PRIVILEGES;
```

#### Step 22: 공백으로 설정한 root 패스워드의 MySQL 접속 테스트
```
$ sudo mysql
# 접속 성공
```

#### Step 23: 회사의 기존 MySQL 계정과 패스워드의 접속 테스트
```
$ sudo mysql -u <회사 계정의 기존 아이디 입력> -p
> <회사 계정의 기존 패스워드 입력>
# 접속 성공
```

#### Step 24: mysql 단독 데몬을 종료 후 MySQL 서비스 시작
```
$ sudo killall mysqld
$ sudo service mysql start
$ sudo service mysql status
# MySQL 서비스 구동 성공
```

#### Step 25: `https://isensortech.co.kr` 웹서버에 `wget`으로 접속 테스트
```
$ rm -f index.html
$ wget https://isensortech.co.kr
$ vim index.html
# 에러 메시지: Connect Error: Server sent charset unknown to the client. Please, report to the developers
```

#### Step 26: 최신 MySQL이 구식 php 버전과 호환되도록 MySQL의 기본 문자열을 구식으로 다운그레이드
```
$ sudo vim /etc/mysql/mysql.conf.d/mysqld.cnf
  +++ character-set-server=utf8
  +++ collation-server=utf8_general_ci 
# 구형버전 php v5.6.40과 호환되도록 MySQL v8.0의 character set을 강제로 downgrade하기
$ sudo service mysql restart
```

#### Step 27: `https://isensortech.co.kr` 웹서버에 `wget`으로 접속 테스트
```
$ rm -f index.html
$ wget https://isensortech.co.kr
$ vim index.html
# 에러 메시지: Connect Error: The server requested authentication method unknown to the client
```

#### Step 28: 최신 MySQL이 구식 php 버전과 호환되도록 MySQL의 패스워드 인증방식을 다운그레이드
```
$ sudo vim /etc/mysql/mysql.conf.d/mysqld.cnf
  +++ default_authentication_plugin=mysql_native_password
# 구형버전 php v5.6.40과 호환되도록 MySQL v8.0의 character set을 강제로 downgrade하기
$ sudo service mysql restart
```

#### Step 29: `https://isensortech.co.kr` 웹서버에 `wget`으로 접속 테스트
```
$ rm -f index.html
$ wget https://isensortech.co.kr
$ vim index.html
# 정상 웹페이지 내용 확인
```

#### Step 30: `https://isensortech.co.kr` 웹서버에 브라우저로 접속 테스트
- 브라우저로 https://isensortech.co.kr 방문하기
- 정상 웹페이지 확인

## 현재 시스템 상태
- Ubuntu 버전 18.04에서 20.04로 업그레이드
- MySQL 버전 5.7에서 8.0으로 업그레이드
- Apache2 버전 v2.4.29에서 v2.4.57로 업그레이드
- php 버전은 v5.6.40 그대로
- 
