# Exercice 3 — DHCP : obtenir une adresse IP automatiquement

**Niveau :** ⭐⭐ · **Durée :** ~1 h
**📖 Fiche théorique à lire avant :** `https://<votre-site>.pythonanywhere.com/dhcp`

## 🎯 Objectifs

- Constater qu'une machine **sans adresse IP** ne peut pas communiquer.
- Capturer les **4 messages DHCP** (Discover, Offer, Request, ACK = « DORA »).
- Retrouver dans ces messages la configuration reçue : IP, masque, passerelle, DNS, durée du bail.

> 🖥️ Préparation : lab démarré. Ouvrez **deux terminaux client** 💻 et gardez un terminal Codespace 🖥️
> (à la racine du dépôt).

---

## Partie A — Un client sans adresse

### Étape 1 — État de départ

💻 **Terminal client n°1** — notez l'adresse actuelle du client :

```bash
ip -br addr show eth0
```

👀 Vous voyez `172.20.1.50/24`. Cette adresse a été fixée **à la main** (par la configuration
du lab) : c'est une adresse **statique**.

### Étape 2 — Supprimer l'adresse

💻 **Terminal client n°1** — retirez l'adresse IP et faites oublier au client tout bail DHCP précédent :

```bash
ip addr flush dev eth0
rm -f /var/lib/dhcp/dhclient.leases
ip -br addr show eth0
```

💻 Essayez de joindre le routeur :

```bash
ping -c 2 172.20.1.254
```

**Question A.1.** Quel message obtenez-vous ? Pourquoi le client ne peut-il plus rien envoyer ?

> 💬 **Votre réponse :**
>
> _Car nous avons supprimer l'adresse Ip du client il ne peut donc plus communiquer avec le routeur._

---

## Partie B — Capturer l'échange DORA

### Étape 3 — Lancer la capture

💻 **Terminal client n°1** — enregistrez les paquets DHCP (ports UDP 67 et 68) :

```bash
rm -f /tmp/dhcp.pcap
tcpdump -i eth0 -U -nn -c 4 -w /tmp/dhcp.pcap port 67 or port 68
```

👀 `listening on eth0…` puis attente.

### Étape 4 — Demander une adresse

💻 **Terminal client n°2** — lancez le client DHCP en mode bavard (`-v`) :

```bash
dhclient -v eth0
```

👀 Vous devez voir les 4 étapes, dans cet ordre :

```
DHCPDISCOVER on eth0 to 255.255.255.255 port 67 ...
DHCPOFFER of 172.20.1.1xx from 172.20.1.2
DHCPREQUEST for 172.20.1.1xx on eth0 to 255.255.255.255 port 67
DHCPACK of 172.20.1.1xx from 172.20.1.2
bound to 172.20.1.1xx -- renewal in ... seconds.
```

Et dans le terminal n°1, tcpdump s'arrête : `4 packets captured`.

**Question B.1.** Recopiez les 5 lignes affichées par `dhclient`. Quelle adresse IP avez-vous obtenue ?
Qui vous l'a donnée ?

> 💬 **Votre réponse :**
>
> _DHCPDISCOVER on eth0 to 255.255.255.255 port 67 interval 3
DHCPOFFER of 172.20.1.147 from 172.20.1.2
DHCPREQUEST for 172.20.1.147 on eth0 to 255.255.255.255 port 67
DHCPACK of 172.20.1.147 from 172.20.1.2
bound to 172.20.1.147 -- renewal in 18290 seconds._

### Étape 5 — Analyser les 4 messages

🖥️ **Terminal Codespace** — listez les paquets capturés :

```bash
./lab/exercises/osi_inspect.py --pcap /tmp/dhcp.pcap
```

🖥️ Puis détaillez chacun d'eux (paquets 1 à 4) :

```bash
./lab/exercises/osi_inspect.py 1 --pcap /tmp/dhcp.pcap
./lab/exercises/osi_inspect.py 2 --pcap /tmp/dhcp.pcap
./lab/exercises/osi_inspect.py 3 --pcap /tmp/dhcp.pcap
./lab/exercises/osi_inspect.py 4 --pcap /tmp/dhcp.pcap
```

**Question B.2.** Complétez le tableau DORA avec **votre** capture :

| Étape       | IP source | IP destination    | MAC destination   | Type DHCP (opt 53) | Transaction ID |
| ----------- | --------- | ----------------- | ----------------- | ------------------ | -------------- |
| 1. Discover | `0.0.0.0` | `255.255.255.255` | ff:ff:ff:ff:ff:ff              | Discover             | 0x2daead74              |
| 2. Offer    | 172.20.1.2         | 172.20.1.147                 | fe:b8:ec:36:77:63                 | Offer                 | 0x2daead74              |
| 3. Request  | 0.0.0.0         | 255.255.255.255                 | ff:ff:ff:ff:ff:ff                 | Request                  | 0x2daead74              |
| 4. ACK      | 172.20.1.2         | 172.20.1.147                 | fe:b8:ec:36:77:63                 | ACK                  | 0x2daead74              |

**Question B.3.** Dans le paquet **Offer** (n°2), relevez la configuration proposée par le serveur :

| Information              | Option DHCP | Valeur |
| ------------------------ | ----------- | ------ |
| Adresse IP proposée      | (yiaddr)    | 172.20.1.147     |
| Masque de sous-réseau    | 1           | 255.255.255.0     |
| Passerelle (routeur)     | 3           | 172.20.1.254      |
| Serveurs DNS             | 6           | 1.1.1.1, 8.8.8.8      |
| Durée du bail (secondes) | 51          | 43200      |

Convertissez la durée du bail en heures.

> 💬 **Votre réponse (conversion) :**
>
> _12H._

---

## Partie C — Vérifier la configuration reçue

### Étape 6 — Le client est configuré

💻 **Terminal client n°2** :

```bash
ip -br addr show eth0
ip route
cat /etc/resolv.conf
ping -c 2 172.20.1.254
```

**Question C.1.** Retrouvez-vous dans `ip addr`, `ip route` et `resolv.conf` les valeurs de la
question B.3 ? Le ping vers le routeur fonctionne-t-il à nouveau ?

> 💬 **Votre réponse :**
>
> _Le ping vers le routeur fonctionne car nous avons la configuration réseau._

### Étape 7 — Le point de vue du serveur

🖥️ **Terminal Codespace** — lisez le journal du serveur DHCP, en ne gardant que les lignes des 4 étapes :

```bash
docker logs lab_dhcp_server 2>&1 | grep -E "DHCP(DISCOVER|OFFER|REQUEST|ACK)"
```

👀 Chaque ligne contient le *transaction ID* (en décimal), l'étape (`DHCPDISCOVER`, `DHCPOFFER`…),
l'adresse IP et l'adresse MAC concernées.

> 💡 Pour voir **tout** le journal (options envoyées, etc.) : `docker logs --tail 40 lab_dhcp_server`.

**Question C.2.** Retrouvez les 4 étapes DORA dans le journal. Quelle **adresse MAC** le serveur
a-t-il associée à l'IP attribuée ?

> 💬 **Votre réponse :**
>
> _Oui, on retrouve exactement les mêmes paramètres réseau que ceux proposés par le serveur DHCP dans le paquet Offer (question B.3) :

ip addr : l'interface eth0 a bien reçu l'adresse IP 172.20.1.147/24 (soit le masque 255.255.255.0).

ip route : la passerelle par défaut configurée est bien 172.20.1.254 (default via 172.20.1.254 dev eth0).

/etc/resolv.conf : les serveurs DNS configurés sont bien 1.1.1.1 et 8.8.8.8.._

---

## Partie D — Comprendre

**Question D.1.** Pourquoi le client utilise-t-il `0.0.0.0` comme **IP source** dans le Discover ?

> 💬 **Votre réponse :**
>
> _0.0.0.0 est utiliser pour dire que c'est une adresse non spécifier  pour communiquer avec le serveur car il ne poss._

**Question D.2.** Pourquoi le Discover est-il envoyé à `255.255.255.255` et à la MAC `ff:ff:ff:ff:ff:ff` ?
Que signifient ces adresses ?

> 💬 **Votre réponse :**
>
> _Les deux sont utiliser pour les broadcasts ._

**Question D.3.** À quoi sert le **Transaction ID**, identique dans les 4 messages ?
(Imaginez 30 PC qui démarrent en même temps sur le même réseau.)

> 💬 **Votre réponse :**
>
> _Il permet à chaque client d'identifier les messages qui lui sont destinés. Si 30 PC démarrent en même temps, chaque machine utilise son propre Transaction ID unique pour ignorer les réponses destinées aux autres.._

**Question D.4.** Que se passe-t-il quand le **bail expire** ? Pourquoi les baux ont-ils une durée
limitée plutôt qu'infinie ?

> 💬 **Votre réponse :**
>
> _À l'expiration du bail (sans renouvellement), le client perd son IP et doit recommencer un cycle DORA complet. La durée est limitée pour recycler automatiquement les adresses des appareils déconnectés et éviter la pénurie d'IP sur le réseau._

---

## 🚀 Pour aller plus loin (facultatif)

**P.1 — Rendre son adresse.** 💻 Libérez le bail puis redemandez-en un :

```bash
dhclient -r -v eth0
ip -br addr show eth0
dhclient -v eth0
```

Quel message est envoyé par `dhclient -r` ? Obtenez-vous la même adresse ensuite ? Pourquoi, d'après vous ?

> 💬 **Votre réponse :**
>
> _Remplacez ce texte par votre réponse._

**P.2 — Panne du serveur DHCP.** 🖥️ Arrêtez le serveur, puis 💻 redemandez une adresse :

```bash
# 🖥️ Terminal Codespace (dans le dossier lab)
docker compose stop dhcp-server
```

```bash
# 💻 Terminal client
dhclient -r eth0
rm -f /var/lib/dhcp/dhclient.leases
timeout 30 dhclient -v -1 eth0
```

Que se passe-t-il ? Quelles conséquences dans une entreprise si le serveur DHCP tombe en panne ?
**Pensez à le redémarrer** : 🖥️ `docker compose start dhcp-server`.

> 💬 **Votre réponse :**
>
> _Remplacez ce texte par votre réponse._

---

## ✅ Ce qu'il faut retenir

- Sans adresse IP, une machine ne peut pas communiquer : DHCP la configure **automatiquement**.
- L'échange se fait en 4 messages : **D**iscover, **O**ffer, **R**equest, **A**CK.
- Le client commence en **broadcast** (il ne connaît encore personne) depuis `0.0.0.0`.
- Le serveur fournit IP, masque, passerelle, DNS, et une **durée de bail**.

## 🆘 En cas de problème

| Problème | Solution |
| -------- | -------- |
| Seulement 2 paquets (REQUEST/ACK), pas de Discover | Le client se souvenait d'un ancien bail. 💻 `dhclient -r eth0; rm -f /var/lib/dhcp/dhclient.leases` puis recommencez l'étape 3 |
| `hostname: you must be root to change the host name` | Message sans importance, ignorez-le |
| `dhclient` attend sans fin | Le serveur DHCP est-il démarré ? 🖥️ `docker compose ps` |
| Remettre le client dans son état initial (`172.20.1.50`) | 🖥️ `docker compose restart client` |
