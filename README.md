# Starter Application for Hyperledger Fabric 1.1

Create a network to jump start development of your decentralized application.

The network can be deployed to multiple docker containers on one host for development or to multiple hosts for testing 
or production.

Scripts of this starter generate crypto material and config files, start the network and deploy your chaincodes. 
Developers can use admin web app of 
[REST API server](https://github.com/Altoros/fabric-rest/tree/master/server/www-admin) 
to invoke and query chaincodes, explore blocks and transactions.

What's left is to develop your chaincodes and place them into the [chaincode](./chaincode) folder, 
and user interface as a single page web app that you can serve by by placing the sources into the [www](./www) folder. 
You can take web app code or follow patterns of the 
[admin app](https://github.com/Altoros/fabric-rest/tree/master/server/www-admin) to enroll users, 
invoke chaincodes and subscribe to events.

Most of the plumbing work is taken care of by this starter.

## Members and Components

Network consortium consists of:

- Orderer organization `example.com`
- Peer organization org1 `a` 
- Peer organization org2 `b` 
- Peer organization org3 `c`

They transact with each other on the following channels:

- `common` involving all members and with chaincode `reference` deployed
- bilateral confidential channels between pairs of members with chaincode `relationship` deployed to them
  - `a-b`
  - `a-c`
  - `b-c`

Both chaincodes are copies of [chaincode_example02](https://github.com/hyperledger/fabric/tree/release/examples/chaincode/go/chaincode_example02).
Replace these sources with your own.

Each organization starts several docker containers:

- **peer0** (ex.: `peer0.a.example.com`) with the anchor [peer](https://github.com/hyperledger/fabric/tree/release/peer) runtime
- **peer1** `peer1.a.example.com` with the secondary peer
- **ca** `ca.a.example.com` with certificate authority server [fabri-ca](https://github.com/hyperledger/fabric-ca)
- **api** `api.a.example.com` with [fabric-rest](https://github.com/Altoros/fabric-rest) API server
- **www** `www.a.example.com` with a simple http server to serve members' certificate files during artifacts generation and setup
- **cli** `cli.a.example.com` with tools to run commands during setup

## Local deployment

Deploy docker containers of all member organizations to one host, for development and testing of functionality. 

All containers refer to each other by their domain names and connect via the host's docker network. The only services 
that need to be available to the host machine are the `api` so you can connect to admin web apps of each member; 
thus their `4000` ports are mapped to non conflicting `4000, 4001, 4002` ports on the host.

Generate artifacts:
```bash
./network.sh -m generate
```

Generated crypto material of all members, block and tx files are placed in shared `artifacts` folder on the host.

Start docker containers of all members:
```bash
./network.sh -m up
```

After all containers are up, browse to each member's admin web app to transact on their behalf: 

- org1 [http://localhost:4000/admin](http://localhost:4000/admin)
- org2 [http://localhost:4001/admin](http://localhost:4001/admin)
- org3 [http://localhost:4002/admin](http://localhost:4002/admin)

Tail logs of each member's docker containers by passing its name as organization `-o` argument:
```bash
# orderer
./network.sh -m logs -m example.com

# members
./network.sh -m logs -m a
./network.sh -m logs -m b
```
Stop all:
```bash
./network.sh -m down
```
Remove dockers:
```bash
./network.sh -m clean
```

## Decentralized deployment

Deploy containers of each member to separate hosts connecting via internet.

Note the docker-compose files don't change much from the local deployment and containers still refer to each other by 
domain names `api.a.example.com`, `peer1.c.example.com` etc. However they can no longer discover each other within a local
docker network and need to resolve these names to real ips on the internet. We use `extra_hosts` setting in docker-compose 
files to map domain names to real ips which come as args to the script. Specify member hosts ip addresses 
in [network.sh](network.sh) file or by env variables:
```bash
export IP_ORDERER=54.235.3.243 IP1=54.235.3.231 IP2=54.235.3.232 IP3=54.235.3.233
```  

The setup process takes several steps whose order is important.

Each member generates artifacts on their respective hosts (can be done in parallel):
```bash
# organization a on their host
./network.sh -m generate-peer -o a

# organization b on their host
./network.sh -m generate-peer -o b

# organization c on their host
./network.sh -m generate-peer -o c
```

After certificates are generated each script starts a `www` docker instance to serve them to other members: the orderer
 will download the certs to create the ledger and other peers will download to use them to secure communication by TLS.  

Now the orderer can generate genesis block and channel tx files by collecting certs from members. On the orderer's host:
```bash
./network.sh -m generate-orderer
```

And start the orderer:
```bash
./network.sh -m up-orderer
```

When the orderer is up, each member can start services on their hosts and their peers connect to the orderer to create 
channels. Note that in Fabric one member creates a channel and others join to it via a channel block file. 
Thus channel _creator_ members make these block files available to _joiners_ via their `www` docker instances. 
Also note the starting order of members is important, especially for bilateral channels connecting pairs of members, 
for example for channel `a-b` member `a` needs to start first to create the channel and serve the block file, 
and then `b` starts, downloads the block file and joins the channel. It's a good idea to order organizations in script
arguments alphabetically, ex.: `ORG1=aorg ORG2=borg ORG3=corg` then the channels are named accordingly 
`aorg-borg aorg-corg borg-corg` and it's clear who creates, who joins a bilateral channel and who needs to start first.

Each member starts:
```bash
# organization a on their host
./network.sh -m up-1

# organization b on their host
./network.sh -m up-2

# organization c on their host
./network.sh -m up-3
```

## How it works

The script [network.sh](network.sh) uses substitution of values and names to create config files out of templates:

- [cryptogentemplate-orderer.yaml](artifacts/cryptogentemplate-orderer.yaml) 
and [cryptogentemplate-peer.yaml](artifacts/cryptogentemplate-peer.yaml) for `cryptogen.yaml` to drive 
[cryptogen](https://github.com/hyperledger/fabric/tree/release/common/tools/cryptogen) tool to generate members' crypto material: 
private keys and certificates
- [configtxtemplate.yaml](artifacts/configtxtemplate.yaml) for `configtx.yaml` with definitions of 
the consortium and channels to drive [configtx](https://github.com/hyperledger/fabric/tree/release/common/configtx) tool to generate 
genesis block file to start the orderer, and channel config transaction files to create channels
- [network-config-template.json](artifacts/network-config-template.json) for `network-config.json` file used by the 
API server and web apps to connect to the members' peers and ca servers
- [docker-composetemplate-orderer.yaml](ledger/docker-composetemplate-orderer.yaml) 
and [docker-composetemplate-peer.yaml](ledger/docker-composetemplate-peer.yaml) for `docker-compose.yaml` files for 
each member organization to start docker containers

During setup the same script uses `cli` docker containers to create and join channels, install and instantiate chaincodes.

And finally it starts members' services via the generated `docker-compose.yaml` files.

## Customize and extend

Customize domain and organization names by editing [network.sh](network.sh) file or by setting env variables. 
Note organization names are ordered alphabetically:

```bash
export DOMAIN=myapp.com ORG1=bar ORG2=baz ORG3=foo
```  

The topology of one `common` channel open to all members and bilateral ones is an example and a starting point: 
you can change channel members by editing [configtxtemplate.yaml](artifacts/configtxtemplate.yaml) to create wider 
channels, groups, triplets etc.

It's also relatively straightforward to extend the scripts from the preset `ORG1`, `ORG2` and `ORG3` to take an arbitrary 
number of organizations and figure out possible permutations of bilateral channels: see `iterateChannels` function in 
[network.sh](network.sh).

## Chaincode development

There are commands for working with chaincodes in `chaincode-dev` mode where a chaincode is not managed within its docker 
container but run separately as a stand alone executable or in a debugger. The peer does not manage the chaincode but 
connects to it to invoke and query.

The dev network is composed of a minimal set of peer, orderer and cli containers and uses pre-generated artifacts
checked into the source control. Channel and chaincodes names are `myc` and `mycc` and can be edited in `network.sh`.

Start containers for dev network:
```bash
./network.sh -m devup
./network.sh -m devinstall
```

Start your chaincode in a debugger with env variables:
```bash
CORE_CHAINCODE_LOGGING_LEVEL=debug
CORE_PEER_ADDRESS=0.0.0.0:7051
CORE_CHAINCODE_ID_NAME=mycc:0
```

Now you can instantiate, invoke and query your chaincode:
```bash
./network.sh -m devinstantiate
./network.sh -m devinvoke
./network.sh -m devquery
```

You'll be able to modify the source code, restart the chaincode, test with invokes without rebuilding or restarting 
the dev network. 

Finally:
```bash
./network.sh -m devdown
```

## Testing

[Caliper](https://github.com/hyperledger/caliper) is using as testing framework. Please find how to setup test [here](test/readme.md).

## Acknowledgements

This environment uses a very helpful [fabric-rest](https://github.com/Altoros/fabric-rest) API server developed separately and 
instantiated from its docker image.

The scripts are inspired by [first-network](https://github.com/hyperledger/fabric-samples/tree/release/first-network) and 
 [balance-transfer](https://github.com/hyperledger/fabric-samples/tree/release/balance-transfer) of Hyperledger Fabric samples.
 
 
 # MODIFICATIONS
 -e "s/WWW_PORT/$DEFAULT_WWW_PORT/g"  (line 167) ----------this is I have put again cuz port error 


uncomment all default assigments


export IP_ORDERER=130.147.175.222 IP1=130.147.175.222 IP2=130.147.175.150 IP3=130.147.175.150    on both system

Created 4 seperate folders for a & orderer on 222 and b & c on 150
USEEE GUAAKKKKEEEEEE for commands only
USE NORMAL TERMINAL for cat & nano commands cuzz 1000 line limit on guake :(

----b scp
 scp network.sh philips@130.147.175.150:/home/philips/fabric/Multinode-al/fabric-starter-b

---c scp
scp network.sh philips@130.147.175.150:/home/philips/fabric/Multinode-al/fabric-starter-c

-------------Steps to Run the multi node network

./network.sh -m generate-peer -o a -a 4000 -w 8081 -c 7054 -0 7051 -1 7053 -2 7056 -3 7058  
./network.sh -m generate-peer -o b -a 4001 -w 8082 -c 8054 -0 8051 -1 8053 -2 8056 -3 8058
./network.sh -m generate-peer -o c -a 4002 -w 8083 -c 9054 -0 9051 -1 9053 -2 9056 -3 9058

./network.sh -m generate-orderer

./network.sh -m up-orderer

./network.sh -m up-1 
./network.sh -m up-2 
./network.sh -m up-3 

// oderer port is 7050 and orderer www port is 8080

------Modifications
----------------------function downloadartifactsorderer for loop #line 743

"for ORG in ${ORG1} ${ORG2} ${ORG3};
 do 
 if [ $ORG = $ORG1 ] then DEFAULT_WWW_PORT=AWWWPORT elif [ $ORG == $ORG2 ] then DEFAULT_WWW_PORT=BWWWPORT elif [ $ORG == $ORG3 ] then DEFAULT_WWW_PORT=CWWWPORT fi 
 wget ${WGET_OPTS} --directory-prefix crypto-config/peerOrganizations/\${ORG}.$DOMAIN/peers/$one_peer.\${ORG}.$DOMAIN/tls http://www.\${ORG}.$DOMAIN:$DEFAULT_WWW_PORT/crypto-config/peerOrganizations/\${ORG}.$DOMAIN/peers/$one_peer.\${ORG}.$DOMAIN/tls/ca.crt; 
 done"

wget ${WGET_OPTS} --directory-prefix crypto-config/peerOrganizations/a.example.com/peers/$one_peer.\${ORG}.$DOMAIN/tls http://www.\${ORG}.$DOMAIN:$DEFAULT_WWW_PORT/crypto-config/peerOrganizations/\${ORG}.$DOMAIN/peers/$one_peer.\${ORG}.$DOMAIN/tls/ca.crt;
wget ${WGET_OPTS} --directory-prefix crypto-config/peerOrganizations/b.example.com/peers/$one_peer.\${ORG}.$DOMAIN/tls http://www.\${ORG}.$DOMAIN:$DEFAULT_WWW_PORT/crypto-config/peerOrganizations/\${ORG}.$DOMAIN/peers/$one_peer.\${ORG}.$DOMAIN/tls/ca.crt;
wget ${WGET_OPTS} --directory-prefix crypto-config/peerOrganizations/c.$DOMAIN/peers/$one_peer.\${ORG}.$DOMAIN/tls http://www.\${ORG}.$DOMAIN:$DEFAULT_WWW_PORT/crypto-config/peerOrganizations/\${ORG}.$DOMAIN/peers/$one_peer.\${ORG}.$DOMAIN/tls/ca.crt;


: ${OGa:="a"}
: ${ORG1:="a" }
: ${ORG2:="b" }
if [ $ORG1 == $OGa ]
then ORG1=ORG2 && echo $ORG2
else echo "222"
fi



for one_peer in 'peer0' 'peer1'; do
          c="for ORG in ${ORG1} ${ORG2} ${ORG3}; 
          do if [ $ORG = $ORG1 ] then DEFAULT_WWW_PORT=AWWWPORT elif [ $ORG == $ORG2 ] then DEFAULT_WWW_PORT=BWWWPORT elif [ $ORG == $ORG3 ] then DEFAULT_WWW_PORT=CWWWPORT fi 
          wget ${WGET_OPTS} --directory-prefix crypto-config/peerOrganizations/\${ORG}.$DOMAIN/peers/$one_peer.\${ORG}.$DOMAIN/tls http://www.\${ORG}.$DOMAIN:$DEFAULT_WWW_PORT/crypto-config/peerOrganizations/\${ORG}.$DOMAIN/peers/$one_peer.\${ORG}.$DOMAIN/tls/ca.crt; done"
          echo ${c}
          f="$GENERATED_DOCKER_COMPOSE_FOLDER/docker-compose-$DOMAIN.yaml"
          docker-compose --file ${f} run --rm "cli.$DOMAIN" bash -c "${c} && chown -R $UID:$GID ."
      done

####  if else not working bcuz string is passed as varriable to bash in the container hence hard coded different ports to different string variable

wget ${WGET_OPTS} --directory-prefix crypto-config/peerOrganizations/a.example.com/peers/peer0.a.example.com/tls http://www.a.example.com:8081/crypto-config/peerOrganizations/a.example.com/peers/peer0.a.example.com/tls/ca.crt && wget ${WGET_OPTS} --directory-prefix crypto-config/peerOrganizations/a.example.com/peers/peer1.a.example.com/tls http://www.a.example.com:8081/crypto-config/peerOrganizations/a.example.com/peers/peer1.a.example.com/tls/ca.crt

wget ${WGET_OPTS} --directory-prefix crypto-config/peerOrganizations/b.example.com/peers/peer0.b.example.com/tls http://www.b.example.com:8082/crypto-config/peerOrganizations/b.example.com/peers/peer0.b.example.com/tls/ca.crt && wget ${WGET_OPTS} --directory-prefix crypto-config/peerOrganizations/b.example.com/peers/peer1.b.example.com/tls http://www.b.example.com:8082/crypto-config/peerOrganizations/b.example.com/peers/peer1.b.example.com/tls/ca.crt

wget ${WGET_OPTS} --directory-prefix crypto-config/peerOrganizations/c.example.com/peers/peer0.c.example.com/tls http://www.c.example.com:8083/crypto-config/peerOrganizations/c.example.com/peers/peer0.c.example.com/tls/ca.crt && wget ${WGET_OPTS} --directory-prefix crypto-config/peerOrganizations/c.example.com/peers/peer1.c.example.com/tls http://www.c.example.com:8083/crypto-config/peerOrganizations/c.example.com/peers/peer1.c.example.com/tls/ca.crt


------------function  downloadmember msp  #line 624

wget --verbose -N --directory-prefix crypto-config/peerOrganizations/a.example.com/peers/peer0.a.example.com/tls http://www.a.example.com:8081/crypto-config/peerOrganizations/a.example.com/peers/peer0.a.example.com/tls/ca.crt
$OGa="a"
$OGb="b"
$OGc="c"
if [ $ORG = $ORG1 ] then DEFAULT_WWW_PORT=AWWWPORT elif [ $ORG == $ORG2 ] then DEFAULT_WWW_PORT=BWWWPORT elif [ $ORG == $ORG3 ] then DEFAULT_WWW_PORT=CWWWPORT fi


for ORG in ${@}; 
do 
if [ $ORG = $ORG1 ] then DEFAULT_WWW_PORT=AWWWPORT elif [ $ORG == $ORG2 ] then DEFAULT_WWW_PORT=BWWWPORT elif [ $ORG == $ORG3 ] then DEFAULT_WWW_PORT=CWWWPORT fi 
wget ${WGET_OPTS} --directory-prefix crypto-config/peerOrganizations/\$ORG.$DOMAIN/msp/admincerts http://www.\$ORG.$DOMAIN:$DEFAULT_WWW_PORT/crypto-config/peerOrganizations/\$ORG.$DOMAIN/msp/admincerts/Admin@\$ORG.$DOMAIN-cert.pem 
&& wget ${WGET_OPTS} --directory-prefix crypto-config/peerOrganizations/\$ORG.$DOMAIN/msp/cacerts http://www.\$ORG.$DOMAIN:$DEFAULT_WWW_PORT/crypto-config/peerOrganizations/\$ORG.$DOMAIN/msp/cacerts/ca.\$ORG.$DOMAIN-cert.pem 
&& wget ${WGET_OPTS} --directory-prefix crypto-config/peerOrganizations/\$ORG.$DOMAIN/msp/tlscacerts http://www.\$ORG.$DOMAIN:$DEFAULT_WWW_PORT/crypto-config/peerOrganizations/\$ORG.$DOMAIN/msp/tlscacerts/tlsca.\$ORG.$DOMAIN-cert.pem
; 
done


wget ${WGET_OPTS} --directory-prefix crypto-config/peerOrganizations/a.example.com/msp/admincerts http://www.a.example.com:8081/crypto-config/peerOrganizations/a.example.com/msp/admincerts/Admin@a.example.com-cert.pem 
&& wget ${WGET_OPTS} --directory-prefix crypto-config/peerOrganizations/a.example.com/msp/cacerts http://www.a.example.com:8081/crypto-config/peerOrganizations/a.example.com/msp/cacerts/ca.a.example.com-cert.pem 
&& wget ${WGET_OPTS} --directory-prefix crypto-config/peerOrganizations/a.example.com/msp/tlscacerts http://www.a.example.com:8081/crypto-config/peerOrganizations/a.example.com/msp/tlscacerts/tlsca.a.example.com-cert.pem

&& wget ${WGET_OPTS} --directory-prefix crypto-config/peerOrganizations/b.example.com/msp/admincerts http://www.b.example.com:8082/crypto-config/peerOrganizations/b.example.com/msp/admincerts/Admin@b.example.com-cert.pem 
&& wget ${WGET_OPTS} --directory-prefix crypto-config/peerOrganizations/b.example.com/msp/cacerts http://www.b.example.com:8082/crypto-config/peerOrganizations/b.example.com/msp/cacerts/ca.b.example.com-cert.pem 
&& wget ${WGET_OPTS} --directory-prefix crypto-config/peerOrganizations/b.example.com/msp/tlscacerts http://www.b.example.com:8082/crypto-config/peerOrganizations/b.example.com/msp/tlscacerts/tlsca.b.example.com-cert.pem

&& wget ${WGET_OPTS} --directory-prefix crypto-config/peerOrganizations/c.example.com/msp/admincerts http://www.c.example.com:8083/crypto-config/peerOrganizations/c.example.com/msp/admincerts/Admin@c.example.com-cert.pem 
&& wget ${WGET_OPTS} --directory-prefix crypto-config/peerOrganizations/c.example.com/msp/cacerts http://www.c.example.com:8083/crypto-config/peerOrganizations/c.example.com/msp/cacerts/ca.c.example.com-cert.pem 
&& wget ${WGET_OPTS} --directory-prefix crypto-config/peerOrganizations/c.example.com/msp/tlscacerts http://www.c.example.com:8083/crypto-config/peerOrganizations/c.example.com/msp/tlscacerts/tlsca.c.example.com-cert.pem

wget ${WGET_OPTS} --directory-prefix crypto-config/peerOrganizations/a.example.com/msp/admincerts http://www.a.example.com:8081/crypto-config/peerOrganizations/a.example.com/msp/admincerts/Admin@a.example.com-cert.pem && wget ${WGET_OPTS} --directory-prefix crypto-config/peerOrganizations/a.example.com/msp/cacerts http://www.a.example.com:8081/crypto-config/peerOrganizations/a.example.com/msp/cacerts/ca.a.example.com-cert.pem && wget ${WGET_OPTS} --directory-prefix crypto-config/peerOrganizations/a.example.com/msp/tlscacerts http://www.a.example.com:8081/crypto-config/peerOrganizations/a.example.com/msp/tlscacerts/tlsca.a.example.com-cert.pem && wget ${WGET_OPTS} --directory-prefix crypto-config/peerOrganizations/b.example.com/msp/admincerts http://www.b.example.com:8082/crypto-config/peerOrganizations/b.example.com/msp/admincerts/Admin@b.example.com-cert.pem && wget ${WGET_OPTS} --directory-prefix crypto-config/peerOrganizations/b.example.com/msp/cacerts http://www.b.example.com:8082/crypto-config/peerOrganizations/b.example.com/msp/cacerts/ca.b.example.com-cert.pem && wget ${WGET_OPTS} --directory-prefix crypto-config/peerOrganizations/b.example.com/msp/tlscacerts http://www.b.example.com:8082/crypto-config/peerOrganizations/b.example.com/msp/tlscacerts/tlsca.b.example.com-cert.pem && wget ${WGET_OPTS} --directory-prefix crypto-config/peerOrganizations/c.example.com/msp/admincerts http://www.c.example.com:8083/crypto-config/peerOrganizations/c.example.com/msp/admincerts/Admin@c.example.com-cert.pem && wget ${WGET_OPTS} --directory-prefix crypto-config/peerOrganizations/c.example.com/msp/cacerts http://www.c.example.com:8083/crypto-config/peerOrganizations/c.example.com/msp/cacerts/ca.c.example.com-cert.pem && wget ${WGET_OPTS} --directory-prefix crypto-config/peerOrganizations/c.example.com/msp/tlscacerts http://www.c.example.com:8083/crypto-config/peerOrganizations/c.example.com/msp/tlscacerts/tlsca.c.example.com-cert.pem



ports sorting for orderer up & peer up commands-----------------------------

    orderer up no issues ---------------

    ----------------Peer Up -----------------
    
    downloadArtifactsMember function             #line 782
        function downloadChannelTxFiles          #line 662       ---
                                                                    }
        function downloadNetworkConfig           #line 649          }---->  from DEFAULT_WWW_PORT to  8080 (orderer's www port ) to download these files from orderer
                                                                    }
        function downloading orderer cert file   #line 714       ---

        downloading member cert files  #TODO download not from all members but from the orderer  #line 721
                from individual orgs cypto files to orderers crypto files cuz of that comment (http://orderer.example.com:7050) 7050 is orderer port
                    doesnt work ---^
                    so hard coded downloading certs from members itself---v
                                                                          v
                                                                          v                                                                    
            wget ${WGET_OPTS} --directory-prefix crypto-config/peerOrganizations/a.example.com/peers/peer0.a.example.com/tls http://orderer.example.com:7050/crypto-config/peerOrganizations/a.example.com/peers/peer0.a.example.com/tls/ca.crt && wget ${WGET_OPTS} --directory-prefix crypto-config/peerOrganizations/a.example.com/peers/peer1.a.example.com/tls http://orderer.example.com:7050/crypto-config/peerOrganizations/a.example.com/peers/peer1.a.example.com/tls/ca.crt
            wget ${WGET_OPTS} --directory-prefix crypto-config/peerOrganizations/b.example.com/peers/peer0.b.example.com/tls http://orderer.example.com:7050/crypto-config/peerOrganizations/b.example.com/peers/peer0.b.example.com/tls/ca.crt && wget ${WGET_OPTS} --directory-prefix crypto-config/peerOrganizations/b.example.com/peers/peer1.b.example.com/tls http://orderer.example.com:7050/crypto-config/peerOrganizations/b.example.com/peers/peer1.b.example.com/tls/ca.crt
            wget ${WGET_OPTS} --directory-prefix crypto-config/peerOrganizations/c.example.com/peers/peer0.c.example.com/tls http://orderer.example.com:7050/crypto-config/peerOrganizations/c.example.com/peers/peer0.c.example.com/tls/ca.crt && wget ${WGET_OPTS} --directory-prefix crypto-config/peerOrganizations/c.example.com/peers/peer1.c.example.com/tls http://orderer.example.com:7050/crypto-config/peerOrganizations/c.example.com/peers/peer1.c.example.com/tls/ca.crt
 


info -- (installALL function & instantiateChaincode function) chaincode installation is done through peer0 of org1 port 7051 always (hard coded by altoros)

x509 error
        uncommented remove artifacts from generate-peer and generate-orderer and all rm commands in remove artifacts function
        there were existing certs in orderer folder hence mismatch of certs 




