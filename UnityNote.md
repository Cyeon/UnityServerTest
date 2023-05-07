# 유니티 연동 강의 노트

## #1

본격적인 유니티 연동 시작! 기존 사용하던 코드들을 어느 정도 재사용 할 수 있음. But 정확히 다 똑같이 유니티에서 사용할 순 없다. 

dln?을 바로 가져오면 기능을 사용할 수 있지만 디버깅이 어렵기 때문에 복사해서 가져오는 걸로 함. 

#### GenPacket.cs

**Span관련에서 오류 발생. -> 예전 방식대로 segment를 쪼개서 해결**

```cs
		ushort count = 0;
		count += sizeof(ushort);
		count += sizeof(ushort);
		this.playerId = BitConverter.ToInt32(segment.Array, segment.Offset + count);
```

**TryWriteBytes 오류 -> Array.Copy**

```cs
		count += sizeof(ushort);
		Array.Copy(BitConverter.GetBytes((ushort)PacketID.C_LeaveGame), 0, segment.Array, segment.Offset + count, sizeof(ushort));

		count += sizeof(ushort);
		Array.Copy(BitConverter.GetBytes(count), 0, segment.Array, segment.Offset, sizeof(ushort));
```



**PacketFormat 수정**

(수업에서도 파일 가져왔기 때문에 생략)



**NetworkManager.cs 생성**

DummyClient쪽 코드를 복붙해오고 수정해준다. 

```cs
    void Start()
    {
		// DNS (Domain Name System)
		string host = Dns.GetHostName();
		IPHostEntry ipHost = Dns.GetHostEntry(host);
		IPAddress ipAddr = ipHost.AddressList[0];
		IPEndPoint endPoint = new IPEndPoint(ipAddr, 7777);

		Connector connector = new Connector();

		connector.Connect(endPoint,
			() => { return _session; },
			1);
	}
```

## #2

PacketHandler 쪽에서 로그를 찍어도 찍히지 않음. 왜? 메인스레드 문제!

유니티는 지정한 게임 실행 스레드 외의 다른 스레드가 게임 오브젝트에 접근하는 것을 통제, 차단함. 따라서 로그가 찍히지 않음.

즉, 메인 스레드(지정한 게임 실행 스레드)에서 동작이 되도록 수정 해야함!

**PacketQueue**를 만들어 해결해준다 

```cs
using System.Collections;
using System.Collections.Generic;
using UnityEngine;

public class PacketQueue
{
    public static PacketQueue Instance { get; } = new PacketQueue();

    Queue<IPacket> _packetQueue = new Queue<IPacket>();
    object _lock = new object();

    public void Push(IPacket packet)
    {
        lock (_lock)
        {
            _packetQueue.Enqueue(packet);
        }
    }

    public IPacket Pop()
    {
        lock (_lock)
        {
            if (_packetQueue.Count == 0)
                return null;

            return _packetQueue.Dequeue();
        }
    }

    public List<IPacket> PopAll()
    {
        List<IPacket> list = new List<IPacket>();

        lock (_lock)
        {
            while (_packetQueue.Count > 0)
                list.Add(_packetQueue.Dequeue());
        }

        return list;
    }
}

```

이후 PacketQueue를 사용하도록 코드를 수정. 



MakePacket의 기능을 분리한다! 
패킷생성 -> 게임 외부 스레드. 패킷 처리 -> 게임 내부 스레드.

**ClientPacketManager.cs**

```cs
	T MakePacket<T>(PacketSession session, ArraySegment<byte> buffer) where T : IPacket, new()
	{
		T pkt = new T();
		pkt.Read(buffer);
		return pkt;
	}

	public void HandlePacket(PacketSession session, IPacket packet)
	{
		Action<PacketSession, IPacket> action = null;
		if (_handler.TryGetValue(packet.Protocol, out action))
			action.Invoke(session, packet);
	}
```

코드 수정을 모두 끝내면 PacketQueue에서 꺼내와서 작업을 처리하게 됨 



## #3

간단한 채팅 프로그램 말고 미니 프로젝트 작업하여 돌릴 예정 

PDL.xml 수정

서버 쪽 코드에서 클라이언트가 보낸 패킷 처리를 해줌 

클라 쪽 코드에서는 간단한 테스트만 할 거라 빌드 통과용 코드 작성 
중요한 건 유니티 쪽에서 할 거임 



## #4

이 강의에서 드디어 유니티를 좀 만짐 (야호!)

Player만 하는 게 아니라 다른 Player도 출력을 해서 이동하는 것까지 구현 하는 게 목표 



네트워크 매니저 말고 Player 스크립트 따로 만들어서 해줄거임 (Player/MyPlayer) 

MyPlayer에서 패킷을 보내주는 게 맞음! 왜냐하면 이동패킷이란 건 플레이어랑 관련 있는 거니까 ㅇㅇ

클라-서버 이동 동기화 방법 2가지

1.서버 쪽에서 허락 패킷이 왔을 때 이동

2. 클라에서 플레이어를 먼저 이동시키고 서버에서 응답이 왔을 때 보정



일단은 1번 쓴다고 함 개인적으로 생각해봤을땐 보통 게임에선 2번 쓸듯?



**Player.cs**

```cs
using UnityEngine;

public class Player : MonoBehaviour
{
    public int PlayerId { get; set; }

}
```

**MyPlayer.cs**

```cs
using System.Collections;
using System.Collections.Generic;
using UnityEngine;

public class MyPlayer : Player
{
    NetworkManager _network;
    void Start()
    {
        StartCoroutine("CoSendPacket");
        _network = GameObject.Find("NetworkManager").GetComponent<NetworkManager>();
    }

    /// <summary>
    /// 코루틴으로 0.25초 마다 패킷 전송 
    /// </summary>
    /// <returns></returns>
    IEnumerator CoSendPacket()
    {
        while (true)
        {
            yield return new WaitForSeconds(0.25f); // 이동 패킷은 실시간으로 (0.25sec)

            C_Move movePacket = new C_Move();
            movePacket.posX = UnityEngine.Random.Range(-50, 50);
            movePacket.posY = 0;
            movePacket.posZ = UnityEngine.Random.Range(-50, 50);
            _network.Send(movePacket.Write()); // 네트워크 매니저로 좌표 정보 전송
        }
    }
}

```

PlayerManager를 만들어서 플레이어들을 관리해줄거임 (Mono안 붙임!!)

**PlayerManager.cs**

```cs
using System.Collections;
using System.Collections.Generic;
using UnityEngine;

public class PlayerManager
{
    MyPlayer _myPlayer;
    Dictionary<int, Player> _players = new Dictionary<int, Player>(); // 플레이어 목록 

    public static PlayerManager Instance { get; } = new PlayerManager();
    /// <summary>
    /// 플레이어 리스트 생성/갱신 상황
    /// </summary>
    /// <param name="packet"></param>
    public void Add(S_PlayerList packet)
    {
        Object obj = Resources.Load("Player");

        foreach (S_PlayerList.Player p in packet.players)
        {
            GameObject go = Object.Instantiate(obj) as GameObject;

            if (p.isSelf)
            {
                MyPlayer myPlayer = go.AddComponent<MyPlayer>();
                myPlayer.PlayerId = p.playerId;
                myPlayer.transform.position = new Vector3(p.posX, p.posY, p.posZ);
                _myPlayer = myPlayer;
            }
            else
            {
                Player player = go.AddComponent<Player>();
                player.PlayerId = p.playerId;
                player.transform.position = new Vector3(p.posX, p.posY, p.posZ);
                _players.Add(p.playerId, player);
            }
        }
    }
    /// <summary>
    /// 나 혹은 누군가가 움직임
    /// </summary>
    /// <param name="packet"></param>
    public void Move(S_BroadcastMove packet)
    {
        if (_myPlayer.PlayerId == packet.playerId)
        {
            _myPlayer.transform.position = new Vector3(packet.posX, packet.posY, packet.posZ);
        }
        else
        {
            Player player = null;
            if (_players.TryGetValue(packet.playerId, out player))
            {
                player.transform.position = new Vector3(packet.posX, packet.posY, packet.posZ);
            }
        }
    }
    /// <summary>
    /// 나 혹은 누군가가 새로 접속한 경우
    /// </summary>
    /// <param name="packet"></param>
    public void EnterGame(S_BroadcastEnterGame packet)
    {
        if (packet.playerId == _myPlayer.PlayerId)
            return;

        Object obj = Resources.Load("Player");
        GameObject go = Object.Instantiate(obj) as GameObject;

		Player player = go.AddComponent<Player>();
		player.transform.position = new Vector3(packet.posX, packet.posY, packet.posZ);
		_players.Add(packet.playerId, player);
	}
    /// <summary>
    /// 나 혹은 누군가가 게임을 떠난 경우
    /// </summary>
    /// <param name="packet"></param>
    public void LeaveGame(S_BroadcastLeaveGame packet)
    {
        if (_myPlayer.PlayerId == packet.playerId)
        {
            GameObject.Destroy(_myPlayer.gameObject);
            _myPlayer = null;
        }
        else
        {
            Player player = null;
            if (_players.TryGetValue(packet.playerId, out player))
            {
                GameObject.Destroy(player.gameObject);
                _players.Remove(packet.playerId);
            }
        }
    }
}

```


