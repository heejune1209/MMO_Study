## TRANSACTION

생각보다 매우 대단한(?) 작업이기 때문에 어떻게 동작하는지 간단하게 집고 넘어가보자

![image](https://github.com/user-attachments/assets/99673f99-9391-4ef2-b8bd-37b9e63f53e5)

TRANSACTION의 ACID 특성

1) A (Atomicity)

원자성

ALL or Nothing = 애매하게 반만 되는 경우는 없다.

2) C (Consistency)

일관성

데이터 간의 일관성 보장 (ex. 데이터와 인덱스 간 불일치 등)

3) I (Isolation)

고립성

트랜잭션을 단독으로 실행하나, 다른 트랜잭션과 함께 실행하나 똑같다.

4) D (Durability)

지속성

장애가 발생하더라도 데이터는 반드시 복구 가능

![image](https://github.com/user-attachments/assets/0cc2dfd0-b468-4ff5-96da-ef7b01c76ce2)
만약에 플레이어 골드 감소까지 실제로 데이터가 적용을 했으면 Undo로그가 있을거니까 과거로 돌아갈수 있는 방법이 기록되어있으니까 
그다음에 아이템을 인벤토리에 추가가 실패했다고 하면은 Undo를 해서 과거상태로 돌이킬 수 있게 된다는 얘기 

결론

1) 실제 데이터를 바로 하드디스크에 반영하지 않고 로그를 이용한다.

2) 로그에 있는 두가지 기능 

⇒ REDO(Before → After) : 아직 하드에 반영이 안되어 있다면 저장된 로그를 통해 업뎃

⇒ Undo(After → Before) : 상황에 따라 과거로 돌아감

3) 미래로 간다 (Roll Forward)

4) 과거로 간다 (Roll Back)

**5) 따라서 데이터 베이스에 장애가 발생하더라도 로그를 이용해 복부/롤백**
