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

| Étape       | Émetteur (IP src) | Destinataire (IP dst) | MAC src / dst                                 | Options DHCP notables |
| ----------- | ----------------- | --------------------- | -------------                                 | --------------------- |
| 1. Discover | `0.0.0.0`         | `255.255.255.255`     | … b2:94:fc:87:4d:e6  ff:ff:ff:ff:ff:ff        | option 53 = …, option 55 = … |
| 2. Offer    | … 172.20.1.2      | …    172.20.1.124     | … 46:91:4e:00:fb:e0 → b2:94:fc:87:4d:e6       | Option 53 = Offer Your-IP = 172.20.1.124                                                                                                                       LeaseTime = 43200s (12h) |
| 3. Request  | … 0.0.0.0         | …   255.255.255.255   | … b2:94:fc:87:4d:e6 → ff:ff:ff:ff:ff:ff       | Option 53 = Request
                                                                                                                Option 50 (Requested IP) = 172.20.1.124
                                                                                                                    Option 54 (Server ID) = 172.20.1.2 |
| 4. ACK      | … 172.20.1.2      | …   172.20.1.124      | … 46:91:4e:00:fb:e0 → b2:94:fc:87:4d:e6       | Option 53 = ACK Attribution officielle des paramètres (DNS, GW, Masque) |

### 2. Configuration finale du client

```bash
docker exec lab_client ip -4 addr show eth0
docker exec lab_client ip route
docker exec lab_client cat /etc/resolv.conf   # peut être vide si non géré par dhclient
```

Notez **l'IP attribuée, le masque, la passerelle, les DNS, la durée de bail**.

> 💬 **Votre réponse :**
>
> IP attribuée : 172.20.1.124

Masque de sous-réseau : /24 (soit 255.255.255.0)

Passerelle (Gateway) : 172.20.1.1

Serveurs DNS : 1.1.1.1 et 8.8.8.8

Durée de bail (Lease Time) : 43200 secondes (soit exactement 12 heures, comme indiqué par le valid_lft d'environ 42929s restantes).
### 3. Questions de réflexion

**Question 1.** Pourquoi le client utilise-t-il **`0.0.0.0` comme IP
source** pour le Discover, alors que c'est une adresse non routable&nbsp;?
Que se passerait-il avec n'importe quelle autre adresse&nbsp;?

> 💬 **Votre réponse :**
>
> car c'est pour voir tous les réseaux

**Question 2.** Pourquoi le **Request** est-il **rediffusé en broadcast**
alors que le client connaît déjà l'IP du serveur après l'Offer&nbsp;?

> 💬 **Votre réponse :**
>
> Quand il accepté une offre en cas de plusieurs DHCP il faut indiqué qu'il à déjà accepté une offre et donc aussi renvoyé a réponse au sevruer qu quel il a accetpé

**Question 3.** À quoi sert le **transaction ID (xid)** présent dans les
4 paquets&nbsp;? Que se passerait-il s'il était omis dans un réseau avec
plusieurs serveurs DHCP&nbsp;?

> 💬 **Votre réponse :**
>
> sert d'identifiant unique pour l'ensemble d'un échange DORA. Il permet au client d'associer avec certitude les réponses du serveur (Offer et ACK) à sa propre demande (Discover).

**Question 4.** Que renvoie le serveur si vous demandez explicitement une
adresse hors du pool (essayez `dhclient -v -s 172.20.1.99 eth0`)&nbsp;?
Justifiez.

> 💬 **Votre réponse :**
>
> docker exec lab_client dhclient -v -s 172.20.1.2 172.20.1.99 eth0
Internet Systems Consortium DHCP Client 4.4.3-P1
Copyright 2004-2022 Internet Systems Consortium.
All rights reserved.
For info, please visit https://www.isc.org/software/dhcp/

Cannot find device "172.20.1.99"
Listening on LPF/eth0/b2:94:fc:87:4d:e6
Sending on   LPF/eth0/b2:94:fc:87:4d:e6
Failed to get interface index: No such device

If you think you have received this message due to a bug rather
than a configuration issue please read the section on submitting
bugs on either our web page at www.isc.org or in the README file
before submitting a bug.  These pages explain the proper
process and the information we find helpful for debugging.

exiting.

**Question 5.** La directive `dhcp-authoritative` est active sur notre
serveur. Quel est son effet **comportemental** sur les NAK&nbsp;?

> 💬 **Votre réponse :**
>
> si il voit une ip anormal venat d'un dhcp il lui ofrce à prendre une nouvelle ip

### 4. Renouvellement de bail (T1/T2)

Le bail est de 12&nbsp;h, T1 (renouvellement) à 6&nbsp;h, T2 (rebind) à 10&nbsp;h30.
En **2-3 phrases**, décrivez la différence entre un renouvellement T1 et
un rebind T2 (destinataire du paquet, comportement attendu).

> 💬 **Votre réponse :**
>
> La premiere demande renouvellement est à 50% si pas réussit la deuxiemme demande est à 87.5%
