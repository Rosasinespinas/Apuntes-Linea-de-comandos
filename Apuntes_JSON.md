# Apuntes de JSON

##Obtener un valor de un JSON

* Usar el comando jq -r y luego entre comillas simples '.nombrearray.dato'
* Esto nos da el resultado del dato.
Con -r le quito las comillas.
Ejemplo:

```
bitcoin-cli getblockchaininfo | jq -r '.blocks'
```

##Obtener varios datos en un array

* Para obtener todos los valores utiliza [] .
Ejemplo:

```
bitcoin-cli listunspent | jq -r '.[]|.amount'
```

* Ejemplo para acceder a un valor en concreto:

```
amount=$(bitcoin-cli listunspent | jq -r '[]|.amount')
echo ${amount[0]}
```

Así obtendré el monto del primer UTXO.

##Escribir un JSON

* Primero construimos los arrays pequeños usando --arg

```
inner1=$(jq -n  --arg txid "$Tradertxid"\
                --arg vout "$Tradervout"\
                '$ARGS.named'
)
inner2=$(jq -n  --arg txid "$Minertxid"\
                --arg vout "$Minervout"\
                '$ARGS.named'
)
```

* Para el array final

```
final=$(jq -n   --argjson input "[$inner1, $inner2]"\
                --arg Fees "$fees"\
                --arg Weight "$weight"\
                '$ARGS.named'
)
```
