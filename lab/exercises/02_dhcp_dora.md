# Exercice 2 — DHCP : observer DORA paquet par paquet

**Durée estimée :** 45 min
**Objectif :** capturer un échange DHCP complet (Discover / Offer / Request /
ACK), identifier les options portées par chaque message, et comprendre
pourquoi DHCP utilise un broadcast L2 alors qu'IP n'est pas encore
configuré.

## Manipulation

Côté `dhcp-server`, démarrez une capture filtrée sur les ports DHCP (67/68)&nbsp;:

```bash
docker exec -it lab_dhcp_server tcpdump -i eth0 -nn -e -v port 67 or port 68
```

> Note&nbsp;: `-e` affiche les adresses MAC, indispensables pour comprendre
> le broadcast L2.

Côté `client`, déclenchez une nouvelle demande de bail&nbsp;:

```bash
docker exec lab_client bash -c "dhclient -r eth0 2>/dev/null; dhclient -v eth0"
```

Observez les **4 paquets** DORA dans la capture, puis arrêtez tcpdump (Ctrl+c).

Affichez aussi les journaux applicatifs du serveur&nbsp;:

```bash
docker logs --tail 40 lab_dhcp_server
```

## À rendre — répondez directement dans ce fichier

### 1. Tableau DORA

Complétez en vous appuyant sur **votre propre capture**&nbsp;: 

Capture d'écran des paquets DORA :

Affiche l'exécution de la commande dhclient où l'on voit le client passer par les quatre étapes et confirmer qu'il est "bound" (lié) à l'IP 172.20.1.133 : 

<img width="1025" height="260" alt="image" src="https://github.com/user-attachments/assets/485e31b4-c7e9-4819-8e54-e0aa6b668757" />

Fournit l'analyse tcpdump montrant la structure interne des paquets : 
<img width="1030" height="741" alt="image" src="https://github.com/user-attachments/assets/d1e495ba-9495-4863-b9d0-f2b9687cea61" />


| Étape       | Émetteur (IP src) | Destinataire (IP dst) | MAC src / dst | Options DHCP notables |
| ----------- | ----------------- | --------------------- | ------------- | --------------------- |
| 1. Discover | `0.0.0.0`         | `255.255.255.255`     | `7a:46:ce:08:66:01`   | `option 53 = Discover, option 12 = client, Option 55 = ....` |
| 2. Offer    | `172.20.1.2`      | `172.20.1.133`         | `7a:46:ce:08:66:01`      |`option 53 = Offer , option 54 = 172.20.1.2, Option 51 = 43200s ` |
| 3. Request  | `0.0.0.0`         | `255.255.255.255`     | `7a:46:ce:08:66:01 / ff:f..`    | `option 53 = Request , option 50 =172.20.1.133, option 54 = 172.20.1.2 `|
| 4. ACK      | `172.20.1.2`      | `172.20.1.133`         | `7a:46:ce:08:66:01`    |`option 53 = ACK , option 1 = 255.255.255.0, option 3 = 172.20.1.254, Option 6 =  1.1.1.1, 8.8.8.8 `|


### 2. Configuration finale du client

```bash
docker exec lab_client ip -4 addr show eth0
docker exec lab_client ip route
docker exec lab_client cat /etc/resolv.conf   # peut être vide si non géré par dhclient
```

Notez **l'IP attribuée, le masque, la passerelle, les DNS, la durée de bail**.

> 💬 **Votre réponse :**
> Configuration finale du client :
>
> <img width="802" height="353" alt="image" src="https://github.com/user-attachments/assets/de86f887-9bc7-4c31-9f5f-ff07fd298d12" />
>
> 

### 3. Questions de réflexion

**Question 1.** Pourquoi le client utilise-t-il **`0.0.0.0` comme IP
source** pour le Discover, alors que c'est une adresse non routable&nbsp;?
Que se passerait-il avec n'importe quelle autre adresse&nbsp;?

> 💬 **Votre réponse :**
>
> _Remplacez ce texte par votre réponse._

**Question 2.** Pourquoi le **Request** est-il **rediffusé en broadcast**
alors que le client connaît déjà l'IP du serveur après l'Offer&nbsp;?

> 💬 **Votre réponse :**
>
> _Remplacez ce texte par votre réponse._

**Question 3.** À quoi sert le **transaction ID (xid)** présent dans les
4 paquets&nbsp;? Que se passerait-il s'il était omis dans un réseau avec
plusieurs serveurs DHCP&nbsp;?

> 💬 **Votre réponse :**
>
> _Remplacez ce texte par votre réponse._

**Question 4.** Que renvoie le serveur si vous demandez explicitement une
adresse hors du pool (essayez `dhclient -v -s 172.20.1.99 eth0`)&nbsp;?
Justifiez.

> 💬 **Votre réponse :**
>
> _Remplacez ce texte par votre réponse._

**Question 5.** La directive `dhcp-authoritative` est active sur notre
serveur. Quel est son effet **comportemental** sur les NAK&nbsp;?

> 💬 **Votre réponse :**
>
> _Remplacez ce texte par votre réponse._

### 4. Renouvellement de bail (T1/T2)

Le bail est de 12&nbsp;h, T1 (renouvellement) à 6&nbsp;h, T2 (rebind) à 10&nbsp;h30.
En **2-3 phrases**, décrivez la différence entre un renouvellement T1 et
un rebind T2 (destinataire du paquet, comportement attendu).

> 💬 **Votre réponse :**
>
> _Remplacez ce texte par votre réponse._
