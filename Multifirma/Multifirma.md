# Multifirma usando la linea de comandos de Bitcoin Core.

## Introduccion
En este tutorial vamos a explicar las funciones que podemos usar para crear y gastar una multifirma usando la linea de comandos de bitcoin core.

## Requisitos:
Sistema operativo Linux. Para este ejemplo he usado Debian 12 y Ubuntu 24
Bitcoin core. Para este ejemplo he utilizado la version 28.0
Configurar Bitcoin Core en modo regtest.
JSON

## Antecedentes:
En las últimas versiones de  Bitcoin Core se ha fomentado el uso de segwit  en detrimento de  las direcciones Legacy. Tambien se han introducido los miniscripts. Algunas funciones  como  addmultisigaddress, importmulti, importprivkey o dumpkey solo funcionan en Legacy por lo tanto debemos usar los descriptors. Para poder hacer  una transaccion multifirma en segwit, tenemos  que trabajar con los descriptors y miniscript.

### Vamos a ver un ejemplo: Alice y Bob crean una dirección multifirma y gastan en una transaccion.

## Entendiendo los comandos
En la página oficial https://bitcoincore.org/en/doc/28.0.0/ encontrarás los comandos que vamos a utilizar para entender mejor este tutorial: listdescriptors, getdescriptorinfo, importdescriptors, walletcreatefundedpsbt, walletprocesspsbt, combinepsbt, finalizepsbt.

## Paso a paso

Creamos dos direcciones  de Alice y Bob

```
$bitcoin-cli createwallet "Alice"
$bitcoin-cli createwallet "Bob"
```

Obtenemos los descriptors de Alice y Bob, según recomendado por los desarrolladores  seleccionaremos los obtenidos con la derivacion de pkh.

```
$descAint=$(bitcoin-cli -rpcwallet=Alice listdescriptors | jq -r '.descriptors | [.[] | select(.desc | startswith("pkh") and contains("/1/*"))][0] | .desc')
$descAext=$(bitcoin-cli -rpcwallet=Alice listdescriptors | jq -r '.descriptors | [.[] | select(.desc | startswith("pkh") and contains("/0/*"))][0] | .desc')
```

Tomaremos la  llave  tpub  y le quitamos la informacion de la derivacion que aparece delante y el checksum que tenemos al final

```
$descAint=$(echo $descAint | awk '{ print substr( $0, 1, length($0)-10 ) }')
$descAint=$(echo $descAint | awk '{ print substr ($0, 25 ) }')
$descAext=$(echo $descAext | awk '{ print substr( $0, 1, length($0)-10 ) }')
$descAext=$(echo $descAext | awk '{ print substr ($0, 25 ) }')
```

Repetimos el proceso con el wallet de Bob

```
$descBint=$(bitcoin-cli -rpcwallet=Bob listdescriptors | jq -r '.descriptors | [.[] | select(.desc | startswith("pkh") and contains("/1/*"))][0] | .desc')
$descBext=$(bitcoin-cli -rpcwallet=Bob listdescriptors | jq -r '.descriptors | [.[] | select(.desc | startswith("pkh") and contains("/0/*"))][0] | .desc')
$descBint=$(echo $descBint | awk '{ print substr( $0, 1, length($0)-10 ) }')
$descBint=$(echo $descBint | awk '{ print substr ($0, 25 ) }')
$descBext=$(echo $descBext | awk '{ print substr( $0, 1, length($0)-10 ) }')
$descBext=$(echo $descBext | awk '{ print substr ($0, 25 ) }')
```

Ahora vamos a crear el descriptor de nuestro wallet multifirma para ello usaremos la funcion  miniscript multi

```
$extdesc="wsh(multi(2,$descAext,$descBext))"
$intdesc="wsh(multi(2,$descAint,$descBint))"
```

Necesitamos añadir el checksum  de nuestro descriptor multifirma, esto lo conseguimos  con getdescriptorinfo

```
$extdescsum=$(bitcoin-cli getdescriptorinfo $extdesc | jq -r  '.descriptor')
$intdescsum=$(bitcoin-cli  getdescriptorinfo $intdesc | jq -r '.descriptor')
```

Creamos un wallet en blanco para poder importar nuestro descriptor

```
$bitcoin-cli -named createwallet wallet_name="multi" disable_private_keys=true blank=true
```

Importamos los descriptors

```
$bitcoin-cli  -rpcwallet="multi" importdescriptors "[{\"desc\": \"$extdescsum\",\"timestamp\": \"now\",\"active\": true,\"watching-only\": true,\"internal\": false,\"range\": [0,999]} , {\"desc\": \"$intdescsum\",\"timestamp\": \"now\",\"active\": true,\"watching-only\": true,\"internal\": true,\"range\": [0,999]}]"

$bitcoin-cli -rpcwallet="multi" getwalletinfo
```

Ya podemos obtener una direccion de nuestro wallet

```
$direccionmulti=$(bitcoin-cli -rpcwallet="multi" getnewaddress)
```

Enviamos unos BTC regtest a nuestro wallet multi y ahora vamos a enviar desde nuestro  wallet Multi a Alice

```
$cambiomulti=$(bitcoin-cli -rpcwallet="multi" getnewaddress)
$direccionAlice=$(bitcoin-cli -rpcwallet="Alice" getnewaddress)
```

Creamos la transaccion psbt

```
$psbt=$(bitcoin-cli -rpcwallet="multi" -named walletcreatefundedpsbt inputs="[{\"txid\": \"$identtx\",\"vout\":0}]" outputs="[{\"$cambiomulti\":10},{\"$direccionAlice\":39.999}]" | jq -r '.psbt')

$bitcoin-cli analyzepsbt $psbt
```

Alice y Bob  pueden firmar por separado y combinar las firmas

```
$psbtA=$(bitcoin-cli -rpcwallet="Alice" walletprocesspsbt $psbt | jq -r '.psbt')
$psbtB=$(bitcoin-cli -rpcwallet="Bob" walletprocesspsbt $psbt | jq -r '.psbt')

$combinedpsbt=$(bitcoin-cli combinepsbt "[\"$psbtA\", \"$psbtB\"]")

$finalizedpsbt=$(bitcoin-cli finalizepsbt $combinedpsbt | jq -r '.hex')

$bitcoin-cli sendrawtransaction $finalizedpsbt
```

Podemos minar unos Bitcoin en Regtest para ver los saldos de Alice y Multi para comprobar que se ha realizado la transaccion

## Solucion

[Solucion](/Multifirma/Solucion.sh)

## Conclusion
Con este tutorial hemos aprendido:
Hacer una transaccion multifirma
Como puedes observar Alice y Bob están offline, luego puedes usar lo aprendido para tener un wallet offline
Descriptors
Miniscripts

## Enlaces
Enlaces de interes usados para hacer este tutorial
https://diyhpl.us/wiki/transcripts/advancing-bitcoin/2020/2020-02-06-andrew-chow-descriptor-wallets/ En este enlace puedes aprender mas detalles sobre los descriptors.
https://github.com/bitcoin/bitcoin/blob/master/doc/descriptors.md#multisig
https://github.com/bitcoin/bitcoin/blob/master/doc/multisig-tutorial.md
https://github.com/bitcoin/bitcoin/blob/master/doc/descriptors.md En este enlace verás una lista de miniscripts implementados en Bitcoin Core.
