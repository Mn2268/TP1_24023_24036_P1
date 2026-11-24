# TP1 – SOAP et WSDL

Binôme : Nom1 (24023) – Nom2 (24036)

# Partie A – Code-first

## Étape 1 – Créer et lancer le service

**Q1. Quelle annotation fait de `EmpruntService` un service web ? Laquelle fait de `LivreIndisponibleException` une erreur déclarée dans le contrat ?**

- `@WebService` sur la classe `EmpruntService` en fait un service web : JAX-WS en déduit le WSDL.
- `@WebFault` sur `LivreIndisponibleException` en fait une erreur du contrat. La clause `throws LivreIndisponibleException` sur `emprunterLivre` la rattache à l'opération.

**Q2. Pourquoi les classes de données doivent-elles avoir un constructeur vide ?**

JAXB (qui convertit Java ↔ XML) crée les objets par réflexion avec `new Emprunt()`, puis remplit les champs. Sans constructeur sans argument, il ne peut pas instancier la classe.

## Étape 2 – Lire le WSDL généré

**Q1. URL du service, opérations, messages d'entrée et de sortie.**

- URL du service (`soap:address location`) : `http://localhost:8080/bib/EmpruntService`
- Opération `emprunterLivre` :
  - entrée : message `emprunterLivre` (élément `tns:emprunterLivre`)
  - sortie : message `emprunterLivreResponse` (élément `tns:emprunterLivreResponse`)
  - fault : `LivreIndisponibleException` (élément `tns:LivreIndisponible`)
- Opération `rendreLivre` :
  - entrée : message `rendreLivre` (élément `tns:rendreLivre`)
  - sortie : message `rendreLivreResponse` (élément `tns:rendreLivreResponse`)
  - aucun fault déclaré.

Lecture de bas en haut : `service EmpruntService` → `port EmpruntPort` → `binding EmpruntPortBinding` → `portType EmpruntService` → `message` → `types`.

**Q2. Style, use, version SOAP.**

- `style="document"` (dans `<soap:binding ... style="document"/>`)
- `use="literal"` (dans `<soap:body use="literal"/>`)
- SOAP 1.1 : le préfixe `soap` est lié à `http://schemas.xmlsoap.org/wsdl/soap/` (pour SOAP 1.2 ce serait `.../wsdl/soap12/`).

**Q3. Types XSD de `isbn` et `dateRetour`. La règle de format de l'ISBN apparaît-elle ?**

- `isbn` : `xs:string`
- `dateRetour` : `xs:string` (et non `xs:date`)
- La règle de format de l'ISBN **n'apparaît pas**. L'expression régulière `97[89]-\d-\d{2,5}-\d{2,6}-\d` n'existe que dans le code Java (`Pattern ISBN`). Le contrat ne la communique pas aux clients : ils ne découvrent l'erreur qu'à l'exécution.

**Q4. `idAdherent` et `isbn` portent-ils un préfixe d'espace de noms dans la requête SoapUI ?**

Non. Seul `<bib:emprunterLivre>` est qualifié (espace de noms `http://supnum.mr/bib`). Ses enfants sont écrits `<idAdherent>` et `<isbn>` sans préfixe : le schéma n'a pas `elementFormDefault="qualified"`, donc les éléments locaux sont non qualifiés.

Requête SoapUI :

```xml
<soapenv:Envelope xmlns:soapenv="http://schemas.xmlsoap.org/soap/envelope/"
                  xmlns:bib="http://supnum.mr/bib">
  <soapenv:Header/>
  <soapenv:Body>
    <bib:emprunterLivre>
      <idAdherent>A-117</idAdherent>
      <isbn>978-1-234-5678-9</isbn>
    </bib:emprunterLivre>
  </soapenv:Body>
</soapenv:Envelope>
```

Réponse reçue :

```xml
<S:Envelope xmlns:S="http://schemas.xmlsoap.org/soap/envelope/">
  <S:Body>
    <ns2:emprunterLivreResponse xmlns:ns2="http://supnum.mr/bib">
      <emprunt>
        <numero>E-2026-0812</numero>
        <dateRetour>2026-12-08</dateRetour>
      </emprunt>
    </ns2:emprunterLivreResponse>
  </S:Body>
</S:Envelope>
```

Remarque : un premier envoi sans remplacer les `?` de SoapUI a donné une Fault `S:Client` avec `faultstring` = `ISBN mal formé : ?`, car `?` ne respecte pas le motif de l'ISBN.

**Q5. Où voit-on HTTP, XML, SOAP dans l'onglet Raw ? Rôle de chaque standard et du WSDL.**

- **HTTP** : les lignes d'en-tête (`POST /bib/EmpruntService HTTP/1.1`, `Content-Type`, `SOAPAction`) et, dans la réponse, la ligne de statut (`HTTP/1.1 200 OK`). Rôle : transport, il achemine le message jusqu'à l'URL du service.
- **XML** : tout le corps du message. Rôle : syntaxe commune, lisible par toute plateforme et tout langage.
- **SOAP** : la structure `Envelope` / `Header` / `Body`. Rôle : format du message (métadonnées dans le Header, appel ou réponse dans le Body, erreurs sous forme de `Fault`).
- **WSDL** : absent du message. C'est le contrat qui décrit où appeler (`service`), comment emballer (`binding`), quelles opérations existent (`portType`) et quelles données circulent (`message`, `types`). SoapUI s'en sert pour fabriquer la requête.

## Étape 3 – Envoyer des messages SOAP

### Envoi de `01-emprunter.xml` (SOAP 1.1)

```
curl.exe -i -X POST http://localhost:8080/bib/EmpruntService -H "Content-Type: text/xml; charset=utf-8" -H "SOAPAction: \"\"" --data-binary @xml/01-emprunter.xml
```

Réponse :

```
HTTP/1.1 200 OK
Content-type: text/xml; charset=utf-8

<S:Envelope xmlns:S="http://schemas.xmlsoap.org/soap/envelope/"><S:Body><ns2:emprunterLivreResponse xmlns:ns2="http://supnum.mr/bib"><emprunt><numero>E-2026-0813</numero><dateRetour>2026-12-08</dateRetour></emprunt></ns2:emprunterLivreResponse></S:Body></S:Envelope>
```

Remarque : sous PowerShell, la même commande a renvoyé une Fault `S:Client` (« Unexpected EOF in prolog ») car PowerShell transmet mal l'argument `SOAPAction: \"\"` et le corps arrive vide. Sous cmd, elle fonctionne.

### Envoi de `02-emprunter-12.xml` (SOAP 1.2)

Réponse :

```
HTTP/1.1 500 Internal Server Error
Content-type: text/xml; charset=utf-8

<S:Envelope xmlns:S="http://schemas.xmlsoap.org/soap/envelope/"><S:Body><S:Fault><faultcode>S:VersionMismatch</faultcode><faultstring>Impossible de créer le message SOAP. Attente de Envelope dans l'espace de noms http://schemas.xmlsoap.org/soap/envelope/, mais http://www.w3.org/2003/05/soap-envelope obtenu</faultstring></S:Fault></S:Body></S:Envelope>
```

### Questions

**Q1. Pourquoi la requête utilise-t-elle POST, même pour lire une information ?**

SOAP transporte une enveloppe XML dans le corps de la requête, et seul POST permet d'envoyer un corps. Toutes les opérations passent par la même URL : HTTP ne dit pas quelle action est demandée, c'est le contenu SOAP (l'élément du Body) qui la décrit. GET n'a pas de corps et sert à lire une ressource identifiée par son URL, ce qui n'est pas le modèle de SOAP.

**Q2. Le bloc `jeton` du Header a-t-il empêché le traitement ? Pourquoi ?**

Non : réponse 200, emprunt créé. Le bloc n'a pas `mustUnderstand="1"`, donc le service n'est pas obligé de le comprendre et l'ignore. Le Header porte des informations de traitement (authentification, transaction) qui ne font pas partie de l'appel lui-même.

**Q3. Que répond le service au message SOAP 1.2 ? Expliquez avec le binding lu à l'étape 2.**

Une Fault `S:VersionMismatch` avec HTTP 500. Le binding de l'étape 2 utilise l'espace de noms `http://schemas.xmlsoap.org/wsdl/soap/`, donc SOAP 1.1 : le service attend une enveloppe dans `http://schemas.xmlsoap.org/soap/envelope/`. Il reçoit `http://www.w3.org/2003/05/soap-envelope` (SOAP 1.2) et refuse.


**Q4. Les cinq erreurs du dernier message et leur correction.**

| N° | Erreur | Correction | Réaction du serveur |
|---|---|---|---|
| 1 | `<soap:envelope>` (casse) | `<soap:Envelope>` | Fault `S:Client` HTTP 500 : « balise XML inattendue ; attendue Envelope, obtenue envelope » |
| 2 | `</rendreLivre>` sans préfixe | `</bib:rendreLivre>` | Fault `S:Server` HTTP 500 : `UnmarshalException … Unexpected close tag </rendreLivre>; expected </bib:rendreLivre>` |
| 3 | `Header` placé après `Body` | `Header` avant `Body` | **Aucune** : HTTP 200 |
| 4 | namespace `http://supnum.mr/bibliotheque` | `http://supnum.mr/bib` | Fault `S:Client` HTTP 500 : « Méthode de répartition introuvable pour {http://supnum.mr/bibliotheque}rendreLivre » |
| 5 | `<bib:numero>` | `<numeroEmprunt>` (sans préfixe) | **Aucune** : HTTP 200 avec `<penalite>0</penalite>` |

Envois successifs : (0) message du sujet → erreur 1 ; (1) après correction 1 →
erreur 4 (le serveur s'arrête au routage, avant de lire la balise fermante) ;
(2) après correction 4 → erreur 2 ; (3) après correction 2 → HTTP 200,
`<penalite>0</penalite>`.

**Erreurs sans réaction du serveur : les n° 3 et 5.**
- Erreur 3 : le serveur ne contrôle pas l'ordre Header/Body, alors que SOAP impose
  le Header avant le Body. Le bloc `jeton` n'a pas `mustUnderstand`, donc il est ignoré.
- Erreur 5 : `bib:numero` est un élément inconnu, que JAXB ignore. `numeroEmprunt`
  vaut `null`, et la méthode renvoie `0` sans rien vérifier : la réponse paraît
  correcte, alors que la donnée n'a pas été lue.
Ces deux erreurs ne sont pas des erreurs de syntaxe XML mais de contrat. Sans
validation de schéma (étape 6, `@SchemaValidation`), rien ne les détecte.

Message final correct (`xml/04-rendre-corrige.xml`) : voir le fichier.

Envoi 4 (après correction 3, Header avant Body) : HTTP 200, `<penalite>0</penalite>`.
Réponse identique à l'envoi 3 : l'ordre Header/Body n'était pas contrôlé.

Envoi 5 (message final `xml/04-rendre-corrige.xml`, avec `<numeroEmprunt>`) : HTTP 200,
`<penalite>0</penalite>`. Là encore la réponse est identique : `rendreLivre` renvoie
toujours `0` et ne lit pas son paramètre. La différence entre `bib:numero` et
`numeroEmprunt` ne sera visible qu'à l'étape 4, quand on ajoutera le contrôle
`E-2026-`.


# Étape 4 – Provoquer et construire des Faults

### Cas 1 : ISBN invalide (`978-2-10`)

```
HTTP/1.1 500 Internal Server Error
Content-type: text/xml; charset=utf-8

<S:Envelope xmlns:S="http://schemas.xmlsoap.org/soap/envelope/"><S:Body><S:Fault><faultcode>S:Client</faultcode><faultstring>ISBN mal formé : 978-2-10</faultstring></S:Fault></S:Body></S:Envelope>
```

### Cas 2 : livre déjà emprunté (`978-2-10-074444-4`)

```
HTTP/1.1 500 Internal Server Error
Content-type: text/xml; charset=utf-8

<S:Envelope xmlns:S="http://schemas.xmlsoap.org/soap/envelope/"><S:Body><S:Fault><faultcode>S:Server</faultcode><faultstring>Livre indisponible</faultstring><detail><ns2:LivreIndisponible xmlns:ns2="http://supnum.mr/bib"><isbn>978-2-10-074444-4</isbn><retourPrevu>2026-12-02</retourPrevu></ns2:LivreIndisponible></detail></S:Fault></S:Body></S:Envelope>
```

### Questions

**Q1. Pour les cas 1 et 2, quels faultcode et faultstring recevez-vous, et quel code HTTP ?**

| Cas | faultcode | faultstring | HTTP |
|---|---|---|---|
| 1 (ISBN `978-2-10`) | `S:Client` | `ISBN mal formé : 978-2-10` | 500 |
| 2 (ISBN `978-2-10-074444-4`) | `S:Server` | `Livre indisponible` | 500 |

Dans les deux cas le code HTTP est 500 : en SOAP 1.1, toute Fault est renvoyée avec ce code, même une faute du client.

**Q2. Dans le cas 2, que contient `<detail>` ? Où sa structure est-elle décrite dans le WSDL ?**

`<detail>` contient l'élément `<ns2:LivreIndisponible>` (namespace `http://supnum.mr/bib`) avec `<isbn>978-2-10-074444-4</isbn>` et `<retourPrevu>2026-12-02</retourPrevu>`. Sa structure est décrite dans le WSDL par :
- `portType` › opération `emprunterLivre` › `<fault name="LivreIndisponibleException" message="tns:LivreIndisponibleException"/>` ;
- le `message` `LivreIndisponibleException`, dont la `part` pointe vers l'élément `tns:LivreIndisponible` ;
- le schéma XSD (`?xsd=1`), où cet élément a le type `livreIndisponibleInfo` (`isbn` et `retourPrevu`, de type `xs:string`).

Le cas 1 n'a pas de `<detail>` : cette Fault est construite à la main et n'est pas déclarée dans le contrat.

**Q3. Quel code JAX-WS a-t-il choisi pour le livre indisponible ? Client ou serveur ? Que choisiriez-vous ?**

JAX-WS a choisi `S:Server`, car toute exception applicative (`@WebFault`) est traduite en Fault `Server` par défaut. Ce choix est discutable : la requête est bien formée et valide, mais l'état de la ressource (livre sorti) empêche de la satisfaire. Ce n'est pas un bogue du serveur.
Je choisirais plutôt `Client` : c'est la demande du client (emprunter ce livre) qui ne peut pas aboutir, et le client ne réussira pas en la renvoyant telle quelle. `Server` suggère une panne du serveur, réessayable plus tard. Ici l'information utile est dans le `detail` (`retourPrevu`), et un client doit la traiter comme une erreur métier, pas comme une panne.


### Test `mustUnderstand`

Avec `soap:mustUnderstand="1"` sur `<bib:jeton>` :

```
HTTP/1.1 500 Internal Server Error
Content-type: text/xml; charset=utf-8

<S:Envelope xmlns:S="http://schemas.xmlsoap.org/soap/envelope/"><S:Body><S:Fault><faultcode>S:MustUnderstand</faultcode><faultstring>MustUnderstand headers:[{http://supnum.mr/bib}jeton] are not understood</faultstring></S:Fault></S:Body></S:Envelope>
```

Avec `soap:mustUnderstand="0"` : *(à compléter avec votre réponse)*

**Q4. Que se passe-t-il avec `mustUnderstand="1"` ? Et avec `"0"` ?**

- Avec `"1"` : le récepteur doit comprendre le bloc d'en-tête ou rejeter tout le message. Le service ne connaît pas `jeton` (aucun handler), il renvoie donc une Fault `S:MustUnderstand` (HTTP 500) qui nomme le bloc non compris. La méthode n'est pas appelée : aucun emprunt n'est créé.

Avec `soap:mustUnderstand="0"` : le bloc `jeton` n'a pas été rejeté, le service a traité
la requête jusqu'à la logique métier. Le premier essai (ISBN `978-2-07-036822-8`) a donné
la Fault `S:Server` « Livre indisponible » (`retourPrevu` 2026-12-08), car ce livre avait
déjà été emprunté à l'étape 3 (l'état est gardé en mémoire). Avec un ISBN encore
disponible : [collez ici la réponse 200 si vous faites l'essai].


### Point 4 : Fault Client dans `rendreLivre`

Ajout dans `EmpruntService.java` :

```java
if (numeroEmprunt == null || !numeroEmprunt.startsWith("E-2026-"))
    throw faultClient("Numéro d'emprunt inconnu : " + numeroEmprunt);
```

Après redémarrage du serveur :

| Fichier | Réponse |
|---|---|
| `08-rendre-invalide.xml` (`X-1`) | HTTP 500, `S:Client`, `Numéro d'emprunt inconnu : X-1` |
| `03-erreurs.xml` (`bib:numero`) | HTTP 500, `S:Client`, `Numéro d'emprunt inconnu : null` |
| `04-rendre-corrige.xml` (`E-2026-0812`) | HTTP 200, `<penalite>0</penalite>` |

Le deuxième cas prouve l'erreur silencieuse n° 5 de l'étape 3 : l'élément `bib:numero`
était ignoré, `numeroEmprunt` valait `null`, et l'ancienne version renvoyait `0` sans
rien lire. Le contrôle rend l'erreur visible.


### Point 5 : Fault SOAP 1.2 écrite à la main (`xml/03-fault-sender-12.xml`)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<soap:Envelope xmlns:soap="http://www.w3.org/2003/05/soap-envelope"
               xmlns:bib="http://supnum.mr/bib">
  <soap:Body>
    <soap:Fault>
      <soap:Code>
        <soap:Value>soap:Sender</soap:Value>
      </soap:Code>
      <soap:Reason>
        <soap:Text xml:lang="fr">ISBN mal formé : 978-2-10</soap:Text>
      </soap:Reason>
      <soap:Detail>
        <bib:erreurIsbn>
          <bib:isbn>978-2-10</bib:isbn>
        </bib:erreurIsbn>
      </soap:Detail>
    </soap:Fault>
  </soap:Body>
</soap:Envelope>
```

Vérification : `xmllint` n'étant pas installé sous Windows, le fichier a été chargé avec
`[xml](Get-Content xml\03-fault-sender-12.xml -Raw -Encoding UTF8)` : aucune erreur,
le XML est bien formé.

**Q5. Quelle différence voyez-vous entre votre Fault 1.2 et les Faults 1.1 reçues ?**

| Élément | SOAP 1.1 (reçu) | SOAP 1.2 (écrit) |
|---|---|---|
| Espace de noms de l'enveloppe | `http://schemas.xmlsoap.org/soap/envelope/` | `http://www.w3.org/2003/05/soap-envelope` |
| Code | `<faultcode>S:Client</faultcode>` (texte simple) | `<soap:Code><soap:Value>soap:Sender</soap:Value></soap:Code>` (élément structuré) |
| Valeurs du code | `Client` / `Server` | `Sender` / `Receiver` |
| Message lisible | `<faultstring>` | `<soap:Reason><soap:Text xml:lang="fr">` : texte avec langue obligatoire |
| Détail | `<detail>` (minuscule) | `<soap:Detail>` |
| Éléments de la Fault | sans namespace (`faultcode`, `faultstring`, `detail`) | tous qualifiés par l'espace de noms SOAP 1.2 |

En résumé : SOAP 1.2 structure et qualifie les éléments de la Fault, rend la langue du message explicite (`xml:lang`) et renomme `Client`/`Server` en `Sender`/`Receiver`.