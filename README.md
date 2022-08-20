## Kubernetes con Hyperledger Fabric


## Lanzar Kubernetes Cluster

Para empezar a desplegar nuestra red Fabric tenemos que tener un cluster de Kubernetes. Para ello vamos a utilizar KinD.

```bash
kind create cluster
```

## Instalar operador de Kubernetes

En este paso vamos a instalar el operador de kubernetes para Fabric, esto instalara:
- CRD (Custom resource definitions) para desplegar Peers, Orderers y Autoridades de certification Fabric
- Desplegara el programa para desplegar los nodos en Kubernetes

Para instalar helm: [https://helm.sh/es/docs/intro/install/](https://helm.sh/es/docs/intro/install/)

```bash
helm repo add kfs https://kfsoftware.github.io/hlf-helm-charts --force-update 

helm install hlf-operator --version=1.7.0 kfs/hlf-operator
```


### Instalar plugin de Kubectl

Para instalar el plugin de kubectl, hay que instalar primero Krew:
[https://krew.sigs.k8s.io/docs/user-guide/setup/install/](https://krew.sigs.k8s.io/docs/user-guide/setup/install/)


Despues, se podra instalar el plugin con la siguiente instruccion:
```bash
kubectl krew install hlf
```


## Desplegar una organizacion `Peer`

### Variables de entorno

```bash
export PEER_IMAGE=hyperledger/fabric-peer
export PEER_VERSION=2.4.3

export ORDERER_IMAGE=hyperledger/fabric-orderer
export ORDERER_VERSION=2.4.3

```


### Desplegar una autoridad de certificacion

```bash
kubectl hlf ca create --storage-class=standard --capacity=1Gi --name=org1-ca \
    --enroll-id=enroll --enroll-pw=enrollpw
kubectl wait --timeout=180s --for=condition=Running fabriccas.hlf.kungfusoftware.es --all
```

Registrar un usuario en la autoridad certificacion de la organizacion peer (Org1MSP)

```bash
# registrar usuario en la CA para los peers
kubectl hlf ca register --name=org1-ca --user=peer --secret=peerpw --type=peer \
 --enroll-id enroll --enroll-secret=enrollpw --mspid Org1MSP
```


### Desplegar un peer

```bash
kubectl hlf peer create --statedb=couchdb --image=$PEER_IMAGE --version=$PEER_VERSION --storage-class=standard --enroll-id=peer --mspid=Org1MSP \
        --enroll-pw=peerpw --capacity=5Gi --name=org1-peer0 --ca-name=org1-ca.default
kubectl wait --timeout=180s --for=condition=Running fabricpeers.hlf.kungfusoftware.es --all
```


## Desplegar una organizacion `Orderer`

Para desplegar una organizacion `Orderer` tenemos que:

1. Crear una autoridad de certificacion
2. Registrar el usuario `orderer` con password `ordererpw`
3. Crear orderer

### Crear la autoridad de certificacion
```bash
kubectl hlf ca create --storage-class=standard --capacity=1Gi --name=ord-ca \
    --enroll-id=enroll --enroll-pw=enrollpw
kubectl wait --timeout=180s --for=condition=Running fabriccas.hlf.kungfusoftware.es --all

```

### Registrar el usuario `orderer`

```bash
kubectl hlf ca register --name=ord-ca --user=orderer --secret=ordererpw \
    --type=orderer --enroll-id enroll --enroll-secret=enrollpw --mspid=OrdererMSP
```

### Desplegar orderer

```bash
kubectl hlf ordnode create --image=$ORDERER_IMAGE --version=$ORDERER_VERSION \
    --storage-class=standard --enroll-id=orderer --mspid=OrdererMSP \
    --enroll-pw=ordererpw --capacity=2Gi --name=ord-node1 --ca-name=ord-ca.default

kubectl wait --timeout=180s --for=condition=Running fabricorderernodes.hlf.kungfusoftware.es --all
```

Comprobar que el orderer esta ejecutandose:
```bash
kubectl get pods
```

Ver si el pod que empieza por `ord-node1` esta ejecutandose.


## Preparar cadena de conexion para interactuar con el orderer

Para preparar la cadena de conexion, tenemos que:

- Obtener la cadena de conexion sin usuarios
- Registrar un usuario en la autoridad de certificacion para firma
- Obtener los certificados utilizando el usuario creado anteriormente
- Adjuntar el usuario a la cadena de conexion

1. Obtener la cadena de conexion sin usuarios
```bash
kubectl hlf inspect --output ordservice.yaml -o OrdererMSP
```

2. Registrar un usuario en la autoridad de certificacion TLS

```bash
kubectl hlf ca register --name=ord-ca --user=admin --secret=adminpw \
    --type=admin --enroll-id enroll --enroll-secret=enrollpw --mspid=OrdererMSP
```

3. Obtener los certificados utilizando el certificado

```bash
kubectl hlf ca enroll --name=ord-ca --user=admin --secret=adminpw --mspid OrdererMSP \
        --ca-name ca  --output admin-ordservice.yaml 
```

4. Adjuntar el usuario a la cadena de conexion
```
kubectl hlf utils adduser --userPath=admin-ordservice.yaml --config=ordservice.yaml --username=admin --mspid=OrdererMSP
```

## Crear el canal

Para crear un canal tenemos que:
1. Generar el bloque genesis del canal
2. Obtener el certificado para la autoridad de certificacion TLS
3. Unir los orderers al canal

1. Generar el bloque genesis del canal
```bash
kubectl hlf channel generate --output=demo.block --name=demo --organizations Org1MSP --ordererOrganizations OrdererMSP
```

2. Obtener el certificado para la autoridad de certificacion TLS
```bash
# enroll using the TLS CA
kubectl hlf ca enroll --name=ord-ca --namespace=default --user=admin --secret=adminpw --mspid OrdererMSP \
        --ca-name tlsca  --output admin-tls-ordservice.yaml 
```

3. Unir los orderers al canal
```bash
kubectl hlf ordnode join --block=demo.block --name=ord-node1 --namespace=default --identity=admin-tls-ordservice.yaml
```


# Unir a los peers al canal

## Preparar cadena de conexion para un peer

Para preparar la cadena de conexion, tenemos que:

1. Obtener la cadena de conexion sin usuarios para la organizacion Org1MSP y OrdererMSP
2. Registrar un usuario en la autoridad de certificacion para firma (register)
3. Obtener los certificados utilizando el usuario creado anteriormente (enroll)
4. Adjuntar el usuario a la cadena de conexion

1. Obtener la cadena de conexion sin usuarios para la organizacion Org1MSP y OrdererMSP

```bash
kubectl hlf inspect --output org1.yaml -o Org1MSP -o OrdererMSP
```

2. Registrar un usuario en la autoridad de certificacion para firma
```bash
kubectl hlf ca register --name=org1-ca --user=admin --secret=adminpw --type=admin \
 --enroll-id enroll --enroll-secret=enrollpw --mspid Org1MSP  
```

3. Obtener los certificados utilizando el usuario creado anteriormente
```bash
kubectl hlf ca enroll --name=org1-ca --user=admin --secret=adminpw --mspid Org1MSP \
        --ca-name ca  --output peer-org1.yaml
```

4. Adjuntar el usuario a la cadena de conexion
```bash
kubectl hlf utils adduser --userPath=peer-org1.yaml --config=org1.yaml --username=admin --mspid=Org1MSP
```



## Unir peer al canal

Este comando habra que ejecutarlo cada vez que queramos que un peer nuevo 

```bash
kubectl hlf channel join --name=demo --config=org1.yaml \
    --user=admin -p=org1-peer0.default
```

## Inspeccionar el canal

Podemos inspeccionar la configuracion JSON del canal que acabamos de crear con el siguiente comando:

```bash
kubectl hlf channel inspect --channel=demo --config=org1.yaml \
    --user=admin -p=org1-peer0.default > demo.json
```

## Adjuntar anchor peer

Necesitamos anchor peers para descubrir peers de otras organizaciones, si no tenemos un anchor peer por organizacion, la red no funcionara correctamente.

```bash
kubectl hlf channel addanchorpeer --channel=demo --config=org1.yaml \
    --user=admin --peer=org1-peer0.default
```


## Ver altura de bloques de cada peer


```bash
kubectl hlf channel top --channel=demo --config=org1.yaml \
    --user=admin -p=org1-peer0.default
```


## Instalar un chaincode

### Crear fichero de metadata

```bash
# remove the code.tar.gz chaincode.tgz if they exist
rm code.tar.gz chaincode.tgz
export CHAINCODE_NAME=asset
export CHAINCODE_LABEL=asset
cat << METADATA-EOF > "metadata.json"
{
    "type": "ccaas",
    "label": "${CHAINCODE_LABEL}"
}
METADATA-EOF
## chaincode as a service
```

### Preparar fichero de conexion

```bash
cat > "connection.json" <<CONN_EOF
{
  "address": "${CHAINCODE_NAME}:7052",
  "dial_timeout": "10s",
  "tls_required": false
}
CONN_EOF

tar cfz code.tar.gz connection.json
tar cfz chaincode.tgz metadata.json code.tar.gz
export PACKAGE_ID=$(kubectl hlf chaincode calculatepackageid --path=chaincode.tgz --language=node --label=$CHAINCODE_LABEL)
echo "PACKAGE_ID=$PACKAGE_ID"

kubectl hlf chaincode install --path=./chaincode.tgz \
    --config=org1.yaml --language=golang --label=$CHAINCODE_LABEL --user=admin --peer=org1-peer0.default

```


## Instalar contenedor chaincode en el cluster
El siguiente comando creará o actualizará el CRD en función del packageID, el nombre del chaincode y la imagen docker.

```bash
kubectl hlf externalchaincode sync --image=kfsoftware/chaincode-external:latest \
    --name=$CHAINCODE_NAME \
    --namespace=default \
    --package-id=$PACKAGE_ID \
    --tls-required=false \
    --replicas=1
```


## Consultar chaincodes instalados
```bash
kubectl hlf chaincode queryinstalled --config=org1.yaml --user=admin --peer=org1-peer0.default
```

## Aprobar chaincode
```bash
export SEQUENCE=1
export VERSION="1.0"
kubectl hlf chaincode approveformyorg --config=org1.yaml --user=admin --peer=org1-peer0.default \
    --package-id=$PACKAGE_ID \
    --version "$VERSION" --sequence "$SEQUENCE" --name=asset \
    --policy="OR('Org1MSP.member')" --channel=demo
```

## Commit chaincode
```bash
kubectl hlf chaincode commit --config=org1.yaml --user=admin --mspid=Org1MSP \
    --version "$VERSION" --sequence "$SEQUENCE" --name=asset \
    --policy="OR('Org1MSP.member')" --channel=demo
```


## Invocar una transaction en el canal
```bash
kubectl hlf chaincode invoke --config=org1.yaml \
    --user=admin --peer=org1-peer0.default \
    --chaincode=asset --channel=demo \
    --fcn=initLedger -a '[]'
```

## Consultar assets en el canal
```bash
kubectl hlf chaincode query --config=org1.yaml \
    --user=admin --peer=org1-peer0.default \
    --chaincode=asset --channel=demo \
    --fcn=GetAllAssets -a '[]'
```

# Añadir una segunda organizacion


## Desplegar una autoridad de certificacion

```bash
kubectl hlf ca create --storage-class=standard --capacity=1Gi --name=org2-ca \
    --enroll-id=enroll --enroll-pw=enrollpw
kubectl wait --timeout=180s --for=condition=Running fabriccas.hlf.kungfusoftware.es --all
```


Registrar un usuario en la autoridad certificacion de la organizacion peer (Org2MSP)

```bash
# registrar usuario en la CA para los peers
kubectl hlf ca register --name=org2-ca --user=peer --secret=peerpw --type=peer \
 --enroll-id enroll --enroll-secret=enrollpw --mspid Org2MSP
```

## Desplegar un peer

```bash
kubectl hlf peer create --statedb=couchdb --image=$PEER_IMAGE --version=$PEER_VERSION --storage-class=standard --enroll-id=peer --mspid=Org2MSP \
        --enroll-pw=peerpw --capacity=5Gi --name=org2-peer0 --ca-name=org2-ca.default

kubectl wait --timeout=180s --for=condition=Running fabricpeers.hlf.kungfusoftware.es --all
```


## Añadir una segunda organizacion al canal

Descargar todo el material criptografico
```bash
kubectl hlf org inspect -o Org2MSP --output-path=crypto-config
```

Añadir una segunda organizacion al canal
```bash
kubectl hlf channel addorg --peer=org1-peer0.default --name=demo \
    --config=org1.yaml --user=admin --msp-id=Org2MSP --org-config=./configtx.yaml
```

Comprobar que la organizacion se ha añadido:
```bash
kubectl hlf channel inspect --channel=demo --config=org1.yaml \
   --user=admin -p=org1-peer0.default > demo.json
```


## Preparar cadena de conexion para el Org2MSP

1. Obtener la cadena de conexion sin usuarios para la organizacion Org1MSP, Org2MSP y OrdererMSP
2. Registrar un usuario en la autoridad de certificacion para firma
3. Obtener los certificados utilizando el usuario creado anteriormente
4. Adjuntar el usuario a la cadena de conexion


1. Obtener la cadena de conexion sin usuarios para la organizacion Org1MSP, Org2MSP y OrdererMSP

```bash
kubectl hlf inspect --output org2.yaml  -o Org1MSP -o Org2MSP -o OrdererMSP
```

2. Registrar un usuario en la autoridad de certificacion para firma
```bash
kubectl hlf ca register --name=org2-ca --user=admin --secret=adminpw --type=admin \
 --enroll-id enroll --enroll-secret=enrollpw --mspid Org2MSP  
```

3. Obtener los certificados utilizando el usuario creado anteriormente
```bash
kubectl hlf ca enroll --name=org2-ca --user=admin --secret=adminpw --mspid Org2MSP \
        --ca-name ca  --output peer-org2.yaml
```

4. Adjuntar el usuario a la cadena de conexion
```bash
kubectl hlf utils adduser --userPath=peer-org2.yaml --config=org2.yaml --username=admin --mspid=Org2MSP
```

### Unir peer de la org2 al canal

```bash
kubectl hlf channel join --name=demo --config=org2.yaml \
    --user=admin -p=org2-peer0.default
```
### Anchor peer para org2 en el canal `demo`

```bash
kubectl hlf channel addanchorpeer \
   --channel=demo --config=org2.yaml \
   --user=admin --peer=org2-peer0.default
```

## Instalar chaincode en peer de Org2MSP

### Crear fichero de metadata

```bash
# remove the code.tar.gz chaincode.tgz if they exist
rm code.tar.gz chaincode.tgz
export CHAINCODE_NAME=asset
export CHAINCODE_LABEL=asset
cat << METADATA-EOF > "metadata.json"
{
    "type": "ccaas",
    "label": "${CHAINCODE_LABEL}"
}
METADATA-EOF

```

### Preparar fichero de conexion

```bash
cat > "connection.json" <<CONN_EOF
{
  "address": "${CHAINCODE_NAME}:7052",
  "dial_timeout": "10s",
  "tls_required": false
}
CONN_EOF

tar cfz code.tar.gz connection.json
tar cfz chaincode.tgz metadata.json code.tar.gz
export PACKAGE_ID=$(kubectl hlf chaincode calculatepackageid --path=chaincode.tgz --language=node --label=$CHAINCODE_LABEL)
echo "PACKAGE_ID=$PACKAGE_ID"

kubectl hlf chaincode install --path=./chaincode.tgz \
    --config=org2.yaml --language=golang --label=$CHAINCODE_LABEL --user=admin --peer=org2-peer0.default

```

## Aprobar y commit del chaincode

### Aprobar chaincode en las 2 organizaciones

```bash
export SEQUENCE=4
export VERSION="1.0"
kubectl hlf chaincode approveformyorg --config=org1.yaml --user=admin --peer=org1-peer0.default \
    --package-id=$PACKAGE_ID \
    --version "$VERSION" --sequence "$SEQUENCE" --name=asset \
    --policy="OR('Org1MSP.member', 'Org2MSP.member')" --channel=demo


kubectl hlf chaincode approveformyorg --config=org2.yaml --user=admin --peer=org2-peer0.default \
    --package-id=$PACKAGE_ID \
    --version "$VERSION" --sequence "$SEQUENCE" --name=asset \
    --policy="OR('Org1MSP.member', 'Org2MSP.member')" --channel=demo
```



### Commit chaincode
```bash
kubectl hlf chaincode commit --config=org1.yaml --user=admin --mspid=Org1MSP \
    --version "$VERSION" --sequence "$SEQUENCE" --name=asset \
    --policy="OR('Org1MSP.member', 'Org2MSP.member')" --channel=demo
```

## Probar chaincode

### Invocar una transaction en el canal
```bash
kubectl hlf chaincode invoke --config=org2.yaml \
    --user=admin --peer=org2-peer0.default \
    --chaincode=asset --channel=demo \
    --fcn=initLedger -a '[]'
```

### Consultar assets en el canal
```bash
kubectl hlf chaincode query --config=org2.yaml \
    --user=admin --peer=org2-peer0.default \
    --chaincode=asset --channel=demo \
    --fcn=GetAllAssets -a '[]'
```
