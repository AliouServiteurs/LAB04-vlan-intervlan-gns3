# LAB04 — VLAN et routage inter-VLAN avec GNS3

> **Niveau :** intermédiaire  
> **Technologies :** GNS3, Cisco 2691, switch Ethernet intégré, VPCS, VLAN 802.1Q  
> **Objectif :** segmenter deux postes dans des VLAN distincts et permettre leur communication via un routeur configuré en *Router-on-a-Stick*.

---

## 1. Présentation du laboratoire

Ce laboratoire met en œuvre une segmentation logique du réseau avec deux VLAN :

- **VLAN 10** : réseau `192.168.10.0/24`, utilisé par PC1.
- **VLAN 20** : réseau `192.168.20.0/24`, utilisé par PC2.

Le routeur Cisco 2691 assure le routage entre ces deux sous-réseaux à travers une seule interface physique et deux sous-interfaces logiques. Les sous-interfaces utilisent l'encapsulation IEEE 802.1Q pour identifier le VLAN auquel appartient chaque trame.

### Objectifs pédagogiques

À la fin du laboratoire, il doit être possible de :

- expliquer l'intérêt de la segmentation en VLAN ;
- associer un port d'accès à un VLAN ;
- configurer les sous-interfaces d'un routeur ;
- comprendre le rôle de l'encapsulation 802.1Q ;
- vérifier les interfaces et la table de routage ;
- tester la connectivité vers les passerelles et entre les hôtes.

## 2. Topologie

La topologie utilisée comprend deux VPCS, un switch Ethernet intégré à GNS3 et un routeur Cisco 2691.

```text
 PC1 (VLAN 10)                         PC2 (VLAN 20)
 192.168.10.10/24                      192.168.20.10/24
 GW 192.168.10.1                       GW 192.168.20.1
       |                                     |
      e0                                    e1
       \                                     /
        +--------- Switch Ethernet ---------+
                         e2
                          |
                    Fa0/0 — R1
                    /           \
          Fa0/0.10                 Fa0/0.20
          VLAN 10                  VLAN 20
       192.168.10.1             192.168.20.1
```

> **À insérer :** capture de la topologie GNS3.  
> Nom conseillé : `captures/01-topologie-lab04.png`

## 3. Plan d'adressage

| Équipement | Interface / port | VLAN | Adresse IP | Passerelle |
|---|---|---:|---|---|
| PC1 | VPCS, port e0 du switch | 10 | `192.168.10.10/24` | `192.168.10.1` |
| R1 | `FastEthernet0/0.10` | 10 | `192.168.10.1/24` | — |
| PC2 | VPCS, port e1 du switch | 20 | `192.168.20.10/24` | `192.168.20.1` |
| R1 | `FastEthernet0/0.20` | 20 | `192.168.20.1/24` | — |

L'interface physique `FastEthernet0/0` du routeur est reliée au port `e2` du switch. Elle ne porte pas directement d'adresse IP : les adresses sont configurées sur les sous-interfaces logiques.

## 4. Configuration du switch GNS3

Dans la configuration du switch Ethernet intégré :

| Port | Connexion | Configuration attendue |
|---|---|---|
| `e0` | PC1 | Port d'accès, VLAN 10 |
| `e1` | PC2 | Port d'accès, VLAN 20 |
| `e2` | R1 `FastEthernet0/0` | Trunk 802.1Q, VLAN 10 et VLAN 20 |

Le trunk permet de transporter sur un même lien les trames associées aux VLAN 10 et 20. Les ports d'accès relient chacun un poste à son VLAN.

> **À insérer :** capture de la configuration des ports/VLAN du switch.  
> Nom conseillé : `captures/02-configuration-vlan-switch.png`

> **Note de reproductibilité :** le switch Ethernet intégré de GNS3 n'a pas nécessairement les mêmes commandes CLI qu'un switch Cisco IOS. La configuration s'effectue ici via ses paramètres dans l'interface graphique. Les noms exacts des options peuvent varier selon la version de GNS3.

## 5. Configuration du routeur Cisco 2691

La configuration suivante utilise `FastEthernet0/0` comme interface reliée au trunk du switch.

```text
enable
configure terminal

interface FastEthernet0/0
 no ip address
 no shutdown
exit

interface FastEthernet0/0.10
 encapsulation dot1Q 10
 ip address 192.168.10.1 255.255.255.0
exit

interface FastEthernet0/0.20
 encapsulation dot1Q 20
 ip address 192.168.20.1 255.255.255.0
exit

end
```

### Explication des commandes

- `interface FastEthernet0/0` : sélectionne l'interface physique connectée au switch.
- `no ip address` : retire l'adresse IP de l'interface physique afin d'utiliser les sous-interfaces pour le routage VLAN.
- `no shutdown` : active l'interface physique.
- `interface FastEthernet0/0.10` : crée/sélectionne la sous-interface logique associée au VLAN 10.
- `encapsulation dot1Q 10` : associe les trames portant le tag 802.1Q du VLAN 10 à cette sous-interface.
- `ip address ...` : configure l'adresse de passerelle du VLAN.
- La sous-interface `.20` applique le même principe au VLAN 20.

**Précaution :** ne pas exécuter `no ip address` sur une interface qui assure déjà une autre connexion nécessaire au laboratoire. Vérifier d'abord le câblage et `show ip interface brief`.

## 6. Configuration des VPCS

### PC1 — VLAN 10

```text
ip 192.168.10.10/24 192.168.10.1
```

### PC2 — VLAN 20

```text
ip 192.168.20.10/24 192.168.20.1
```

Ces commandes définissent l'adresse IPv4, le masque `/24` et la passerelle par défaut de chaque VPCS.

## 7. Vérification de la configuration

Sur R1, les commandes de contrôle sont :

```text
show ip interface brief
show running-config
show ip route
```

La sortie de `show ip interface brief` obtenue pendant le laboratoire indiquait :

```text
Interface                  IP-Address      OK? Method Status                Protocol
FastEthernet0/0            unassigned      YES unset  up                    up
FastEthernet0/0.10         192.168.10.1    YES manual up                    up
FastEthernet0/0.20         192.168.20.1    YES manual up                    up
FastEthernet0/1            unassigned      YES unset  administratively down down
```

Les sous-interfaces `FastEthernet0/0.10` et `FastEthernet0/0.20` étaient `up/up`. Cela confirme leur état opérationnel, mais les tests de connectivité restent nécessaires pour vérifier le chemin de bout en bout.

> **À insérer :** capture des sous-interfaces et de la table de routage.  
> Noms conseillés : `captures/03-sous-interfaces-routeur.png` et `captures/06-table-routage.png`

## 8. Tests de connectivité

### Tests depuis PC1

```text
ping 192.168.10.1
ping 192.168.20.1
ping 192.168.20.10
```

### Tests depuis PC2

```text
ping 192.168.20.1
ping 192.168.10.1
ping 192.168.10.10
```

### Résultats observés

Les tests effectués depuis PC1 et PC2 vers les deux passerelles ont réussi : chaque ping de cinq requêtes a reçu cinq réponses. Les pings directs entre PC1 et PC2 ont également été confirmés par l'utilisateur comme fonctionnels.

| Test | Résultat |
|---|---|
| PC1 → `192.168.10.1` | Réussi, 5 réponses sur 5 |
| PC1 → `192.168.20.1` | Réussi, 5 réponses sur 5 |
| PC2 → `192.168.20.1` | Réussi, 5 réponses sur 5 |
| PC2 → `192.168.10.1` | Réussi, 5 réponses sur 5 |
| PC1 ↔ PC2 | Réussi, confirmé lors du test final |

> **À insérer :** captures des tests de ping.  
> Noms conseillés : `captures/04-ping-passerelles.png` et `captures/05-ping-inter-vlan.png`

## 9. Comment fonctionne le routage inter-VLAN ?

1. PC1 envoie un paquet destiné à une adresse située hors de son réseau `192.168.10.0/24` vers sa passerelle `192.168.10.1`.
2. Le switch transmet le trafic sur le trunk vers R1, avec l'identification VLAN appropriée.
3. La sous-interface `FastEthernet0/0.10` reçoit le trafic du VLAN 10.
4. R1 consulte sa table de routage et transmet le paquet vers le réseau `192.168.20.0/24` via `FastEthernet0/0.20`.
5. Le routeur transmet le paquet sur le trunk avec l'identification du VLAN 20.
6. Le switch livre la trame à PC2 sur son port d'accès du VLAN 20.

La communication en sens inverse suit le chemin correspondant dans l'autre direction.

## 10. Dépannage

Si les pings échouent, vérifier dans cet ordre :

1. Les adresses IP, masques et passerelles des VPCS.
2. Le port d'accès de PC1 dans le VLAN 10 et celui de PC2 dans le VLAN 20.
3. Le port relié à R1 configuré comme trunk 802.1Q, transportant les VLAN 10 et 20.
4. Le câblage vers `FastEthernet0/0` du routeur.
5. L'état des interfaces avec `show ip interface brief`.
6. Les adresses et encapsulations des sous-interfaces avec `show running-config`.
7. La présence des réseaux directement connectés avec `show ip route`.

Une interface `up/up` ne suffit pas, à elle seule, à garantir que le trunk transporte correctement les deux VLAN. Il faut valider avec les tests de bout en bout.

## 11. Captures d'écran recommandées

Créer un dossier `captures/` à la racine du laboratoire et y placer les images suivantes :

```text
captures/
├── 01-topologie-lab04.png
├── 02-configuration-vlan-switch.png
├── 03-sous-interfaces-routeur.png
├── 04-ping-passerelles.png
├── 05-ping-inter-vlan.png
└── 06-table-routage.png
```

N'inclure que des captures réelles issues de votre environnement. Les noms de fichiers ci-dessus sont des suggestions pour organiser le dépôt.

## 12. Conclusion

Ce laboratoire a permis de mettre en place deux réseaux logiquement séparés au moyen des VLAN 10 et 20, puis d'assurer leur communication grâce au routage inter-VLAN sur un routeur Cisco 2691. La configuration des sous-interfaces 802.1Q et les tests de connectivité ont validé le fonctionnement de la topologie.

Ce type de configuration constitue une base pour approfondir la segmentation réseau, les ACL inter-VLAN, la supervision du trafic et les mécanismes de sécurité réseau.

---

## Structure GitHub suggérée

```text
LAB04-vlan-intervlan-gns3/
├── README.md
├── captures/
│   ├── 01-topologie-lab04.png
│   ├── 02-configuration-vlan-switch.png
│   ├── 03-sous-interfaces-routeur.png
│   ├── 04-ping-passerelles.png
│   ├── 05-ping-inter-vlan.png
│   └── 06-table-routage.png
└── projet/
    └── lab04.gns3
```

Le nom exact du fichier de projet GNS3 peut être différent. Copiez le fichier de projet et ses éventuels fichiers associés dans `projet/` en conservant la structure produite par GNS3. Ne publiez pas d'images IOS propriétaires ni d'autres fichiers soumis à licence sans autorisation.
