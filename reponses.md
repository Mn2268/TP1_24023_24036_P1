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


# Partie B – Contract-first

## Introduction

Dans cette deuxième partie du TP, nous avons utilisé l'approche **Contract-first**.

Contrairement à l'approche Code-first de la partie A, nous commençons ici par définir le contrat du service dans un fichier WSDL. Ensuite, l'outil `wsimport` génère automatiquement les classes Java nécessaires à partir de ce contrat.

L'objectif est de vérifier que le service respecte le contrat défini, notamment les types de données, les opérations, les messages SOAP et les erreurs déclarées.

Le projet créé est :

`emprunt-contractfirst`

---

# Étape 5 – Générer le code depuis le WSDL

## 1. Création du contrat WSDL

Nous avons créé le fichier :

`src/main/resources/wsdl/EmpruntService.wsdl`

Ce fichier définit le contrat du service avec les caractéristiques suivantes :

| Élément | Valeur |
|---|---|
| Namespace | `http://supnum.mr/bib` |
| Service | `EmpruntService` |
| Port | `EmpruntPort` |
| PortType | `EmpruntPortType` |
| Opération | `emprunterLivre` |
| Binding | Document/Literal |
| SOAPAction | `http://supnum.mr/bib/emprunterLivre` |
| Adresse du service | `http://localhost:8080/bib/EmpruntService` |

Le contrat contient également les types suivants :

- `IsbnType` : type personnalisé qui impose un format précis pour l'ISBN.
- `EmpruntType` : contient le numéro d'emprunt et la date de retour.
- `LivreIndisponible` : décrit les informations retournées lorsqu'un livre est indisponible.

Le motif de validation de l'ISBN est :

```text
97[89]-\d-\d{2,5}-\d{2,6}-\d
```

Le champ `dateRetour` est défini avec le type `xs:date`.

Le schéma utilise également :

```xml
elementFormDefault="qualified"
```

Cela signifie que les éléments XML locaux doivent appartenir au namespace du contrat.

## 2. Génération des classes Java avec wsimport

Nous avons exécuté la commande :

```bash
mvn clean generate-sources
```

<badge color="success">BUILD SUCCESS</badge>

La génération des sources s'est terminée avec succès.

Les classes ont été créées automatiquement dans :

`target/generated-sources/wsimport/mr/supnum/bib/contrat/`

Les principales classes générées sont :

| Classe | Rôle |
|---|---|
| `EmpruntPortType.java` | Interface Java du service |
| `EmpruntType.java` | Objet représentant un emprunt |
| `LivreIndisponible.java` | Objet contenant les informations du fault |
| `LivreIndisponibleFault.java` | Exception déclarée dans le contrat |
| `EmprunterLivre.java` | Représentation de la requête |
| `EmprunterLivreResponse.java` | Représentation de la réponse |
| `EmpruntService.java` | Classe cliente du service |
| `ObjectFactory.java` | Création des objets JAXB |
| `package-info.java` | Informations du namespace XML |

Le dossier `target/` est généré automatiquement par Maven. Nous ne l'avons pas modifié manuellement.

## Questions de l'étape 5

### Q1. Dans ce WSDL, retrouvez l'opération, son message de fault et l'action SOAP à envoyer.

Dans le fichier `EmpruntService.wsdl`, nous avons identifié :

- Nom de l'opération : `emprunterLivre`.
- Message de requête : `emprunterLivreRequest`.
- Message de réponse : `emprunterLivreResponse`.
- Message de fault : `LivreIndisponibleFault`.
- Élément XML du fault : `LivreIndisponible`.
- Action SOAP : `http://supnum.mr/bib/emprunterLivre`.

Le message de fault est déclaré dans le WSDL et permet de transmettre les informations relatives à un livre indisponible.

### Q2. Quelle est la signature de emprunterLivre dans EmpruntPortType ?

Après la génération du code, nous avons ouvert le fichier `EmpruntPortType.java`.

La signature obtenue est :

```java
@WebMethod(action = "http://supnum.mr/bib/emprunterLivre")
public EmpruntType emprunterLivre(
    String idAdherent,
    String isbn
) throws LivreIndisponibleFault;
```

Cette méthode reçoit deux paramètres :

- `idAdherent` de type `String`.
- `isbn` de type `String`.

Elle retourne un objet `EmpruntType` et peut déclencher l'exception `LivreIndisponibleFault`.

L'annotation `@WebMethod` contient également l'action SOAP définie dans le contrat WSDL.

### Q3. Quel type Java a été généré pour dateRetour ? Comparez avec la partie A.

Dans la partie B, le type Java généré pour `dateRetour` est :

```java
XMLGregorianCalendar
```

Dans la partie A (Code-first), le champ `dateRetour` était de type `String`.

Cette différence vient du contrat WSDL : le type XML `xs:date` est traduit en `XMLGregorianCalendar` par l'outil `wsimport`.

L'approche Contract-first permet donc de contrôler plus précisément les types de données échangés entre le service et ses clients.

### Q4. Quel est le nom de la classe d'exception générée, et d'où vient ce nom dans le WSDL ?

La classe générée est :

`LivreIndisponibleFault`

Son nom provient du message de fault déclaré dans le WSDL :

```xml
<wsdl:message name="LivreIndisponibleFault">
    <wsdl:part name="fault" element="bib:LivreIndisponible"/>
</wsdl:message>
```

Cette classe représente l'exception métier qui est déclenchée lorsqu'un livre n'est pas disponible.

---

# Étape 6 – Implémenter et comparer

## 1. Implémentation du service

Après la génération des classes Java, nous avons créé :

`src/main/java/mr/supnum/bib/impl/EmpruntServiceImpl.java`

Cette classe implémente l'interface générée :

`EmpruntPortType`

Nous avons utilisé les classes générées par `wsimport` au lieu de recréer manuellement les objets du contrat.

La méthode `emprunterLivre` réalise deux traitements principaux :

**Cas 1 : livre disponible**

Si l'ISBN ne correspond pas au livre indisponible défini dans notre implémentation, le service crée un objet `EmpruntType` et retourne :

- Numéro d'emprunt : `E-2026-0812`.
- Date de retour : date actuelle + 14 jours.

**Cas 2 : livre indisponible**

Si l'ISBN est :

`978-2-10-074444-4`

le service crée un objet `LivreIndisponible`, renseigne l'ISBN et la date de retour prévue `2026-12-02`, puis déclenche :

```java
throw new LivreIndisponibleFault(
    "Livre indisponible",
    info
);
```

## 2. Démarrage du serveur

Le serveur a été publié à l'adresse :

`http://localhost:8080/bib/EmpruntService`

Le message affiché au démarrage était :

```text
Service Contract-first demarre !
WSDL : http://localhost:8080/bib/EmpruntService?wsdl
```

Le WSDL était accessible depuis le navigateur.

**Remarque technique :**

Lors du lancement avec `mvn exec:java`, nous avons rencontré un problème de chargement de l'implémentation JAXB (`ContextFactory`).

Après vérification des dépendances Maven, nous avons lancé le serveur avec le classpath explicite :

```powershell
$cp = (Get-Content .\classpath.txt -Raw).Trim()
$cp = ".\target\classes;" + $cp
java -cp $cp mr.supnum.bib.impl.Serveur
```

Cette méthode a permis de démarrer correctement le service et de tester les réponses SOAP, y compris les Faults métier.

## 3. Premier test : emprunt réussi

Nous avons envoyé une requête SOAP 1.1 contenant un ISBN valide et disponible.

La requête utilise l'en-tête :

```text
Content-Type: text/xml; charset=utf-8
SOAPAction: "http://supnum.mr/bib/emprunterLivre"
```

Le test a été effectué avec `curl.exe`.

Commande utilisée :

```powershell
curl.exe -i -X POST "http://localhost:8080/bib/EmpruntService" `
-H "Content-Type: text/xml; charset=utf-8" `
-H 'SOAPAction: "http://supnum.mr/bib/emprunterLivre"' `
--data-binary "@xml/01-emprunter-contractfirst.xml"
```

**Résultat obtenu :**

```text
HTTP/1.1 200 OK
Content-type: text/xml; charset=utf-8
```

Corps de la réponse :

```xml
<emprunterLivreResponse xmlns="http://supnum.mr/bib">
    <emprunt>
        <numero>E-2026-0812</numero>
        <dateRetour>2026-12-08</dateRetour>
    </emprunt>
</emprunterLivreResponse>
```

**Interprétation :**

Le code HTTP `200 OK` indique que la requête a été traitée correctement.

Le service a retourné un numéro d'emprunt et une date de retour valide. Le test confirme le bon fonctionnement de l'opération `emprunterLivre` pour un livre disponible.

## 4. Deuxième test : livre indisponible

Nous avons envoyé une deuxième requête avec l'ISBN :

`978-2-10-074444-4`

Cet ISBN correspond au livre déclaré indisponible dans l'implémentation Java.

**Résultat obtenu :**

```text
HTTP/1.1 500 Internal Server Error
Content-type: text/xml; charset=utf-8
```

Corps de la réponse :

```xml
<?xml version='1.0' encoding='UTF-8'?>
<S:Envelope xmlns:S="http://schemas.xmlsoap.org/soap/envelope/">
    <S:Body>
        <S:Fault xmlns:ns4="http://www.w3.org/2003/05/soap-envelope">
            <faultcode>S:Server</faultcode>
            <faultstring>Livre indisponible</faultstring>
            <detail>
                <LivreIndisponible xmlns="http://supnum.mr/bib">
                    <isbn>978-2-10-074444-4</isbn>
                    <retourPrevu>2026-12-02</retourPrevu>
                </LivreIndisponible>
            </detail>
        </S:Fault>
    </S:Body>
</S:Envelope>
```

**Interprétation :**

Le serveur a retourné une SOAP Fault avec le code `S:Server`.

Le message indique que le livre est indisponible.

Le bloc `<detail>` contient les informations métier structurées :

- ISBN du livre : `978-2-10-074444-4`.
- Date de retour prévue : `2026-12-02`.

Contrairement à une erreur technique générale, cette Fault transmet une information métier précise au client.

Le code HTTP `500` est attendu pour cette Fault SOAP 1.1.

---

## Questions de l'étape 6

### Q1. Le WSDL publié est-il celui que vous avez fourni ? Contient-il le motif de l'ISBN et le type xs:date ?

Lorsque l'annotation `@WebService` contient `wsdlLocation = "wsdl/EmpruntService.wsdl"`, le service utilise le contrat WSDL fourni.

Le contrat initial contient :

- Le type personnalisé `IsbnType`.
- Le motif ISBN : `97[89]-\d-\d{2,5}-\d{2,6}-\d`.
- Le type `xs:date` pour `dateRetour`.
- Le type `xs:date` pour `retourPrevu`.
- Le namespace `http://supnum.mr/bib`.

Le motif ISBN est donc présent dans le contrat initial.

### Q2. Avez-vous dû modifier 01-emprunter.xml pour qu'il fonctionne ici ? Pourquoi ?

Oui, nous avons adapté le fichier XML de la requête pour respecter le contrat de la partie B.

La différence essentielle vient de la déclaration :

```xml
elementFormDefault="qualified"
```

Dans la partie A, les éléments enfants de `emprunterLivre` étaient non qualifiés.

Dans la partie B, les éléments locaux doivent appartenir au namespace `http://supnum.mr/bib`.

Nous avons donc utilisé le préfixe `bib` pour les éléments enfants :

```xml
<soap:Envelope
    xmlns:soap="http://schemas.xmlsoap.org/soap/envelope/"
    xmlns:bib="http://supnum.mr/bib">

    <soap:Body>
        <bib:emprunterLivre>
            <bib:idAdherent>A-117</bib:idAdherent>
            <bib:isbn>978-1-234-5678-9</bib:isbn>
        </bib:emprunterLivre>
    </soap:Body>
</soap:Envelope>
```

Cette modification permet de respecter la structure et les espaces de noms exigés par le schéma XSD du contrat.

### Q3. Sans @SchemaValidation, l'ISBN 978-2-10 est-il refusé ? Et avec ? Quelle Fault obtenez-vous ?

L'annotation utilisée est :

```java
@com.sun.xml.ws.developer.SchemaValidation
```

**Sans `@SchemaValidation` :**

La validation automatique du message SOAP par rapport au schéma XSD n'est pas activée.

Dans notre implémentation, la méthode ne contient pas de contrôle manuel du motif ISBN. Par conséquent, un ISBN comme `978-2-10` peut atteindre la logique métier et être traité comme un ISBN disponible.

Nous n'avons toutefois pas conservé de résultat expérimental séparé pour ce cas sans annotation.

**Avec `@SchemaValidation` :**

Nous avons ajouté l'annotation à la classe `EmpruntServiceImpl`, puis redémarré le serveur.

Nous avons envoyé une requête contenant l'ISBN invalide :

`978-2-10`

Commande utilisée :

```powershell
curl.exe -i -X POST "http://localhost:8080/bib/EmpruntService" `
-H "Content-Type: text/xml; charset=utf-8" `
-H 'SOAPAction: "http://supnum.mr/bib/emprunterLivre"' `
--data-binary "@xml/03-isbn-invalide.xml"
```

**Résultat réellement obtenu :**

```text
HTTP/1.1 500 Internal Server Error
Content-type: text/xml; charset=utf-8
```

Réponse SOAP :

```xml
<?xml version='1.0' encoding='UTF-8'?>
<S:Envelope xmlns:S="http://schemas.xmlsoap.org/soap/envelope/">
    <S:Body>
        <S:Fault xmlns:ns4="http://www.w3.org/2003/05/soap-envelope">
            <faultcode>S:Client</faultcode>
            <faultstring>
                org.xml.sax.SAXParseException;
                cvc-pattern-valid: Value '978-2-10' is not facet-valid
                with respect to pattern
                '97[89]-\d-\d{2,5}-\d{2,6}-\d'
                for type 'IsbnType'.
            </faultstring>
        </S:Fault>
    </S:Body>
</S:Envelope>
```

**Interprétation :**

Le serveur a rejeté la requête avant le traitement normal de l'opération.

- HTTP : `500 Internal Server Error`.
- `faultcode` : `S:Client`.
- Erreur de validation : `cvc-pattern-valid`.

La valeur `978-2-10` ne respecte pas l'expression régulière définie dans `IsbnType`.

Ce test démontre que `@SchemaValidation` permet d'appliquer les contraintes du schéma XML au message SOAP reçu.

### Q4. Retirez wsdlLocation de l'annotation et redémarrez : qu'est-ce qui change dans le WSDL publié ?

Nous avons supprimé la propriété :

```java
wsdlLocation = "wsdl/EmpruntService.wsdl"
```

L'annotation `@WebService` est alors devenue :

```java
@WebService(
    serviceName = "EmpruntService",
    portName = "EmpruntPort",
    targetNamespace = "http://supnum.mr/bib",
    endpointInterface = "mr.supnum.bib.contrat.EmpruntPortType"
)
```

Après recompilation et redémarrage, le serveur a continué à fonctionner.

Le WSDL était toujours accessible à l'adresse :

`http://localhost:8080/bib/EmpruntService?wsdl`

Nous avons enregistré le WSDL publié avec :

```powershell
curl.exe -s "http://localhost:8080/bib/EmpruntService?wsdl" -o wsdl-publie.wsdl
```

Puis nous avons examiné ses éléments principaux.

**Résultats observés dans le WSDL publié :**

| Élément | Valeur observée |
|---|---|
| Namespace | `http://supnum.mr/bib` |
| Service | `EmpruntService` |
| Port | `EmpruntPort` |
| PortType | `EmpruntPortType` |
| Opération | `emprunterLivre` |
| SOAPAction | `http://supnum.mr/bib/emprunterLivre` |
| Adresse | `http://localhost:8080/bib/EmpruntService` |

Le WSDL publié contient notamment :

```xml
<xsd:schema>
    <xsd:import
        namespace="http://supnum.mr/bib"
        schemaLocation="http://localhost:8080/bib/EmpruntService?xsd=1"/>
</xsd:schema>
```

Nous avons ensuite téléchargé le XSD publié :

```powershell
curl.exe -s "http://localhost:8080/bib/EmpruntService?xsd=1"
```

**Résultats réellement observés dans le XSD :**

1. Le schéma utilise `elementFormDefault="qualified"`.
2. Le champ `isbn` de la requête est publié comme `xs:string`.
3. Le type personnalisé `IsbnType` et son motif ISBN ne sont plus présents dans ce XSD.
4. Le champ `dateRetour` est toujours de type `xs:date`.
5. Le champ `retourPrevu` est également de type `xs:date`.

Extrait du XSD obtenu :

```xml
<xs:element name="emprunterLivre">
    <xs:complexType>
        <xs:sequence>
            <xs:element name="idAdherent" type="xs:string"/>
            <xs:element name="isbn" type="xs:string"/>
        </xs:sequence>
    </xs:complexType>
</xs:element>
```

Et pour le type `EmpruntType` :

```xml
<xs:complexType name="EmpruntType">
    <xs:sequence>
        <xs:element name="numero" type="xs:string"/>
        <xs:element name="dateRetour" type="xs:date"/>
    </xs:sequence>
</xs:complexType>
```

**Interprétation :**

Après la suppression de `wsdlLocation`, Metro génère automatiquement le WSDL à partir de l'interface Java.

Les informations principales du service sont conservées, mais toutes les contraintes du contrat initial ne sont pas reproduites.

En particulier, le motif ISBN disparaît du XSD publié, tandis que les types de dates restent correctement définis.

Cette comparaison montre l'importance de conserver le contrat WSDL lorsqu'on souhaite imposer des règles précises aux clients du service.

---

# Conclusion – Choisir et rendre

### Q1. Citez deux différences que vous avez observées entre le WSDL de la partie A et celui de la partie B.

Nous avons observé plusieurs différences entre les deux approches.

**Première différence : la manière de créer le contrat.**

- Dans la partie A (Code-first), le service est développé en Java et le WSDL est généré automatiquement à partir du code.
- Dans la partie B (Contract-first), le WSDL est défini en premier, puis les classes Java sont générées avec `wsimport`.

**Deuxième différence : les types et les contraintes des données.**

Dans la partie A, le champ `dateRetour` était publié comme `xs:string`.

Dans la partie B, le contrat initial définit `dateRetour` comme `xs:date` et utilise un type personnalisé `IsbnType` avec une expression régulière de validation.

Nos tests ont également montré qu'après la suppression de `wsdlLocation`, le XSD généré automatiquement ne conservait plus le motif ISBN.

### Q2. Si on renomme le champ dateRetour dans le code, quel projet voit son contrat changer ? Quel risque pour les clients ?

Dans le projet Code-first, le WSDL est généré à partir du code Java. Par conséquent, si le champ `dateRetour` est renommé dans le code, le WSDL généré peut également changer.

Dans le projet Contract-first, le WSDL constitue la référence du contrat. Une modification du code Java ne doit pas modifier automatiquement le contrat fourni.

Le principal risque est une rupture de compatibilité avec les clients qui utilisent l'ancien nom du champ.

Ces clients peuvent ne plus réussir à lire ou à envoyer correctement les données attendues.

### Q3. Quelle approche choisiriez-vous pour le service Paiement, appelé par Scolarité (.NET), Bibliothèque (PHP) et le portail (Node.js) ? Et pour un prototype interne de deux jours ?

Pour le service Paiement, je choisirais l'approche **Contract-first**.

En effet, le service est utilisé par plusieurs systèmes développés avec des technologies différentes : .NET, PHP et Node.js.

Un contrat WSDL commun permet à toutes les équipes de connaître précisément :

- Les opérations disponibles.
- Les paramètres des requêtes.
- Les formats des réponses.
- Les types de données.
- Les erreurs possibles.

Cela facilite l'interopérabilité et réduit les risques d'incompatibilité entre les systèmes.

Pour un prototype interne de deux jours, je choisirais plutôt **Code-first**.

Cette approche est plus rapide à mettre en place, car le développeur peut commencer directement par le code Java et laisser le framework générer le WSDL.

Elle convient donc mieux à un prototype limité, avec peu de consommateurs et des besoins susceptibles de changer rapidement.

---

## Bilan de la Partie B

Au cours de cette partie, nous avons réalisé les opérations suivantes :

- Création du contrat `EmpruntService.wsdl`.
- Génération des classes Java avec `wsimport`.
- Implémentation de l'interface générée `EmpruntPortType`.
- Publication du service SOAP sur le port 8080.
- Test d'un emprunt réussi avec une réponse HTTP 200.
- Test d'un livre indisponible avec une SOAP Fault métier.
- Activation de `@SchemaValidation`.
- Rejet d'un ISBN invalide avec une erreur `cvc-pattern-valid`.
- Suppression de `wsdlLocation`.
- Comparaison du WSDL et du XSD publiés avant et après cette suppression.

**Conclusion générale :**

Cette partie nous a permis de comprendre que l'approche Contract-first donne une place centrale au contrat WSDL. Elle permet de définir à l'avance les opérations, les types de données et les contraintes que les clients doivent respecter.

Elle est particulièrement adaptée aux services web utilisés par plusieurs applications ou équipes travaillant avec des langages et des technologies différents.