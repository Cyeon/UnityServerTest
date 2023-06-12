## 1강 - 커맨드 패턴

**람다의 캡처**

```cs
    public void Func(){
        GameRoom room;
        Action action = () =>
            {
                room.Init();
            };
    }
```

action 안쪽의 람다식에선 room을 모르니 오류가 나는 것이 당연한데 오류가 나지 않음. 
람다의 캡쳐라고 람다 함수에서 **람다식 외부에서 정의된 변수**를 참조하는 변수를 람다식 내부에 저장하고 사용하는 동작임.

이게 조금 위험한게 람다의 이 기능을 이용한 Action을 만들 때까지만 해도 null이 아니었는데 다시 사용할 땐 null이라던c가... 할 수도 있어서 이 방식으로 만들던 JobQueue를 다른 방식으로 만들 예정 

우리가 하고 싶은 건 결국 클라에서 요청이 왔을 때 그걸 캡슐화 해가지고 어떤 클래스로 걔가 요청한 걸 포함해주고 싶은 거 
함수를 어떻게 캡슐화 하냐가 관건

**Sever - Job.cs**

```cs
    public interface IJob
    {
        void Excute();
    }

    public class Job : IJob
    {
        Action _action = null;

        public Job(Action action)
        {
            _action = action;
        }

        public void Excute()
        {
            _action.Invoke();
        }
    }
```

인자가 하나도 없는 경우 이런 식으로 단순하게 캡슐화를 해주면 되는데 우리가 쓰는 함수들은 그렇게 단순한 함수들이 아님. 제네릭으로 인자를 받는 것도 만들어줄거. 

```cs
  public class Job<T1> : IJob
    {
        Action<T1> _action = null;
        T1 _t1;

        public Job(Action<T1> action, T1 t1)
        {
            _action = action;
            _t1 = t1;
        }

        public void Excute()
        {
            _action.Invoke(_t1);
        }
    }
```

이런 식으로 완성을 했으면 Job들을 모아서 실행시켜주는 클래스를 만들 예정. JobSerializer라는 이름으로 cs 파일 생성. 

JobQueue와 굉장히 유사하게 만들어질 예정. 내용을 복사해오고, JobQueue는 더 이상 사용하지 않을 예정이므로 삭제한다. 

지금까지는 람다 캡쳐를 이용하여 Action으로 캡슐화 하여 게임 서버에서 이용하고 있었는데 이제는 Job을 이용할 거기 때문에 복사해온 코드에서 Action들을 IJob으로 변경. 다른 코드들도 그에 맞추어 변경. 

**Sever - JobSerializer.cs**

```cs
using System;
using System.Collections.Generic;
using System.Text;

namespace Server.Game
{
    public class JobSerializer
    {
        Queue<IJob> _jobQueue = new Queue<IJob>();
        object _lock = new object(); // 멀티 스레드 환경에서 작동하므로 lock을 들고 있다. JobQueue 관련으로만 락 걸고 있음 
        bool _flush = false;

        public void Push(IJob job)
        {
            bool flush = false;

            lock (_lock)
            {
                _jobQueue.Enqueue(job);
                if (_flush == false) // 실행 중인 작업이 없으면 
                    flush = _flush = true; // 바로 실행 시키도록 
            }

            if (flush)
                Flush();
        }

        void Flush()
        {
            while (true) // 작업이 없어질 때까지 무한 실행 
            {
                IJob job = Pop();
                if (job == null)
                    return;

                job.Excute();
            }
        }

        IJob Pop()
        {
            lock (_lock)
            {
                if (_jobQueue.Count == 0) // 작업이 더 없으면
                {
                    _flush = false; // 실행 가능하도록 하고 
                    return null; // 널 반환 
                }
                return _jobQueue.Dequeue();
            }
        }
    }
}
```

또, 외부에서 Job을 일일히 만들어 주는 게 아니라 외부에서는 똑같이 Action으로 Push 하지만 내부에서 Job으로 변환하도록 래핑 함수 추가까지 해준다. 제네릭 버전 별로 총 4개 만들어준다. 

```cs
public void Push(Action action) { Push(new Job(action)); }
public void Push<T1>(Action<T1> action, T1 t1) { Push(new Job<T1>(action, t1)); }
public void Push<T1, T2>(Action<T1, T2> action, T1 t1, T2 t2) { Push(new Job<T1, T2>(action, t1, t2)); }
public void Push<T1, T2, T3>(Action<T1, T2, T3> action, T1 t1, T2 t2, T3 t3) { Push(new Job<T1, T2, T3>(action, t1, t2, t3)); }
```

여기까지 했으면 Job에 대한 부분은 끝났고 이제 이걸 이용하여 GameRoom을 수정할 차례. 

## 2강 - JobQueue

GameRoom에 JobSerializer를 적용시킬 거임. 일단은 상속 시켜서 사용.

그리고 GameRoom의 lock을 지워준다. (중요함)

왜냐하면 앞으론 일일히 락을 걸어주는 게 아니라 JobSerializer를 통하여 제어할 거기 때문. 이게 이번 작업의 핵심임. 

그냥 호출했던 GameRoom의 함수들을 Push 함수를 통하여 밀어넣도록 변경한다 

**Sever - RoomManager.cs / public GameRoom Add(int mapId)**

```cs
            gameRoom.Push(gameRoom.Init, mapId);
            //gameRoom.Init(mapId);
```

<details>

<summary> 참조 확인 팁 </summary>

<div markdown="1">

강의에는 public을 지우고 오류 나는 거 확인하라는 식으로 제대로 된 방법은 안 나왔는데 visual studio community(보라색)을 쓴다면 참조를 확인할 수 있는 방법이 2가지 있다.

1. 함수명 윗쪽에 참조를 눌러서 확인![](C:\Users\USER\AppData\Roaming\marktext\images\2023-06-09-10-25-31-image.png) 

2. shift+F12를 누르고서 하단부에서 확인

![](C:\Users\USER\AppData\Roaming\marktext\images\2023-06-09-10-26-12-image.png)

</div>

</details>

코드 다 바꿔줌. 

지금까지의 함수들(Update, EnterGame)은 언제 실행될 지 모르므로 모두 반환형이 없는 void 였음. 

근데 FindPlayer는 경우가 좀 다름. 몬스터에서 공격할 때 플레이어를 찾아오기 위해 씀. 이게 만약에 GameRoom이랑 관련이 없는 곳에서 하면 오류가 날텐데 지금은 다행히 그런 부분이 아님. 그래서 일단 넘어감. 

이번에는 Broadcast 이렇게 바로 사용해도 되는지 고민. 얘도 내부에서는 FindPlayer처럼 안전하게 사용할 수 있는데 외부로 넘어가면 위험해짐. 근데 지금은 안전한 곳에서만 호출하고 있으니 패스. 

서버랑 클라를 실행시켜봤을 때 예전이랑 다름 없이 똑같음. 서버 코드를 제대로 바꿨다는 뜻. 

게임 룸.CS 에서 함수 실행시키는 건 Push를 이용하지 않았는데 이건 안 고쳐도 문제 없음. 근데 고쳐주는 게 더 좋으니 수정해줌. 

그 담에 Map의 ApplyLeave 쪽이 위험해보여서 예외처리 해주고 그에 맞게 코드 수정해줌. 

~~서버 코드가 굉장히 얼레벌레 돌아가듯 강의도 그런 거 같다... 근데 이건 모든 프로그래머가 그런듯~~

## 3강 JobTimer

업데이트를 하면서 계속 틱을 확인했던 무식한 방법을 고칠 거임. 이번에는 기존과는 좀 다른 방법으로 관리 할 예정. 원래 있던 JobTimer를 옮기면서 시작. 

일단은 Action을 Ijob으로 바꾸고 관련 코드만 수정. 

이제 이 jobtimer를 어떻게 바꿀 거냐면 JobSerializer 안에 낑겨넣어서 관리를 같이하도록 할 거임.  

실질적인 당장 해야하는 일들은 JobQueue로 관리. 미래에 할 일들은 다 JobTimer에 넣었다가 꺼내면서 실행할 수 있는 애들은 실행하고 실행해야할 일감들을 실행시키는 방식. (말을 이상하게 해서 뭐라는 건지 좀 헷갈림...)

Jobtimer를 쓰기 위한 PushAfter 등의 함수를 추가하고, 기존 JobQueue 기능의 Push는 자기가 첫 작업이면 Flush도 알아서 실행하는 형식이었는데 이를 수정. Push는 Push만 Flush는 Flush만 하게 하고 다른 곳에서 작업을 실행시키도록 변경. 

**JobSerializer.cs**

```cs
        public void PushAfter(int tickAfter, Action action) { PushAfter(tickAfter, new Job(action)); }
        public void PushAfter<T1>(int tickAfter, Action<T1> action, T1 t1) { PushAfter(tickAfter, new Job<T1>(action, t1)); }
        public void PushAfter<T1, T2>(int tickAfter, Action<T1, T2> action, T1 t1, T2 t2) { PushAfter(tickAfter, new Job<T1, T2>(action, t1, t2)); }
        public void PushAfter<T1, T2, T3>(int tickAfter, Action<T1, T2, T3> action, T1 t1, T2 t2, T3 t3) { PushAfter(tickAfter, new Job<T1, T2, T3>(action, t1, t2, t3)); }

        public void PushAfter(int tickAfter, IJob job)
        {
            _timer.Push(job, tickAfter);
        }


         public void Push(IJob job)
        {
            lock (_lock)
            {
                _jobQueue.Enqueue(job);
            }
        }
```

그럼 이걸 어디서 Flush 하냐? GameRoom의 Update문. Update는 틱 단위로 주기적으로 호출되는 함수니까 ㅇㅇ. 

**GameRoom.cs**

```cs
        public void Update()
        {
            foreach (Monster monster in _monsters.Values)
            {
                monster.Update();
            }

            foreach (Projectile projectile in _projectiles.Values)
            {
                projectile.Update();
            }

            Flush();
        }
```

그럼 다음 문제는 Update를 어디서 호출 하는 게 제일 좋을까? while문에서 계속 호출을 하는 건 무식하니까 C#의 기능을 이용할 거임. 

이용하는 건 System.Timers의 Timer 클래스. 

```cs
        static void TickRoom(GameRoom room, int tick = 100)
        {
            var timer = new System.Timers.Timer();
            timer.Interval = 100;
            timer.Elapsed += ((s, e) => { room.Update(); });
        }
```

Interval -> 실행 시키는 주기 / Elapsed -> 뭐 실행 시킬건지 추가 

이 담에 AutoReset이랑 Enable 변수도 true로 설정해준다. 또, 혹시 모르니 리스트 만들어서 넣어서 보관해줌. 그리고 Main에서 만든 함수를 실행도 시켜줍니다. 틱은 50틱으로. 

이러면 50틱마다 room의 Update가 실행되고, 그 업데이트에서 Flush가 실행되는 구조 완성. 

## 4강 마무리

버그를 수정할 예정. 먼저, 몬스터가 죽었는데 계속 죽는 이펙트가 나오는 버그. 

    -> 저번에 GameRoom 수정할 때 몬스터 부분 코드 줄을 서로 바꿔주지 않아서 그럼. 바꿔주면 해결.

JobTimer를 추가하고 Tick을 하면서부터 버그가 생겼으므로 Tick 간격을 50->10으로 줄여서 자주 업데이트 해줌. 

지금 결국 문제가 되는 부분은, 예전에는 클라이언트에서 이동 패킷을 보내면 거의 바로 돌려줘서 클라/서버가 서로 거의 일치했음. 근데 지금은 경우에 따라서 딜레이가 생길 수 있게 됨(JobQueue 때문에). 

이걸 서버에 패킷을 보내고, 돌아왔을 때 포지션을 바꾸는 방식으로도 수정할 수 있겠지만 지금 문제는 이게 아니라 서버가 S_MovePacket을 보내는 시점이 문제임. 

핸들 무브를 보면 엄청나게 검증을 하진 않고 정보를 넣어서 보내줌. 거의 에코 서버. 문제는 클라는 이미 이동을 했는데 좀 뒤늦게 움직임을 반환함. 이게 문제가 뭐냐면 클라쪽 핸들러를 보면 서버에서 정보를 받아서 대입해주는데 이게 플레이어 본인도 해당됨. 이전 좌표로 설정이 될 수 잇다는 것. 일반적으론 여기서 내가 컨트롤하는 플레이어면 예외처리를 해서 움직여주지 않음. 이걸 코드에 적용시키자. 

```cs
    public static void S_MoveHandler(PacketSession session, IMessage packet)
    {
        S_Move movePacket = packet as S_Move;

        GameObject go = Managers.Object.FindById(movePacket.ObjectId);
        if (go == null)
            return;

        if (Managers.Object.MyPlayer.Id == movePacket.ObjectId)
            return;

        BaseController bc = go.GetComponent<BaseController>();
        if (bc == null)
            return;

        bc.PosInfo = movePacket.PosInfo;
    }
```

예외처리 추가 후 Tick 간격 롤백.

이후에 그리드 단위가 아닌 다른 게임은 어떻게 이동 동기화를 하는지 설명이랑 실제 MMORPG는 방 단위가 아니라 구역 단위로 한다거나 설명 나왔다. 그리고 다음 강의 설명이랑 강사 분의 과거 경험과 노하우도 좀 나왔다.  