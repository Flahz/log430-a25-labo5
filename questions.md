## Question 1 : Quelle réponse obtenons-nous à la requête à POST /payments ? Illustrez votre réponse avec des captures d'écran/terminal.

```bash
$ docker compose exec store_manager curl -s -i \
>   -H 'Content-Type: application/json' \
>   -d '{"user_id":1,"order_id":999,"total_amount":42.5}' \
>   http://api-gateway:8080/payments-api/payments
HTTP/1.1 200 OK
Cache-Control: public, max-age=300
Content-Type: application/json; charset=utf-8
X-Krakend: Version 2.4.6
X-Krakend-Completed: true
Date: Mon, 27 Oct 2025 18:59:12 GMT
Content-Length: 16

{"payment_id":3}
```

### Question 2 : Quel type d'information envoyons-nous dans la requête à POST payments/process/:id ? Est-ce que ce serait le même format si on communiquait avec un service SOA, par exemple ? Illustrez votre réponse avec des exemples et des captures d'écran/terminal.

Dans cette requête, nous envoyons un corps JSON contenant les données de carte de crédit nécessaires pour simuler la transaction (`cardNumber`, `cardCode`, `expirationDate`). Il s'agit d'un format typique pour une API REST moderne, léger et directement compatible avec le `Content-Type: application/json`.

Pour un service SOA de type SOAP, le même échange se ferait plutôt au moyen d'un message XML enveloppé, par exemple :

```xml
<soapenv:Envelope xmlns:soapenv="http://schemas.xmlsoap.org/soap/envelope/" xmlns:pay="http://example.com/payment">
  <soapenv:Header/>
  <soapenv:Body>
    <pay:ProcessPaymentRequest>
      <pay:PaymentId>3</pay:PaymentId>
      <pay:CardNumber>9999999999999</pay:CardNumber>
      <pay:CardCode>123</pay:CardCode>
      <pay:ExpirationDate>2030-01-05</pay:ExpirationDate>
    </pay:ProcessPaymentRequest>
  </soapenv:Body>
</soapenv:Envelope>
```

La principale différence repose donc sur le format (JSON vs XML) et sur l'enveloppe supplémentaire imposée par SOAP dans un contexte SOA traditionnel.

### Question 3 : Quel résultat obtenons-nous de la requête à POST payments/process/:id ?

```bash
$ docker compose exec store_manager curl -s -i \
>   -H 'Content-Type: application/json' \
>   -d '{"cardNumber":9999999999999,"cardCode":123,"expirationDate":"2030-01-05"}' \
>   http://api-gateway:8080/payments-api/payments/3
HTTP/1.1 200 OK
Cache-Control: public, max-age=300
Content-Type: application/json; charset=utf-8
X-Krakend: Version 2.4.6
X-Krakend-Completed: true
Date: Mon, 27 Oct 2025 19:03:53 GMT
Content-Length: 46

{"is_paid":true,"order_id":999,"payment_id":3}
```

### Question 4 : Quelle méthode avez-vous dû modifier dans log430-a25-labo05-payment et qu'avez-vous modifié ?

```python
# ../log430-a25-labo5-payment/src/controllers/payment_controller.py
# Avant
def process_payment(payment_id, credit_card_data):
    _process_credit_card_payment(credit_card_data)
    update_result = update_status_to_paid(payment_id)
    print(f"Updated order {update_result['order_id']} to paid={update_result}")
    result = {
        "order_id": update_result["order_id"],
        "payment_id": update_result["payment_id"],
        "is_paid": update_result["is_paid"]
    }

    return result


# Après
def process_payment(payment_id, credit_card_data):
    _process_credit_card_payment(credit_card_data)
    update_result = update_status_to_paid(payment_id)
    print(f"Updated order {update_result['order_id']} to paid={update_result}")
    result = {
        "order_id": update_result["order_id"],
        "payment_id": update_result["payment_id"],
        "is_paid": update_result["is_paid"]
    }

    if result["is_paid"] and result["order_id"]:
        try:
            result["order_update"] = _notify_store_manager_order_paid(result["order_id"])
        except requests.RequestException as exc:
            result["order_update_error"] = str(exc)

    return result


def _notify_store_manager_order_paid(order_id: int):
    response = requests.put(
        "http://api-gateway:8080/store-api/orders",
        json={"order_id": order_id, "is_paid": True},
        headers={"Content-Type": "application/json"},
        timeout=5
    )
    response.raise_for_status()
    return response.json()
```

**Différence** : la version modifiée continue de marquer le paiement comme complété, mais elle notifie aussi le microservice Store Manager via `PUT /store-api/orders` (à travers KrakenD) pour aligner l’état de la commande (`is_paid = true`). L’appel HTTP est isolé dans `_notify_store_manager_order_paid`, ce qui permet de consigner la réponse ou l’erreur et d’assurer la synchronisation des données entre services.


### Question 5 : Que se passe-t-il dans le navigateur quand vous faites une requête avec un délai supérieur au timeout configuré (5 secondes) ? Quelle est l'importance du timeout dans une architecture de microservices ? Justifiez votre réponse avec des exemples pratiques.

Le serveur (via KrakenD) retourne une erreur 500. Par exemple :

```bash
$ curl -i http://api-gateway:8080/store-api/test/slow/10
HTTP/1.1 500 Internal Server Error
X-Krakend: Version 2.4.6
X-Krakend-Completed: false
Date: Mon, 27 Oct 2025 21:05:12 GMT
Content-Length: 0
```

Le navigateur affiche donc un échec après environ cinq secondes, même si le backend `store_manager` poursuit son `time.sleep(10)`. Le timeout sert de filet de sécurité : il empêche les appels lents de monopoliser les ressources et informe rapidement le client que la requête a échoué.

Dans une architecture de microservices, c’est essentiel pour éviter l’« effet domino » :
- **Protection des clients** : l’UI (ou un service appelant) peut présenter un message clair ou réessayer un endpoint alternatif au lieu d’attendre indéfiniment.
- **Observabilité** : des timeouts configurés permettent aux métriques (ex. Prometheus) de détecter rapidement les lenteurs et de déclencher des alertes.
