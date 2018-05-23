# Part 3. 영속성 및 CLI

## 소개

우리는 [지금까지](/proof-of-work) 채굴이 가능한 작업 증명 시스템을 지닌 블록체인을 구현했다. 좀 더 완벽한 기능을 갖춘 블록체인에 가까워지고 있지만 여전히 일부 중요한 기능들이 부족하다. 이번 파트에서는 블록체인을 데이터베이스에 저장하고 블록체인 연산을 수행하기 위한 간단한 커맨드라인 툴을 만들어 볼 것이다. 블록체인은 본질적으로 데이터베이스이다. 지금은 "분산 (distributed)"부분을 생략하고 "데이터베이스"부분에 집중할 것이다.

## 데이터베이스 선택

지금까지 구현한 블록체인에는 데이터베이스가 없기 때문에 매번 프로그램을 실행할때마다 블록을 생성하여 메모리에 저장하고 있다. 따라서 블록체인을 재사용할 수 없으며 다른 사람들과도 공유할 수 없기 때문에 이를 디스크에 저장해야할 필요가 있다.

그렇다면 어떤 데이터베이스가 필요할까? 사실 아무거나 써도된다. [비트코인 논문](https://bitcoin.org/bitcoin.pdf)에서도 특정 데이터베이스에 대한 언급은 없으며 이는 개발자의 선택에 달렸다. 사토시 나카모토가 초창기에 배포하고 현재 비트코인 구현체의 레퍼런스로 있는 [비트코인 코어](https://github.com/bitcoin/bitcoin)에서는 [LevelDB](https://github.com/google/leveldb) (물론 이는 2012년에만 도입되었지만)를 사용한다. 그리고 우리는 다음 데이터베이스를 사용할 것이다.

## BoltDB

BoltDB를 사용하는 이유는 다음과 같다.

1. 단순하고 가볍다.
2. Go로 작성되었다.
3. 별도의 서버가 필요하지 않다.
4. 데이터 구조 설계가 자유롭다.

BoltDB의 [Github README.md](https://github.com/boltdb/bolt)에 따르면

> Bolt는 Howard Chu의 LMDB 프로젝트에서 영감을 받아 순수 Go로 작성된 키/값 스토어이다. 이는 Postgres나 MySQL과 같이 데이터베이스 서버가 필요하지 않은 프로젝트를 위해 간단하고 빠르며 안정적인 데이터베이스를 제공하고자 만들어졌다.
>
> Bolt는 저수준의 기능으로 사용되기 때문에 단순함이 중요하다. API가 적으며 값을 얻고 설정하는데에만 초점을 맞추고 있다. 이게 끝이다.

우리의 요구사항에 완벽히 들어맞는다! 조금만 더 살펴보자.

BoltDB는 키/값 스토리지로 SQL RDBMS (MySQL, PostgreSQL등)와 같은 테이블과 행 및 열이 필요없음을 의미한다. 대신 데이터는 Go에서의 map과 같이 키-값의 쌍으로 저장된다. 키-값 쌍은 유사한 쌍을 그룹화 하기위한 버킷에 저장된다 (이는 RDBMS에서의 테이블과 유사하다). 따라서 값을 가져오려면 버킷과 키 모두 알아야한다.

BoltDB에서 한가지 중요한 점은 데이터 타입이 없다는 것이다. 키와 값은 바이트 배열로 저장된다. 우리는 Go 구조체 (특히 **Block**)를 저장할 것이기 때문에 직렬화가 필요하다. Go 구조체를 바이트 배열로 변환하고 바이트 배열로부터 구조체를 복원하는 메커니즘을 구현해야한다. 우리는 [encoding/gob](https://golang.org/pkg/encoding/gob/)을 사용할 것이다. 이 패키지는 **JSON**, **XML** 및 **Protocol Buffers**와도 사용할 수 있다.

## 데이터베이스 구조

영속성 로직을 구현하기 전에, 우선 DB에 데이터를 저장하는 방식부터 결정해야한다. 비트코인 코어의 방식을 참조해보자.

간단히 말하면 비트코인 코어는 데이터를 저장하기 위해 두 개의 "버킷"을 사용한다.

1. **blocks**는 체인의 모든 블록을 설명하는 메타데이터를 저장한다.
2. **chainstate**는 체인의 상태를 저장한다. 이 상태는 현재 사용되지 않은 모든 트랙잭션과 일부 메타데이터를 포함한다.

또한 블록들은 디스크에 분리된 파일들로 저장된다. 이는 성능 때문인데 하나의 블록을 읽기 위해 전체 혹은 일부 데이터를 메모리에 로드할 필요는 없기 때문이다. 우리는 이 부분은 구현하지 않을 것이다.

**block**에는 다음과 같은 **키->값** 쌍들이 있다.

1. **'b' + 32바이트 블록 해시 -> 블록 인덱스 레코드**
2. **'f' + 4바이트 파일 번호 -> 파일 정보 레코드**
3. **'l' -> 4바이트 파일 번호: 마지막으로 사용된 블록의 파일 번호**
4. **'R' -> 1바이트 부울: 재색인 작업 진행 여부**
5. **'F' + 1바이트 플래그명 길이 + 플래그명 -> 1바이트 부울: 끄고 켤 수 있는 여러 플래그들**
6. **'t' + 32바이트 트랙잭션 해시 -> 트랜잭션 인덱스 레코드**

**chainstate**에는 다음과 같은 **키->값** 쌍들이 있다.

1. **'c' + 32바이트 트랜잭션 해시 -> 해당 트랜잭션에 대한 미사용 트랜잭션 출력 레코드**
2. **'B' -> 32바이트 블록 해시: 데이터베이스가 미사용 트랜잭션 출력을 나타내는 블록 해시**

*(자세한 설명은 [여기](https://en.bitcoin.it/wiki/Bitcoin_Core_0.11_%28ch_2%29:_Data_Storage)에서 볼 수 있다)*

우리에겐 아직 트랜잭션이 없기 때문에 **blocks** 버킷만 가질 것이다. 또한 앞서 언급했듯이 우리는 블록들을 여러 파일에 저장하지 않고 전체 DB를 하나의 파일로 저장할 것이다. 따라서 우리는 파일 번호와 관련된 데이터는 필요하지 않다. 따라서 우리가 사용할 키-값 쌍은 다음과 같다.

1. **32바이트 블록 해시 -> 직렬화된 블록 구조체**
2. **'l' -> 체인의 마지막 블록의 해시**

이제 영속성 메커니즘을 구현하는데 필요한 것들을 모두 살펴보았다.

## 직렬화

이전에도 말했듯이 BoltDB에서는**[]byte** 타입의 값만 사용할 수 있으며 우리는 DB에 **Block** 구조체를 저장하려고 한다. 우리는 구조체 직렬화에 [encoding/gob](https://golang.org/pkg/encoding/gob/)을 사용할 것이다.

**Block**의 **Serialize** 메서드를 구현해보자. (단순한 구현을 위해 에러 처리는 생략한다)

```go
func (b *Block) Serialize() []byte {
        var result bytes.Buffer

        encoder := gob.NewEncoder(&result)
        err := encoder.Encode(b)

        return result.Bytes()
}
```

아주 직관적인 코드로, 우선 직렬화된 데이터를 저장할 버퍼를 선언하고 **gob** 인코더를 초기화 한 뒤 블록을 인코딩하면 바이트 배열이 반환된다.

다음으로 바이트 배열을 받아 **Block**을 반환하는 역직렬화 함수가 필요하다. 이는 메서드가 아닌 독립적인 함수로 구현한다.

```go
func DeserializeBlock(d []byte) *Block {
        var block Block

        decoder := gob.NewDecoder(bytes.NewReader(d))
        err := decoder.Decode(&block)

        return *blcok
}
```

직렬화 관련 작업을 끝났다!

## 영속성

이제 **NewBlockchain** 함수를 살펴보자. 이 함수는 현재 새로운 **Blockchain** 인스턴스를 생성하고 여기에 제네시스 블록을 추가하고 있다. 우리가 원하는 기능은 다음과 같다.

1. DB 파일을 연다.
2. 저장된 블록체인이 있는지 확인한다.
3. 블록체인이 존재하면
    1. 새로운 **Blockchain** 인스턴스를 생성한다.
    2. **Blockchain** 인스턴스의 끝부분을 DB에 저장된 마지막 블록의 해시로 설정한다.
4. 블록체인이 존재하지 않으면
    1. 제네시스 블록을 생성한다.
    2. DB에 저장한다.
    3. 제네시스 블록의 해시를 마지막 블록의 해시로 저장한다.
    4. 제네시스 블록을 끝부분으로 하는 새로운 **Blockchain** 인스턴스를 생성한다.

코드로보면 다음과 같다.

```go
func NewBlockchain() *Blockchain {
        var tip []byte
        db, err := bolt.Open(dbFile, 0600, nil)

        err = db.Update(func(tx *bolt.Tx) error {
                b := tx.Bucket([]byte(blocksBucket))
                if b == nil {
                        genesis := NewGenesisBlock()
                        b, err := tx.CreateBucket([]byte(blocksBucket))
                        err = b.Put(genesis.Hash, genesis.Serialize())
                        err = b.Put([]byte("l"), genesis.Hash)
                        tip = genesis.Hash
                } else {
                        tip = b.Get([]byte("l"))
                }
                return nil
        })

        bc := Blockchain{tip, db}
        return &bc
}
```

한 줄씩 살펴보자.

```go
db, err := bolt.Open(dbFile, 0600, nil)
```

이는 BoltDB 파일을 여는 표준 방식이다. 파일이 없어도 오류를 반환하고 있지 않음을 볼 수 있다.

```go
err = db.Update(func(tx *bolt.Tx) error {
...
})
```

BoltDB에서 데이터베이스 연산은 트랜잭션내에서 실행된다. 읽기 전용과 읽기-쓰기의 두 가지 타입의 트랜잭션이 있다. 위 코드에서는 DB에 제네시스 블록을 써넣을수도 있기 때문에 읽기-쓰기 트랜잭션 (**db.Update(...)**)을 연다.

```go
b := tx.Bucket([]byte(blocksBucket))
if b == nil {
		genesis := NewGenesisBlock()
		b, err := tx.CreateBucket([]byte(blocksBucket))
		err = b.Put(genesis.Hash, genesis.Serialize())
		err = b.Put([]byte("l"), genesis.Hash)
		tip = genesis.Hash
} else {
		tip = b.Get([]byte("l"))
}
```

이 코드는 함수의 핵심이다. 블록을 저장하고 있는 버킷을 가져와 버킷이 존재하면 **l** 키를 읽고 존재하지 않으면 제네시스 블록을 생성하고, 버킷을 만들어 블록을 저장한뒤 체인의 마지막 블록의 해시를 저장하는 **l** 키를 업데이트한다.

여기서 **Blockchain**을 생성하는 또 다른 방식을 볼 수 있다.

```
bc := Blockchain{tip, db}
```

더 이상 모든 블록을 저장하지 않고 체인의 끝 블록의 해시만 저장한다. 또한 프로그램이 실행되는동안 한 번 열어둔 데이터베이스를 유지하기 위해 DB 커넥션을 저장한다. 따라서 **Blockchain** 구조체는 다음과 같아진다.

```go
type Blockchain struct {
		tip []byte
		db  *bolt.DB
}
```

다음으로 수정할 메서드는 **AddBlock** 메서드이다. 이제 체인에 블록을 추가하는건 배열에 원소를 추가하는 것만큼 쉽지않다. 지금부터는 블록을 DB에 저장할 것이다.

```go
func (bc *Blockchain) AddBlock(data string) {
		var lastHash []byte

		err := bc.db.View(func(tx *bolt.Tx) error {
				b := tx.Bucket([]byte(blocksBucket))
				lastHash = b.Get([]byte("l"))
				return nil
		})

		newBlock := NewBlock(data, lastHash)

		err = bc.db.Update(func(tx *bolt.Tx) error {
				b := tx.Bucket([]byte(blocksBucket))
				err := b.Put(newBlock.Hash, newBlock.Serialize())
				err = b.Put([]byte("l"), newBlock.Hash)
				bc.tip = newBlock.Hash
				return nil
		})
}
```

한 줄씩 살펴보자.

```go
err := bc.db.View(func(tx *bolt.Tx) error {
		b := tx.Bucket([]byte(blocksBucket))
		lastHash = b.Get([]byte("l"))
		return nil
})
```

이는 BoltDB의 읽기 전용 트랜잭션이다. 새로운 블록의 해시를 채굴하기 위해 DB로부터 마지막 블록의 해시값을 가져온다.

```go
newBlock := NewBlock(data, lastHash)
b := tx.Bucket([]byte(blocksBucket))
err := b.Put(newBlock.Hash, newBlock.Serialize())
err = b.Put([]byte("l"), newBlock.Hash)
bc.tip = newBlock.Hash
```

새로운 블록의 채굴이 끝나면 직렬화하여 DB에 저장하고 새로운 블록의 해시를 저장하는 **l** 키를 업데이트한다.

끝났다! 그리 어렵지는 않았다.

## 블록체인 탐색

이제 모든 블록을 데이터베이스에 저장하고 있기 때문에 블록체인을 다시 열어 새로운 블록을 추가할 수 있다. 그러나 이를 구현한 뒤로 우리는 멋진 기능 하나를 잃었다. 더 이상 블록을 배열에 저장하고 있지 않기 때문에 블록체인의 블록들을 출력할 수 없다. 이제 이 결함을 수정해보자!

BoltDB는 버킷의 모든 키를 순회할 수 있는 기능을 제공하지만 키들은 바이트 순서로 정렬되어있다. 그러나 우리는 블록체인에 삽입된 순서대로 블록을 출력하고싶다. 또한 (블록체인 DB가 거대해질 수 있으므로) 모든 블록을 메모리에 로드하고 싶지는 않기 때문에 블록들을 하나씩 읽을 것이다. 이를 위해선 블록체인 반복자 (Iterator)가 필요하다.

```go
type BlockchainIterator struct {
		currentHash []byte
		db          *bolt.DB
}
```

반복자는 블록체인의 블록을 반복할때마다 만들어지며 이는 현재 반복의 블록 해시와 DB 커넥션을 저장한다. 블록의 해시와 DB 커넥션이 필요하기 때문에 반복자는 논리적으로 블록체인과 연결되어 있으며 따라서 이는 **Blockchain**의 메서드로 만든다.

```go
func (bc *Blockchain) Iterator() *BlockchainIterator {
		bci := &BlockchainIterator{bc.tip, bc.db}
		return bci
}
```

반복자는 초기에 블록체인의 끝을 가리킨다. 따라서 블록은 위에서 아래로, 최신 블록부터 오래된 블록의 순서로 얻을 수 있다. 사실 **블록체인의 끝을 선택한다는 것은 블록체인에 대한 "투표"를 의미한다** . 블록체인은 여러개의 가지 (브랜치)를 가질 수 있으며 가장 긴 체인이 메인 체인으로 간주된다. 블록체인의 끝을 얻으면 (이는 블록체인의 어느 블록도 가능하다) 전체 블록체인을 재구성하여 체인의 길이와 체인을 만드는데 필요한 작업을 찾을 수 있다.

**BlockchainIterator**는 블록체인으로부터 다음 블록을 반환하는 단 한 가지 일만 수행한다.

```go
func (i *BlockchainIterator) Next() *Block {
		var block *Block

		err := i.db.View(func(tx *bolt.Tx) error {
			b := tx.Bucket([]byte(blocksBucket))
			encodedBlock := b.Get(i.currentHash)
			block = DeserializeBlock(encodedBlock)

			return nil
		})

		i.currentHash = block.PrevBlockHash
		return block
}
```

DB 파트가 끝났다!

## CLI

지금까지 우리가 만든 구현체는 프로그램과 상호 작용할 수 있는 그 어떤 인터페이스도 제공하지 않았다. 우리는 여태 단순히 **main** 함수에서 **NewBlockchain**과 **bc.AddBlock**을 실행시켰었다. 이제 이를 개선해보자! 우리는 다음 명령어들이 필요하다.

```bash
blockchain_go addblock "Pay 0.31337 for a coffee"
blockchain_go printchain
```

커맨드라인과 관련된 모든 연산들은 **CLI** 구조체에 의해 처리된다.

```go
type CLI struct {
		bc *Blockchain
}
```

CLI의 엔트리포인트는 **Run** 함수이다.

```go
func (cli *CLI) Run() {
		cli.validateArgs()

		addBlockCmd := flag.NewFlagSet("addblock", flag.ExitOnError)
		printChainCmd := flag.NewFlagSet("printchain", flag.ExitOnError)
		addBlockData := addBlockCmd.String("data", "", "Block data")

		switch os.Args[1] {
		case "addblock":
				err := addBlockCmd.Parse(os.Args[2:])
		case "printchain":
				err := printChainCmd.Parse(os.Args[2:])
		default:
				cli.printUsage()
				os.Exit(1)
		}

		if addBlockCmd.Parsed() {
				if *addBlockData == "" {
						addBlockCmd.Usage()
						os.Exit(1)
				}
				cli.addBlock(*addBlockData)
		}

		if printChainCmd.Parsed() {
				cli.printChain()
		}
}
```

표준 [flag](https://golang.org/pkg/flag/) 패키지를 사용해 커맨드라인 인자를 파싱한다.

```go
addBlockCmd := flag.NewFlagSet("addblock", flag.ExitOnError)
printChainCmd := flag.NewFlagSet("printchain", flag.ExitOnError)
addBlockData := addBlockCmd.String("data", "", "Block data")
```

우선, **addBlock**과 **printchain**이라는 두 개의 하위 커맨드를 만들고 **addBlock**에 **-data** 플래그를 추가한다. **printchain**에는 플래그가 없다.

```go
switch os.Args[1] {
case "addblock":
		err := addBlockCmd.Parse(os.Args[2:])
case "printchain":
		err := printChainCmd.Parse(os.Args[2:])
default:
		cli.printUsage()
		os.Exit(1)
}
```

다음엔 사용자가 입력한 커맨드를 검사하고 관련된 **flag** 하위커맨드를 파싱한다.

```go
if addBlockCmd.Parsed() {
		if *addBlockData == "" {
				addBlockCmd.Usage()
				os.Exit(1)
		}
		cli.addBlock(*addBlockData)
}

if printChainCmd.Parsed() {
		cli.printChain()
}
```

다음으로 어떤 하위커맨드가 파싱되었는지 확인한 뒤 관련 함수를 실행한다.

```go
func (cli *CLI) addBlock(data string) {
		cli.bc.AddBlock(data)
		fmt.Println("Success!")
}

func (cli *CLI) printChain() {
		bci := cli.bc.Iterator()

		for {
				block := bci.Next()
				fmt.Printf("Prev. hash: %x\n", block.PrevBlockHash)
				fmt.Printf("Data: %s\n", block.Data)
				fmt.Printf("Hash: %x\n", block.Hash)

				pow := NewProofOfWork(block)
				fmt.Printf("PoW: %s\n", strconv.FormatBool(pow.Validate()))
				fmt.Println()

				if len(block.PrevBlockHash) == 0 {
						break
				}
		}
}
```

이 코드는 이전에 봤던 코드와 아주 유사하다. 유일한 차이점은 블록체인의 블록을 순회하기 위해 **BlockchainIterator**를 사용하고 있다는 것이다.

이에 맞게 **main** 함수를 수정하는 것도 잊지 말자.

```go
func main() {
		bc := NewBlockchain()
		defer bc.db.Close()

		cli := CLI{bc}
		cli.Run()
}
```

커맨드라인 인자 입력과 무관하게 새로운 **Blockchain**이 만들어지고 있음을 볼 수 있다.

끝났다! 모든 기능이 잘 동작하는지 확인해보자.

```
$ blockchain_go printchain
No existing blockchain found. Creating a new one...
Mining the block containing "Genesis Block"
000000edc4a82659cebf087adee1ea353bd57fcd59927662cd5ff1c4f618109b

Prev. hash:
Data: Genesis Block
Hash: 000000edc4a82659cebf087adee1ea353bd57fcd59927662cd5ff1c4f618109b
PoW: true

$ blockchain_go addblock -data "Send 1 BTC to Ivan"
Mining the block containing "Send 1 BTC to Ivan"
000000d7b0c76e1001cdc1fc866b95a481d23f3027d86901eaeb77ae6d002b13

Success!

$ blockchain_go addblock -data "Pay 0.31337 BTC for a coffee"
Mining the block containing "Pay 0.31337 BTC for a coffee"
000000aa0748da7367dec6b9de5027f4fae0963df89ff39d8f20fd7299307148

Success!

$ blockchain_go printchain
Prev. hash: 000000d7b0c76e1001cdc1fc866b95a481d23f3027d86901eaeb77ae6d002b13
Data: Pay 0.31337 BTC for a coffee
Hash: 000000aa0748da7367dec6b9de5027f4fae0963df89ff39d8f20fd7299307148
PoW: true

Prev. hash: 000000edc4a82659cebf087adee1ea353bd57fcd59927662cd5ff1c4f618109b
Data: Send 1 BTC to Ivan
Hash: 000000d7b0c76e1001cdc1fc866b95a481d23f3027d86901eaeb77ae6d002b13
PoW: true

Prev. hash:
Data: Genesis Block
Hash: 000000edc4a82659cebf087adee1ea353bd57fcd59927662cd5ff1c4f618109b
PoW: true
```

## 결론

다음 파트에서는 주소, 지갑 그리고 트랜잭션을 구현할 것이다.
