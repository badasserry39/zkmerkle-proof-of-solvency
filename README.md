# zkmerkle-proof-of-solvency
## Circuit Design

See the [technical blog](https://gusty-radon-13b.notion.site/Proof-of-solvency-61414c3f7c1e46c5baec32b9491b2b3d) for more details about background and circuit design
## How to run

### Run third-party services
This project needs following third party services:
- mysql: used to store `witness`, `userproof`, `proof` table;
- redis: provide distributed lock for multi provers;
- kvrocks: used to store account tree

We can use docker to run these services:

```shell
docker run -d --name zkpos-redis -p 6379:6379 redis

docker run -d --name zkpos-mysql -p 3306:3306  -v /server/docker_data/mysql_data:/var/lib/mysql -e MYSQL_USER=zkpos -e MYSQL_PASSWORD=zkpos@123 -e MYSQL_DATABASE=zkpos  -e MYSQL_ROOT_PASSWORD=zkpos@123 mysql

docker run -d --name zkpos-kvrocks -p 6666:6666 -v /server/docker_data/kvrocksdata:/var/lib/kvrocks apache/kvrocks
```

where `/server/docker_data/` is directory in the host machine which is used to persist mysql and kvrocks docker data.


### Generate zk keys

The `keygen` service is for generating zk related keys which are used to generate and verify zk proof. The `BatchCreateUserOpsCounts` constant variable in utils package represent how many users can be created in one batch.
The larger the `BatchCreateUserOpsCounts` is, the longer it will take to generate zk-related keys and generate zk proof.

We choose `864` as the `BatchCreateUserOpsCounts` variable. It will take about 6 hours to generate zk-related keys, and 105 seconds to generate zk proof for one batch in a 128GB memory and 32 core virtual machine.

Run the following commands to start `keygen` service:
```
// make sure the BatchCreateUserOpsCounts in utils/constants.go is expected

cd src/keygen; go run main.go
```

After `keygen` service finishes running, there will be several key files generated in the current directory, like the following:
```shell
Total 26G
 -rww-r-- 1 ec2-user ec2-user  69M 12月 26 07:49 zkpor864.ccs.ct.save
-rw-rw-r-- 1 ec2-user ec2-user  16G 12月 26 07:51 zkpor864.ccs.save
-rw-rw-r-- 1 ec2-user ec2-user 1.8G 12月 26 07:51 zkpor864.pk.A.save
-rw-rw-r-- 1 ec2-user ec2-user 1.6G 12月 26 07:51 zkpor864.pk.B1.save
-rw-rw-r-- 1 ec2-use r ec2-user 3.2G 12月 26 07:51 zkpor864.pk.B2.save
-rw-rw-r-- 1 -rw-rw-r-- 1 ec2-user ec2-user  708 12月 26 07:52 zkpor864.vk.
-rw-rw-r-- 1 ec2-user ec2-user 2.1G 12月 26 07:52 The `witness` service is used to generate witness for `prover` service. 

`witness/config/config.json` is the config file `witness` service use. The sample file is as follows:
```json
{
  "MysqlDataSource" : "zkpos:zkpos@123@tcp(127.0.0.1:3306)/zkpos?parseTime=true",
  "UserDataFile": "/server/data/20230118",
  "DbSuffix": "0",
  "TreeDB": {
    "Driver": "redis",
    "Option": {
      "Addr": "127.0.0.1:6666"
    }
  }
}
```

Where

- `MysqlDataSource`: this is the mysql config;
- `UserDataFile`: the directory which contains all users balance sheet files;
- `DbSuffix`: this suffix will be appended to the ending of table name, such as `proof0`, `witness0` table;
- `TreeDB`:
  - `Driver`: `redis` means account tree use kvrocks as its storage engine;
  - `Option`:
    - `Addr`: `kvrocks` service listen address


Run the following command to start `witness` service:
```shell
cd witness; go run main.go
```

The `witness` service supports recovery from unexpected crash. After `witness` service finish running, we can see `witness` from `witness` table.

The performance: about 850 users witness generation per second in a 128GB memory and 32 core virtual machine.

### Generate zk proof

The `prover` service is used to generate zk proof and supports running in parallel. It reads witness from `witness` table generated by `witness` service.

`prover/config/config.json` is the config file `prover` service uses. The sample file is as follows:
```json
{
  "MysqlDataSource" : "zkpos:zkpos@123@tcp(127.0.0.1:3306)/zkpos?parseTime=true",
  "DbSuffix": "0",
  "Redis": {
    "Host": "127.0.0.1:6379",
    "Type": "node"
  },
  "ZkKeyName": "/server/zkmerkle-proof-of-solvency/src/keygen/zkpor864"
}
```

Where

- `MysqlDataSource`: this is the mysql config;
- `DbSuffix`: this suffix will be appended to the ending of table name, such as `proof0`, `witness0` table;
- `Redis`:
  - `Host`: `redis` service listen addr;
  - `Type`: only support `node` type
- `ZkKeyName`: the key name generated by `keygen` service

Run the following command to start `prover` service:
```shell
cd prover; go run main.go
```

To run `prover` service in parallel, just repeat executing above commands.

**Note: After all prover service finishes running, We should use `go run main.go -rerun` command to regenerate proof for unfinished batch**

After the whole `prover` service finished, we can see batch zk proof in `proof` table.

### Generate user proof

The `userproof` service is used to generate and persist user merkle proof. It uses `userproof/config/config.json` as config file, and the sample config is as follows:
```json
{
  "MysqlDataSource" : "zkpos:zkpos@123@tcp(127.0.0.1:3306)/zkpos?parseTime=true",
  "UserDataFile": "/server/data/20230118",
  "DbSuffix": "0",
  "TreeDB": {
    "Driver": "redis",
    "Option": {
      "Addr": "127.0.0.1:6666"
    }
  }
}
```

Where

- `MysqlDataSource`: this is the mysql config;
- `UserDataFile`: the directory which contains all users balance sheet files;
- `DbSuffix`: this suffix will be appended to the ending of table name, such as `proof0`, `witness0` table;
- `TreeDB`:
  - `Driver`: `redis` means account tree use kvrocks as its storage engine;
  - `Option`:
    - `Addr`: `kvrocks` service listen address

Run the following command to run `userproof` service:
```shell
cd userproof; go run main.go
```

After `userproof` service finishes running, we can see every user proof from `userproof` table.

The performance: about 10k users proof generation per second in a 128GB memory and 32 core virtual machine.

### Verifier

The `verifier` service is used to verify batch proof and single user proof.

#### Verify batch proof
The service use `config.json` as its config file, and the sample config is as follows:
```json
{
  "ProofTable": "config/proof.csv",
  "ZkKeyName": "config/zkpor864",
  "CexAssetsInfo": [{"TotalEquity":219971568487,"TotalDebt":9789219,"BasePrice":24620000000},{"TotalEquity":8664493444,"TotalDebt":122580,"BasePrice":1682628000000},{"TotalEquity":67463930749983,"TotalDebt":16127314913,"BasePrice":100000000},{"TotalEquity":68358645578,"TotalDebt":130187,"BasePrice":121377000000},{"TotalEquity":590353015932,"TotalDebt":0,"BasePrice":598900000},{"TotalEquity":255845425858,"TotalDebt":13839361,"BasePrice":6541000000},{"TotalEquity":0,"TotalDebt":0,"BasePrice":99991478},{"TotalEquity":267958065914051,"TotalDebt":501899265949,"BasePrice":100000000},{"TotalEquity":124934670143615,"TotalDebt":1422964747,"BasePrice":34500000}]
}
```
Where
- `ProofTable`: this is proof csv file which can be exported by `proof` table;
- `ZkKeyName`: the key name generated by `keygen` service;
- `CexAssetsInfo`: this is published by CEX, it represents CEX's liability;

You can get `CexAssetsInfo` using `dbtool` command after `witness` service run finished. Run the following command to verify batch proof:
```shell
cd verifier; go run main.go
```

#### Verify user proof
The service use `user_config.json` as its config file, and the sample config is as follows:
```json
{
  "AccountIndex": 9,
  "AccountIdHash": "0000041cb7323211d0b356c2fe6e79fdaf0c27d74b3bb1a4635942f9ae92145b",
  "Root": "29591ef3a9ed02605edd6ab14f5dd49e7dbe0d03e72a27383f929ef3efb7514f",
  "Assets": [{"Index":7,"Equity":123456000,"Debt":0}],
  "Proof": ["DrPpFsm4/5HntRTf8M3dbgpdrxq3Q8lZkB2ngysW2js=","G1WgD/CvmGApQgmIX0rE0BlSifkw6IfNwY9z2DnRazM=","HZm8N563lDMrx//oMjejlXKLzLrZQRqKZKyXwpDnT+I=","A+wqmnw0NfdgAyEieZRqKlczF48VOROxME36YBwLhmY=","BaLA62VkWlLE/FZTxnvx/lwU7WNfDxgdM2cBw27MAog=","EvkbbZF7Y/W8AxPc+49lmzzFNdyPl1QWRu1ZQB/gmz8=","EJzsNkrzgcB6LIctzoqWAetzLHO57vx00FxIlLKhwqg=","D3kGOz1Xcsi5hGXr02LOSC23L+lCOhq6FlXGiPYcYig=","DSAmcCy6GEl1DlRNpP05e4I9ScMCz/4DtJCKpserVL8=","L16LdkqEmjI/4uVNETm4ScZ5uA/Cho59zjbgN1OgI+k=","CaRuyDB3b3JeZlItZw8Txi7MSP5vyRpco0OtHRuO6RQ=","CkfKeaCV6rLcskjWE91vCAxTPKEymsDtJvc4r8aWytw=","JfAF+uLGIBAJklZrWRPjxmJ3lFHyau9oNL5dOwQ72kQ=","JKQm66yeIFuPJcm5OkTUDyohsL5CWFF+fFpW/NiS2O0=","KXs+zCQELaQRJzQoCnYm+G2lr5eKLqmWd3G0xeSPWgc=","AFL9l/1p5puhKsCtS7WTT/hExtl3hmRfXnTofVY9+3s=","A2lctbb1mGsAgbERpnu4/RKX/OkcU06dhU9l4wS7e8k=","AIB1T+2rEytbSyMLfyp1rNhS0RCmPYwITdjhb/34FVQ=","GQmXAAvuoPUvt4zDrQ9qaAcRV5t8TPFXAIAIuwv0vr0=","J8pP9A0l54qj9UfumcTXn3hVBYEh+iBH03newr296io=","LD/99JpY8ZZupXaoRt6p6wKDmQepQyVMsJ3s5rX+Q9g=","EOqvnEUbpnzF0HeEv0PdB/R3KMbNU15jYqwNaBJ/j7M=","KLZwBqvYBNFPBFVPQQOipuYd9bYdLEme+Y5UiDRV5Z8=","IXgr/ZWOF/rduJvkSjOABCUtj5C1+tXjHJ/AwR+f7MM=","MFzAvInNDNdT4+6kp2YKAUT4+sb8fdrha9BxXnwHR1Q=","DdvOYaMZDO/di7mBvQdEocS2CE8BH2bGmIDuSGBaHa8=","MEa67EULCikf5rusfcsUb/kWUfx7NPpHjzHKEo3imho=","KLRU4kqn5Fd3FtjapW9MTJBgWdMtAGKgt3OVLQNHh14="],
  "TotalEquity": 123456000,
  "TotalDebt": 0
}
```

Where

- `AccountIndex`: account index used to verify;
- `AccountIdHash`: account hash id which contains user info
- `Root`: account tree root published by cex;
- `Assets`: all user assets info;
- `Proof`: user merkle proof which uses `base64` encoding;
- `TotalEquity`: user total equity which is calculated by all the assets equity multipy its corresponding price
- `TotalDebt`: user total debt which is calculated by all the assets debt multipy its corresponding price

Run the following command to verify single user proof:
```shell
cd verifier; go run main.go -user
```

### dbtool command

Run the following command to remove only kvrocks data:
```shell
cd src/dbtool; go run main.go -only_delete_kvrocks
```

Run the following command to delete kvrocks data and mysql:
```shell
cd src/dbtool; go run main.go -delete_all
```

Run the following command to get cex assets info in json format:
```shell
cd src/dbtool; go run main.go -query_cex_assets
```

### Check data correctness

#### check account tree construct correctness
`userproof` service provides a command flag `-memory_tree` which can construct account tree in memory
using user balance sheet.

Run the following command:
```shell
cd userproof; go run main.go -memory_tree
```

Compare the account tree root in the output log with the account tree root by `witness` service, if matches, then the account tree is correctly constructed.

**Note: when `userproof` service runs in the `-memory_tree` mode, its performance is about 75k per minute, so 3000w accounts will take about ~7 hours**

#### check mysql table consistency

Suppose the number of users is `accountsNum`, and `batchNumber` is 864, so the number of rows in `witness` and `proof` table is: `(accountNum + 864 - 1) / 864`. And the number of rows in `userproof` table equals to `accountsNum`.

When all services run finished, Check that the number of rows in table is as expected. 