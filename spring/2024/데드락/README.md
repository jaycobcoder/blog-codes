슬랙에 다음과 같은 알림이 떴다.

[그림]

서버 로그를 확인해보니 다음과 같은 로그가 남아 있었다.
```
o.m.jdbc.message.server.ErrorPacket      : Error: 1213-40001: Deadlock found when trying to get lock; try restarting transaction
o.h.engine.jdbc.spi.SqlExceptionHelper   : SQL Error: 1213, SQLState: 40001
o.h.engine.jdbc.spi.SqlExceptionHelper   : (conn=1366188) Deadlock found when trying to get lock; try restarting transaction
could not execute statement [(conn=1366188) Deadlock found when trying to get lock; try restarting transaction] [update ~ set ~=? where ~=?]; SQL [update ~ set ~=? where ~=?]
```
팀장님이 나에게 문제 해결을 맡기셔서 오늘 해결기를 작성해보고자 한다.

클라이언트가 기록을 하면 포인트 보상을 적립해준다. 예외가 터진 부분은 보상을 적립하는 부분이었다.
로그를 보니,
- 2024-11-10T16:09:14.158
- 2024-11-10T16:09:14.231
연속적인 시간대에 기록을 하고 있었다. 그리고 보상을 받는 부분에서 예외가 터져버렸다.

`SHOW ENGINE innodb STATUS;` 명령어를 통해 최근에 발생한 데드락 상황을 알 수 있다. 확인해보자.

```
SHOW ENGINE innodb STATUS;

------------------------
LATEST DETECTED DEADLOCK
------------------------
2024-11-10 16:09:14 0x7fedb6f91640
*** (1) TRANSACTION:
TRANSACTION 584841584, ACTIVE 0 sec starting index read
mysql tables in use 1, locked 1
LOCK WAIT 16 lock struct(s), heap size 1128, 6 row lock(s), undo log entries 4
MariaDB thread id 1366188, OS thread handle 140658953754176, query id 3671160502 아이피 ... Updating
update point set amount = 2200 where uuid=_binary '...'
*** WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 5719 page no 70 n bits 248 index PRIMARY of table `point` trx id 584841584 lock_mode X locks rec but not gap waiting
Record lock, heap no 212 PHYSICAL RECORD: n_fields 7; compact format; info bits 0
...

*** CONFLICTING WITH:
RECORD LOCKS space id 5719 page no 70 n bits 248 index PRIMARY of table `point` trx id 584841583 lock mode S locks rec but not gap
Record lock, heap no 212 PHYSICAL RECORD: n_fields 7; compact format; info bits 0
....

RECORD LOCKS space id 5719 page no 70 n bits 248 index PRIMARY of table `point` trx id 584841584 lock mode S locks rec but not gap
Record lock, heap no 212 PHYSICAL RECORD: n_fields 7; compact format; info bits 0
...

*** (2) TRANSACTION:
TRANSACTION 584841583, ACTIVE 0 sec starting index read
mysql tables in use 1, locked 1
LOCK WAIT 16 lock struct(s), heap size 1128, 6 row lock(s), undo log entries 4
MariaDB thread id 1366175, OS thread handle 140658935838272, query id 3671160499 아이피  Updating
update point set amount=2200 where uuid=_binary '...'
*** WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 5719 page no 70 n bits 248 index PRIMARY of table `point` trx id 584841583 lock_mode X locks rec but not gap waiting
Record lock, heap no 212 PHYSICAL RECORD: n_fields 7; compact format; info bits 0
...

*** CONFLICTING WITH:
RECORD LOCKS space id 5719 page no 70 n bits 248 index PRIMARY of table `point` trx id 584841583 lock mode S locks rec but not gap
Record lock, heap no 212 PHYSICAL RECORD: n_fields 7; compact format; info bits 0
...

RECORD LOCKS space id 5719 page no 70 n bits 248 index PRIMARY of table `point` trx id 584841584 lock mode S locks rec but not gap
Record lock, heap no 212 PHYSICAL RECORD: n_fields 7; compact format; info bits 0
...

*** WE ROLL BACK TRANSACTION (1)
```

```
2024-11-10 16:09:14 0x7fedb6f91640
*** (1) TRANSACTION:
TRANSACTION 584841584, ACTIVE 0 sec starting index read
mysql tables in use 1, locked 1
```
첫 번째 트랜잭션을 분석해보자
```
2024-11-10 16:09:14 0x7fedb6f91640
*** (1) TRANSACTION:
TRANSACTION 584841584, ACTIVE 0 sec starting index read
mysql tables in use 1, locked 1
LOCK WAIT 16 lock struct(s), heap size 1128, 6 row lock(s), undo log entries 4
MariaDB thread id 1366188, OS thread handle 140658953754176, query id 3671160502 아이피 ... Updating
update point set amount = 2200 where uuid=_binary '...'
*** WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 5719 page no 70 n bits 248 index PRIMARY of table `point` trx id 584841584 lock_mode X locks rec but not gap waiting
Record lock, heap no 212 PHYSICAL RECORD: n_fields 7; compact format; info bits 0
...

*** CONFLICTING WITH:
RECORD LOCKS space id 5719 page no 70 n bits 248 index PRIMARY of table `point` trx id 584841583 lock mode S locks rec but not gap
Record lock, heap no 212 PHYSICAL RECORD: n_fields 7; compact format; info bits 0
....

RECORD LOCKS space id 5719 page no 70 n bits 248 index PRIMARY of table `point` trx id 584841584 lock mode S locks rec but not gap
Record lock, heap no 212 PHYSICAL RECORD: n_fields 7; compact format; info bits 0
...
```
실행 쿼리는 다음과 같다.
```
update point set amount = 2200 where uuid=_binary '...'
```
포인트를 업데이트하는 쿼리다.
```
*** WAITING FOR THIS LOCK TO BE GRANTED:
```
그런데 이곳에서 데드락이 발생했다.

```
RECORD LOCKS space id 5719 page no 70 n bits 248 index PRIMARY of table `point` trx id 584841584 lock_mode X locks rec but not gap waiting
Record lock, heap no 212 PHYSICAL RECORD: n_fields 7; compact format; info bits 0
```
트랜잭션 아이디 1584가 x락 얻기 위해 대기 중이다.

```
*** CONFLICTING WITH:
RECORD LOCKS space id 5719 page no 70 n bits 248 index PRIMARY of table `point` trx id 584841583 lock mode S locks rec but not gap
Record lock, heap no 212 PHYSICAL RECORD: n_fields 7; compact format; info bits 0
....

RECORD LOCKS space id 5719 page no 70 n bits 248 index PRIMARY of table `point` trx id 584841584 lock mode S locks rec but not gap
Record lock, heap no 212 PHYSICAL RECORD: n_fields 7; compact format; info bits 0
```
트랜잭션 1583, 1584가 모두 s락을 획득하고 있다.

이제 두 번째 쿼리를 분석하자.
```
*** (2) TRANSACTION:
TRANSACTION 584841583, ACTIVE 0 sec starting index read
mysql tables in use 1, locked 1
LOCK WAIT 16 lock struct(s), heap size 1128, 6 row lock(s), undo log entries 4
MariaDB thread id 1366175, OS thread handle 140658935838272, query id 3671160499 아이피  Updating
update point set amount=2200 where uuid=_binary '...'
*** WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 5719 page no 70 n bits 248 index PRIMARY of table `point` trx id 584841583 lock_mode X locks rec but not gap waiting
Record lock, heap no 212 PHYSICAL RECORD: n_fields 7; compact format; info bits 0
...

*** CONFLICTING WITH:
RECORD LOCKS space id 5719 page no 70 n bits 248 index PRIMARY of table `point` trx id 584841583 lock mode S locks rec but not gap
Record lock, heap no 212 PHYSICAL RECORD: n_fields 7; compact format; info bits 0
...

RECORD LOCKS space id 5719 page no 70 n bits 248 index PRIMARY of table `point` trx id 584841584 lock mode S locks rec but not gap
Record lock, heap no 212 PHYSICAL RECORD: n_fields 7; compact format; info bits 0
```
트랜잭션 1583이 시작되었다. 

```
update point set amount=2200 where uuid=_binary '...'
```
실행한 쿼리는 다음과 같다.

```
*** WAITING FOR THIS LOCK TO BE GRANTED:
```
트랜잭션 1584와 마찬가지로 데드락이 발생했다.
```
RECORD LOCKS space id 5719 page no 70 n bits 248 index PRIMARY of table `point` trx id 584841583 lock_mode X locks rec but not gap waiting
Record lock, heap no 212 PHYSICAL RECORD: n_fields 7; compact format; info bits 0
```
x락 얻기 위해 대기하고 있다.


```
*** CONFLICTING WITH:
RECORD LOCKS space id 5719 page no 70 n bits 248 index PRIMARY of table `point` trx id 584841583 lock mode S locks rec but not gap
Record lock, heap no 212 PHYSICAL RECORD: n_fields 7; compact format; info bits 0
....

RECORD LOCKS space id 5719 page no 70 n bits 248 index PRIMARY of table `point` trx id 584841584 lock mode S locks rec but not gap
Record lock, heap no 212 PHYSICAL RECORD: n_fields 7; compact format; info bits 0
```
트랜잭션 1583, 1584가 모두 s락을 획득하고 있다.

x락 획득 과정
x락을 획득하기 위해서는 s락을 획득해야 한다. s락은 여러 트랜잭션에서 얻을 수 있는 공유락이다.
x락은 배타적 잠금이기 때문에 x락을 얻기 위해서는 다른 트랜잭션이 보유하고 있는 s락이 모두 커밋되거나 롤백되어야 x락을 얻을 수 있다.

[그림]
하지만 tx 1583는 x락을 획득하기 위해 1584의 s락 반납을 대기하고 있고 1584는 x락을 획득하기 위해 1583의 s락 반납을 대기하고 있다. 여기에서 데드락이 발생했다.

