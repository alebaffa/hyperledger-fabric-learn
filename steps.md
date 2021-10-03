## List of steps of a basic chaincode lifecycle on Hyperledger Fabric
### Environment preparation
Open two terminals on `/test-network` directory and run the following scripts to add all the env variables to setup the Organizations and the paths to the Fabric /config and /bin:
- from a terminal run `./network.sh up createChannel -ca -s couchdb`
- from one terminal run `source ./addOrg1.sh`
- from the second terminal run `source ./addOrg2.sh`

### Package the chaincode
This assumes that the *simple_chaincode* source is inside /test-network/chaincode. 
From Org1 terminal run 
- `peer lifecycle chaincode package simple_chaincode.tar.gz --path ../chaincode/simple_chaincode/ --lang node --label simple_chaincode_1.0` 
 
this will create a tar.gz in the /test-network directory that is ready to be installed in the peers. No need to run it in Org2 terminal.

### Install the chaincode in the peers
From both Org1 and Org2 terminal run:
- `peer lifecycle chaincode install simple_chaincode.tar.gz` 
This will install the chaincode into Org1 and Org2 and return its hash code needed for the following steps.

You can check the installation in both peers with:
- `peer lifecycle chaincode queryinstalled`

### Approve the chaincode in both peers
To simplify the approval step, it is better to save the hash code returned before in an env variable: 

- `export PACKAGE_ID=simple_chaincode_1.0:4cee77dfe4b8965e3865d72fea6c0893d9589fb317c543e2f8fe8a13979277cd`

Now run the following command in both Org1 and Org2:
- `peer lifecycle chaincode approveformyorg -o localhost:7050 --ordererTLSHostnameOverride orderer.example.com --channelID mychannel --name simple_chaincode --version 1.0 --package-id $PACKAGE_ID --sequence 1 --tls --cafile ${PWD}/organizations/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem` 

(Note: `--ordererTLSHostnameOverride` is needed only because if use Docker in test-network)

### Commit the chaincode in the channel
Check the commit readiness of the chaincode:
- `peer lifecycle chaincode checkcommitreadiness --channelID mychannel --name simple_chaincode --version 1.0 --sequence 1 --tls --cafile ${PWD}/organizations/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem --output json`

It should return `true` for both Org1 and Org2. If not, check the previous step.

Now commit the chaincode:

- `peer lifecycle chaincode commit -o localhost:7050 --ordererTLSHostnameOverride orderer.example.com --channelID mychannel --name simple_chaincode --version 1.0 --sequence 1 --tls --cafile ${PWD}/organizations/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem --peerAddresses localhost:7051 --tlsRootCertFiles ${PWD}/organizations/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt --peerAddresses localhost:9051 --tlsRootCertFiles ${PWD}/organizations/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt`

You can check the committed chaincode:
- `peer lifecycle chaincode querycommitted -C mychannel -n simple_chaincode`

### Invoke the chaincode
Put a value in the ledger using the `put` function implemented in the chaincode:
- `peer chaincode invoke -o localhost:7050 --ordererTLSHostnameOverride orderer.example.com --tls --cafile ${PWD}/organizations/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem -C mychannel -n simple_chaincode --peerAddresses localhost:7051 --tlsRootCertFiles ${PWD}/organizations/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt --peerAddresses localhost:9051 --tlsRootCertFiles ${PWD}/organizations/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt -c '{"function":"put","Args":["string", "a", "b"]}'`
The TLS are required since the peers have been created with the CA.

### Query the chaincode
Get the value just put from the chaincode:
- `peer chaincode query -C mychannel -n simple_chaincode -c '{"function":"get","Args":["string", "a"]}'`