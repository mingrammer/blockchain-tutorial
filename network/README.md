# Part 7. 네트워크

## 소개

지금까지 우리는 익명의 안전하고 무작위로 생성된 주소, 블록체인 데이터 저장소, 작업 증명 시스템, 신뢰할 수 있는 트랜잭션 저장 방법등의 핵심 기능들을 모두 갖춘 블록체인을 만들었다. 이 기능들도 물론 중요하지만 아직 부족하다. 이 기능들을 실제로 빛나게하고 암호화 화폐를 가능케 하는것은 바로 네트워크이다. 이러한 블록체인을 하나의 컴퓨터에서만 실행하면 무슨 소용인가? 단 한 명의 사용자만 있다면 이러한 암호화 기반의 기능들이 무슨 소용인가? 이 메커니즘들을 작동시키고 유용하게 만드는건 네트워크이다.

이러한 블록체인의 기능들은 하나의 규칙으로 생각할 수 있다. 사람들이 함께 살고 번영하고 싶어서 세우는 사회 질서의 규칙들과 유사하다. 블록체인 네트워크는 네트워크를 활성화시키는 동일한 규칙을 따르는 프로그램들의 커뮤니티이다. 마찬가지로, 사람들이 똑같은 생각을 공유할 때 그들은 강해지고 함께 더 나은 삶을 살 수 있다. 사람들이 서로 다른 규칙을 따르면 그들은 분리된 사회에서 살게된다. 이와 동일하게, 블록체인 노드들이 서로 다른 규칙을 따르면 이는 분리된 네트워크를 형성하게 될 것이다.

**매우 중요함:** 네트워크가 없고 대다수의 노드가 동일한 규칙을 공유하지 않는다면 이러한 규칙들은 쓸모가 없다.

> 아쉽지만, 필자는 P2P 네트워크 프로토타입을 구현할 시간이 없었다. 이 파트에서는 서로 다른 타입의 노드를 포함하는 가장 일반적인 시나리오 하나를 시연해볼 것이다. 이 시나리오를 개선하고 P2P 네트워크를 구현해보는건 여러분에게 훌륭한 도전거리가 될 것이다. 또한, 이 파트에서 구현한 네트워크가 시연할 시나리오 이외의 시나리오에서도 잘 동작할지는 보장할 수 없다. 이 부분은 죄송하다!
>
> 이 파트에서는 중요한 코드 변경사항을 소개하기 때문에 모든 코드를 설명하는건 의미가 없다. 모든 변경사항은 [여기](https://github.com/Jeiwan/blockchain_go/compare/part_6...part_7#files_bucket)를 참조하라.

## 블록체인 네트워크

블록체인 네트워크는 분산되어있어 작업을 수행하는 서버와 서버에서 데이터를 가져와 처리하는 클라이언트가 따로 없다. 블록체인 네트워크에는 노드가 존재하며 각 노드들은 네트워크의 일원으로서 자격을 지닌다. 노드는 클라이언트이자 서버이다. 일반적인 웹 애플리케이션과 매우 다르기 때문에 꼭 명심하길 바란다.

블록체인 네트워크는 P2P (Peer-to-Peer) 네트워크기 때문에 노드들은 서로 직접 연결되어있다. 이 구조의 토폴로지는 평면인데 노드 역할에는 계층이 없기 때문이다. 이를 도식화하면 다음과 같다.

<center> <img src="../images/p2p-network.png" align="center"/> </center>

이러한 네트워크의 노드는 많은 연산을 수행해야하기 때문에 구현하기가 더욱 까다롭다. 각 노드는 다른 여러 노드들과 상호작용 해야하며, 다른 노드의 상태를 요청하고 자신의 상태와 비교해 오래된 값이면 상태를 업데이트 해야한다.

## 노드의 역할

블록체인의 노드들은 모두 완전한 기능을 갖추고 있지만 네트워크에서 서로 다른 역할을 수행할 수 있다. 역할은 다음과 같다.

1. 채굴 노드: 채굴 노드는 ASIC과 같은 강력하고 특수한 하드웨어 위에서 동작하며, 이 노드들의 유일한 목적은 새로운 블록을 가능한한 빠르게 채굴하는 것이다. 채굴 노드는 작업 증명 시스템을 사용하는 블록체인에서만 가능하다. 채굴이란게 실제로는 PoW 퍼즐을 푸는 일이기 때문이다. 한 예로, 지분 증명 (Proof-of-Stake) 블록체인에는 채굴이 없다.

2. 풀 노드: 이 노드는 채굴 노드에 의해 채굴된 블록의 유효성을 확인하고 트랜잭션을 검증한다. 이 작업을 위해선 블록체인의 전체 복제본이 필요하다. 또한 이 노드는 다른 노드들이 서로를 찾을 수 있도록 돕는일과 같은 라우팅 연산을 수행하기도 한다.
풀 노드는 블록 및 트랜잭션의 유효성을 검사하고 의사결정을 하는 노드이기 때문에 네트워크는 많은 풀 노드를 갖는게 중요하다.

3. SPV: SPV는 Simplified Payment Verification의 약자이다. 이 노드는 블록체인의 전체 복사본을 저장하고 있지는 않지만 트랜잭션 검증은 가능하다 (단, 모든 검증 작업이 가능한건 아니며 일부만 가능하다. 예를 들어, 어떤 트랜잭션이 특정 주소에서 전송되었는지 확인하는 검증은 가능하다). SPV 노드는 풀 노드로부터 데이터를 얻어오며 하나의 풀 노드에 여러개의 SPV 노드가 연결될 수 있다. SPV는 지갑 애플리케이션을 가능케한다. 개인은 전체 블록체인을 다운로드 받을 필요는 없으나 여전히 트랜잭션을 검증할 수 있다.

## 네트워크 단순화

우리가 만든 블록체인에서 네트워크를 구현하기 위해선 일부 기능들을 단순화 시켜야한다. 한 가지 문제는 다수의 노드를 가진 네트워크를 시뮬레이션 하기 위해 필요한 충분한 수의 컴퓨터를 가지고 있지 않다는 것이다. 이 문제를 해결하기 위해 가상머신이나 도커를 사용할 수도 있지만 되려 더 어려워질 수가 있다. 블록체인 구현에만 집중하는게 목표인데 가상머신이나 도커를 사용하면서 발생할 수 있는 이슈를 해결해야할 수도 있기 때문이다. 따라서 우리는 하나의 머신에서 여러 블록체인 노드를 동시에 실행하여 각 노드에 서로 다른 주소를 할당해 사용할 것이다. 이를 위해 IP 주소 대신 **포트를 노드 식별자**로 사용할 것이다. 예로 **127.0.0.1:3000**, **127.0.0.1:3001**, **127.0.0.1:3002** 등의 주소를 가진 노드가 있을 수 있다. 우리는 포트를 노드 ID로 두고 **NODE_ID** 환경 변수를 사용해 값을 설정할 것이다. 따라서, 여러대의 터미널창을 띄워 서로 다른 **NODE_ID**를 설정해 독립된 노드들을 실행할 수 있다.

이 방법을 사용하려면 서로 다른 블록체인과 지갑 파일이 필요하다. 이 파일들은 노드 ID를 사용해 **blockchain_3000.db**, **blockchain_3001.db** 및 **wallet_3000.db**, **wallet_3001.db** 등등과 같이 명명할 것이다.

## 구현

비트코인 코어를 다운로드하여 처음 실행하면 어떤 일이 일어날까? 최신 상태의 블록체인을 다운로드 받기 위해 일부 노드와 연결되어야 한다.
컴퓨터가 비트코인의 전체 또는 일부 노드를 인식하지 못한다는것을 고려할 때, 이 노드는 어떤게 될 수 있을까?

비트코인 코어에서 노드 주소를 하드코딩하면 잘못될 수도 있다. 노드는 공격 당하거나 종료되어 새로운 노드가 네트워크에 참여할 수 없게될 수도 있다. 비트코인 코어에서는 대신 [DNS 시드](https://bitcoin.org/en/glossary/dns-seed)가 하드코딩 되어있다. 이는 노드가 아니며 일부 노드의 주소를 알고 있는 DNS 서버이다. 비트코인 코어를 최초로 실행하면 코어는 시드중 하나에 연결되어 블록체인을 다운로드 받을 풀 노드들의 주소가 담긴 리스트를 받아올 것이다.

하지만 우리의 구현에서는 중앙 집중화가 이루어질 것이다. 우리는 세 종류의 노드를 갖는다.

1. 중앙 노드: 모든 노드와 연결되어 있는 노드이며 다른 노드들간에 데이터를 전송하는 노드이다.
2. 채굴자 노드: 이 노드는 새로운 트랜잭션을 mempool에 저장하며 충분한 양의 트랜잭션이 쌓이면 새로운 블록을 채굴한다.
3. 지갑 노드: 이 노드는 지갑간 코인을 전송하는데 사용된다. SPV 노드와는 다르게 블록체인의 전체 복사본을 저장한다.

### 시나리오

목표는 다음 시나리오를 구현하는 것이다.

1. 중앙 노드는 블록체인을 생성한다.
2. 다른 지갑 노드들은 중앙 노드에 연결되어 블록체인을 다운로드 받는다.
3. 채굴자 노드들도 중앙 노드에 연결되어 블록체인을 다운로드 받는다.
4. 지갑 노드는 트랜잭션을 생성한다.
5. 채굴자 노드는 트랜잭션을 받아 메모리풀에 저장해둔다.
6. 메모리풀에 충분한 양의 트랜잭션이 쌓이면 채굴자 노드는 새로운 블록의 채굴을 시작한다.
7. 새로운 블록이 채굴되면, 중앙 노드에 전송된다.
8. 지갑 노드는 중앙 노드와 동기화된다.
9. 지갑 노드의 사용자는 지불이 정상적으로 이루어졌는지 확인한다.

이는 비트코인에서의 시나리오와 유사하다. 물론 실제 P2P 네트워크를 구축하지는 않겠지만 비트코인의 실제로 동작하는 가장 중요한 사용 사례를 구현할 것이다.

### 버전

노드는 메시지를 통해 통신한다. 새로운 노드가 실행되면 DNS 시드에서 여러 노드를 가져와 **version** 메시지를 전송한다. 메시지는 다음과 같이 구현할 수 있다.

```go
type version struct {
        Version    int
        BestHeight int
        AddrFrom   string
}
```

우리의 블록체인은 단 하나의 버전만 가지기 때문에 **Version** 필드는 중요하지 않다. **BestHeight**는 노드가 가진 블록체인의 길이를 저장한다. **AddrFrom**은 전송자의 주소를 저장한다.

**version** 메시지를 받은 노드는 무엇을 해야할까? 이 노드는 자신의 **version** 메시지로 응답한다. 이는 핸드셰이크의 일종으로 서로간 인사말 메시지를 주고받기 전까진 상호작용을 할 수 없다. 이는 단순히 공손함을 표현하는게 아니다. **version**은 더 긴 블록체인을 찾는데 사용된다. 노드가 **version** 메시지를 받으면 이는 자신이 가진 블록체인이 **BestHeight**보다 더 긴지 확인한다. 더 짧은 경우, 노드는 누락된 블록을 요청하여 다운로드 받는다.

메시지를 받으려면 서버가 필요하다.

```go
var nodeAddress string
var knownNodes = []string{"localhost:3000"}

func StartServer(nodeID, minerAddress string) {
        nodeAddress = fmt.Sprintf("localhost:%s", nodeID)
        miningAddress = minerAddress
        ln, err := net.Listen(protocol, nodeAddress)
        defer ln.Close()

        bc := NewBlockchain(nodeID)

        if nodeAddress != knownNodes[0] {
                sendVersion(knownNodes[0], bc)
        }

        for {
                conn, err := ln.Accept()
                go handleConnection(conn, bc)
        }
}
```

먼저 중앙 노드의 주소를 하드코딩한다. 모든 노드는 처음에 연결할 노드를 알아야한다. **minerAddress** 인자는 채굴 보상을 받을 주소를 지정한다.

```go
if nodeAddress != knownNodes[0] {
        sendVersion(knownNodes[0], bc)
}
```

현재 노드가 중앙 노드가 아니면 자신의 블록체인이 최신 데이터인지 확인하기위해 중앙 노드에 **version** 메시지를 전송해야한다.

```go
func sendVersion(addr string, bc *Blockchain) {
        bestHeight := bc.GetBestHeight()
        payload := gobEncode(version{nodeVersion, bestHeight, nodeAddress})

        request := append(commandToBytes("version"), payload...)
        sendData(addr, request)
}
```

메시지는 로우 레벨에서 보면 바이트의 시퀀스이다. 첫 12 바이트는 커맨드명 (여기선 "version")을 나타내며 이어지는 바이트는 **gob**으로 인코딩된 메시지 구조체이다. **commandToBytes**는 다음과 같이 구현한다.

```go
func commandToBytes(command string) []byte {
        var bytes [commandLength]byte

        for i, c := range command {
                bytes[i] = byte(c)
        }
        return bytes[:]
}
```

이 함수는 12 바이트의 버퍼를 만들어 커맨드명을 채워넣은 뒤 나머지 바이트는 빈 상태 그대로 둔다. 반대로 바이트 시퀀스를 커맨드로 변환하는 함수도 있다.

```go
func bytesToCommand(bytes []byte) string {
        var command []byte

        for _, b := range bytes {
                if b != 0x0 {
                        command = append(command, b)
                }
        }
        return fmt.Sprintf("%s", command)
}
```

노드가 커맨드를 수신하면, **bytesToCommand**를 통해 커맨드명을 가져와 적절한 핸들러로 커맨드 내용을 처리한다.

```go
func handleConnection(conn net.Conn, bc *Blockchain) {
        request, err := ioutil.ReadAll(conn)
        command := bytesToCommand(request[:commandLength])
        fmt.Printf("Received %s command\n", command)

        switch command {
        ...
        case "version":
                handleVersion(request, bc)
        default:
                fmt.Println("Unknown command!")
        }

        conn.Close()
}
```

**version** 커맨드 핸들러는 다음과 같다.

```go
func handleVersion(request []byte, bc *Blockchain) {
        var buff bytes.Buffer
        var payload version

        buff.Write(request[commandLength:])
        dec := gob.NewDecoder(&buff)
        err := dec.Decode(&payload)

        myBestHeight := bc.GetBestHeight()
        foreignerBestHeight := payload.BestHeight

        if myBestHeight < foreignerBestHeight {
                sendGetBlocks(payload.AddrFrom)
        } else if myBestHeight > foreignerBestHeight {
                sendVersion(payload.AddrFrom, bc)
        }

        if !nodeIsKnown(payload.AddrFrom) {
                knownNodes = append(knownNodes, payload.AddrFrom)
        }
}
```

우선 요청을 디코딩하여 페이로드를 가져온다. 이는 모든 핸들러의 공통사항이므로, 앞으로 작성할 코드 스니펫에서 이 코드는 생략할 것이다.

그 다음 노드는 자신의 **BestHeight**와 메시지에서 받은 값을 비교한다. 노드가 가진 블록체인이 더 길면 **version** 메시지로 회신하며, 짧으면 **getblocks** 메시지를 전송한다.

### getblocks

```go
type getblocks struct {
        AddrFrom string
}
```

**getblocks**는 "네가 가지고 있는 블록을 보여줘"라는 의미를 지닌다 (비트코인에서는 좀 더 복잡하다). 이는 모든 블록들을 가져오는게 아닌 블록들의 해시 리스트를 요청하는것임에 주의해야한다. 이는 네트워크 로드를 낮추기 위함인데 블록들은 다른 노드들에서도 가져올 수 있으므로, 한 노드에서 수십 기가바이트에 해당하는 블록들을 다운로드할 필요는 없기 때문이다.

커맨드 처리는 아주 쉽다.

```go
func handleGetBlocks(request []byte, bc *Blockchain) {
        ...
        blocks := bc.GetBlockHashes()
        sendInv(payload. AddrFrom, "block", blocks)
}
```

우리의 간소화된 구현에서 이는 **모든 블록의 해시**를 반환한다.

### inv

```go
type inv struct {
        AddrFrom string
        Type     string
        Items    [][]byte
}
```

비트코인은 **inv**를 사용해 현재 노드가 가진 블록 및 트랜잭션을 다른 노드에 표시한다. 다시 말하지만, 이는 전체 블록과 트랜잭션이 아니라 그저 블록과 트랜잭션의 해시값만 포함하고있다.

**inv** 처리는 조금 복잡하다.

```go
func handleInv(request []byte, bc *Blockchain) {
        ...
        fmt.Printf("Received inventory with %d %s\n", len(payload.Items), payload.Type)

        if payload.Type == "block" {
                blocksInTransit = payload.Items

                blockHash := payload.Items[0]
                sendGetData(payload.AddrFrom, "block", blockHash)

                newInTransit := [][]byte{}
                for _, b := range blocksInTransit {
                        if bytes.Compare(b, blockHash) != 0 {
                                newInTransit = append(newInTransit, b)
                        }
                }
                blocksInTransit = newInTransit
        }

        if payload.Type == "tx" {
                txID := payload.Items[0]
                if mempool[hex.EncodeToString(txID)].ID == nil {
                        sendGetData(payload.AddrFrom, "tx", txID)
                }
        }
}
```

블록 해시값들을 전달받으면 다운로드한 블록을 추적하기 위해 **blocksInTransit** 변수에 저장한다. 이렇게 하면 다른 노드에서 블록을 다운로드 할 수 있다. 블록을 전송 상태로 전환한 직후에 **inv** 메시지를 보낸 노드에게 **getdata** 커맨드를 전송하고 **blockInTransit**을 갱신한다. 실제 P2P 네트워크에서는 메시지를 보낸 노드만이 아니라 서로 다른 노드에서 블록을 전송하려고 한다.

우리의 구현에서는 **inv**에 다중 해시를 전송하지 않는다. 이게 **payload.Type == "tx"**에서 첫 해시만 취해 검사하는 이유이다. mempool에 해시가 존재하는지 확인한 뒤, 없으면 **getdata** 메시지를 전송한다.

### getdata

```go
type getdata struct {
        AddrFrom string
        Type     string
        ID       []byte
}
```

**getdata**는 특정 블록 및 트랜잭션에 대한 요청이며, 단 하나의 블록 및 트랜잭션의 ID만을 포함할 수 있다.

```go
func handleGetData(request []byte, bc *Blockchain) {
        ...
        if payload.Type == "block" {
                block, err := bc.GetBlock([]byte(payload.ID))
                sendBlock(payload.AddrFrom, &block)
        }

        if payload.Type == "tx" {
                txID := hex.EncodeToString(payload.ID)
                tx := mempool[txID]
                sendTx(payload.AddrFrom, &tx)
        }
}
```

요청이 블록이면 블록을, 트랜잭션이면 트랜잭션을 반환한다. 노드가 블록 또는 트랜잭션을 가지고 있는지 확인하는 과정이 없는데 이는 결함이다.

### block과 tx

```go
type block struct {
        AddrFrom string
        Block    []byte
}

type tx struct {
        AddrFrom    string
        Transaction []byte
}
```

실제로 데이터를 전달하는 메시지 구조체이다.

**block** 메시지 처리는 쉽다.

```go
func handleBlock(request []byte, bc *Blockchain) {
        ...

        blockData := payload.Block
        block := DeserializeBlock(blockData)

        fmt.Println("Received a new block!")
        bc.AddBlock(block)

        fmt.Printf("Added block %x\n", block.Hash)

        if len(blocksInTransit) > 0 {
                blockHash := blocksInTransit[0]
                sendGetData(payload.AddrFrom, "block", blockHash)
                blocksInTransit = blocksInTransit[1:]
        } else {
                UTXOSet := UTXOSet{bc}
                UTXSOSet.Reindex()
        }
}
```

새로운 블록을 받으면 자신의 블록체인에 추가한다. 다운로드할 블록이 더 있다면 이전 블록을 다운로드 받은 동일한 노드에게 블록을 요청한다. 블록이 모두 다운로드되면 UTXO 집합의 재색인이 이루어진다.

> TODO: 무조건 신뢰하는 대신, 블록체인에 추가하기 전에 수신하는 모든 블록의 유효성을 검사해야한다.
>
> TODO: 블록체인이 크기 때문에 UTXOSet.Reindex() 대신 UTXOSet.Update(block)을 사용해야한다. 전체 UTXO 집합을 재색인하는데에는 아주 많은 시간이 소요된다.

**tx** 메시지 처리는 조금 복잡하다.

```go
func handleTx(request []byte, bc *Blockchain) {
    ...
    txData := payload.Transaction
    tx := DeserializeTransaction(txData)
    mempool[hex.EncodeToString(tx.ID)] = tx

    if nodeAddress == knownNodes[0] {
        for _, node := range knownNodes {
            if node != nodeAddress && node != payload.AddrFrom {
                sendInv(node, "tx", [][]byte{tx.ID})
            }
        }
    } else {
        if len(mempool) >= 2 && len(miningAddress) > 0 {
        MineTransactions:
            var txs []*Transaction

            for id := range mempool {
                tx := mempool[id]
                if bc.VerifyTransaction(&tx) {
                    txs = append(txs, &tx)
                }
            }

            if len(txs) == 0 {
                fmt.Println("All transactions are invalid! Waiting for new ones...")
                return
            }

            cbTx := NewCoinbaseTX(miningAddress, "")
            txs = append(txs, cbTx)

            newBlock := bc.MineBlock(txs)
            UTXOSet := UTXOSet{bc}
            UTXOSet.Reindex()

            fmt.Println("New block is mined!")

            for _, tx := range txs {
                txID := hex.EncodeToString(tx.ID)
                delete(mempool, txID)
            }

            for _, node := range knownNodes {
                if node != nodeAddress {
                    sendInv(node, "block", [][]byte{newBlock.Hash})
                }
            }

            if len(mempool) > 0 {
                goto MineTransactions
            }
        }
    }
}
```

처음에 할 일은 새로운 트랜잭션을 mempool에 추가하는 것이다 (다시 말하지만, 트랜잭션은 mempool에 추가되기 전에 반드시 검증되어야한다). 다음 코드를 살펴보자.

```go
if nodeAddress == knownNodes[0] {
    for _, node := range knownNodes {
        if node != nodeAddress && node != payload.AddFrom {
            sendInv(node, "tx", [][]byte{tx.ID})
        }
    }
}
```

현재 노드가 중앙 노드인지를 확인한다. 우리의 구현체에서 중앙 노드는 블록을 채굴하지 않는다. 대신 새로운 트랜잭션을 네트워크상의 다른 노드들에게 전달해준다.

그 다음 코드는 채굴자 노드에만 해당한다. 각 부분별로 살펴보자.

```go
if len(mempool) >= 2 && len(miningAddress) > 0 {}
```

**miningAddress**은 채굴자 노드에만 있다. 현재 (채굴자) 노드의 mempool에 두 개 이상의 트랜잭션이 있을때 채굴을 시작한다.

```go
for id := range mempool {
    tx := mempool[id]
    if bc.VerifyTransaction(&tx) {
        txs = append(txs, &tx)
    }
}

if len(txs) == 0 {
    fmt.Println("All transactions are invalid! Waiting for new ones...")
    return
}
```

먼저 mempool의 모든 트랜잭션이 검증된다. 유효하지 않은 트랜잭션은 무시되며, 유효한 트랜잭션이 없는 경우 채굴은 중단된다.

```go
cbTx := NewCoinbaseTX(miningAddress, "")
txs = append(txs, cbTx)

newBlock := bc.MineBlock(txs)
UTXOSet := UTXOSet{bc}
UTXOSet.Reindex()

fmt.Println("New block is mined!")
```

검증된 트랜잭션은 보상을 가진 코인베이스 트랜잭션과 함께 블록에 추가된다. 블록 채굴이 끝나면 UTXO 집합은 재색인된다.

> TODO: 마찬가지로, UTXOSet.Reindex 대신 UTXOSet.Update를 사용하는게 더 좋다.

```go
for _, tx := range txs {
    txID := hex.EncodeToString(tx.ID)
    delete(mempool, txID)
}

for _, node := range knownNodes {
    if node != nodeAddress {
        sendInv(node, "block", [][]byte{newBlock.Hash})
    }
}

if len(mempool) > 0 {
    goto MineTransactions
}
```

트랜잭션은 채굴된 후 mempool에서 제거된다. 현재 노드가 알고있는 모든 노드들은 새로운 블록 해시가 담긴 **inv** 메시지를 받는다. 이 노드들은 메시지를 처리한 후에 블록을 요청할 수 있다.

## 결과

앞에서 정의했던 시나리오를 시연해보자.

먼저 첫 번째 터미널에서 **NODE_ID**를 3000으로 설정 (**export NODE_ID=3000**)한다. 커맨드를 실행하는 노드를 알 수 있도록 설명하기 전마다 **NODE 3000**이나 **NODE 3001**과 같이 표시를 해두겠다.

**NODE 3000**

지갑과 블록체인을 생성하자.

```
$ blockchain_go createblockchain -address CENTRAL_NODE
```

(간결한 설명을 위해 가짜 주소를 사용하겠다)

커맨드 실행이 끝나면 블록체인은 하나의 제네시스 블록을 갖게될 것이다. 그 다음 다른 노드에서 사용할 수 있도록 블록을 저장해야한다. 제네시스 블록은 블록체인의 식별자로 사용된다 (비트코인의 제네시스 블록은 하드코딩 되어있다).

```
$ cp blockchain_3000.db blockchain_genesis.db
```

**NODE 3001**

다음으로, 새 터미널을 열어 노드 ID를 3001로 설정한다. 이는 지갑 노드이다. **blockchain_go createwallet**으로 몇 개의 주소를 생성해 각 주소를 **WALLET_1**, **WALLET_2**, **WALLET_3**라고 부르자.

**NODE 3000**

지갑 주소로 코인을 전송해보자.

```
$ blockchain_go send -from CENTREAL_NODE -to WALLET_1 -amount 10 -mine
$ blockchain_go send -from CENTREAL_NODE -to WALLET_2 -amount 10 -mine
```

**-mine** 플래그는 블록이 전송 노드에 의해 즉시 채굴됨을 의미한다. 초기에는 네트워크에 채굴자 노드가 없기 때문에 이 플래그를 사용해야한다.
이제 노드를 실행해보자.

```
$ blockchain_go startnode
```

이 노드는 반드시 시나리오가 끝날때까지 계속 실행되어야 한다.

**NODE 3001**

위에서 저장한 제네시스 블록으로 노드의 블록체인을 시작한다.

```
$ cp blockchain_genesis.db blockchain_3001.db
```

노드를 실행하자.

```
$ blockchain_go startnode
```

이 노드는 중앙 노드로부터 모든 블록을 다운로드 받는다. 잘 동작하는지 확인하기 위해 노드를 중지하고 잔고를 확인해보자.

```
$ blockchain_go getbalance -address WALLET_1
Balance of 'WALLET_1': 10

$ blockchain_go getbalance -address WALLET_2
Balance of 'WALLET_2': 10
```

3001 노드는 이제 자체 블록체인을 가지고 있기 때문에 **CENTRAL_NODE** 주소의 잔고도 확인할 수 있다.

```
$ blockchain_go getbalance -address CENTRAL_NODE
Balance of 'CENTRAL_NODE': 10
```

**NODE 3002**

새 터미널을 열어 노드 ID를 3002로 설정하고 지갑을 생성한다. 이는 채굴자 노드이다. 블록체인을 초기화하자.

```
$ cp blockchain_genesis.db blockchain_3002.db
```

노드를 실행하자.

```
$ blockchain_go startnode -miner MINER_WALLET
```

**NODE 3001**

코인을 전송해보자.

```
$ blockchain_go send -from WALLET_1 -to WALLET_3 -amount 1
$ blockchain_go send -from WALLET_2 -to WALLET_4 -amount 1
```

**NODE 3002**

채굴자 노드로 전환하면 새로운 블록을 채굴하는걸 볼 수 있을 것이다. 그리고 중앙 노드의 출력을 확인해보아라.

**NODE 3001**

지갑 노드로 전환하고 노드를 실행하자.

```
$ blockchain_go startnode
```

노드를 실행하면 새로 채굴된 블록을 다운로드할 것이다.

노드를 중지하고 잔고를 확인해보자.

```
$ blockchain_go getbalance -address WALLET_1
Balance of 'WALLET_1': 9

$ blockchain_go getbalance -address WALLET_2
Balance of 'WALLET_2': 9

$ blockchain_go getbalance -address WALLET_3
Balance of 'WALLET_3': 1

$ blockchain_go getbalance -address WALLET_4
Balance of 'WALLET_4': 1

$ blockchain_go getbalance -address MINER_WALLET
Balance of 'MINER_WALLET': 10
```

다 되었다!

## 결론

드디어 시리즈가 끝났다. P2P 네트워크의 실제 프로토타입을 구현하는 포스트를 더 작성하고 싶었지만 시간이 부족했다. 이 시리즈로 비트코인 기술에 대한 질문에 답을 얻고 새로운 질문을 제기하여 여러분 스스로가 답을 할 수 있기를 바란다. 비트코인 기술에는 더 많은 흥미로운 주제들이 숨겨져 있다! 행운을 빈다!

P.S [비트코인 네트워크 프로토콜](https://en.bitcoin.it/wiki/Protocol_documentation)에 설명되어 있는 **addr** 메시지를 구현함으로써 네트워크를 개선해볼 수 있다. 이는 노드들이 서로를 탐색할 수 있도록 만들어주기 때문에 아주 중요하다. 필자 또한 구현중이지만 아직 완성되진 않았다.

링크:

1. [소스 코드](https://github.com/Jeiwan/blockchain_go/tree/part_7)
2. [비트코인 네트워크](https://en.bitcoin.it/wiki/Network)
