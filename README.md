  # VoIP

## Introduction
La VoIP (Voice over Internet Protocol) permet la transmission de la voix via Internet ou tout réseau utilisant le protocole IP. Contrairement à la téléphonie traditionnelle (RTC), elle convertit les signaux vocaux en paquets de données. Elle est largement utilisée dans les communications personnelles, professionnelles et industrielles.

## Évolution de la VoIP
- **Années 1970** : Premières tentatives de transmission vocale sur IP.
- **Années 1990** : Émergence avec l'essor d'Internet.
- **Aujourd'hui** : Solution majeure en entreprise grâce à l'amélioration des protocoles (SIP, RTP, H.323) et de la bande passante.

Les premières tentatives de transmission vocale sur IP remontent aux années 1970, mais la technologie a réellement émergé dans les années 1990 avec l’essor d’Internet. Initialement limitée en qualité et en adoption, la VoIP a connu une amélioration progressive grâce à l’augmentation de la bande passante, à l’optimisation des protocoles (SIP, RTP, H.323) et à la généralisation du haut débit.
Aujourd’hui, elle est devenue une alternative majeure à la téléphonie classique et constitue la base des communications modernes en entreprise.


## Installation d'Asterisk
### Prérequis
```bash
sudo apt update && sudo apt upgrade
sudo apt install build-essential libxml2-dev libncurses5-dev \
linux-headers-$(uname -r) libsqlite3-dev libssl-dev libedit-dev \
uuid-dev libjansson-dev
```

Installation de Asterisk :
```
- sudo apt update && sudo apt upgrade
- sudo apt install build-essential libxml2-dev libncurses5-dev linux-headers-$(uname -r) libsqlite3-dev libssl-dev libedit-dev uuid-dev libjansson-dev (Pour les dépendances)
- mkdir /usr/src/asterisk
- cd /usr/src/asterisk
- wget https://downloads.asterisk.org/pub/telephony/asterisk/asterisk-22.2.0.tar.gz
- tar -xvzf asterisk-22.2.0.tar.gz
- cd asterisk-22.2.0
- ./configure
- make
- make install
- make samples
- make config
```

## Configuration
### Utilisateurs - `/etc/asterisk/pjsip.conf`
Création des utilisateurs dans /etc/asterisk/pjsip.conf :
```
Exemple :
[6001]
 type=endpoint
 context=from-internal
 disallow=all
 allow=ulaw
 auth=6001
 aors=6001

[6001]
 type=auth
 auth_type=userpass
 password=motdepasse6001
 username=6001

[6001]
 type=aor
 max_contacts=1
```

### Extensions - `/etc/asterisk/extensions.conf`
```ini
exten => 6099,1,VoiceMailMain()
exten => 6001,2,VoiceMail(6001)
exten => 6002,2,VoiceMail(6002)
```

Configuration de la boîte mail dans /etc/asterisk/voicemail.conf :
```
[6001]
 password=6001
 fullname=Utilisateur 6001
 email=6001@example.com
```
Dans /etc/asterisk/extensions.conf :
```
- exten=>6099,1,VoiceMailMain() ; 6099 Numéro de téléphone du répondeur
- exten=>6001,2,VoiceMail(6001) ; Appel répondeur compte 6001
- exten=>6002,2,VoiceMail(6002) ; Appel répondeur compte 6002
```

### Sécurisation avec TLS - `/etc/asterisk/pjsip.conf`
Configuration de TLS (Transport Layer Security) :
```
[transport-tls]
 type=transport
 protocol=tls
 bind=0.0.0.0:5061
 cert_file=/etc/asterisk/keys/asterisk.pem
 priv_key_file=/etc/asterisk/keys/asterisk.key
```

## Automatisation des appels

```
[ivr_1]
exten => s,1,Answer()
exten => s,2,Set(TIMEOUT(response)=10)
exten => s,3,agi(googletts.agi,"Bonjour et bienvenue chez à La Plateforme !",fr,any)
exten => s,4,agi(googletts.agi,"Pour joindre Alice, taper 1.",fr,any)
exten => s,5,agi(googletts.agi,"Pour joindre Bob, taper 2.",fr,any)
exten => s,6,agi(googletts.agi,"Pour joindre Morgan, taper 3.",fr,any)
exten => s,7,WaitExten()

exten => 1,1,Dial(PJSIP/alice,10)
exten => 2,1,Dial(PJSIP/bob,10)
exten => 3,1,Dial(PJSIP/alice,10)

exten => _[04-9*#],1,agi(googletts.agi,"Entrée invalide",fr,any)
exten => _[04-9*#],2,Goto(ivr_1,s,1)
exten => t,1,Goto(ivr_1,s,3)
```
```
[appel-sortant]
exten => s,1,Answer()
 same => n,Wait(1)
 same => n,AGI(googletts.agi,"Bonjour, ceci est un appel automatique.",fr)
 same => n,Hangup()
```

Pour qu’un automate appelle nos utilisateurs automatiquement :
1. Créez un fichier csv nommé contacts.csv qui répertorie les utilisateurs.
2. Créez un script random.sh contenant les appels automatiques.
   
Exemple de random.sh :
```
#!/bin/bash

CSV_FILE="contacts.csv"
CALLS_DIR="/var/spool/asterisk/outgoing/"
CALLER_ID="9000"

# Vérifier si le fichier CSV existe
if [[ ! -f "$CSV_FILE" ]]; then
    echo "Erreur : le fichier $CSV_FILE n'existe pas."
    exit 1
fi

# Lire le fichier CSV et générer un appel pour chaque contact
tail -n +2 "$CSV_FILE" | while IFS=, read -r NAME NUMBER; do
    # Vérifier que les champs ne sont pas vides
    if [[ -z "$NAME" || -z "$NUMBER" ]]; then
               continue
    fi

    CALL_FILE="/tmp/call_$NUMBER.call"

    echo "Génération de l'appel pour $NAME ($NUMBER)..."

    cat <<EOF > "$CALL_FILE"
Channel: PJSIP/$NUMBER
CallerID: "Prospection Automatique" <$CALLER_ID>
MaxRetries: 2
RetryTime: 60
WaitTime: 30
Context: auto_calls
Extension: s
Priority: 1
EOF

    # Vérifier que le fichier a été créé
    if [[ ! -f "$CALL_FILE" ]]; then
        echo "Erreur : Impossible de créer le fichier d'appel pour $NAME ($NUMBER)."
        continue
    fi

    # Déplacer le fichier avec les bonnes permissions
    chmod 777 "$CALL_FILE"
    mv "$CALL_FILE" "$CALLS_DIR/"

    echo "Appel généré pour $NAME ($NUMBER)"
done
```

## Avantages et Inconvénients
### ✅ Avantages
1. Coût réduit : Moins cher pour les appels internationaux.
2. Flexibilité : Appels depuis n'importe quel appareil connecté.
3. Fonctionnalités avancées : Messagerie, conférence, suivi des appels.
4. Évolutivité : Ajout/suppression facile de lignes.
5. Intégration avec des CRM et autres outils collaboratifs.


### ❌ Inconvénients
1. Dépendance à Internet : La qualité dépend de la connexion.
2. Besoins en électricité : Risque de coupure si panne de courant.
3. Problèmes d'interopérabilité entre systèmes différents.
4. Limitations sur les appels d’urgence.


## Sécurité
Pour sécuriser la VoIP :
- Utilisez SRTP (Secure Real-Time Transport Protocol) pour la voix.
- Activez TLS pour sécuriser les connexions.
- Utilisez un VPN pour un tunnel sécurisé.

## Solutions sur le marché
### Open Source
- Asterisk : Flexible et personnalisable.
- FreeSWITCH : Polyvalent avec des fonctionnalités avancées.
- Kamailio : Orienté vers la gestion de gros volumes d'appels.

### Payantes
- Cisco Webex
- Microsoft Teams
- Zoom Phone
- RingCentral
  
## Exemples de Tests
### ✅ Création d'utilisateur
Cas de test : Création d’un utilisateur VoIP
- Fonctionnalité testée : Ajout d’un utilisateur PJSIP.
- Comportement attendu : L’utilisateur s’enregistre avec succès.

### 📞 Appel interne
Cas de test : Appel interne entre deux utilisateurs SIP
- L'utilisateur A appelle l'utilisateur B.
- Vérification : Audio clair et bidirectionnel.


### 🌐 Connectivité externe
Test de connectivité :
- Un utilisateur essaie de se connecter depuis un réseau externe.
- Comportement attendu : Connexion et appel réussis

### 🔄 Redémarrage serveur
```bash
sudo systemctl restart asterisk
```
- Après redémarrage, les utilisateurs se reconnectent automatiquement.
- Vérification des logs pour s'assurer de la persistance des configurations.

## Conclusion
La VoIP est une solution moderne et économique adaptée aux besoins actuels des entreprises. Elle allie flexibilité, évolutivité et intégration avec des outils numériques tout en nécessitant une infrastructure réseau fiable et sécurisée.


