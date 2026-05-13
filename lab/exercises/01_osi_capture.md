# Exercice 1 — OSI à travers une vraie capture de paquets

**Durée estimée :** 45 min
**Objectif :** relier chaque couche du modèle OSI à un champ concret observable
dans une capture réseau effectuée sur le lab.

## Préparation

Configurez le client (s'il ne l'a pas déjà été par DHCP — voir exercice 2)&nbsp;:

```bash
docker exec lab_client bash -c "ip route del default 2>/dev/null; ip route add default via 172.20.1.254"
```

Vérifiez l'accès au site « public »&nbsp;:

```bash
docker exec lab_client curl -s http://172.20.0.10/whoami
```

## Manipulation

**Étape 1 — supprimez une éventuelle capture précédente** (sinon échec en
*Permission denied* car tcpdump abandonne ses privilèges vers l'utilisateur
`tcpdump` après ouverture du fichier) :

```bash
docker exec lab_client rm -f /tmp/http.pcap
```

**Étape 2 — lancez la capture côté client.** Les flags importants&nbsp;:
`-U` (écriture non bufferisée, indispensable si la capture est interrompue),
`-c 30` (s'arrête automatiquement après 30 paquets, ≈ 2 à 3 requêtes HTTP) :

```bash
docker exec lab_client tcpdump -i eth0 -U -w /tmp/http.pcap -nn -c 30 host 172.20.0.10
```

> ⚠️ Cette commande **bloque** le terminal tant que les 30 paquets ne sont
> pas capturés. Ouvrez un **second terminal** pour l'étape 3.

**Étape 3 — dans un second terminal, déclenchez du trafic** *pendant* que
tcpdump tourne&nbsp;:

```bash
docker exec lab_client curl -v http://172.20.0.10/
docker exec lab_client curl -v http://172.20.0.10/whoami
docker exec lab_client curl -s http://172.20.0.10/        # complément pour atteindre 30 paquets
```

tcpdump s'arrête seul dès le 30e paquet et affiche un résumé du type
`30 packets captured / 0 packets dropped by kernel`.

**Étape 4 — vérifiez que la capture est exploitable** :

```bash
docker exec lab_client capinfos /tmp/http.pcap
```

`Number of packets` doit être > 0. Si la capture est vide, voir la section
*Pièges fréquents* en bas de cet énoncé.

**Étape 5 — analysez la capture**. Trois vues utiles&nbsp;:

```bash
# Vue compacte : une ligne par paquet (utile pour repérer les n° de frames)
docker exec lab_client tshark -r /tmp/http.pcap

# Vue détaillée d'un paquet précis (ex. la requête GET = frame n°4)
docker exec lab_client tshark -r /tmp/http.pcap -V -Y 'frame.number == 4'

# Vue détaillée complète (pager : utilisez 'q' pour quitter)
docker exec -it lab_client sh -c "tshark -r /tmp/http.pcap -V | less"
```

> `-V` produit la décomposition complète couche par couche
> (Frame → Ethernet → IP → TCP → HTTP). Le filtre `-Y` cible un paquet
> par son numéro pour éviter d'avoir à scroller dans toute la trace.

## Visualisation assistée : `osi_inspect.py`

La sortie brute de `tshark -V` est dense (plusieurs centaines de lignes par
paquet). Un script Python est fourni dans ce dossier pour vous présenter,
pour chaque trame, un **tableau structuré par couche OSI** avec en plus
une **colonne d'explication pédagogique** pour chaque champ.

### Lister les trames

Depuis la racine du dépôt (l'hôte, pas l'intérieur du conteneur) :

```bash
./lab/exercises/osi_inspect.py
```

Vous obtenez la même vue compacte que `tshark` mais sans avoir à taper la
commande complète. Repérez la trame qui vous intéresse — typiquement la
trame portant `GET /` (ligne marquée `HTTP 141 GET / HTTP/1.1`).

### Détailler une trame

```bash
./lab/exercises/osi_inspect.py 4         # trame n°4 — la requête HTTP
./lab/exercises/osi_inspect.py 1         # trame n°1 — le SYN du handshake
./lab/exercises/osi_inspect.py 8         # trame n°8 — la réponse 200 OK
```

Le script affiche, pour chaque couche OSI **présente** dans la trame, les
champs clés avec **3 informations** :

| Colonne       | Contenu                                       |
| ------------- | --------------------------------------------- |
| `Champ`       | Nom du champ tel qu'extrait par tshark        |
| `Valeur`      | Valeur réelle observée dans **votre** capture |
| `Explication` | À quoi sert ce champ, comment l'interpréter   |

C'est exactement la matière dont vous avez besoin pour remplir le tableau
*« Couche / Élément observé / Valeur exemple »* demandé dans la section
suivante.

### Réutilisation pour les exercices suivants

Le script est générique. Pour disséquer une capture DHCP (exercice 2) ou
NAT (exercice 3), pointez-le vers le bon conteneur et le bon fichier :

```bash
./lab/exercises/osi_inspect.py 1 --pcap /tmp/dhcp.pcap --container lab_dhcp_server
./lab/exercises/osi_inspect.py 3 --pcap /tmp/nat.pcap  --container lab_nat_router
```

### Travail demandé avec ce script

1. Lancez `./lab/exercises/osi_inspect.py` pour obtenir la liste des trames.
2. Identifiez **une trame contenant du HTTP** (typiquement la requête `GET /`)
   et **une trame de contrôle TCP** (SYN, ACK seul, ou FIN).
3. Lancez le script avec le n° de chaque trame et **copiez la sortie**
   dans le README de votre fork (bloc de code).
4. Pour chacune des deux trames, **comptez et nommez** les couches OSI
   visibles (utilisez la ligne `Pile présente : …` en en-tête). Expliquez
   en 1 phrase pourquoi la couche 7 est absente sur la trame de contrôle TCP.

> 💬 **Votre réponse (sorties du script + analyse) :**
>
> une trame contenant du HTTP :

La trame 4 montre une requête HTTP complète traversant 4 couches : le client 172.20.1.50 envoie via Ethernet (MAC 7a:46:ce:08:66:01) un paquet IPv4 TCP sur le port 50300 → 80 avec les flags PSH+ACK, portant en couche applicative la méthode GET / en HTTP/1.1 émise par curl/7.88.1. 


<img width="1089" height="393" alt="image" src="https://github.com/user-attachments/assets/20f0297c-afad-4402-9e98-d0a6afa75970" />



 une trame de contrôle TCP :

La trame 1 est un paquet de contrôle TCP pur qui ne contient que 3 couches : le client 172.20.1.50 initie une connexion vers le port 80 avec le flag SYN et un numéro de séquence à 0, sans aucune donnée applicative — c'est pourquoi la couche 7 (HTTP) est absente.


 <img width="1076" height="378" alt="image" src="https://github.com/user-attachments/assets/9048524a-d043-4656-828d-e2ee0b464ae4" />



## À rendre — répondez directement dans ce fichier

Pour **chaque couche OSI**, donnez **un exemple concret extrait de votre
capture** (champ, valeur observée). Justifiez en 1-2 phrases.

| Couche OSI         | Élément observé dans la capture | Valeur exemple | 
| ------------------ | ------------------------------- | -------------- |
| 7 — Application    | _ex. méthode HTTP_              | `GET /whoami HTTP/1.1` |
| 6 — Présentation   | _ex. encodage / Content-Type_   | `Content-Type: text/html` |
| 5 — Session        | _ex. Keep-Alive, cookies_       | `Connection: keep-alive `  |
| 4 — Transport      | _ex. port TCP, flags_           | `50300 → 80 [PSH, ACK]`|
| 3 — Réseau         | _ex. IP source / destination_   |`172.20.1.50 → 172.20.0.10`|
| 2 — Liaison        | _ex. adresses MAC_              |`7a:46:ce:08:66:01 → 1a:f6:ac:76:c7:f6` |
| 1 — Physique       | _non visible — pourquoi&nbsp;?_ |` ...   ` |


7 - Couche Application :  La méthode GET et l'URI / sont les champs applicatifs HTTP visibles dans la trame 4 — ils portent le sens métier de la requête entre curl et le serveur.

6 - Couche Présentation : L'en-tête Content-Type: text/html dans la réponse nginx indique le format d'encodage des données renvoyées au client 

5 - Couche Session : L'en-tête Connection: keep-alive maintient la connexion TCP ouverte entre les trois requêtes curl successives

4 - Couche Transport : Le port source 50300 est le port éphémère choisi par le client, 80 identifie le service HTTP.

3 - Couche Réseau : L'IP 172.20.1.50 est l'adresse logique du client sur le LAN.

2 - Couche Liason : La MAC source 7a:46:ce:08:66:01 est l'interface eth0 du client, la MAC destination 1a:f6:ac:76:c7:f6 est l'interface LAN du nat-router

1 - Couche Physique : La couche Physique est gérée exclusivement par le hardware de la carte réseau.


## Questions de réflexion

**Question 1.** Pourquoi l'**adresse MAC source** observée n'est-elle
**pas** celle du serveur `internet` mais celle du `nat-router`&nbsp;? Que
vous apprend cette observation sur la portée de chaque couche&nbsp;?

> 💬 Votre réponse :
>Cela s'explique par la portée locale de la couche 2 : une adresse MAC n'est valable que sur un seul segment Ethernet (bridge (lan)). À chaque fois qu'un paquet franchit un routeur, celui-ci réécrit les adresses MAC pour les adapter au lien suivant, tandis que les adresses IP (Couche 3) restent inchangées de bout en bout.

**Question 2.** Vous capturez sur `eth0` du client (côté LAN). Dans votre
trace, l'**IP source** sortante est `172.20.1.50`. Pourtant, `curl /whoami`
rapporte que le serveur perçoit `172.20.0.254`. Expliquez cette différence
et indiquez **où** il faudrait capturer pour voir l'IP réécrite.
*Astuce&nbsp;:* `docker exec lab_nat_router tcpdump -i any -nn -c 10 host 172.20.0.10`.

> 💬 Votre réponse : Sur eth0 du client, l'IP source est bien 172.20.1.50 — c'est l'adresse privée du client sur le LAN. Mais avant de sortir vers le réseau 172.20.0.0/24, le nat-router applique une règle qui remplace cette IP privée par son IP WAN 172.20.0.254. Le serveur nginx ne voit donc jamais 172.20.1.50, seulement 172.20.0.254.
>
> Une commande permet cela : On y verra 172.20.0.254 comme IP source dans les paquets sortants.
>
> <img width="1045" height="271" alt="image" src="https://github.com/user-attachments/assets/2b50ab0d-9a19-4623-b19e-8b9a6230a831" />

> 

**Question 3.** Lancez `curl -v https://...` vers un site HTTPS public
(depuis l'hôte, pas le lab). Quelle couche change visiblement par
rapport au HTTP du lab&nbsp;? Quelles couches **disparaissent** de votre
visibilité&nbsp;?

> 💬 Votre réponse : Lancement de la commande -  La couche 6 (Présentation) apparaît clairement avec HTTPS : le handshake TLS 1.3 est entièrement visible
>
> <img width="941" height="719" alt="image" src="https://github.com/user-attachments/assets/f0f6ea69-554f-40d6-8e7b-0abb8ef6288f" />

>
>En revanche, la couche 7 (Application) disparaît de la visibilité d'une capture tcpdump : les headers et le corps sont chiffrés par TLS et ne seraient plus lisibles, Seul curl -v peut les afficher ici car il opère au-dessus de TLS
>

**Question 4.** La couche 5 (Session) est très peu visible dans une
capture HTTP/1.1. Donnez **deux mécanismes applicatifs** qui jouent le
rôle de la couche session, et expliquez pourquoi ils sont implémentés
« plus haut »&nbsp;dans la pile.

> 💬 Votre réponse : 
>
>1 : Les cookies HTTP (Set-Cookie / Cookie:) : HTTP est sans état par nature, chaque requête est indépendante et le serveur ne reconnaît pas naturellement un client d'une requête à l'autre. Les cookies >permettent au serveur d'identifier un même client à travers plusieurs requêtes successives et de maintenir une session logique persistante — c'est exactement le rôle de la couche 5 : ouvrir, maintenir et >identifier un dialogue.
>
>2 : Les streams HTTP/2 : comme on l'a observé dans la capture curl -v https..., HTTP/2 multiplex plusieurs échanges numérotés sur une seule connexion TCP. Chaque stream est une >session individuelle identifiée, ouverte et fermée indépendamment — sans avoir à rouvrir une connexion TCP à chaque fois.
>
>Ces deux mécanismes sont implémentés en couche 7
>
>

## Pièges fréquents

* **Capture vide (`Number of packets: 0`)** — vous avez lancé `curl` *avant*
  tcpdump, ou tcpdump a été tué avant d'écrire son buffer. Solution : utilisez
  bien `-U -c 30` (étape 2) et déclenchez le trafic *après* le message
  `listening on eth0…`.
* **`tcpdump: /tmp/http.pcap: Permission denied`** — un fichier appartenant à
  l'utilisateur `tcpdump` (créé par une capture précédente) bloque
  l'écriture. Solution : `docker exec lab_client rm -f /tmp/http.pcap`.
* **`tshark … | less` n'affiche rien** — vous êtes dans un environnement
  sans TTY (script, pipeline). Retirez `| less` ou utilisez
  `docker exec -it lab_client sh -c "… | less"`.
